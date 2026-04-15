# Claude Code 内存与助手系统 - 架构图与流程说明

> 本文档包含所有核心架构图的详细说明和流程解释

---

## 图 1: 三层记忆系统架构概览

**核心概念**：

```
主对话循环（同步）
  ↓
  ├→ 长期记忆：扫描 memdir，AI 选择相关→嵌入系统提示
  │
  └→ 短期记忆 + 后台助手：[停止钩子]
     ├→ Session Memory: 异步 fork，缓存高效
     ├→ Extract Memories: 异步 fork，自动保存发现
     └→ Auto Dream: 定期 fork，跨会话融合
```

**关键洞察**：
- 长期记忆在**对话开始加载**（每次会话）
- 短期记忆在**对话进行中更新**（阈值驱动）
- 后台助手**永远不阻塞主对话**（异步 fork）

---

## 图 2: Forked Agent 提示缓存机制

**这是整个系统成本优化的核心！**

缓存工作原理：

```
API 接收的请求数据：
┌ System Prompt (5KB)
├ Tool Definitions (2KB)
├ Message Prefix [msg1-5] (8KB)
└ New Messages [new] (0.5KB)
  
缓存键 = hash(System + Tools + MsgPrefix)
```

**成本计算**：

| 请求 | 缓存状态 | 成本 |
|------|-------|------|
| 主对话 | 第一次 | 100% = 15.5K tokens |
| Extract fork | 缓存命中 ✓ | 10% = 0.5K tokens (cache reads free!) |
| Dream fork | 缓存命中 ✓ | 10% = 0.5K tokens |

**为什么后台工作几乎免费**？

Anthropic API 缓存策略（自 2024 年）：
- 缓存命中时，缓存部分的读取完全免费  
- 只需付费"新内容"部分（0.1x 倍的基础费率）
- 因此 fork 复用父请求的前缀 → 90% 节省

**代码关键点**：

```typescript
// CacheSafeParams：强制同步这些，否则缓存失效
export type CacheSafeParams = {
  systemPrompt,      // ← 必须完全相同
  userContext,       // ← 必须完全相同
  systemContext,     // ← 必须完全相同
  toolUseContext,    // ← 包括工具配置，必须相同
  forkContextMessages // ← 消息*前缀*必须相同
};

// 后续 fork 时，只改这个
promptMessages: [
  ...params.forkContextMessages,  // 前缀复用（缓存命中）
  ...params.promptMessages         // 新 fork 特定的提示
]
```

---

## 图 3: 长期记忆系统完整流程

**对话开始时的 5 步加载流程**：

### 步骤 1-2: 扫描和列表

```
User: "实现 caching layer"
  ↓
1. scanMemoryFiles(memoryDir)
   - 读取所有 .md 文件的前 30 行 (frontmatter)
   - 按 mtime 排序，最新优先
   - 最多 200 个文件

2. 生成清单
   [file] [type] [mtime] [description]
   user/expertise.md [user] 2026-04-10 "后端工程师，6年经验"
   feedback/refactor.md [feedback] 2026-04-05 "单一PR方案优于多个"
   project/roadmap.md [project] 2026-03-20 "Q2周期冻结"
```

### 步骤 3: 相关性选择（使用 Sonnet fork）

```
3. Sonnet 选择相关
   输入：用户查询 + 清单 (≤ 5)
   输出：精选的相关记忆
   
   策略：
   - 排除已看过的（alreadySurfaced）
   - 排除最近使用的工具文档（防止重复）
   - 宁可遗漏，不过选（保持信噪比）
   
   示例决定：
   Query: "caching layer"
   选出:
     ✓ feedback/refactor.md (关于大改的指导)
     ✓ project/roadmap.md (截止日期限制)
     ✗ user/expertise.md (与缓存无关)
```

### 步骤 4: 新鲜度检查

```
4. 为选出的每个记忆检查新鲜度
   memoryAgeDays(mtime):
   
   今天: "Expertise added today"
   昨天: "Modified yesterday"
   3天前: ⚠️ "3 days ago.
            Memories are point-in-time observations, 
            not live state — verify against current code"
```

### 步骤 5: 嵌入系统提示

```
5. 嵌入
   
   系统提示构建：
   ───────────────────────────────
   You are Claude Code, an AI assistant...
   
   [如下是相关记忆：]
   
   ### feedback/refactor.md (yesterday)
   Single PR for refactoring is better than many...
   
   ### project/roadmap.md (3 days ago)
   Q2 merge freeze starts 2026-03-05...
   ⚠️ This memory is 3 days old. Verify against current state.
   ───────────────────────────────
```

---

## 图 4: 短期记忆系统 - 阈值驱动更新

**问题**：长对话中消息会爆炸性增长，需要压缩。

**解决方案**：三重阈值触发后台提取

### 阈值详解

```
TOKEN COUNT 变化时间轴：

时间 → | Turn 1 | Turn 2 | Turn 3 | Turn 4 |
token → | 8K   | 15K  | 22K   | 28K   |
       | ──── | ──── | ──── | ──── |
```

**门 1：初始化阈值**
```
if (!isInitialized && tokenCount >= 8K) {
  markSessionMemoryInitialized();
  // 只计算一次
}
```

**门 2：更新间隔**
```
if (tokenCount - tokenCountAtLastExtraction >= 5K) {
  // token 增长了 5000，可以提取了
  shouldExtract = true;
}
```

**门 3：工具调用数**
```
const toolCalls = countToolCallsSince(messages, lastExtractUuid);
if (toolCalls >= 2) {
  // 至少运行了 2 个工具，有内容可提取
  shouldExtract = true;
}
```

**全部满足时**：

```
shouldExtract() → 
  1. 立即返回给用户（不等待）
  2. queueAsyncWork(() => {
       runForkedAgent({
         promptMessages: [
           { role: 'user', 
             content: buildSessionMemoryUpdatePrompt(
               previousSessionMemory,
               recentMessages,
               recentTools
             )}
         ],
         cacheSafeParams: createCacheSafeParams(context),
         canUseTool: createAutoMemCanUseTool(),
         forkLabel: 'session_memory'
       })
     })
  3. 返回的 fork 生成 markdown 摘要
  4. 写入 sessionMemory.md（增量，不覆盖）
```

### 内容示例

**提取后的 sessionMemory.md**：

```markdown
---
type: session
generated: 2026-04-15T14:32Z
---

# Session Memory

## Current Task
实现缓存层以提高 API 响应速度。目前在 `/src/cache` 目录。

## Recent Discoveries
1. Redis 服务在 dev 环境可用 (localhost:6379)
2. 现有代码在三个地方做缓存：
   - UserService.ts:42 (LRU for profiles)
   - AuthService.ts:88 (token cache)
3. Token 过期时间不一致 (user:1h, auth:30m)

## Decisions Made
- 决定统一使用 30 分钟过期 (考虑了安全性)
- 选择 Redis 而非内存缓存 (多进程共享)
- 拒绝了添加缓存预热 (复杂度高，收益小)

## Known Issues
- Redis 连接池在高并发下有泄漏
- 需要添加缓存失效机制

## Next Steps
- [ ] 实现 Redis adapter
- [ ] 添加失效监听器
- [ ] 集成测试
```

---

## 图 5: 后台助手完整流程

### 时间轴

```
[用户输入] → [工具执行] → [生成响应] → [return to user]
                                           ↓
                                    [Stop Hook]
                                    
(这时用户已看到响应，不等待后台)

在后台：
Extract Thread              Session Memory Thread         Auto Dream Thread
    ↓                              ↓                           ↓
检查条件                        检查条件                      时间 24h?
    ↓                              ↓                           ↓
触发 Fork              触发 Fork                   检查 5 会话?
    ↓                              ↓                           ↓
解析记忆           生成摘要                      获取锁
    ↓                              ↓                           ↓
保存 memdir        保存 md                    融合记忆
```

### Extract Memories 的特殊之处

**防重复机制**：

```
不是简单地把所有对话都提取。而是：

lastMemoryMessageUuid = "msg-uuid-from-previous-extract"

新提取时：
  const newMessages = getMessagesSince(lastMemoryMessageUuid);
  // 只看新增部分
  
  const previousExtractions = await readExtractionLog();
  // 避免提取相同的记忆
  
  const toExtract = filterDuplicates(
    newMessages,
    previousExtractions
  );
```

### 隔离设计矩阵

**后台 fork 的隔离程度**：

| 资源 | Extract | Session | Dream | 说明 |
|-----|---------|---------|-------|------|
| readFileState | 克隆 | 克隆 | 克隆 | 缓存副本，防污染 |
| setAppState | no-op | no-op | no-op | 后台工作无 UI |
| abort | 链接* | 链接 | 链接 | 父中止传播，反之不行 |
| 消息 | 继承前缀+新 | 继承前缀+新 | 继承前缀 | 缓存复用 |

\* "链接"：`createChildAbortController()` 创建一个受父控制的子控制器

---

## 图 6: 记忆类型与生命周期

### 四种记忆的对比

| 特征 | user | feedback | project | reference |
|-----|------|----------|---------|-----------|
| **定义** | 用户身份 | 工作指导 | 进行中的工作 | 外部系统 |
| **作用域** | 仅私有 | 默认私有 | 偏向团队 | 多为团队 |
| **何时保存** | 学习特征 | 纠正或确认 | 听到截止/目标 | 提及工具 |
| **何时访问** | 定制回答 | 每项工作 | 评估影响 | 查找时 |
| **例** | "6年Go经验" | "单PR优于多PR" | "3/5冻结" | "Linear→INGEST" |

### 生命周期（CRUD）

```
C: Create
───────────
用户编辑文件：
  ~/.claude/projects/<slug>/memory/user/expertise.md
  
或 Extract Agent 自动保存：
  postSamplingHook('extract-memories') 
    → runForkedAgent(...)
    → parseMemories()
    → saveMemoryFile()

R: Read
───────────
对话开始：
  buildSystemPrompt()
    → findRelevantMemories(query)
      → scanMemoryFiles()
      → Sonnet选择(≤5)

U: Update
───────────
用户编辑：
  手动在 memory/ 修改文件

自动提取：
  Extract Agent 在对话末重新保存

D: Delete
───────────
用户手动删除：
  rm ~/.claude/projects/<slug>/memory/feedback/old-rule.md
  
系统不自动删除（保守）
```

---

## 关键设计权衡总结

### 1. 记忆持久化策略

**选择**：磁盘文件 (`memdir`)

**为什么**？
- ✅ 用户可以 git 版本控制（便于审查发展）
- ✅ 可跨机器同步（通过 dotfiles repo）
- ✅ 用户可以手动编辑合并矛盾
- ❌ 需要文件 I/O（但不是关键路径，启动时一次）

### 2. 检索策略

**选择**：扫描所有 frontmatter + Sonnet 选择

**为什么**？
- ✅ 语义理解（vs 关键字搜索）
- ✅ 支持"不确定则不选"（保持信噪比）
- ✅ Sonnet fork 利用缓存，成本低
- ❌ 多一个 API 调用（但缓存命中，所以免费）

### 3. 后台执行模式

**选择**：异步非阻塞 fork

**为什么**？
- ✅ 用户立即看到响应（最优体验）
- ✅ 通过缓存实现低成本（90% 节省）
- ✅ 隔离设计防止副作用
- ❌ 日志复杂（tracking fork 的状态）

### 4. 缓存加热

**选择**：从不缓存，总是计算

**为什么**？
- ✅ 无需复杂的缓存失效逻辑
- ✅ 保证结果新鲜（文件可能被编辑）
- ✅ 内存占用低（无需驻留）
- ❌ 启动时的 frontmatter 扫描不可避免

---

## 实现复杂度分析

### 代码量估算

| 模块 | 文件数 | 关键复杂性 |
|-----|-------|----------|
| `memdir/` | 8 | memoryTypes 的 4 个类型定义 + Sonnet 选择逻辑 |
| `SessionMemory/` | 3 | 阈值计算 + Forked Agent 集成 |
| `extractMemories/` | 2 | 去重逻辑 + 提示模板 |
| `autoDream/` | 4 | 时间 + 会话 + 锁 的三重门 |
| `forkedAgent/` | 1 | CacheSafeParams 验证 + 隔离上下文 |

### 集成点

1. **系统提示构建** (`context.ts` → `getUserContext()`)
   - 调用 `findRelevantMemories(query)`
   - 嵌入到最终系统提示

2. **停止钩子** (`postSamplingHooks`)
   - 触发 Extract Memories fork
   - 触发 Session Memory fork

3. **启动时** (`main.tsx`)
   - 初始化 Extract Memories
   - 初始化 Session Memory
   - 初始化 Auto Dream

---

## 性能指标预期

### Token 成本

**所有数字基于缓存命中的前提**：

| 操作 | Token 消耗 | 频率 | 总成本 |
|-----|-----------|------|-------|
| 对话开始扫描 memory | 100 | 1x/session | 100 |
| Sonnet 选择相关 | 200 (缓存命中 50) | 1x/session | 50 |
| Session Memory fork | 100 (缓存命中 10) | 2-3x/session | 20-30 |
| Extract fork | 200 (缓存命中 20) | 1-2x/session | 20-40 |
| Auto Dream fork | 300 (缓存命中 30) | 1x/week | <5 |

**总额外成本**：~100-150 tokens/session（相比没有记忆系统）

**节省**：无需重复解释背景知识（可节省 1000+ tokens）

### 延迟

| 操作 | 延迟 | 阻塞? |
|-----|------|-------|
| 扫描 memdir | ~10ms | 否（缓存） |
| Sonnet 选择 | ~1-2s | 否（fork） |
| 嵌入系统提示 | <1ms | 是（关键路径） |
| 提取/摘要 fork | ~1-2s | 否（后台） |

---

## 故障和边界情况

### 内存文件损坏

```
if (readFrontmatter fails) {
  // scanMemoryFiles 会捕获错误
  // 返回 []（此路径被跳过）
  // 其他记忆继续加载（graceful degrade）
}
```

### 新鲜度问题

```
记忆 5 个月前：
  memoryFreshnessNote(mtimeMs) 返回警告
  系统提示中：
  ⚠️ "This memory is 152 days old. 
      Verify against current code."
  
  模型看到警告 → 更加谨慎
```

### 缓存失效

```
用户修改了系统提示模板？
  → 缓存自动失效（hash 改变）
  → 下一个 fork 重新计算
  → 无需手动操作
```

---

## 与 Anthropic 其他产品的关系

### 与 Vision 的关系

如果 memory 中有图表/图片，Sonnet 选择器如何处理？
- 当前：frontmatter only（看不到图片）
- 后续可能：指标化图片，投入 frontmatter description

### 与 Files 的关系

大文件如何存储？
- 不在 memory/ （太重）
- 改为存储在项目 `.claude` 目录下
- memory/ 只存引用

### 与 Projects 的关系

不同项目的 memory 如何共享？
- 当前：完全隔离（per-project slug）
- 未来可能：team memory （TEAMMEM feature）

---

## 总结与启示

这个系统设计体现了以下工程哲学：

1. **User First**：永远不打断用户体验（异步）
2. **Cost-Conscious**：充分利用 API 提示缓存（90% 折扣）
3. **Graceful**：部分失败不影响整体（边界情况处理）
4. **Observable**：每个决定都可追踪（新鲜度标注、日志）
5. **Structural**：投资框架而非特例（4 种类型、3 重门）

它是**AI + 工程成熟度**的典范实现。

