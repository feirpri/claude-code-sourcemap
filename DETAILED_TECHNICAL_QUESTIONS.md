# Claude Code 内存与助手系统 - 详细补充与文件路径

> 本文档补充关键实现细节、完整的文件映射、以及 7 个核心问题的深度回答

---

## 📍 关键要点与文件路径映射

### 长期记忆系统

| 要点 | 文件路径 | 关键函数 |
|------|--------|--------|
| 记忆类型定义 (user/feedback/project/reference) | `memdir/memoryTypes.ts` | `parseMemoryType()`, `MEMORY_TYPES` |
| 内存目录扫描 | `memdir/memoryScan.ts` | `scanMemoryFiles()`, `formatMemoryManifest()` |
| 内容截断逻辑 (200行/25KB) | `memdir/memdir.ts` | `truncateEntrypointContent()`, `ensureMemoryDirExists()` |
| Sonnet 相关性选择 | `memdir/findRelevantMemories.ts` | `findRelevantMemories()`, `selectRelevantMemories()` |
| 新鲜度计算 | `memdir/memoryAge.ts` | `memoryAgeDays()`, `memoryFreshnessNote()` |
| 系统提示注入 | `context.ts` | `getUserContext()`, `getSystemContext()` |
| 内存目录路径解析 | `memdir/paths.ts` | `getAutoMemPath()`, `isAutoMemoryEnabled()` |

### 短期记忆系统

| 要点 | 文件路径 | 关键函数 |
|------|--------|--------|
| 会话摘要主逻辑 | `services/SessionMemory/sessionMemory.ts` | `shouldExtractMemory()`, `initSessionMemory()` |
| 提示模板 | `services/SessionMemory/prompts.ts` | `buildSessionMemoryUpdatePrompt()`, `loadSessionMemoryTemplate()` |
| 阈值和状态管理 | `services/SessionMemory/sessionMemoryUtils.ts` | `hasMetInitializationThreshold()`, `hasMetUpdateThreshold()`, `countToolCallsSince()` |
| 与压缩的交互 | `services/compact/sessionMemoryCompact.ts` | `truncateSessionMemoryForCompact()`, `annotateBoundaryWithPreservedSegment()` |

### 后台助手系统

| 要点 | 文件路径 | 关键函数 |
|------|--------|--------|
| Extract Memories 主逻辑 | `services/extractMemories/extractMemories.ts` | `initExtractMemories()`, `spawnMemoryExtraction()` |
| Extract 提示 | `services/extractMemories/prompts.ts` | `buildExtractAutoOnlyPrompt()`, `buildExtractCombinedPrompt()` |
| Auto Dream 融合 | `services/autoDream/autoDream.ts` | `initAutoDream()`, `runAutoDream()` |
| Dream 提示和锁 | `services/autoDream/consolidationPrompt.ts`, `consolidationLock.ts` | `buildConsolidationPrompt()`, `tryAcquireConsolidationLock()` |
| Forked Agent 基础 | `utils/forkedAgent.ts` | `runForkedAgent()`, `createSubagentContext()`, `CacheSafeParams` |

### 压缩系统 (自动和手动)

| 要点 | 文件路径 | 关键函数/常量 |
|------|--------|--------|
| 自动压缩阈值和配置 | `services/compact/autoCompact.ts` | `getAutoCompactThreshold()`, `getEffectiveContextWindowSize()`, `AUTOCOMPACT_BUFFER_TOKENS` |
| 压缩主核心 | `services/compact/compact.ts` | `compactConversation()`, `buildPostCompactMessages()` |
| 警告和状态 | `services/compact/compactWarningState.ts` | `calculateTokenWarningState()` |
| Post-compact 清理 | `services/compact/postCompactCleanup.ts` | `runPostCompactCleanup()` |
| 微压缩优化 | `services/compact/microCompact.ts`, `snipCompact.ts` | 文件级压缩 |
| Query 压缩集成 | `query.ts` | 压缩触发点 |

### 权限控制系统

| 要点 | 文件路径 | 关键函数 |
|------|--------|--------|
| 权限检查入口 | `utils/permissions/permissions.ts` | `canUseTool()` (CanUseToolFn) |
| 权限规则定义 | `utils/permissions/PermissionRule.ts` | 类型定义 |
| 权限模式 | `utils/permissions/PermissionMode.ts` | `auto|manual|ask|deny` |
| 分类器决策 | `utils/permissions/classifierDecision.ts` | Bash 命令分类 |
| 危险模式检测 | `utils/permissions/dangerousPatterns.ts` | `DANGEROUS_BASH_PATTERNS`, `isDangerousBashPermission()` |
| 权限设置加载 | `utils/permissions/permissionSetup.ts`, `permissionsLoader.ts` | `loadAllPermissionRulesFromDisk()` |
| Bash 分类 | `utils/permissions/bashClassifier.ts` | Bash 命令分析 |
| 路径验证 | `utils/permissions/pathValidation.ts` | `validatePath()` |
| 文件系统权限 | `utils/permissions/filesystem.ts` | 文件权限检查 |

### 主循环集成

| 要点 | 文件路径 | 关键函数 |
|------|--------|--------|
| 消息构建和上下文 | `utils/messages.ts` | `createUserMessage()`, `normalizeMessagesForAPI()` |
| 停止钩子 | `query/stopHooks.ts` | `handleStopHooks()` (触发 Extract/Session) |
| Post-sampling 钩子 | `utils/hooks/postSamplingHooks.ts` | `registerPostSamplingHook()` |
| Token 计算 | `utils/tokens.ts` | `tokenCountWithEstimation()`, `getTokenUsage()` |

---

## 1️⃣ 消息时的上下文获取流程

### 第一次对话开始

```
用户启动 Claude Code
  ↓
main.tsx → query.ts → buildQueryConfig()
  ↓
┌─────────────────────────────────────────────┐
│  构建初始系统提示和上下文                      │
│                                             │
│  getSystemPrompt()                          │
│  ├─ 基础系统提示（生成规则）                  │
│  └─ 注入: 如果设置了 systemPromptInjection  │
│                                             │
│  getUserContext()  [memoized]               │
│  ├─ 扫描 memdir/MEMORY.md 索引              │
│  ├─ 调用 findRelevantMemories(query)       │
│  │  └─ 子 fork (Sonnet): 选择 ≤5 个       │
│  ├─ 读取相关记忆文件内容                    │
│  └─ 返回: { relatedMemories: [...] }      │
│                                             │
│  getSystemContext()  [memoized]             │
│  ├─ Git status (if !isRemote || git cmd)  │
│  │  ├─ getBranch()                         │
│  │  ├─ getDefaultBranch()                  │
│  │  ├─ git status --short (≤2000 chars)   │
│  │  └─ git log -n 5                        │
│  └─ 返回: { gitStatus: "..." }            │
│                                             │
│  构建最终系统提示:                           │
│  ┌─ [基础系统提示]                         │
│  │  You are Claude Code, an AI assistant...│
│  ├─ [Git status]                          │
│  │  (if available)                        │
│  ├─ [相关记忆索引 + 内容]                   │
│  │  ## Types of Memory                    │
│  │  [user/feedback/project/reference]     │
│  │  ## Memory Directory                   │
│  │  - [user] expertise.md (2026-04-10)    │
│  │  - [feedback] pr-strategy.md           │
│  ├─ [内存目录已创建警告]                     │
│  │  DIR_EXISTS_GUIDANCE                   │
│  └─ [系统注入（如果有）]                     │
│     (cache breaking, ant-only)            │
└─────────────────────────────────────────────┘
  ↓
normlizeMessagesForAPI(messages)
  ├─ Prepend: prependUserContext(userContext)
  └─ Append: appendSystemContext(systemContext)
  ↓
API 请求
```

### 关键文件获取工具

```typescript
// 文件 1: utils/attachments.ts
startRelevantMemoryPrefetch(query) 
  → findRelevantMemories()
  → generateMemoryAttachments()
  → filterDuplicateMemoryAttachments()

// 文件 2: utils/claudemd.ts / memdir/memdir.ts
getClaudeMds() / getMemoryFiles()
  → readdir(memoryDir)
  → filterInjectedMemoryFiles()
  → 返回{ frontmatter, content }

// 文件 3: utils/context.ts
getGitStatus() [memoized]
  → execFileNoThrow(git ...)
  → 截断到 MAX_STATUS_CHARS (2000)

// 文件 4: context.ts
getUserContext()
  → findRelevantMemories(query)
     → scanMemoryFiles() + Sonnet fork
  → 最多 5 个相关
```

### Memoization 和缓存

```typescript
// 系统上下文缓存策略
export const getUserContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    // findRelevantMemories
    // 第一次调用: 完整成本
    // 后续调用: 内存缓存，不重新扫描
  },
)

// 清空缓存的条件
setSystemPromptInjection(value) {
  systemPromptInjection = value
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

---

## 2️⃣ 压缩的完整细节

### 压缩阈值和门控

**关键常量** (`services/compact/autoCompact.ts`):

```typescript
// 有效上下文窗口 = 模型窗口 - 输出保留
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000  // p99.99 紧凑摘要

// 自动压缩触发阈值
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000

// 三个严重程度场景：
// 1. 错误 (error): tokens >= effectiveWindow - 20K
//                  → 必须压缩
// 2. 警告 (warning): tokens >= effectiveWindow - 20K
//                   → 提示用户，建议压缩
// 3. 自动压缩 (autocompact): tokens >= effectiveWindow - 13K
//                           → 如果启用，自动压缩
```

**触发流程**:

```
query() 中每个 turn 后：

checkTokenBudget(currentTokens, model)
  ↓
calculateTokenWarningState(tokenUsage, model)
  ├─ percentLeft = (contextWindow - currentTokens) / contextWindow
  ├─ isAboveErrorThreshold? → 停止，报错
  ├─ isAboveWarningThreshold? → 警告
  └─ isAboveAutoCompactThreshold?
     ↓
     if (isAutoCompactEnabled()) {
       reportAutoCompaction()
       triggerCompaction()  ← 异步？还是同步？
     }
```

### 压缩的方式

**三层压缩机制** (`services/compact/`):

```
┌─────────────────────────────────────────────┐
│  Microcompact (每个 tool result)           │
│  服务: microCompact.ts                      │
│  触发: 单个工具结果太大 (>某阈值)            │
│  方式: 截断或摘要单个 result                │
│  成本: 无（本地操作）                        │
│  可逆: 否（结果被修改）                      │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  Session Memory Compact (可选)            │
│  服务: sessionMemoryCompact.ts              │
│  触发: 如果 session memory 太大              │
│  方式: 使用 Fork Agent 重新摘要             │
│  成本: 一个 fork（缓存命中 ~50 tokens）     │
│  可逆: 否（摘要替代原始内容）                │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  全局 Compact (对话压缩)                    │
│  服务: compact.ts                           │
│  触发: 全局 tokens >= 阈值                  │
│  方式: Fork Agent 摘要 + 生成 compact msg   │
│  成本: 一个 fork（缓存命中 ~100 tokens）    │
│  可逆: 是（保留 boundary marker）           │
└─────────────────────────────────────────────┘
```

### 压缩时的上下文

**Compact Fork 接收的信息**:

```typescript
// 文件: services/compact/compact.ts
async function compactConversation(
  context: REPLHookContext,
  messages: Message[],
  model: string,
  signal: AbortSignal
): Promise<CompactionResult> {
  
  // 构建 fork 的输入
  const postCompactMessages = buildPostCompactMessages(
    messages,
    compactBoundary  // ← 找出要压缩的范围
  );
  
  // 传递给 fork 的信息
  const result = await runForkedAgent({
    promptMessages: [
      createUserMessage({
        content: `Please summarize the conversation between:
        
${postCompactMessages
  .map(msg => `${msg.type}: ${getText(msg)}`)
  .slice(startIdx, endIdx)
  .join('\n\n')}

Key things to preserve:
- Important discoveries
- User decisions and preferences
- Task context and goals
- Any unresolved issues
`
      })
    ],
    cacheSafeParams: {...},  // 缓存上下文
    canUseTool: createAutoMemCanUseTool(),
    forkLabel: 'compact',
    maxOutputTokens: COMPACT_MAX_OUTPUT_TOKENS  // ← 限制摘要长度
  });
  
  return { compactSummary, boundaryMarker };
}
```

### 压缩比例和结果

**预期压缩效果**:

```
典型场景：Long conversation with many tools

输入:
- 消息数: 100+
- Token 数: 200K (超过 128K 窗口)
- Tool results: 50+ 个 (Bash, FileRead, etc)

压缩后:
- 消息数: 20-30 (关键消息+工具摘要)
- Token 数: 30-50K (75-85% 压缩率)
- Tool results: 10-15 (摘要后)

压缩方式:
1. 早期对话中的工具结果 → 摘要成一行
2. 重复或冗长的 assistant 响应 → 1-2 行总结
3. 连续的用户输入 → 合并
4. 保留: 最后的 context window 大小
```

### 压缩结果存放位置和使用时机

**存放位置**:

```
内存中 (messages 数组):
  在 query() 中被替换：
  const messages = [
    ...messagesBeforeCompaction,
    [SystemCompactBoundaryMessage] ← marker
    [CompactSummaryMessage]  ← 摘要
    ...messagesAfterCompaction
  ];
  
文件系统 (可选):
  ~/.claude/projects/<slug>/.transcripts/
    └─ <turn-id>_compact.md  (如果 DUMP_PROMPTS=1)

不持久化到 memdir!
  (Session Memory 会另外更新 sessionMemory.md)
```

**使用时机**:

```
压缩后的下一个 turn：

query() 
  ↓
normalizeMessagesForAPI(messages)
  ├─ 包含压缩边界 marker
  ├─ 包含压缩摘要
  └─ 从边界之后的新消息
  ↓
API 请求
  ├─ [System Prompt]
  ├─ [Compressed Summary]
  ├─ [New messages since boundary]
  └─ [Tool use]

重点：压缩摘要变成"新的独轮"，不能再压缩
  (避免无限嵌套压缩)
```

---

## 3️⃣ Fork Agent 触发和任务分配

### 何时 Fork？

**主动触发的 Fork**:

```
时机 1: 消息开始时 (相关记忆选择)
  条件: findRelevantMemories(query) 被调用
  任务: Sonnet 从记忆列表中选择 ≤5 个
  成本: ~50-200 tokens (缓存命中)
  上下文: 当前查询 + 记忆清单
  文件: memdir/findRelevantMemories.ts

时机 2: 每个对话末 (Extract Memories)
  条件: shouldExtractMemory(messages)
    - 初始化: >= 8K tokens
    - 更新: >= 5K 增长
    - 工具: >= 2 个调用
  任务: 从对话中提取可保存的记忆
  成本: ~100-200 tokens (缓存命中 90%)
  上下文: 对话历史 + 已有记忆
  文件: services/extractMemories/extractMemories.ts

时机 3: 每个对话末 (Session Memory)
  条件: 同上 + isSessionMemoryGateEnabled()
  任务: 更新当前对话的工作摘要
  成本: ~100-200 tokens (缓存命中 90%)
  上下文: 对话历史 + 前次摘要
  文件: services/SessionMemory/sessionMemory.ts

时机 4: 全局压缩时
  条件: tokens >= autoCompactThreshold
  任务: 生成对话摘要，压缩历史
  成本: ~300-500 tokens (缓存命中 90%)
  上下文: 待压缩消息范围
  文件: services/compact/compact.ts

时机 5: 定期梦想整合 (Auto Dream)
  条件: 24小时过期 && 5+个新会话 && 获取锁
  任务: 跨会话知识融合
  成本: ~150-250 tokens (缓存命中 90%)
  上下文: 所有相关记忆文件
  文件: services/autoDream/autoDream.ts

时机 6: 压缩后清理 (Post-Compact)
  条件: compaction 完成
  任务: 可选的 cleanup（如果启用）
  成本: 无或很小
  文件: services/compact/postCompactCleanup.ts
```

### 任务分配详表

```typescript
// 文件: utils/forkedAgent.ts

interface ForkedAgentParams {
  promptMessages: Message[];      // 新的用户/系统提示
  cacheSafeParams: CacheSafeParams;  // 缓存父请求
  canUseTool: CanUseToolFn;       // 权限检查
  querySource: QuerySource;       // 'extract_memories'|'session_memory'|...
  forkLabel: string;              // 日志标签
  maxOutputTokens?: number;       // 输出上限（可选）
  onMessage?: (message: Message) => void;  // 流式回调（通常无）
  skipTranscript?: boolean;       // 跳过转录（通常否）
  skipCacheWrite?: boolean;       // 跳过缓存写（通常否）
}
```

### 具体的 Fork 任务

**任务 1: Sonnet 记忆选择**
```typescript
// Input
promptMessages: [
  { role: 'user', content: `
You are selecting memory files that will be useful to Claude Code.
Given:
- Query: "${query}"
- Recent tools: ["mcp__bash", "file_read"]
- Available memories:
  - [user] expertise.md (2026-04-10): 后端工程师，6年经验...
  - [feedback] pr-strategy.md (2026-04-05): 单 PR vs 多 PR...
  ...

Select up to 5 filenames that are clearly useful.
  ` }
]

// Output (parsed)
selected = ['user/expertise.md', 'feedback/pr-strategy.md']
```

**任务 2: Extract Memories**
```typescript
// Input
promptMessages: [
  { role: 'user', content: buildExtractAutoOnlyPrompt(
    messages,     // 对话历史
    previousMemories,  // 已保存的
    tools     // 使用过的工具
  ) }
]

// Output (parsed and saved)
saveMemoryFile('user/typescript-patterns.md', content);
saveMemoryFile('project/deadline-2026-03-20.md', content);
```

**任务 3: Session Memory Update**
```typescript
// Input
promptMessages: [
  { role: 'user', content: buildSessionMemoryUpdatePrompt(
    previousSessionMemory,  // "# Session Memory\n## Current Task..."
    recentMessages,        // 最后 50 条消息
    usedTools             // 最后使用的工具
  ) }
]

// Output (parsed)
updated = `
# Session Memory

## Current Task
实现缓存层...

## Recent Discoveries
1. Redis 在 dev...
...
`
// 写入 sessionMemory.md (增量)
```

**任务 4: Compact Conversation**
```typescript
// Input
promptMessages: [
  { role: 'user', content: `
Please create a concise summary of this conversation section:

User: "实现 caching"
Assistant: "我来帮助..."
Tool: Bash executed 5 commands
...

Preserve:
- Key decisions
- Discoveries
- Unresolved issues
  ` }
]

// Output
summary = `
用户要求实现缓存层。发现 Redis 可用...
决定使用 30min 过期时间...
`
// 替换消息范围为 summary
```

**任务 5: Auto Dream (Multi-session consolidation)**
```typescript
// Input
promptMessages: [
  { role: 'user', content: buildConsolidationPrompt(
    allMemoryFiles,        // 全部记忆文件
    sessionsSinceLast       // 上次整合后的会话
  ) }
]

// Output
consolidated = {
  'user/emerging-patterns.md': "跨项目发现的设计模式...",
  'project/team-patterns.md': "整个团队的共识..."
}
```

---

## 4️⃣ 压缩与缓存失效的关系处理

### 压缩是否导致缓存失效？

**答案：取决于压缩的位置和方式**

#### 情况 1: Microcompact (单个工具结果）

```
是否失效缓存？ 否
理由: 只修改了消息数组中的单个元素
      不改变系统提示、工具定义或消息前缀的哈希
效应: 后续 fork 仍然使用原缓存
成本: 零额外开销
```

#### 情况 2: Session Memory Compact

```
是否失效缓存？ 否
理由: 作为一个单独的消息被插入，不改变前缀
      前缀仍然是 "真实对话的起点"
效应: Extract/Dream fork 继续命中缓存
成本: 零额外开销
实现: sessionMemoryCompact() 返回新消息，插入到消息数组
```

#### 情况 3: 全局压缩 (Compact Conversation)

```
是否失效缓存？ 不完全失效！

核心机制：
1. 压缩生成摘要，作为一条消息插入
2. 建立"紧凑边界标记" (SystemCompactBoundaryMessage)
3. 边界之前的内容被摘要替代
4. 边界之后的内容保持不变

所以：
┌─────────────────────────────────┐
│ Before:  [msg1, msg2, ..., msg100]  │
├─────────────────────────────────┤
│ Compact: [msg1-99 → SUMMARY]        │
│          [boundary marker]          │
│          [msg100]                   │
└─────────────────────────────────┘

后续 fork 的消息前缀：
  [boundary marker + summary + msg100]

这是"新的缓存起点"，与之前不同：
- 消息前缀内容变了 → 缓存键不同 → 缓存失效
- 但节省的 token > 缓存重建成本
```

### 如何处理缓存失效

**机制 1: API 通知** (`services/api/promptCacheBreakDetection.ts`)

```typescript
// 压缩完成后通知 API
notifyCompaction({
  turnsBefore: turnCountBeforeCompaction,
  tokensBeforeCached: ...,
  tokensSaved: ...,
  ...
});

// Anthropic API 内部可能记录这个事件
// 帮助优化缓存策略
```

**机制 2: saveCacheSafeParams (Post-Compact)**

```typescript
// 文件: utils/forkedAgent.ts

// 压缩后，更新"最后的缓存安全参数"
saveCacheSafeParams(lastCacheSafeParams);

// 下一个非 fork 对话请求时，会使用新的参数
// 建立新的缓存前缀

// 后续 fork 会使用这个新前缀
// 而不是旧的（已过期的）缓存
```

**机制 3: 缓存命中但内容不同的情况**

```
罕见情况：

如果用户手动编辑了 systemPromptInjection：
  setSystemPromptInjection("new value")
    → 清空 getUserContext 缓存
    → 清空 getSystemContext 缓存

后续请求：
  systemPrompt 改变 → 缓存键改变 → 自动失效和重建
  成本: 新的 API 调用（无增量费用，完整成本）
```

---

## 5️⃣ 后台记忆整合：一致性模式

### 整合是增量还是重写？

**答案：增量更新，但有陷阱**

#### Extract Memories (增量)

```typescript
// 文件: services/extractMemories/extractMemories.ts

// 防重复机制
const previousExtractions = await readExtractionLog();  // 追踪之前提取了什么
const messagesSinceLastExtraction = messages.slice(
  indexOf(messages, lastMemoryMessageUuid)
);  // 只看新增部分

const newMemories = filterDuplicates(
  extractedMemories,
  previousExtractions  // 去重
);

// 保存
for (const mem of newMemories) {
  await saveMemoryFile(mem.path, mem.content);
}
```

**结果**：
```
内存文件中：
  user/expertise.md (2026-04-10)
  user/typescript-patterns.md (2026-04-15) ← 新增
  feedback/pr-strategy.md (2026-04-05)

没有重写，只是追加了新文件。
现有文件不修改（除非重复）。
```

#### Session Memory (增量覆盖)

```typescript
// 文件: services/SessionMemory/sessionMemory.ts

// 不是追加，而是覆盖同一个 sessionMemory.md

const previousSessionMemory = await getSessionMemoryContent();
const updated = await forkAgent.summarize(
  previousSessionMemory,  // 旧摘要作为输入
  recentMessages
);

await saveFile(sessionMemoryPath, updated);  // 覆盖
```

**结果**：
```
sessionMemory.md （单一文件，反复更新）

turn 1-2:
  # Session Memory
  ## Current Task
  实现 caching
  ## Discoveries
  Redis 可用

turn 3-4:  (增量更新)
  # Session Memory
  ## Current Task
  实现 caching + 测试
  ## Discoveries
  Redis 可用 + found token mismatch
  ## Decisions
  (新增的决策)
```

#### Auto Dream (陈旧覆盖)

```typescript
// 文件: services/autoDream/autoDream.ts

const consolidated = await dreamFork.consolidate(
  allMemoryFiles,
  sessionsSinceLast
);

// 陈旧覆盖：消除并重建相关的记忆文件
for (const [path, newContent] of Object.entries(consolidated)) {
  if (shouldConsolidate(path)) {
    await saveMemoryFile(path, newContent);  // 覆盖
  }
}
```

**风险**：信息丢失

```
示例：
之前: user/typescript-patterns.md
  "从项目 A 学到的 TypeScript 模式"

更新: 24小时后，新的 dream
  从 5 个会话中选择性地重新编写
  可能：部分模式被简化或删除

结果: 信息丢失的潜在性
```

### 一致性保证

**情况 1: 从用户的角度（内存文件）**

```
强一致性：写入立即可见
  - 用户手动编辑 / 或看到 Extract 后的文件
  - 下一个对话立即加载新内容

最终一致性：不存在
  - 每个 fork 都同步到 API（等待结果）
  - 没有"异步写入"的延迟
```

**情况 2: 从 API 对话的角度（消息流）**

```
最终一致性：缓存和新消息
  - 消息被立即发送给 API
  - 后台 fork 可能在"压缩时"改变后续对话的前缀
  - 但前一个对话的响应已经生成，不受影响

具体：
┌─────────────────────────┐
│ Turn 1: User → API      │  ← 同步，用户看到响应
│ [Fork: Extract memories]│  ← 异步，后台进行
│                         │
│ Turn 2: User → API      │  ← 可能以压缩后的前缀开始
│ [Fork: Session update]  │  ← 异步
└─────────────────────────┘
```

**情况 3: 记忆文件的一致性**

```
CAP 定理的应用：

一致性 (Consistency):
  - 新提取的记忆可能与旧记忆有矛盾
  - Dream 可能改写旧记忆（不是 100% 保留）
  - 解决: 用户可以手动合并或选择

可用性 (Availability):
  - 记忆文件 mount 在本地，总是可用
  - 网络故障不影响本地访问
  
容错性 (Partition tolerance):
  - 多个 Agent fork 可能同时写入同一文件
  - 当前: 无锁定，最后写入获胜

实现: 没有强 ACID 保证，依赖于：
  1. 单线程主循环（大部分写入串行化）
  2. 后台 fork 的写入不太频繁
  3. 文件级别的原子操作（write + rename）
```

---

## 6️⃣ 权限控制实现

### 是基于系统权限还是 Prompt 提示？

**答案：两者都有，混合模式**

#### 层 1: 系统权限 (文件系统）

```typescript
// 文件: utils/permissions/filesystem.ts

// 检查文件系统权限
canAccessPath(path, accessMode) {
  const realPath = resolveAndNormalize(path);
  
  // 白名单检查
  if (!isAllowedPath(realPath)) {
    throw PermissionDeniedError;
  }
  
  // 符号链接检查 (避免逃逸)
  if (isSymlink(realPath)) {
    if (!mayFollowSymlink(realPath)) {
      throw SymlinkNotAllowedError;
    }
  }
  
  // 系统级权限检查
  try {
    fs.accessSync(realPath, accessMode);
  } catch {
    throw SystemPermissionDeniedError;
  }
}

// 示例
canAccessPath('/home/user/project/file.ts', 'read')
  // ✓ 允许：在项目目录内

canAccessPath('/etc/passwd', 'read')
  // ✗ 拒绝：超出项目范围
  
canAccessPath('/home/user/.ssh/id_rsa', 'read')
  // ✗ 拒绝：敏感目录
```

#### 层 2: Prompt 提示权限（工具使用）

```typescript
// 文件: utils/permissions/permissions.ts
// 类型: CanUseToolFn

export type CanUseToolFn = (
  toolName: string,
  toolInput: unknown,
  context: ToolUseContext
): Promise<PermissionResult>;

// 使用场景

async function canUseTool(
  toolName: 'bash',
  toolInput: { command: 'rm -rf /' },  // 危险命令
  context: ToolUseContext
): Promise<PermissionResult> {
  
  // 1. 检查全局规则
  const rule = getPermissionRule(toolName);
  if (rule.mode === 'deny') {
    return { decision: 'deny', reason: 'tool_blocked' };
  }
  if (rule.mode === 'allow') {
    return { decision: 'allow', reason: 'blanket_rule' };
  }
  
  // 2. 检查详细规则（Bash 分类）
  if (toolName === 'bash') {
    const classification = classifyBashCommand(toolInput.command);
    if (classification.isDangerous) {
      // 3. 看权限模式
      const mode = context.getAppState().permissionMode;
      
      if (mode === 'auto') {
        // 使用 Bash 分类器
        if (classification.autoApprove) {
          return { decision: 'allow', reason: 'auto_approved' };
        } else {
          // 降级到 ask（需要用户确认）
          return {
            decision: 'ask',
            reason: 'classifier_uncertain',
            explanation: `Command seems ${classification.risk}...`
          };
        }
      } else if (mode === 'manual') {
        // 总是问
        return {
          decision: 'ask',
          reason: 'manual_mode',
          explanation: 'Command classification: ' + classification.classification
        };
      } else if (mode === 'deny') {
        return { decision: 'deny', reason: 'mode_deny' };
      }
    }
  }
  
  return { decision: 'allow', reason: 'safe_command' };
}
```

#### 层 3: 权限规则存储和加载

```typescript
// 文件: utils/permissions/permissionsLoader.ts

// 从多个来源加载规则（优先级）
loadAllPermissionRulesFromDisk():
  1. 项目级 settings.json (高优先级)
     ~/.claude/projects/<slug>/.claude/settings.json
     {
       "permissions": {
         "bash": "manual",
         "rules": [
           { "tool": "bash", "pattern": "npm test", "allow": true },
           { "tool": "bash", "pattern": "rm", "deny": true }
         ]
       }
     }
  
  2. 用户级 settings.json (中优先级)
     ~/.claude/settings.json
     {
       "permissions": { ... }
     }
  
  3. 默认规则 (低优先级)
     内置的安全规则集

// 运行时应用
applyPermissionRulesToPermissionContext(
  rules,
  toolPermissionContext
):
  → 更新 getAppState() 中的 permissionContext
  → 下一个工具检查时使用
```

#### 层 4: 分类器决策（智能判断）

```typescript
// 文件: utils/permissions/bashClassifier.ts + classifierDecision.ts

classifyBashCommand(command: string):
  → 分析命令结构
  → 检测危险模式 (dangerousPatterns.ts)
  → 返回: { isDangerous, riskLevel, autoApprove, classification }

// 危险模式示例
const DANGEROUS_BASH_PATTERNS = [
  /^(rm|dd|mkfs|>.+\/dev\/).*-rf/,  // 递归删除，格式化
  /^(exec|source|eval|.+\$\(|\${SHELL})/,  // 注入执行
  /^.*sudo.*(?!test|check|--version)/,  // 提权
  ...
];

// 自动批准的模式
const AUTO_APPROVE_PATTERNS = [
  /^cd\s/,           // 目录切换
  /^(ls|cat|grep)/,  // 读操作
  /^npm test/,       // 测试
  ...
];
```

#### 权限决定流程图

```
用户要求: "bash: rm -rf node_modules"
  ↓
canUseTool('bash', {...})
  ├─ 全局规则检查?
  │  ├─ mode === 'deny' → [拒绝]
  │  ├─ mode === 'allow' → [允许]
  │  └─ mode === 'auto/manual' → 继续
  │
  ├─ Bash 分类
  │  ├─ 模式匹配: "rm -rf" → 危险
  │  ├─ 自动批准? 否
  │  └─ 分类: [高风险]
  │
  ├─ 权限模式
  │  ├─ 'auto' → classifier 说"不确定" → [问]
  │  ├─ 'manual' → [总是问]
  │  └─ 'deny' → [拒绝]
  │
  └─ 返回: { decision: 'ask', reason: '...', explanation: '...' }
      ↓
  提示用户 (UI):
    "⚠️ Command seems high-risk: rm -rf node_modules
     reason: matches dangerous deletion pattern
     [Allow] [Deny] [Add rule]"
  ↓
  用户选择 "Allow"
  ↓
  执行工具
```

---

## 7️⃣ 非代码文件的完整清单

### 目录结构和优先级

```
~/.claude/
├─ projects/
│  └─ <project-slug>/          [PER-PROJECT SCOPE]
│     ├─ memory/               [长期记忆库 - 高优先级]
│     │  ├─ MEMORY.md          (索引, ≤200行/25KB)
│     │  ├─ user/
│     │  │  ├─ expertise.md
│     │  │  ├─ preferences.md
│     │  │  └─ ...
│     │  ├─ feedback/
│     │  │  ├─ refactor-rules.md
│     │  │  ├─ testing-policy.md
│     │  │  └─ ...
│     │  ├─ project/
│     │  │  ├─ roadmap.md
│     │  │  ├─ deadlines.md
│     │  │  └─ ...
│     │  └─ reference/
│     │     ├─ external-tools.md
│     │     ├─ team-channels.md
│     │     └─ ...
│     │
│     ├─ .claude/              [项目配置]
│     │  ├─ settings.json      (权限, 工具配置, 偏好)
│     │  ├─ sessionMemory.md   (单次对话摘要 - 临时)
│     │  ├─ .transcripts/      (对话记录 - 调试用)
│     │  │  ├─ <session-id>_*.md
│     │  │  ├─ <turn-id>_compact.md
│     │  │  └─ ...
│     │  ├─ .tasks/            (背景任务持久化)
│     │  │  └─ <task-id>.json
│     │  └─ .state/            (会话状态)
│     │     ├─ lastSessionId.txt
│     │     ├─ compactionState.json
│     │     └─ ...
│     │
│     └─ plan.md              (当前 plan 模式的计划 - 临时)
│
├─ settings.json              [全局用户设置 - 中优先级]
│  ├─ permissions
│  ├─ defaultModel
│  ├─ tools
│  ├─ autoMemoryEnabled
│  └─ ...
│
├─ models/                    [已下载的离线模型 - 运行时]
│  └─ ...
│
├─ extensions/               [VS Code 扩展配置 - 可选]
│  └─ ...
│
└─ cache/                     [临时缓存 - 运行时]
   └─ ...
```

### 文件详解

#### 1. Long-term Memory Files

**MEMORY.md** (索引)
- 位置: `~/.claude/projects/<slug>/memory/MEMORY.md`
- 大小限制: ≤200 行 + ≤25 KB
- 格式:
  ```markdown
  # Memory Index
  
  ## Available Memories
  - [user] expertise.md (2026-04-10): TypeScript + React 5년
  - [feedback] pr-strategy.md (2026-04-05): Single PR for refactors
  - [project] deadline.md (2026-03-20): Release freeze on 2026-03-05
  
  ## Custom Instructions
  (Optional: 用户自定义指导)
  ```
- 优先级: 最高（对话开始时立即加载）
- 刷新: 每个对话开始时扫描（通过 Sonnet 选择相关）
- 修改: Extract Memories fork 可能增加条目；梦想可能更新

**user/*.md** (用户身份)
- 文件名: expertise.md, background.md, preferences.md, ...
- 格式:
  ```markdown
  ---
  type: user
  description: 用户的 TypeScript 和 React 经验
  ---
  
  # User Profile
  
  ...
  ```
- 何时保存: Extract Agent 在学习用户偏好时
- 何时读: 对话开始时 Sonnet 选择，或手动指定
- 优先级: 中

**feedback/*.md** (工作指导)
- 文件名: refactor-rules.md, testing-policy.md, communication-style.md, ...
- 格式:
  ```markdown
  ---
  type: feedback
  description: 重构时该用单 PR 还是多 PR
  ---
  
  ## 规则
  单 PR 优于多 PR 用于：...
  原因: 减少 CI/CD 费用
  
  何时应用: 大规模重构
  反例: ...
  ```
- 何时保存: Extract Agent 在捕获用户纠正时
- 何时读: 每次工作开始时相关选择
- 优先级: 高（"如何工作")

**project/*.md** (进行中的工作)
- 文件名: roadmap.md, current-sprint.md, blockers.md, ...
- 格式:
  ```markdown
  ---
  type: project
  description: Q2 发布冻结时间表
  ---
  
  ## 重要日期
  - 2026-03-05: 合并冻结开始
  - 2026-03-20: 发布
  
  原因: 移动端需要稳定期
  影响: 非紧急 PR 暂停
  ```
- 何时保存: Extract Agent 在听到截止日期/目标时
- 何时读: 评估工作优先级或时间限制时
- 优先级: 中（时间敏感）

**reference/*.md** (外部系统指针)
- 文件名: linear-project.md, slack-channel.md, docs-link.md, ...
- 格式:
  ```markdown
  ---
  type: reference
  description: 管道错误在这里追踪
  ---
  
  ## Linear Project "INGEST"
  - URL: linear.app/team/INGEST
  - 用途: 所有数据管道 bug
  - 所有者: Data Platform 团队
  ```
- 何时保存: Extract Agent 当用户提及工具/系统
- 何时读: 需要找外部信息时
- 优先级: 低（支持）

#### 2. Session Memory (临时，单次对话)

**sessionMemory.md**
- 位置: `~/.claude/projects/<slug>/.claude/sessionMemory.md` (或内存)
- 生命周期: 单次对话内
- 刷新: 每 5K-8K token 增长后更新
- 格式:
  ```markdown
  # Session Memory
  
  ## Current Task
  实现缓存层以提高 API 响应...
  
  ## Discoveries
  1. Redis 在 dev 环 (localhost:6379)
  2. 现有缓存在 3 个地方
  
  ## Decisions
  - 统一 30min 过期
  - 選 Redis vs 内存
  
  ## Known Issues
  - Redis 连接池泄漏
  
  ## Next Steps
  - [ ] 实现 Redis adapter
  ```
- 优先级: 中（当前对话的工作状态）
- 删除: 对话结束（不持久化到 memdir）或被压缩摘要替代

#### 3. 项目配置

**settings.json** (项目级)
- 位置: `~/.claude/projects/<slug>/.claude/settings.json`
- 格式:
  ```json
  {
    "autoMemoryEnabled": true,
    "permissionMode": "auto|manual|deny",
    "permissions": {
      "bash": "custom",
      "rules": [
        {
          "tool": "bash",
          "pattern": "npm test",
          "action": "allow"
        },
        {
          "tool": "bash",
          "pattern": "rm",
          "action": "deny"
        }
      ]
    },
    "tools": {
      "enabled": ["bash", "file_read", "file_edit"],
      "disabled": ["power_shell"]
    },
    "defaultModel": "claude-3-5-sonnet-20241022",
    "context": {
      "maxTokens": 150000
    }
  }
  ```
- 优先级: 项目级配置最高（覆盖全局）
- 刷新: 会话开始时加载
- 修改: 用户手动编辑或通过 CLI 命令

**settings.json** (全局用户)
- 位置: `~/.claude/settings.json`
- 格式: 同上，但为全局默认
- 优先级: 低（被项目级覆盖）

#### 4. 运行时状态

**Transcripts (会话记录)**
- 位置: `~/.claude/projects/<slug>/.claude/.transcripts/`
- 文件: `<session-id>_turn-<n>.md`, `<turn-id>_compact.md`
- 用途: 调试 + 审计
- 清除: 用户可以手動刪除
- 启用: `DUMP_PROMPTS=1` 环境变量

**Compaction State**
- 位置: `~/.claude/projects/<slug>/.claude/.state/compactionState.json`
- 格式:
  ```json
  {
    "lastCompactedAt": 1713139200,
    "lastCompactedMessageId": "msg-uuid-xxx",
    "consecutiveFailures": 0,
    "postCompactCleanupDone": true
  }
  ```
- 用途: 追踪压缩历史，防止重复压缩同一范围
- 生命周期: 跨会话持久化

**Session State**
- 位置: `~/.claude/projects/<slug>/.claude/.state/sessionState.json`
- 内容:
  ```json
  {
    "sessionId": "...",
    "createdAt": ...,
    "lastTurnAt": ...,
    "autoCompactTracking": {
      "compacted": false,
      "turnCounter": 5,
      "consecutiveFailures": 0
    },
    "sessionMemoryExtracted": true
  }
  ```
- 用途: 恢复会话，追踪自动压缩状态

**Task Output (后台任务)**
- 位置: `~/.claude/projects/<slug>/.claude/.tasks/<task-id>.json`
- 用途: 追踪 Dream/Extract/Session 的状态
- 生命周期: 任务完成后删除

#### 5. 临时文件

**Plan (计划模式)**
- 位置: `<project-root>/plan.md`
- 生命周期: 计划模式期间
- 格式: Markdown，用户写的任务列表
- 删除: 计划完成或模式退出

**Session Lock (分布式)**
- 位置: `~/.claude/projects/<slug>/.claude/.state/dream.lock`
- 用途: 防止多个进程同时运行 auto-dream
- 生命周期: Dream 运行期间持有，完成/失败时释放

#### 6. 清单和优先级总结

| 文件 | 位置 | 范围 | 优先级 | 生命周期 | 何时读 | 何时写 |
|-----|------|------|-------|---------|-------|-------|
| MEMORY.md | memory/ | 项目 | 最高 | 跨会话 | 对话开始 | Extract/Dream |
| user/*.md | memory/user/ | 项目 | 高 | 跨会话 | 需要定制 | Extract |
| feedback/*.md | memory/feedback/ | 项目 | 最高 | 跨会话 | 每项工作 | Extract |
| project/*.md | memory/project/ | 项目 | 中 | 跨会话 | 评估影响 | Extract |
| reference/*.md | memory/reference/ | 项目 | 低 | 跨会话 | 需要外部信息 | Extract |
| sessionMemory.md | .claude/ | 会话 | 中 | 单会话 | 压缩/恢复 | Session fork |
| settings.json | .claude/ | 项目 | 高 | 跨会话 | 会话开始 | 用户手動 |
| transcripts | .transcripts/ | 会话 | 低 | 会话 | 调试 | System |
| compactionState.json | .state/ | 项目 | 中 | 跨会话 | 压缩触发 | System |
| sessionState.json | .state/ | 会话 | 中 | 会话 | 恢复 | System |
| dream.lock | .state/ | 项目 | 高 | 临时 | Dream | System |

---

## 总结对比表

### 7 个核心问题的答案速查表

| # | 问题 | 关键答案 | 关键文件 |
|----|------|--------|--------|
| 1 | 上下文来源 | Sonnet fork 选择 ≤5 相关，系统提示注入 | context.ts, findRelevantMemories.ts |
| 2 | 压缩详情 | 三层: Micro/SM/全局; 13K buffer; 缓存边界保留 | autoCompact.ts, compact.ts |
| 3 | Fork 时机 | 6 个场景: 选择/提取/摘要/压缩/梦想/清理 | forkedAgent.ts, 各 fork 文件 |
| 4 | 缓存失效 | Microcompact 无，全局紧凑边界后重建 | notifyCompaction, saveCacheSafeParams |
| 5 | 一致性 | 最终一致，无强 ACID；增量写入防重复 | 各模块的 +去重和覆盖逻辑 |
| 6 | 权限 | 混合: 规则+Bash分类+提示确认 | permissions.ts, bashClassifier.ts |
| 7 | 文件结构 | 7 层: memdir/settings/.claude/.state/.transcripts | 详见文件清单 |

---

## 关键洞察

### 1. 消息流的"上下文交割"
```
对话前: getuserContext() [Sonnet fork] → memdir 扫描
对话中: normalizeMessagesForAPI() → 注入 userContext + systemContext
对话中: Bash 权限检查 [Prompt] → 若有分类器不确定则询问
对话末: 压缩 [fork] → saveCompactSafeParams() → 更新缓存基线
```

### 2. 多重并发而非并行
```
主线程: 用户看到响应（立即）
后台 1: Extract fork（无阻塞）
后台 2: Session fork（无阻塞）
定期 3: Dream fork（24h + 5会话）

没有真正的 parallelism，依赖 event loop 的异步调度。
```

### 3. 缓存的分层
```
第一次请求: 100% 成本
Extract fork: 10% 成本 (缓存命中)
Session fork: 10% 成本 (缓存命中)
全局压缩后: 缓存失效 → 新基线 → 后续 fork 又是 10%
```

### 4. 权限的三层防护
```
层 1: 系统权限（文件实际存在和可访问）
层 2: 规则权限（settings.json 策略）
层 3: Prompt 权限（分类器 + 实时确认）

设计哲学: 宽进细出（允许优先），用户可覆盖。
```

### 5. 文件系统作为"记忆存储"
```
不是数据库，而是 Markdown + JSON
- 用户可以 git commit
- 用户可以手动编辑和合并
- 跨机器同步（dotfiles）
- 导出友好（Plain text）

成本: 启动时扫描，但通常 <100ms
```

