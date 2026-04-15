# Claude Code 内存与助手系统深度研究

> 基于源码分析（版本 2.1.88），还原自 `@anthropic-ai/claude-code` npm 包

## 📋 目录
1. [长期记忆系统（Memory Directory）](#长期记忆系统)
2. [短期记忆系统（Session Memory）](#短期记忆系统)
3. [后台助手工作原理](#后台助手工作原理)
4. [架构设计与权衡](#架构设计与权衡)
5. [设计哲学](#设计哲学)

---

## 长期记忆系统

### 🎯 是什么

Claude Code 的长期记忆系统（`memdir`）是一个**文件系统级的知识库**，存储项目相关的持久化信息。系统自动将其注入到每次对话的系统提示中，帮助助手理解项目背景和历史决策。

**存储位置**：`~/.claude/projects/<project-slug>/memory/`

**核心文件**：
- `MEMORY.md` — **索引文件**，最多 200 行 25KB，记录所有记忆的引用和摘要
- `*.md` — 单个记忆文件，最多 200 个（按修改时间最新优先排序）

### ❓ 为什么/解决了什么问题

#### 问题背景

在长期项目中，AI 助手面临以下挑战：

1. **上下文窗口有限**：无法在每次对话中包含完整的项目历史
2. **知识遗忘**：无法跨会话保持对项目决策的理解
3. **决策不一致**：每次对话可能基于过时的假设
4. **低效的搜索**：需要手动重复解释同样的项目信息

#### 解决方案

通过文件系统中的持久化记忆库：
- ✅ **大幅减少 token 消耗**：仅加载最相关的记忆，不是全部
- ✅ **跨会话一致性**：历史决策和学习的经验保留下来
- ✅ **主动发现**：系统自动扫描和选择相关记忆，无需用户手动指定
- ✅ **灵活的优先级**：按修改时间和关键字相关性动态选择

### 🏗️ 如何实现

#### 1. **记忆类型分类** (`memoryTypes.ts`)

四种正交的记忆类型，约束记忆内容为**不可从代码推导**的信息：

```typescript
type MemoryType = 'user' | 'feedback' | 'project' | 'reference'
```

**详细说明**：

| 类型 | 作用域 | 用途 | 示例 |
|------|-------|------|------|
| **user** | 仅私有 | 用户角色、专业背景、偏好 | "用户是后端工程师，新接触 React" |
| **feedback** | 偏向私有 | 对 AI 工作方式的反馈；包括纠正和确认 | "不要在每个响应末尾总结，我可以读代码" |
| **project** | 偏向团队 | 进行中的工作、目标、截止日期、事件 | "合并冻结从 2026-03-05 开始，移动端发布" |
| **reference** | 多为团队 | 指向外部系统（Bug 追踪、Slack 等）的指针 | "Linear 项目 'INGEST' 追踪所有管道 Bug" |

**关键约束**：代码、架构、git 历史、文件结构**不应该**保存，因为它们可从工作目录推导。

#### 2. **内容嵌入与截断** (`memdir.ts`)

系统提示中嵌入记忆内容时的处理策略：

```
前缀：200 行 + 25KB 字节双重限制
策略：
  1. 按行数截断（200 行）
  2. 再按字节数截断（25KB，从最后一个完整行）
  3. 添加警告，建议索引条目<200 字符
```

**为什么双限制**？
- **行数限制**：保护短上下文窗口，避免单一记忆淹没列表
- **字节限制**：防止长行索引（如长代码段或 JSON）绕过行数限制

```typescript
function truncateEntrypointContent(raw: string): EntrypointTruncation {
  // 先按行数，再按字节，双保险
  if (lineCount > 200 || byteCount > 25KB) {
    return truncated + `
      > WARNING: MEMORY.md is ${reason}. Only part was loaded. Keep entries
        to one line under ~200 chars; move detail into topic files.`
  }
}
```

#### 3. **相关记忆自动选择** (`findRelevantMemories.ts`)

**两步骤流程**：

```
步骤 1: 扫描内存目录
  ↓
  使用 scanMemoryFiles() 读取所有 .md 文件的 frontmatter（最多 200 个）
  按修改时间排序，最新优先
  提取：描述、类型、文件名

步骤 2：用 Sonnet 选择相关记忆
  ↓
  系统提示：给定用户查询和记忆清单，选择≤5 个明确有用的记忆
  策略：宁可遗漏，不要过度选择（避免噪音）
  过滤已显示过的文件（避免重复）
```

**核心代码**：

```typescript
async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]> {
  // 1. 扫描记忆目录，每个文件读取前 30 行（frontmatter）
  const memories = await scanMemoryFiles(memoryDir, signal);
  
  // 2. 排除已显示过的
  const candidates = memories
    .filter(m => !alreadySurfaced.has(m.filePath));
  
  // 3. 用 Sonnet 选择最相关的
  const selected = await selectRelevantMemories(
    query,
    candidates,
    signal,
    recentTools // 防止重复推荐最近使用的工具文档
  );
  
  return selected.map(m => ({ path: m.filePath, mtimeMs: m.mtimeMs }));
}
```

**集成点**：在系统提示构建时，通过 `context.ts` 的 `getUserContext()` 自动调用此函数，并将选定的记忆文件嵌入消息流。

#### 4. **新鲜度跟踪** (`memoryAge.ts`)

防止过时记忆被断言为事实：

```typescript
function memoryFreshnessText(mtimeMs: number): string {
  const days = memoryAgeDays(mtimeMs);
  if (days <= 1) return '';  // 最近的记忆不需要警告
  return `This memory is ${days} days old. Memories are point-in-time 
          observations, not live state — claims about code behavior or 
          file:line citations may be outdated. Verify against current code 
          before asserting as fact.`;
}
```

**使用场景**：当记忆中包含文件:行号引用（如 "UserService 在 src/user.ts:42"）时，如果是 3 天前的记忆，系统会添加警告，避免代码已更改导致的错误信息。

#### 5. **Frontmatter 规范**

所有 .md 记忆文件应包含：

```markdown
---
type: user|feedback|project|reference
description: 一句话摘要（显示在索引中）
---

# 记忆标题

详细内容...
```

### ⚖️ 实现中的权衡

| 权衡 | 选择 | 理由 |
|------|------|------|
| **持久化 vs 易失性** | 文件系统 | 跨项目/会话持久化，用户可自助编辑和版本控制 |
| **自动 vs 手动提取** | 自动提取（后台 Agent） | 用户不需要手动 `/memory save`；自动化减少开销 |
| **检索方案** | 语义+关键字混合（Sonnet） | Sonnet 的 few-shot 理解优于词匹配；支持"不确定则不选"的谨慎策略 |
| **内存缓存** | cloudflare 部分缓存 | 系统提示在对话内缓存；跨对话需重新扫描（避免内存泄漏） |
| **浪费 vs 智能加载** | 总是加载索引+动态选择记忆 | 索引很小（200 行上限），记忆选择由 Sonnet 优化可达性 |

---

## 短期记忆系统

### 🎯 是什么

**会话记忆**（Session Memory）是对话过程中持续更新的自动摘要，记录每个会话中的关键上下文和发现。不同于长期记忆的持久化，会话记忆是**单次对话的临时工作状态**。

**存储位置**：内存中（可选写入临时文件进行调试）

**主要作用**：
1. 跟踪当前对话中被发现的关键事实
2. 在长对话中通过 token 限制前压缩历史信息
3. 传递上下文给 Copilot 扩展的外部工具

### ❓ 为什么/解决了什么问题

#### 问题背景

在长对话中面临挑战：

1. **上下文窗口爆炸**：100+ 条消息会快速填满窗口
2. **重复信息**：相同的事实被重复讨论
3. **中间发现遗失**：用户提出的临时发现在深度对话中被埋没
4. **工具链信息丢失**：外部工具（如 Copilot 扩展）无法获知助手的中间发现

#### 解决方案

通过自动摘要机制：
- ✅ **token 高效**：用紧凑的要点替代完整转录
- ✅ **主动更新**：在后台 Agent 中异步更新，不中断对话
- ✅ **用户可查看**：保存为 markdown 文件，用户可审查和编辑
- ✅ **上下文保留**：始终作为系统提示的一部分注入

### 🏗️ 如何实现

#### 1. **自动提取流程** (`sessionMemory.ts`)

触发机制是**基于阈值的后台 Agent**：

```typescript
function shouldExtractMemory(messages: Message[]): boolean {
  const currentTokenCount = tokenCountWithEstimation(messages);
  
  // 初始化阈值：等到足够的上下文（如 8000 tokens）
  if (!isSessionMemoryInitialized()) {
    if (!hasMetInitializationThreshold(currentTokenCount)) {
      return false;  // 太早，跳过
    }
    markSessionMemoryInitialized();
  }

  // 更新阈值：从上次提取后增长了足够的 token
  if (!hasMetUpdateThreshold(currentTokenCount)) {
    return false;  // 未增长足够，跳过
  }

  // 工具调用阈值：至少运行了 2 个工具
  const toolCallCount = countToolCallsSince(messages, lastMemoryMessageUuid);
  if (toolCallCount < 2) {
    return false;  // 工作量不足
  }

  return true;  // 条件满足，触发提取
}
```

**三重门阈值**：

| 阈值 | 值 | 目的 |
|------|-----|------|
| **初始化** | ~8000 tokens | 避免在简短对话中开启提取 |
| **更新间隔** | ~5000-10000 tokens 增长 | 防止过度提取消耗 |
| **工具调用** | ≥2 个工具调用 | 确保有实质发现可提取 |

#### 2. **Forked Agent 提取** (`runForkedAgent`)

关键创新：**使用 fork of parent agent**，共享提示缓存

```
主对话                      后台提取 Agent
┌─────────────────────┐    ┌──────────────────────────┐
│ User query          │    │ Forked context (isolated)│
│   ↓                 │    │                          │
│ Tool calls (Bash)   │    │ System: "Extract and     │
│   ↓                 │    │  update these notes:"    │
│ Tool results        │    │ Messages: [conversation] │
│   ↓                 │    │ Previous session memory  │
│ Model output        │    │   ↓                      │
│                     │    │ Model generates markdown │
│                     │    │   ↓                      │
│  [触发]             │    │ Write to sessionMemory.md│
│   └─────────────────┼────→ (isolated writes)       │
└─────────────────────┘    └──────────────────────────┘
  └─ 共享提示缓存 ──────────────→ 高效重用
```

**代码流程**：

```typescript
export async function runSessionMemoryExtraction(
  context: REPLHookContext,
  messages: Message[]
): Promise<void> {
  // 构建隔离的 forked 上下文
  const subagentContext = createSubagentContext(parentContext, {
    // 关键：不分享 setAppState 或 abortController
    // 因为提取是后台工作，不应影响主对话
  });

  const cacheSafeParams = createCacheSafeParams(context);

  // 运行完全隔离的 forked Agent
  const result = await runForkedAgent({
    promptMessages: [
      createUserMessage({
        content: buildSessionMemoryUpdatePrompt(previousMemory, messages)
      })
    ],
    cacheSafeParams,  // ← 共享提示缓存，高效
    canUseTool: createAutoMemCanUseTool(),
    querySource: 'session_memory_extraction',
    forkLabel: 'session_memory'
  });

  // 只读取提取的记忆，不中断主对话
  const extracted = extractSessionMemoryFromResults(result.messages);
  await writeSessionMemory(extracted);
}
```

#### 3. **提示指令** (`prompts.ts`)

对 Session Memory Agent 的核心指令：

```markdown
## Session Memory Update Prompt

You are updating the session memory document with discoveries from 
the conversation so far.

Previous session memory:
${previousMemory}

## Recent conversation (last 50 messages):
${recentMessages}

## Your task:
1. Review what has been discovered or changed in recent messages
2. Update key sections:
   - **Current Task**: What is the user working on right now?
   - **Recent Discoveries**: Key findings from tools, code exploration
   - **Decisions**: Important choices made (e.g., "decided not to refactor 
     because...")
   - **Next Steps**: What should happen next
3. Keep entries concise (~100 words per section)
4. Preserve previous learning but update for new information
5. Remove items that are resolved or outdated

Output ONLY the updated session memory markdown.
```

#### 4. **与 extractMemories 的交互**

Session Memory 与 Long-term Memory 的协调：

```typescript
// 在 extractMemories Agent 中
async function extractMemories(/* ... */) {
  // 排除已保存到 SessionMemory 的内容
  const previousSessionMemories = 
    await getSessionMemoryContent();  // 已写入的要点
  
  const messageSinceLastExtraction = messages.slice(
    indexOf(messages, lastMemoryMessageUuid)
  );
  
  // 专注于新的、持久化价值的发现（供跨会话使用）
  const memoriesToExtract = selectNonduplicate(
    messageSinceLastExtraction,
    previousSessionMemories  // 去重
  );
  
  // 只提取有跨会话价值的
  return memoriesToExtract.filter(m => 
    m.type !== 'temporary' && m.importance > threshold
  );
}
```

### ⚖️ 实现中的权衡

| 权衡 | 选择 | 理由 |
|------|------|------|
| **同步 vs 异步提取** | 异步（后台 fork） | 不打断用户对话；利用 fork 的提示缓存高效 |
| **多少频度更新** | 阈值驱动（3 重） | 避免过度提取（token 浪费）；确保实质发现 |
| **何处存储** | 内存 + 可选文件 | 内存快速访问；文件支持用户编辑和调试 |
| **覆盖旧内容** | 增量更新（不清除） | 保持上下文连续性；用户可看到演进 |

---

## 后台助手工作原理

### 🎯 是什么

Claude Code 中的后台助手是一系列**自动化的异步 Agent 任务**，在主对话进行中在后台悄悄运行，处理知识保存、优化和学习。这些助手共享一个统一的技术基础：**forked agent 模式**。

**主要角色**：
1. **Extract Memories** — 从对话中自动提取可保存的记忆
2. **Session Memory** — 维护当前对话的工作摘要
3. **Auto Dream** — 定期跨会话知识融合

### ❓ 为什么/解决了什么问题

#### 问题背景

1. **用户疲劳**：强制用户手动 `/memory save` 每个发现
2. **遗忘窗口**：有价值的信息在对话后立即丢失
3. **知识孤岛**：单个会话的发现无法跨项目应用
4. **对话中断**：任何后台工作都会暂停主对话的 token 流

#### 解决方案

通过后台 Agent 架构：
- ✅ **零用户开销**：自动识别和保存，无需手动命令
- ✅ **无中断**：后台工作与主对话真正异步
- ✅ **智能化**：由 AI 识别什么值得保存（优于规则库）
- ✅ **可追踪**：用户可以看到后台提取了什么

### 🏗️ 如何实现

#### 1. **Forked Agent 的核心机制** (`forkedAgent.ts`)

Forked Agent 是关键创新，实现了**共享提示缓存的后台工作**：

```typescript
export type CacheSafeParams = {
  systemPrompt: SystemPrompt;
  userContext: { [k: string]: string };
  systemContext: { [k: string]: string };
  toolUseContext: ToolUseContext;
  forkContextMessages: Message[];  // 父对话的消息前缀
};

export async function runForkedAgent(params: ForkedAgentParams): 
  Promise<ForkedAgentResult> {
  // 关键：fork 启动新的 API 请求，但继承：
  // 1. 完全相同的系统提示 → 缓存命中
  // 2. 完全相同的消息前缀 → 缓存命中
  // 3. 完全相同的模型和工具配置 → 缓存命中
  
  const query = await query({
    messages: [
      ...params.forkContextMessages,  // 从父对话继承
      ...params.promptMessages         // fork 特定的新消息
    ],
    systemPrompt: params.cacheSafeParams.systemPrompt,
    toolUseContext: params.cacheSafeParams.toolUseContext,
    // ...
  });
  
  return {
    messages: query.messages,
    totalUsage: query.totalUsage
  };
}
```

**为什么这很关键**？

Anthropic API 的提示缓存基于：`hash(system_prompt + tool_definitions + messages_prefix)`

如果 fork 使用精确相同的参数，缓存命中时会：
- ✅ 第一次请求支付完整的 token 成本
- ✅ 后续 fork request 的缓存命中**享受 90% token 折扣**
- ✅ 后台工作几乎免费！

#### 2. **Extract Memories Agent** (`extractMemories.ts`)

在每个对话的**停止钩子**（stop hook）触发：

```typescript
export async function initExtractMemories() {
  registerPostSamplingHook(
    'extract-memories',
    async (context: REPLHookContext) => {
      // 检查条件
      if (!isAutoMemoryEnabled()) return;
      if (!shouldExtractThisSession()) return;
      
      // 触发 fork（不等待结果）
      if (hasToolCallsInLastTurn(context.messages)) {
        // 转到后台
        spawnMemoryExtraction(context);
      }
    }
  );
}

async function spawnMemoryExtraction(context: REPLHookContext) {
  // 立即返回给用户；提取在后台进行
  queueAsyncWork(async () => {
    const result = await runForkedAgent({
      promptMessages: [
        createUserMessage({
          content: buildExtractAutoOnlyPrompt(
            context.messages,
            previousExtractions,
            allTools
          )
        })
      ],
      cacheSafeParams: createCacheSafeParams(context),
      canUseTool: createAutoMemCanUseTool(),
      querySource: 'extract_memories_bg',
      forkLabel: 'extract_memories'
    });

    // 解析并保存提取的记忆
    const extracted = parseMemoriesFromResult(result.messages);
    for (const mem of extracted) {
      await saveMemoryFile(mem.path, mem.content);
    }
  });
}
```

**触发时机**：
```
对话进行中
  ↓
用户: "实现 caching" 
  ↓
Agent: 运行 5 个 Bash 工具
  ↓
Model: 输出最终响应
  ↓
[停止] 触发 postSamplingHook
  ↓
   ├─→ 立即返回给用户 "Done!"
  └─→ queueAsyncWork({
        runForkedAgent with extract prompt
        save results
      })
```

#### 3. **Auto Dream 整合** (`autoDream.ts`)

对多个会话的记忆进行**跨会话融合**：

```typescript
export function initAutoDream() {
  let lastSessionScanAt = 0;
  
  runner = async function runAutoDream(context) {
    // 三层门控（成本递增）
    
    // 1. 时间门控（最便宜）
    const lastConsolidatedAt = await readLastConsolidatedAt();
    const hoursSince = (Date.now() - lastConsolidatedAt) / 3_600_000;
    if (hoursSince < minHours) return;  // 24 小时规则
    
    // 2. 会话门控（需要扫描）
    const sessionsSince = await listSessionsTouchedSince(lastConsolidatedAt);
    if (sessionsSince.length < minSessions) return;  // 5 个会话规则
    
    // 3. 锁门控（防止并发）
    const lock = await tryAcquireConsolidationLock();
    if (!lock) return;  // 已有其他进程在运行
    
    try {
      // 运行 dream agent fork
      const result = await runForkedAgent({
        promptMessages: [
          createUserMessage({
            content: buildConsolidationPrompt(
              allMemoryFiles,
              sessionsSinceLastConsolidation
            )
          })
        ],
        cacheSafeParams,
        canUseTool: createAutoMemCanUseTool(),
        forkLabel: 'auto_dream'
      });
      
      // 更新记忆，标记梦想任务完成
      const consolidated = parseConsolidatedMemories(result.messages);
      await saveConsolidatedMemories(consolidated);
      await updateLastConsolidatedAt();
    } finally {
      await releaseLock(lock);
    }
  };
}
```

**time-based + session-based 的混合策略**：

```
      时间：现在？→ 24h 后
      │
会话 1 → 会话 2 → 会话 3
                  │
                [触发条件满足]
                  │
              [dream Agent]
                  │
            合并相关记忆 + 生成合成知识
                  │
            更新 MEMORY.md 索引
```

#### 4. **隔离与状态管理** (`createSubagentContext`)

后台 Agent 必须严格隔离：

```typescript
export function createSubagentContext(
  parentContext: ToolUseContext,
  overrides?: SubagentContextOverrides
): ToolUseContext {
  // 关键：大部分状态都是隔离的
  
  return {
    // 隔离状态（克隆）
    readFileState: cloneFileStateCache(parentContext.readFileState),
    nestedMemoryAttachmentTriggers: new Set(),
    loadedNestedMemoryPaths: new Set(),
    dynamicSkillDirTriggers: new Set(),
    
    // 隔离回调（no-op 或受限）
    setAppState: overrides?.shareSetAppState 
      ? parentContext.setAppState  // 仅在显式选择时
      : () => {},                   // 后台 agent：no-op
    
    setResponseLength: overrides?.shareSetResponseLength
      ? parentContext.setResponseLength
      : noOpCallback,
    
    // 隔离但链接的中止控制
    abortController: overrides?.shareAbortController
      ? parentContext.abortController
      : createChildAbortController(parentContext.abortController)
      //  ↑ 父中止会传播，但子中止不会影响父
  };
}
```

**隔离矩阵**：

| 资源 | 后台 Extract | 后台 Dream | 交互式 Agent |
|------|-------------|-----------|-----------|
| 文件缓存 | 克隆 | 克隆 | 克隆 |
| setAppState | no-op | no-op | 共享 |
| abortController | 链接 | 链接 | 共享 |
| 消息 | 继承前缀+ 新 | 继承前缀 | 继承前缀 |

### ⚖️ 实现中的权衡

| 权衡 | 选择 | 理由 |
|------|------|------|
| **同步 vs 异步** | 异步非阻塞 | 主对话流畅性优先（用户体验）|
| **缓存高效性 vs 灵活性** | 强制 CacheSafeParams | 缓存命中比灵活性更重要（成本） |
| **提示缓存 vs 新鲜性** | 重用前缀 | 对后台工作足够新鲜；减少 token |
| **状态隔离 vs 共享** | 默认隔离，显式共享 | 防止副作用的安全设计 |
| **并发控制** | 文件锁 + 时间/会话门控 | 防止 CPU 浪费和竞态 |

---

## 架构设计与权衡

### 📊 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     主对话循环（Main REPL Loop）                      │
│  ┌──────────────┐    ┌─────────────────┐    ┌──────────────────┐  │
│  │ User Input   │ → │ System Prompt   │ → │ Claude API       │  │
│  │              │    │ + Long Mem      │    │ (cached)         │  │
│  │              │    │ + Context       │    │                  │  │
│  └──────────────┘    └─────────────────┘    └──────────────────┘  │
│                                                      ↓              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Tool Use Loop:                                              │  │
│  │  1. Parse tool_use blocks                                   │  │
│  │  2. Execute (Bash, FileRead, MCP, etc.)                    │  │
│  │  3. Append results to messages                              │  │
│  │  4. Loop until stop_reason=end_turn                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              ↓                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  [Stop Hook] - postSamplingHooks                            │  │
│  │  ├→ Extract Memories (fork) ──→ [保存到 memdir]              │  │
│  │  ├→ Session Memory (fork) ───→ [更新会话摘要]                │  │
│  │  └→ Other hooks (prompt suggestions, etc.)                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Output rendered to user ← 不等待后台工作                            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│               后台异步任务（Queue + Fork）                        │
│  ┌────────────────────────┐      ┌───────────────────────────┐  │
│  │ Extract Memories Fork  │      │ Session Memory Fork       │  │
│  │ ├─ 系统提示（缓存）    │      │ ├─ 系统提示（缓存）      │  │
│  │ ├─ 消息前缀（缓存）    │      │ ├─ 消息前缀（缓存）      │  │
│  │ ├─ 隔离的 FileState   │      │ ├─ 隔离的 FileState     │  │
│  │ ├─ isAbort: child     │      │ ├─ isAbort: child       │  │
│  │ └─ canUseTool: safe   │      │ └─ canUseTool: safe     │  │
│  │      ↓                 │      │      ↓                   │  │
│  │ Claude API (cached!)   │      │ Claude API (cached!)     │  │
│  │      ↓                 │      │      ↓                   │  │
│  │ parseMemories()        │      │ parseSessionMemory()     │  │
│  │      ↓                 │      │      ↓                   │  │
│  │ ~/.claude/projects/    │      │ sessionMemory.md in      │  │
│  │  <slug>/memory/*.md    │      │ project or memory dir    │  │
│  └────────────────────────┘      └───────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                              ↑
                        [后台系列/并行执行]


┌──────────────────────────────────────────────────────────────────┐
│                  长期记忆库（Persistent）                        │
│                                                                   │
│  ~/.claude/projects/<slug>/memory/                               │
│  ├─ MEMORY.md          [Index, <=200 lines, <=25KB]            │
│  │  ├─ user memory 1: 2026-04-10                               │
│  │  ├─ feedback memory: use single PR for refactors            │
│  │  └─ reference: Linear project INGEST                        │
│  │                                                               │
│  ├─ user/                                                        │
│  │  └─ expertise.md    [type: user]                            │
│  │                                                               │
│  ├─ feedback/                                                    │
│  │  ├─ pr-strategy.md  [type: feedback]                        │
│  │  └─ testing-policy.md                                        │
│  │                                                               │
│  ├─ project/                                                     │
│  │  ├─ current-release.md  [type: project, deadline: 2026-03-20]│
│  │  └─ known-issues.md                                          │
│  │                                                               │
│  └─ reference/                                                   │
│     └─ external-tools.md  [type: reference]                    │
│                                                                   │
│  [访问流程]                                                       │
│  对话开始 → scanMemoryFiles() → 索引 MEMORY.md                   │
│           + 记忆文件 frontmatter                                  │
│           → Sonnet 选择相关 (≤5)                                 │
│           → 嵌入系统提示 + Context                               │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 系统流序图

```
用户                 主对话              后台任务              文件系统
│                     │                   │                    │
├─ 输入命令 ─────────→│                   │                    │
│                     │                   │                    │
│                     ├─ 扫描 memdir ─────→│                    │
│                     │ scanMemoryFiles   └─→ 读 frontmatter    │
│                     │← 记忆列表◄──────────────┤               │
│                     │                         │               │
│                     ├─ Sonnet 选择 (fork) ────→ [隔离运行]    │
│                     │  ├─→ 缓存命中         └─ 返回相关 5 个  │
│                     │  └─ 最相关 ≤5        
│                     │← 相关记忆◄─────────────┤               │
│                     │                         │               │
│                     ├─ 系统提示 build ───────────────────┐   │
│                     │ + MEMORY.md 索引                    │   │
│                     │ + 相关记忆文件                      │   │
│                     │ + 当前上下文                        │   │
│                     │                                      │   │
│                     ├─ Claude API (messages)           ←┘   │
│                     │← 工具调用                             │
│                     │                                      │
│                     ├─ 执行工具 (Bash, FileRead...)      │
│                     │ ├─ 工具 1: Bash "ls"           │
│                     │ │  └─ 结果: "file1.ts file2.ts"│
│                     │ ├─ 工具 2: FileRead "./src"    │
│                     │ │  └─ 结果: file content       │
│                     │ └─ ...                           │
│                     │                                  │
│                     ├─ 循环直到 end_turn              │
│                     │ [Tool use loop]                 │
│                     │                                  │
│                     ├─ 输出响应 ─────→ 端口显示       │
│                     │ [立即返回给用户]                │
│                     │                   │               │
│                     │                   ○ 用户看到响应  │
│                     │                                  │
│        [Stop Hook 触发 - 异步]                         │
│                     │                                  │
│                     ├──┐  Extract Memories Fork        │
│                     │  │  ├─ 系统提示 (cached)        │
│                     │  │  ├─ 消息前缀 (cached hit!)   │
│                     │  │  ├─ 提示: "Extract memories" │
│                     │  │  └─ Claude API (90% 折扣!)  │
│                     │  │       缓存命中   │            │
│                     │  │       └─→ parseMemories()    │
│                     │  │                  │            │
│                     │  │         ┌────────→ 保存       │
│                     │  │         │                     │ 写
│                     │  │         └────────────→ ~/.claude/memory/
│                     │  │                              │
│                     │  │                              │
│                     │  └─ Session Memory Fork   │
│                     │     ├─ 同样的缓存高效          │
│                     │     ├─ 隔离 FileState     │
│                     │     ├─ 生成摘要           │
│                     │     └─ ([写入可选文件])   │
│                     │                         │
│                     [对话继续或结束]           │
│                                              │
│                     [Auto Dream 定期触发]     │
│                     ├─ 条件检查 (24h + 5个会话) │
│                     ├─ Fork Agent with lock  │
│                     └─ 融合多个会话的学习    │
│                                              │
```

### 数据流最优化

**问题**：fork 如果重复传输所有消息，token 成本会很高。

**解决方案**：利用提示缓存的分层结构：

```
API 请求 = [System Prompt] + [Tool Definitions] 
         + [Message Prefix] + [New Messages]
           └─ CACHE KEY ──────┬──────────────┬────────────┘
                              │
                   第一次请求：完整成本
                   后续请求：仅 New Messages 计费
                          (90% 折扣！)

runForkedAgent 的设计：
┌─ 父请求（主对话）
│  └─ 消息: [msg1, msg2, msg3, msg4, msg5]  [完整成本]
│     缓存键: (system + tools + [msg1-5])
│
└─ 子请求 #1（Extract）
   └─ 消息: [msg1, msg2, msg3, msg4, msg5] + [新提示]
      缓存键: (system + tools + [msg1-5]) ✓ 命中
      计费: 仅 [新提示] (~100 tokens)
      
└─ 子请求 #2（Session Memory）
   └─ 消息: [msg1, msg2, msg3, msg4, msg5] + [新提示]
      缓存键: (system + tools + [msg1-5]) ✓ 命中
      计费: 仅 [新提示] (~50 tokens)
```

### 关键权衡总结表

| 权衡点 | 选择 | 成本 | 收益 | 理由 |
|-------|------|------|------|------|
| **记忆持久化** | 磁盘（memdir） | 文件 I/O | 跨会话保留 | 用户可控；支持版本控制 |
| **数据新鲜度** | 文件 mtime | 扫描成本 | 相关排序 | 避免陈旧记忆优先 |
| **检索策略** | Sonnet 选择 | 额外 fork | 语义理解 | vs 关键字搜索效果差 |
| **提取触发** | 阈值驱动 | 复杂条件 | 自动最优 | vs 固定时间浪费 |
| **后台执行** | 异步非阻塞 | 多个 fork | 用户流畅 | 主线程优先 |
| **fork 缓存** | 强制同步参数 | 参数校验 | 90% token 折扣 | 费用最优 |
| **隔离设计** | 默认隔离+ 显式共享 | 代码复杂度 | 安全无副作用 | 防止错误传播 |

---

## 设计哲学

### 核心原则

#### 1. **用户优先，成本次之**

```
主对话流畅性 > token 效率 > 代码简洁性
```

- 后台工作永不阻塞主线程
- 缓存命中是必须的（fork 的参数严格同步）
- 调试友好（提出关于新鲜度的警告，不是沉默失败）

#### 2. **显式优于隐式**

```typescript
// Good: 显式指定哪些状态共享
const ctx = createSubagentContext(parent, {
  shareSetAppState: true,    // 显式选择
  shareAbortController: false // 显式禁用
});

// Bad: 隐式地猜测意图
// const ctx = createSubagentContext(parent);
```

- 隔离优先，共享时必须标记
- 函数签名清晰表达意图（例如 `CacheSafeParams` 名字本身就是契约）

#### 3. **Graceful Degradation**

```typescript
// 记忆不可用？继续对话
if (!isAutoMemoryEnabled()) {
  // 跳过所有内存系统，正常工作
}

// 提取失败？不影响主对话
try {
  await runForkedAgent({ /* ... */ });
} catch (e) {
  logForDebugging(e);
  // 继续，不中断用户
}

// 缓存不可用？直接计费
// runForkedAgent 总是有效，缓存是优化不是必需
```

#### 4. **可观测性优先**

```typescript
// 记忆的新鲜度需要可见
function memoryFreshnessNote(mtimeMs: number): string {
  // 返回警告文本，用户和模型都能看到
  return `<system-reminder>This memory is ${days} days old...</>`
}

// 提取的内容要能追踪
logEvent('memory_extraction', {
  file_count: extracted.length,
  token_usage: result.totalUsage,
  tool_calls: tools_used
});
```

#### 5. **投资于结构，而非优化某个案例**

```
错误：为了节省 token，特殊处理"用户记忆"类型
✓ 正确：建立统一的记忆框架（4 种类型），让阈值和选择器适应
```

- 当新需求（如团队记忆）出现时，框架能支持
- 避免特例；倾向于参数化

#### 6. **Token 成本是架构约束**

```
forkAgent 的设计 100% 围绕缓存命中展开：
- CacheSafeParams 严格同步系统提示
- 消息前缀完全相同（不能添加上下文变量）
- 工具定义不变

这不是个性化的限制，而是 API 成本模型的反映。
```

---

## 实现细节对比

### 长期 vs 短期 vs 后台的特征对比表

| 特征 | 长期记忆 | 短期记忆 | 后台助手 |
|-----|--------|--------|--------|
| **存储** | 磁盘（memdir） | 内存 + 可选文件 | 无（产出写入磁盘） |
| **生命周期** | 跨会话持久 | 单会话 | 触发即消亡 |
| **更新频率** | 低（per 对话末） | 中（per 阈值） | 低（时间 + 会话驱动） |
| **用户可见** | 是（可编辑） | 是（可查看） | 部分（写入的成果可见） |
| **访问路径** | 系统提示 + 消息流 | 系统提示 | 无 |
| **故障优雅性** | 降级到无记忆 | 降级到无摘要 | 不影响主对话 |
| **大小限制** | 200 行 + 25KB | ~无（但会触发压缩） | ~无 |
| **缓存复用** | 部分（MEMORY.md） | 全部（共享前缀） | 全部（CacheSafeParams） |
| **并发修改** | 可能（多 agent） | 无（单 agent） | 防护锁（auto dream） |
| **设计复杂度** | 低 | 中 | 高 |

---

## 代码质量模式

### 错误处理

```typescript
// ✓ 推荐：降级而非崩溃
async function loadMemory(): Promise<MemoryInfo> {
  try {
    return await scanMemory();
  } catch (e) {
    logForDebugging(`Memory scan failed: ${e}`);
    return { memories: [], error: e };  // 返回部分结果
  }
}

// ✗ 不推荐：抛出导致主对话终止
// async function loadMemory() {
//   return await scanMemory();  // 失败 → 主对话中断
// }
```

### 缓存和性能

```typescript
// ✓ 推荐：参数为缓存决策点
const params: CacheSafeParams = {
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  forkContextMessages  // ← 这些必须完全相同
};

// ✗ 不推荐：隐含假设
// runForkedAgent({ /* messages */ })  // 上下文？缓存策略？不清楚
```

### 集成点

```typescript
// ✓ 推荐：清晰的注册模式
registerPostSamplingHook('memory-extraction', async (context) => {
  // 钩子负责触发条件
  if (!shouldExtract(context)) return;
  
  // 返回时还没有结果（异步）
  queueAsyncWork(() => runExtraction(context));
});

// ✗ 不推荐：阻塞钩子
// registerPostSamplingHook('memory-extraction', async (context) => {
//   await runExtraction(context);  // 主对话停止！
// });
```

---

## 总结

Claude Code 的记忆与助手系统是一个**多层次、异步、缓存优化的架构**，体现了以下设计：

1. **长期记忆**：结构化文件库 + AI 驱动的相关性选择，支持跨项目知识积累
2. **短期记忆**：自动摘要，通过后台 agent 无损更新，保持对话上下文
3. **后台助手**：基于 forked agent + 提示缓存的优雅异步模式，成本可控

**关键创新**：
- 📌 **Forked Agent 模式**：共享提示缓存，后台工作几乎免费
- 📌 **三重门阈值**：触发效率与避免浪费的平衡
- 📌 **显式隔离**：防止后台工作的副作用
- 📌 **新鲜度追踪**：避免陈旧信息被错误认为是真相

这是一个**用户优先、成本次之、可观测性优先**的系统设计典范。

