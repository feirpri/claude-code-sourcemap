# Claude Code 源代码文件路径完整索引

> 本文档提供所有关键功能模块的源文件路径，便于深度研究和代码审查

---

## 📁 文件夹结构总览

```
restored-src/src/
├── memdir/                      # 长期记忆系统
├── services/
│  ├── SessionMemory/            # 短期记忆系统
│  ├── extractMemories/          # Extract Memories Agent
│  ├── autoDream/                # Auto Dream 融合
│  ├── compact/                  # 对话压缩系统
│  ├── analytics/                # 分析和遥测
│  └── ...
├── utils/
│  ├── permissions/              # 权限控制系统
│  ├── forkedAgent.ts            # Forked Agent 基础
│  ├── hooks/                    # 钩子系统
│  ├── messages.ts               # 消息操作
│  ├── tokens.ts                 # Token 计算
│  └── ...
├── context.ts                   # 上下文构建入口
├── query.ts                     # 主循环（消息处理）
├── query/                       # 查询协调
└── ...
```

---

## 🔷 长期记忆系统 (Memory Directory)

### 核心模块

| 文件 | 功能 | 关键函数/类 |
|-----|------|----------|
| [memdir/memdir.ts](restored-src/src/memdir/memdir.ts) | 索引管理和内容截断 | `truncateEntrypointContent()`, `ensureMemoryDirExists()`, `DIR_EXISTS_GUIDANCE` |
| [memdir/memoryTypes.ts](restored-src/src/memdir/memoryTypes.ts) | 记忆类型定义 | `MEMORY_TYPES`, `parseMemoryType()`, `TYPES_SECTION_COMBINED` |
| [memdir/memoryScan.ts](restored-src/src/memdir/memoryScan.ts) | 文件扫描和清单生成 | `scanMemoryFiles()`, `formatMemoryManifest()`, `MemoryHeader` |
| [memdir/findRelevantMemories.ts](restored-src/src/memdir/findRelevantMemories.ts) | Sonnet 相关性选择 | `findRelevantMemories()`, `selectRelevantMemories()`, `SELECT_MEMORIES_SYSTEM_PROMPT` |
| [memdir/memoryAge.ts](restored-src/src/memdir/memoryAge.ts) | 新鲜度计算与标注 | `memoryAgeDays()`, `memoryAge()`, `memoryFreshnessText()`, `memoryFreshnessNote()` |
| [memdir/paths.ts](restored-src/src/memdir/paths.ts) | 路径解析和启用控制 | `getAutoMemPath()`, `isAutoMemoryEnabled()`, `getMemoryBaseDir()` |
| [memdir/teamMemPaths.ts](restored-src/src/memdir/teamMemPaths.ts) | 团队记忆路径（可选） | 动态加载（feature: TEAMMEM） |

### 集成点

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [context.ts](restored-src/src/context.ts) | 系统提示构建和上下文注入 | `getUserContext()`, `getSystemContext()`, `getGitStatus()` |
| [utils/attachments.ts](restored-src/src/utils/attachments.ts) | 内存附件生成 | `startRelevantMemoryPrefetch()`, `getAgentListingDeltaAttachment()` |
| [utils/claudemd.ts](restored-src/src/utils/claudemd.ts) | Claude.md 处理 | `getClaudeMds()`, `getMemoryFiles()`, `filterInjectedMemoryFiles()` |

---

## 🔷 短期记忆系统 (Session Memory)

### 核心模块

| 文件 | 功能 | 关键函数/类 |
|-----|------|----------|
| [services/SessionMemory/sessionMemory.ts](restored-src/src/services/SessionMemory/sessionMemory.ts) | 主逻辑和初始化 | `shouldExtractMemory()`, `initSessionMemory()`, `runSessionMemoryExtraction()` |
| [services/SessionMemory/sessionMemoryUtils.ts](restored-src/src/services/SessionMemory/sessionMemoryUtils.ts) | 状态管理 | `hasMetInitializationThreshold()`, `hasMetUpdateThreshold()`, `countToolCallsSince()`, `SessionMemoryConfig` |
| [services/SessionMemory/prompts.ts](restored-src/src/services/SessionMemory/prompts.ts) | 提示模板 | `buildSessionMemoryUpdatePrompt()`, `loadSessionMemoryTemplate()` |

### 与压缩的交互

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [services/compact/sessionMemoryCompact.ts](restored-src/src/services/compact/sessionMemoryCompact.ts) | Session 压缩 | `truncateSessionMemoryForCompact()`, `getSessionMemoryCompactConfig()` |
| [services/compact/compact.ts](restored-src/src/services/compact/compact.ts) | 全局压缩（包含 SM 处理） | 见压缩系统部分 |

---

## 🔷 后台助手系统 (Background Agents)

### Extract Memories

| 文件 | 功能 | 关键函数/类 |
|-----|------|----------|
| [services/extractMemories/extractMemories.ts](restored-src/src/services/extractMemories/extractMemories.ts) | 提取主逻辑 | `initExtractMemories()`, `spawnMemoryExtraction()`, `createAutoMemCanUseTool()` |
| [services/extractMemories/prompts.ts](restored-src/src/services/extractMemories/prompts.ts) | 提示和指导 | `buildExtractAutoOnlyPrompt()`, `buildExtractCombinedPrompt()` |

### Session Memory (在后台)

见短期记忆系统部分

### Auto Dream

| 文件 | 功能 | 关键函数/类 |
|-----|------|----------|
| [services/autoDream/autoDream.ts](restored-src/src/services/autoDream/autoDream.ts) | Dream 主逻辑 | `initAutoDream()`, `runAutoDream()` |
| [services/autoDream/consolidationPrompt.ts](restored-src/src/services/autoDream/consolidationPrompt.ts) | Dream 提示 | `buildConsolidationPrompt()` |
| [services/autoDream/consolidationLock.ts](restored-src/src/services/autoDream/consolidationLock.ts) | 分布式锁和会话扫描 | `tryAcquireConsolidationLock()`, `listSessionsTouchedSince()`, `readLastConsolidatedAt()` |
| [services/autoDream/config.ts](restored-src/src/services/autoDream/config.ts) | 配置管理 | `isAutoDreamEnabled()` |

### Forked Agent 基础

| 文件 | 功能 | 关键函数/类 |
|-----|------|----------|
| [utils/forkedAgent.ts](restored-src/src/utils/forkedAgent.ts) | Fork 实现和缓存管理 | `runForkedAgent()`, `createSubagentContext()`, `CacheSafeParams`, `saveCacheSafeParams()` |
| [utils/sessionStorage.ts](restored-src/src/utils/sessionStorage.ts) | 会话存储和转录 | `recordSidechainTranscript()`, `getTranscriptPath()` |

### 钩子系统

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [query/stopHooks.ts](restored-src/src/query/stopHooks.ts) | 停止钩子协调 | `handleStopHooks()` |
| [utils/hooks/postSamplingHooks.ts](restored-src/src/utils/hooks/postSamplingHooks.ts) | 后采样钩子注册 | `registerPostSamplingHook()`, `executePostSamplingHooks()` |
| [utils/hooks.ts](restored-src/src/utils/hooks.ts) | 其他钩子 | `executePreCompactHooks()`, `executePostCompactHooks()` |

---

## 🔷 压缩系统 (Compaction)

### 自动压缩

| 文件 | 功能 | 关键函数/常量 |
|-----|------|----------|
| [services/compact/autoCompact.ts](restored-src/src/services/compact/autoCompact.ts) | 自动压缩条件和配置 | `getAutoCompactThreshold()`, `getEffectiveContextWindowSize()`, `calculateTokenWarningState()`, `AUTOCOMPACT_BUFFER_TOKENS` |
| [services/compact/compact.ts](restored-src/src/services/compact/compact.ts) | 压缩核心（大文件！） | `compactConversation()`, `buildPostCompactMessages()`, `createPlanAttachmentIfNeeded()` |
| [services/compact/sessionMemoryCompact.ts](restored-src/src/services/compact/sessionMemoryCompact.ts) | Session 记忆压缩 | `truncateSessionMemoryForCompact()`, `SessionMemoryCompactConfig` |

### 压缩工具

| 文件 | 功能 | 关键函数/类 |
|-----|------|----------|
| [services/compact/microCompact.ts](restored-src/src/services/compact/microCompact.ts) | 微级压缩（单工具结果） | `estimateMessageTokens()` |
| [services/compact/snipCompact.ts](restored-src/src/services/compact/snipCompact.ts) | 历史片段压缩（feature: HISTORY_SNIP） |  |
| [services/compact/reactiveCompact.ts](restored-src/src/services/compact/reactiveCompact.ts) | 反应式压缩（feature: REACTIVE_COMPACT） |  |
| [services/compact/apiMicrocompact.ts](restored-src/src/services/compact/apiMicrocompact.ts) | API 级微压缩 | `createApiMicrocompactContext()` |

### 压缩状态和清理

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [services/compact/compactWarningState.ts](restored-src/src/services/compact/compactWarningState.ts) | 警告和状态 |  |
| [services/compact/postCompactCleanup.ts](restored-src/src/services/compact/postCompactCleanup.ts) | 压缩后清理 | `runPostCompactCleanup()` |
| [services/compact/grouping.ts](restored-src/src/services/compact/grouping.ts) | 消息分组 |  |
| [services/compact/prompt.ts](restored-src/src/services/compact/prompt.ts) | 压缩提示 | `getCompactUserSummaryMessage()` |

### 缓存和优化

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [services/api/promptCacheBreakDetection.ts](restored-src/src/services/api/promptCacheBreakDetection.ts) | 缓存失效通知 | `notifyCompaction()` |

---

## 🔷 权限控制系统

### 权限核心

| 文件 | 功能 | 关键函数/类 |
|-----|------|----------|
| [utils/permissions/permissions.ts](restored-src/src/utils/permissions/permissions.ts) | 权限检查入口 | `canUseTool()` (CanUseToolFn) |
| [utils/permissions/PermissionMode.ts](restored-src/src/utils/permissions/PermissionMode.ts) | 权限模式定义 | `auto\|manual\|ask\|deny` |
| [utils/permissions/PermissionRule.ts](restored-src/src/utils/permissions/PermissionRule.ts) | 规则类型定义 | `PermissionRule`, `PermissionRuleValue` |
| [utils/permissions/PermissionResult.ts](restored-src/src/utils/permissions/PermissionResult.ts) | 决策结果 | `PermissionDecision`, `PermissionResult` |

### 权限加载和管理

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [utils/permissions/permissionSetup.ts](restored-src/src/utils/permissions/permissionSetup.ts) | 初始化和配置 | `loadAllPermissionRulesFromDisk()`, `applyPermissionRulesToPermissionContext()` |
| [utils/permissions/permissionsLoader.ts](restored-src/src/utils/permissions/permissionsLoader.ts) | 规则加载 | `loadAllPermissionRulesFromDisk()`, `deletePermissionRuleFromSettings()` |
| [utils/permissions/PermissionUpdate.ts](restored-src/src/utils/permissions/PermissionUpdate.ts) | 规则更新 | `applyPermissionUpdate()`, `persistPermissionUpdates()` |
| [utils/permissions/PermissionUpdateSchema.ts](restored-src/src/utils/permissions/PermissionUpdateSchema.ts) | 更新 schema | (类型定义) |
| [utils/permissions/permissionRuleParser.ts](restored-src/src/utils/permissions/permissionRuleParser.ts) | 规则解析 | `permissionRuleValueFromString()` |

### 分类器和决策

| 文件 | 功能 | 关键函数/类 |
|-----|------|----------|
| [utils/permissions/bashClassifier.ts](restored-src/src/utils/permissions/bashClassifier.ts) | Bash 命令分类 | `classifyBashCommand()` |
| [utils/permissions/classifierDecision.ts](restored-src/src/utils/permissions/classifierDecision.ts) | 分类器决策（feature: TRANSCRIPT_CLASSIFIER） | `getClassifierDecision()` |
| [utils/permissions/dangerousPatterns.ts](restored-src/src/utils/permissions/dangerousPatterns.ts) | 危险模式检测 | `DANGEROUS_BASH_PATTERNS`, `isDangerousBashPermission()` |
| [utils/permissions/yoloClassifier.ts](restored-src/src/utils/permissions/yoloClassifier.ts) | YOLO 模式（自动批准） | `getYoloApprovedPatterns()` |

### 路径和文件系统权限

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [utils/permissions/pathValidation.ts](restored-src/src/utils/permissions/pathValidation.ts) | 路径验证 | `validatePath()`, `isAllowedPath()` |
| [utils/permissions/filesystem.ts](restored-src/src/utils/permissions/filesystem.ts) | 文件系统权限 | `canAccessPath()` |
| [utils/permissions/denialTracking.ts](restored-src/src/utils/permissions/denialTracking.ts) | 拒绝追踪 | `recordDenial()`, `DENIAL_LIMITS` |

### 其他权限工具

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [utils/permissions/permissionExplainer.ts](restored-src/src/utils/permissions/permissionExplainer.ts) | 权限说明 | 解释权限决定 |
| [utils/permissions/bypassPermissionsKillswitch.ts](restored-src/src/utils/permissions/bypassPermissionsKillswitch.ts) | 紧急关闭 |  |
| [utils/permissions/shadowedRuleDetection.ts](restored-src/src/utils/permissions/shadowedRuleDetection.ts) | 规则冲突检测 |  |
| [utils/permissions/autoModeState.ts](restored-src/src/utils/permissions/autoModeState.ts) | Auto 模式状态（feature: TRANSCRIPT_CLASSIFIER） |  |

---

## 🔷 主循环和消息处理

### 核心查询循环

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [query.ts](restored-src/src/query.ts) | **主循环**（大文件） | `query()`, `queryCheckpoint()`, 压缩触发, hooks 协调 |
| [query/config.ts](restored-src/src/query/config.ts) | 查询配置 | `buildQueryConfig()` |
| [query/deps.ts](restored-src/src/query/deps.ts) | 依赖注入 | `QueryDeps`, `productionDeps` |
| [query/transitions.ts](restored-src/src/query/transitions.ts) | 状态转移 | `Terminal`, `Continue` |
| [query/tokenBudget.ts](restored-src/src/query/tokenBudget.ts) | Token 预算检查 | `createBudgetTracker()`, `checkTokenBudget()` |

### 消息操作

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [utils/messages.ts](restored-src/src/utils/messages.ts) | 消息构建和规范化 | `createUserMessage()`, `normalizeMessagesForAPI()`, `isCompactBoundaryMessage()` |
| [utils/api.ts](restored-src/src/utils/api.ts) | API 请求构建 | `prependUserContext()`, `appendSystemContext()` |
| [utils/tokens.ts](restored-src/src/utils/tokens.ts) | Token 计算 | `tokenCountWithEstimation()`, `getTokenUsage()` |

### 工具执行

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [services/tools/StreamingToolExecutor.ts](restored-src/src/services/tools/StreamingToolExecutor.ts) | 流式工具执行 |  |
| [services/tools/toolOrchestration.ts](restored-src/src/services/tools/toolOrchestration.ts) | 工具协调 | `runTools()` |

---

## 🔷 工具使用总结 (Tool Use Summary)

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [services/toolUseSummary/toolUseSummaryGenerator.ts](restored-src/src/services/toolUseSummary/toolUseSummaryGenerator.ts) | 生成工具使用摘要 | `generateToolUseSummary()` |

---

## 🔷 其他关键系统

### 分析和遥测

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [services/analytics/index.ts](restored-src/src/services/analytics/index.ts) | 分析主入口 | `logEvent()` |
| [services/analytics/growthbook.ts](restored-src/src/services/analytics/growthbook.js) | 特性控制和动态配置 | `getFeatureValue_CACHED_MAY_BE_STALE()`, `checkSecurityRestrictionGate()` |

### 状态管理

| 文件 | 功能 | 关键函数 |
|-----|------|--------|
| [state/AppStateStore.ts](restored-src/src/state/AppStateStore.ts) | 全局应用状态 |  |
| [bootstrap/state.ts](restored-src/src/bootstrap/state.ts) | 启动和会话状态 | `getSessionId()`, `getProjectRoot()` |

---

## 📊 文件重要性评分

| 重要性 | 文件数量 | 代表文件 |
|-------|--------|--------|
| ⭐⭐⭐ (铁血) | ~15 | query.ts, memdir/findRelevantMemories.ts, utils/forkedAgent.ts, services/compact/compact.ts, utils/permissions/permissions.ts |
| ⭐⭐ (明星) | ~30 | memdir/*, SessionMemory/*, extractMemories/*, autoDream/*, compact/*, permissions/* |
| ⭐ (支持) | ~20 | context.ts, utils/messages.ts, services/tools/*, state/* |

---

## 🔍 快速导航

### 如果你想理解...

**长期记忆如何工作**：
1. 从 `memdir/memdir.ts` 开始理解索引机制
2. 看 `memdir/findRelevantMemories.ts` 了解 Sonnet 选择
3. 追踪 `context.ts` 的 `getUserContext()` 看注入

**短期记忆和压缩的交互**：
1. `services/SessionMemory/sessionMemory.ts` — 何时提取
2. `services/compact/sessionMemoryCompact.ts` — 压缩时的处理
3. `services/compact/compact.ts` — 全局压缩逻辑

**后台 Agent 的完整流程**：
1. `query/stopHooks.ts` — 何时触发
2. `utils/forkedAgent.ts` — 如何 Fork 并缓存复用
3. 各 Agent 的 prompts.ts — 做什么任务

**权限决策的多层防线**：
1. `utils/permissions/permissions.ts` — 检查入口
2. `utils/permissions/bashClassifier.ts` — 危险等级
3. `utils/permissions/permissionSetup.ts` — 规则加载

**Token 成本和压缩何时触发**：
1. `services/compact/autoCompact.ts` — 阈值计算
2. `query.ts` — 压缩触发点
3. `utils/tokens.ts` — Token 计数

