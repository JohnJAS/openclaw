# OpenClaw Agent-Loop 实现分析

**分析日期:** 2026-03-24  
**代码版本:** OpenClaw (D:\workspace\openclaw)

---

## 一、架构概览

OpenClaw 的 agent-loop 是一个多层次、支持故障转移和上下文管理的智能代理执行系统。整体架构分为以下层次:

```
┌─────────────────────────────────────────────────────────────┐
│                    Gateway Server Layer                      │
│  (src/gateway/server.impl.ts, server-methods/agent.ts)      │
│  - HTTP/WebSocket 请求处理                                    │
│  - 会话管理和路由                                             │
│  - 身份验证和权限控制                                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Agent Command Layer                        │
│  (src/agents/agent-command.ts)                              │
│  - 代理命令入口                                               │
│  - 模型选择和故障转移                                         │
│  - 会话状态管理                                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Embedded PI Agent Loop                       │
│  (src/agents/pi-embedded-runner/run.ts)                     │
│  - 核心 agent 循环                                             │
│  - Auth Profile 管理和故障转移                                │
│  - 上下文溢出处理                                             │
│  - 使用量统计和报告                                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Attempt Layer                              │
│  (src/agents/pi-embedded-runner/run/attempt.ts)             │
│  - 单次 LLM 调用执行                                           │
│  - 工具调用执行循环                                           │
│  - 流式响应处理                                               │
│  - 会话历史管理                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、核心组件详解

### 2.1 Gateway Server 层 (`src/gateway/server-methods/agent.ts`)

**职责:** 处理外部请求，验证参数，分发到 agent 执行系统

**关键函数:**
```typescript
agent: async ({ params, respond, context, client }) => {
  // 1. 参数验证
  if (!validateAgentParams(p)) { ... }
  
  // 2. 权限检查 (模型覆盖需要管理员权限)
  const allowModelOverride = resolveAllowModelOverrideFromClient(client);
  
  // 3. 会话管理
  const { cfg, entry, canonicalKey } = loadSessionEntry(requestedSessionKey);
  
  // 4. 交付计划解析
  const deliveryPlan = resolveAgentDeliveryPlan({ ... });
  
  // 5. 分发执行
  dispatchAgentRunFromGateway({ ingressOpts, runId, respond, context });
}
```

**关键特性:**
- 支持幂等性 (通过 `idempotencyKey`)
- 支持会话重置 (`/new`, `/reset` 命令)
- 支持附件和图片处理
- 支持交付目标解析 (channel, accountId, to, threadId)

---

### 2.2 Agent Command 层 (`src/agents/agent-command.ts`)

**职责:** 代理命令的统一入口，处理模型选择和故障转移

**关键函数:**
```typescript
async function runAgentWithModelFallback(opts: AgentCommandOpts): Promise<...> {
  // 1. 解析配置和会话
  const cfg = loadConfig();
  const sessionEntry = resolveSessionEntry(...);
  
  // 2. 构建 ingress 参数
  const ingressOpts = buildAgentIngressOpts(opts);
  
  // 3. 执行带故障转移的 agent 运行
  return await runWithModelFallback({
    ingressOpts,
    runtime,
    deps,
  });
}
```

**关键特性:**
- 支持模型故障转移 (当配置的模型不可用时自动切换)
- 支持 Thinking Level 自动降级
- 支持 Auth Profile 轮换

---

### 2.3 Embedded PI Agent Loop (`src/agents/pi-embedded-runner/run.ts`)

**职责:** **核心 agent 循环**,管理认证、重试、上下文溢出处理

**主循环结构:**
```typescript
export async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams
): Promise<EmbeddedPiRunResult> {
  
  // === 初始化阶段 ===
  const started = Date.now();
  const usageAccumulator = createUsageAccumulator();
  let runLoopIterations = 0;
  let overflowCompactionAttempts = 0;
  
  // === Auth Profile 解析 ===
  const profileCandidates = resolveAuthProfileOrder({ ... });
  let profileIndex = 0;
  
  // === 主循环 ===
  while (true) {
    // 1. 检查重试限制
    if (runLoopIterations >= MAX_RUN_LOOP_ITERATIONS) {
      return { error: { kind: "retry_limit" } };
    }
    runLoopIterations += 1;
    
    // 2. 执行单次尝试
    const attempt = await runEmbeddedAttempt({ ... });
    
    // 3. 处理上下文溢出
    if (contextOverflowError) {
      if (overflowCompactionAttempts < MAX_OVERFLOW_COMPACTION_ATTEMPTS) {
        // 尝试压缩
        const compactResult = await contextEngine.compact({ ... });
        if (compactResult.compacted) {
          continue; // 重试
        }
      }
      // 尝试截断工具结果
      if (!toolResultTruncationAttempted) {
        const truncResult = await truncateOversizedToolResultsInSession({ ... });
        if (truncResult.truncated) {
          continue; // 重试
        }
      }
      return { error: { kind: "context_overflow" } };
    }
    
    // 4. 处理认证错误
    if (authFailure && await maybeRefreshRuntimeAuthForAuthError(...)) {
      authRetryPending = true;
      continue;
    }
    
    // 5. 处理故障转移
    if (shouldRotate) {
      const rotated = await advanceAuthProfile();
      if (rotated) {
        continue; // 切换到下一个 profile 重试
      }
      if (fallbackConfigured) {
        throw new FailoverError(...); // 触发模型故障转移
      }
    }
    
    // 6. 成功完成
    return {
      payloads,
      meta: {
        agentMeta: {
          sessionId,
          provider,
          model,
          usage: toNormalizedUsage(usageAccumulator),
        },
      },
    };
  }
}
```

**关键特性:**

| 特性 | 实现 |
|------|------|
| **Auth Profile 管理** | 支持多个 API Key 轮换，失败时自动切换到下一个 |
| **上下文溢出处理** | 最多 3 次压缩尝试，失败后尝试截断工具结果 |
| **运行时 Auth 刷新** | 支持动态刷新过期的 API 凭证 |
| **使用量统计** | 累加 input/output/cache tokens，报告最终使用量 |
| **重试限制** | 基础 24 次 + 每 profile 8 次，最多 160 次 |

---

### 2.4 Attempt 层 (`src/agents/pi-embedded-runner/run/attempt.ts`)

**职责:** 执行单次 LLM 调用，处理工具调用循环

**核心流程:**
```typescript
async function runEmbeddedAttempt(params: EmbeddedRunAttemptParams) {
  
  // === 1. 会话初始化 ===
  const sessionManager = await prepareSessionManagerForRun({ ... });
  const activeSession = await createOrLoadSession({ ... });
  
  // === 2. 系统提示构建 ===
  const systemPromptText = await buildEmbeddedSystemPrompt({ ... });
  
  // === 3. 历史消息处理 ===
  const prior = await sanitizeSessionHistory({
    messages: activeSession.messages,
    modelApi: params.model.api,
    policy: transcriptPolicy,
  });
  const limited = limitHistoryTurns(validated, historyLimit);
  activeSession.agent.replaceMessages(limited);
  
  // === 4. 工具定义 ===
  const tools = await createOpenClawCodingTools({ ... });
  const allowedToolNames = collectAllowedToolNames(tools);
  
  // === 5. 流式执行 ===
  const subscription = subscribeEmbeddedPiSession({
    session: activeSession,
    onPartialReply: params.onPartialReply,
    onToolResult: params.onToolResult,
    onAgentEvent: params.onAgentEvent,
    ...
  });
  
  // === 6. 等待完成 ===
  const result = await abortable(subscription.waitForEnd());
  
  // === 7. 结果处理 ===
  return {
    assistantTexts: result.assistantTexts,
    toolMetas: result.toolMetas,
    lastAssistant: result.lastAssistant,
    aborted: result.aborted,
    timedOut: result.timedOut,
    compactionCount: result.compactionCount,
  };
}
```

**工具调用循环:**
```
┌─────────────────────────────────────────────────────────┐
│                   Tool Call Loop                         │
│                                                          │
│  1. LLM 生成响应 (可能包含 tool_use)                       │
│       ↓                                                  │
│  2. 检测 tool_use 块                                      │
│       ↓                                                  │
│  3. 执行工具 (通过 pi-agent-core 调度)                     │
│       ↓                                                  │
│  4. 将 tool_result 添加回会话历史                          │
│       ↓                                                  │
│  5. 继续 LLM 生成 (直到没有 tool_use)                       │
│       ↓                                                  │
│  6. 返回最终响应                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 三、关键机制

### 3.1 故障转移机制 (Failover)

**触发条件:**
- 认证失败 (401/403)
- 速率限制 (429)
- 计费问题 (billing)
- 服务过载 (overloaded)
- 超时 (timeout)

**故障转移流程:**
```
1. 检测错误类型
   ↓
2. 尝试刷新 Auth (如果支持运行时刷新)
   ↓
3. 切换到下一个 Auth Profile
   ↓
4. 如果所有 Profile 都失败 → 触发模型故障转移
   ↓
5. 切换到备用模型 (如果配置了 fallbacks)
```

**代码位置:** `src/agents/pi-embedded-runner/run.ts:advanceAuthProfile()`

---

### 3.2 上下文溢出处理 (Context Overflow)

**处理策略:**
1. **自动压缩** (Auto-compaction): 使用 context engine 压缩会话历史
2. **工具结果截断**: 截断过大的工具结果
3. **错误报告**: 如果都无法解决，返回用户友好的错误消息

**最大尝试次数:** 3 次压缩尝试

**代码位置:** `src/agents/pi-embedded-runner/run.ts:contextOverflowError 处理块`

---

### 3.3 会话状态管理

**会话存储结构:**
```typescript
type SessionEntry = {
  sessionId: string;
  channel?: string;
  lastChannel?: string;
  lastTo?: string;
  lastAccountId?: string;
  thinkingLevel?: ThinkLevel;
  verboseLevel?: VerboseLevel;
  modelOverride?: string;
  providerOverride?: string;
  spawnedBy?: string;
  spawnDepth?: number;
  // ... 更多字段
};
```

**会话生命周期:**
```
创建 → 激活 → (工具调用循环) → 完成 → (可选: 删除/保留)
```

---

### 3.4 子代理 (Subagent) 系统

**子代理公告机制:**
```typescript
// 子代理完成后自动通知父代理
async function runSubagentAnnounceFlow(params: {
  childSessionKey: string;
  childRunId: string;
  requesterSessionKey: string;
  ...
}) {
  // 1. 等待子代理完成
  const settled = await waitForEmbeddedPiRunEnd(childSessionId, timeoutMs);
  
  // 2. 读取子代理输出
  const reply = await readLatestSubagentOutput(childSessionKey);
  
  // 3. 构建内部事件
  const internalEvents: AgentInternalEvent[] = [{
    type: "task_completion",
    source: "subagent",
    result: findings,
    ...
  }];
  
  // 4. 发送到父代理会话
  await deliverSubagentAnnouncement({ ... });
}
```

**代码位置:** `src/agents/subagent-announce.ts`

---

## 四、数据流图

```
用户请求 (HTTP/WS)
       │
       ▼
┌──────────────────┐
│  Gateway Server  │
│  (server.ts)     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  agent Handler   │
│  (agent.ts)      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Agent Command   │
│  (agent-command) │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Embedded PI     │
│  (run.ts)        │◄──── Auth Profile 轮换
└────────┬─────────┘  ◄──── 上下文溢出处理
         │            ◄──── 故障转移
         ▼
┌──────────────────┐
│  Attempt         │
│  (attempt.ts)    │◄──── 工具调用循环
└────────┬─────────┘  ◄──── 流式响应
         │
         ▼
┌──────────────────┐
│  pi-agent-core   │
│  (SDK)           │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  LLM Provider    │
│  (Anthropic/     │
│   OpenAI/...)    │
└──────────────────┘
```

---

## 五、关键配置项

### 5.1 Agent 默认配置
```typescript
agents: {
  defaults: {
    timeoutSeconds: number;        // 超时时间
    subagents: {
      announceTimeoutMs: number;   // 子代理公告超时
      maxSpawnDepth: number;       // 最大子代理深度
    };
    heartbeat: {
      ackMaxChars: number;         // 心跳 ACK 最大字符数
    };
  };
  fallbacks: [...] ;               // 模型故障转移列表
}
```

### 5.2 Auth Profile 配置
```typescript
auth: {
  profiles: {
    "profile-id": {
      provider: string;
      apiKey: string;              // 或 env 引用
      priority?: number;           // 优先级
    };
  };
}
```

---

## 六、错误处理

### 6.1 错误类型分类

| 错误类型 | FailoverReason | 处理策略 |
|---------|----------------|----------|
| 认证失败 | `auth` | 刷新 Auth 或切换 Profile |
| 速率限制 | `rate_limit` | 切换 Profile 或模型 |
| 计费问题 | `billing` | 切换 Profile 或模型 |
| 服务过载 | `overloaded` | 退避后切换 |
| 超时 | `timeout` | 切换 Profile (不标记失败) |
| 上下文溢出 | `context_overflow` | 压缩或截断 |

### 6.2 错误传播
```
Attempt 层 → Run 层 → Agent Command 层 → Gateway Handler → 客户端
    │           │            │              │
    ▼           ▼            ▼              ▼
FailoverError 重试/切换   模型故障转移    HTTP/WS 错误响应
```

---

## 七、性能优化

### 7.1 并发控制
- **Lane 系统**: 通过 `command-queue` 实现并发控制
- **会话级队列**: 同一会话的请求串行执行
- **全局队列**: 限制全局并发数

### 7.2 缓存
- **会话管理器缓存**: 预加载会话文件
- **模型目录缓存**: 缓存模型配置解析结果
- **Auth Profile 缓存**: 缓存 Auth Profile 状态

### 7.3 流式处理
- **增量响应**: 通过 `onPartialReply` 流式返回
- **工具事件**: 通过 WebSocket 实时推送工具执行状态

---

## 八、总结

OpenClaw 的 agent-loop 是一个高度健壮、支持故障转移的智能代理执行系统。其核心特点包括:

1. **多层次架构**: Gateway → Agent Command → Embedded Run → Attempt
2. **强大的故障转移**: Auth Profile 轮换 + 模型故障转移
3. **上下文管理**: 自动压缩 + 工具结果截断
4. **子代理系统**: 支持嵌套子代理和自动公告
5. **流式处理**: 支持增量响应和实时工具事件

整个系统设计考虑了生产环境的各种边界情况，包括 API 限流、认证过期、上下文溢出等，并通过多层次的重试和故障转移机制确保高可用性。

---

## 参考文件

- `src/gateway/server-methods/agent.ts` - Gateway agent 处理器
- `src/agents/agent-command.ts` - Agent 命令入口
- `src/agents/pi-embedded-runner/run.ts` - 核心 agent 循环
- `src/agents/pi-embedded-runner/run/attempt.ts` - 单次尝试执行
- `src/agents/subagent-announce.ts` - 子代理公告系统
- `src/agents/pi-embedded-runner/runs.ts` - 运行状态管理
