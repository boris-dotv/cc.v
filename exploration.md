
## 我关注的问题

### memory
- memdir 里的 .md 文件具体是怎么被注入到每次 query 的？拼在 system prompt 哪一段、什么时机？
- 跨 session / 跨工作目录怎么决定加载哪些 memory？是全量还是按相关性？
- 用户手写的 memory 和 auto-saved 的怎么区分、是否会互相覆盖？

### monitor
- 后台 Monitor 怎么把 stdout 流变成离散通知事件（line buffering / 触发条件）？
- 一个 session 能同时挂几个？资源/调度怎么处理？
- 被监控进程异常退出怎么传回主 agent？persistent 模式跟普通有什么不一样？

### prompt 编排
- 真正发出去的 messages payload 长什么样？哪段是 system、哪段是 user、tool 定义放哪？
- 每轮 system prompt 都重拼？哪些片段 stable、哪些会变？
- prompt cache breakpoint 打在什么位置、为什么打那儿？
- memory / CLAUDE.md / skill descriptions / environment / tool defs 的拼接顺序和原因？
- 想办法 dump 一次真实 payload 看看（mitmproxy / log）。

### token 追踪
- 状态栏那个 token 数字算的是哪部分？input / output / cached 分开吗？
- autocompact 触发阈值写在哪、是绝对值还是百分比？
- 每个 tool call 的 token 消耗能不能单独看到？session 结束后能不能回看？

### automode / 长程自主开发（"让它跑 24 小时"）
- CC 里至少有 5 套机制和"自主运行"相关：permission mode（`bypassPermissions` / 内部 `auto`）、Auto Mode prompt 注入、Proactive Mode + `<TICK>`、`/loop` + cron、scheduleRemoteAgents。它们各自解决什么、什么时候组合？
- Proactive Mode 的"心跳"机制是怎么实现的？harness 怎么在 message queue 空的时候注入 tick 让模型"还活着"？为什么 model 在 tick 上必须调 Sleep 而不能 echo？
- Auto Mode 的指令是怎么"持续生效"的？为什么有 full / sparse 两版？sparse 注入频率由谁决定？
- `terminalFocus` 字段是怎么采集的？为什么 unfocused 时模型可以更激进？这个信号能被 spoof 吗？
- 长跑 24 小时上下文一定爆 —— 实测下来 4 层压缩（snip + microcompact + collapse + autocompact）能撑多久不退化？退化点的失败模式是什么（summary 越来越抽象、丢关键代码片段）？
- 跑飞了怎么停？`maxBudgetUsd` / `maxTurns` / `taskBudget` 各自是什么粒度的 circuit breaker？谁先触发？
- `stuck` skill 是什么时候触发的？harness 怎么检测"模型在原地打转"？检测算法值得偷吗？
- 远程 agent（scheduleRemoteAgents → RemoteTrigger）vs 本地 proactive 模式，分别适合什么场景？为什么不全用远程的？
- 后台 subagent（`BG_SESSIONS`）和主 agent 怎么通信？`shouldAvoidPermissionPrompts` 是怎么让 background agent 不卡在权限弹窗的？
- 24 小时自主任务的"目标定义"放在哪？CLAUDE.md / appendSystemPrompt / 初始 user message —— 哪种活得最久不被 compact 掉？
- 这套 automode 跑出来的 trajectory 有哪些 telemetry signal 被采集？哪些信号能反推"这次 autonomous run 是否高质量"用作 RL reward？

## 推荐我去搞懂的问题

> 读完应该能当场回答出来。答不上来就是没读透。

### 上下文管理（最难也最值得）
- 为什么 `query.ts` 里 snip、microcompact、contextCollapse、autocompact 是 4 个独立 feature gate？它们能同时开吗？谁先触发、谁兜底？
- compact 本身是用 forked agent 跑的，它的 system prompt 和主对话一样吗？cache 怎么处理才不破？
- `taskBudget.remaining` 为什么压缩前不用传、压缩后必须手动传？背后的 server 行为是什么？

### prompt 拼装 + cache
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 之后的内容为什么不能 global-cache？举一个会变的具体例子。
- 一个 user 消息被发送到 API 时，它前面会被 prepend 几条 meta 消息？分别从哪里来？（提示：`prependUserContext` + attachments + memory prefetch）
- MCP tools 一旦存在，整个 system prompt 的 cache scope 为什么会塌到 `org`？
- `<system-reminder>` 为什么是 user message 而不是 system prompt 的一部分？

### Tool 抽象
- `backfillObservableInput` 为什么不能直接 mutate 原始 input？拿掉这个保护会发生什么？
- tool 并发执行的分批规则是什么？什么样的连续 tool calls 会被合并成并发批？
- `shouldDefer` + ToolSearch 这套机制解决的是什么问题？为什么不直接全量发 tool schema？

### subagent
- subagent 是新进程吗？如果不是，它和主 agent 共享什么、隔离什么？
- 父 agent 的 prompt cache 子 agent 能命中吗？需要满足什么条件？（关键词：`CacheSafeParams`）
- sidechain transcript 和主 transcript 的关系？resume 的时候怎么衔接？

### compact prompt 设计
- BASE_COMPACT_PROMPT 为什么要 `<analysis>` + `<summary>` 双 block？只让模型直接出 summary 行不行？
- `formatCompactSummary` 砍掉 analysis 这一步背后的 prompt engineering 原理是什么？这个模式能不能偷来用在自己的 agent 里？

## 探索足迹

> 最新的日期写在最上方。每天一个 `### YYYYMMDD` 小节，记当天读了什么、想通了什么、卡在哪。

### 20260524

#### 主题

`src/query.ts` 里的 `queryLoop` —— CC harness 的主循环。整个 agent 在干的事情，本质就是这个 generator 里 `while (true)` 在驱动。

#### 蓝图

```
ENTER queryLoop
  ├─ destructure 8 个不变参数 (systemPrompt, canUseTool, ...)
  ├─ 初始化 9 字段 state (messages, toolUseContext, turnCount=1, ...)
  ├─ buildQueryConfig() 一次性快照 env/statsig/session
  └─ using pendingMemoryPrefetch (退出时自动 dispose)

  while (true):
    ① destructure state (toolUseContext: let, 其余 const)
    ② yield 'stream_request_start'
    ③ messages → messagesForQuery 改造主干
         getMessagesAfterCompactBoundary
           → applyToolResultBudget
             → microcompact
               → autocompact
                 → blocking check (硬阻断 → return 'blocking_limit')
    ④ for await (msg of deps.callModel({...})) {   ← 流循环体，3 个职责
          (a) tombstone (fallback 触发时清空已 yield)
          (b) clone & yield (tool_use input backfill, 不动原始)
          (c) push & 收集 tool_use → needsFollowUp
       }
    ⑤ if (!needsFollowUp):
         • 处理 withheld 错误（prompt_too_long / max_output_tokens 恢复）
         • 跑 stop hooks
         • return 'completed'   ◄── happy path 终点

       else (needsFollowUp === true):
         落到 ⑥

    ⑥ runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
         → for await update: yield msg, push toolResults
    ⑦ 拼 attachments / memory prefetch consume / skill prefetch consume
       / refresh tools / maxTurns 检查
    ⑧ state = {
         messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
         turnCount: nextTurnCount, ...
       }
    ⑨ continue ────────────────────────────────────┐
                                                    ↓
                                             (回到 while 顶)

EXIT queryLoop
  return Terminal:
    'completed' | 'max_turns' | 'blocking_limit' | 'prompt_too_long' |
    'image_error' | 'model_error' | 'aborted_streaming' | 'aborted_tools' |
    'stop_hook_prevented' | 'hook_stopped'
```

骨架最重要的一行（站点 ⑧）：
`messages: [...messagesForQuery, ...assistantMessages, ...toolResults]`
下一轮的 messages = 这一轮喂 API 的输入 + assistant 回复 + 工具结果。**这就是 ping-pong 累积**。

#### 前置知识

##### 1. 外壳与调用链

- `query()`（line 219-239）是壳子，只负责包一层、管 queue lifecycle。`queryLoop()` 才是节拍器。
- `query()` 对外只 yield 两类东西：流事件（`StreamEvent` / `RequestStartEvent`）和消息（`Message` 系列）。最后 `return` 一个 `Terminal` 告诉外面"为什么结束"。
- `QueryParams` 由 `QueryEngine.submitMessage`（src/QueryEngine.ts:209）构造并传入。REPL、`claude -p`、SDK、`runForkedAgent` 最终都汇聚到 QueryEngine。
- "消费者" = 调用 `for await (const event of query(params))` 的那段代码，也就是 QueryEngine。它把 yield 的事件分发给 UI / SDK / transcript。**消费者不是人，是上一层代码**。

##### 2. `consumedCommandUuids` 与队列消息

用户在 CC 跑长任务时排队的消息（比如 "顺便把 README 也更新一下"）会进 `messageQueueManager`，每条带一个 uuid。queryLoop 跑到 line 1604 `getCommandsByMaxPriority` 时把它们拉出来变 attachment，uuid push 进 `consumedCommandUuids`，同时 `notifyCommandLifecycle(uuid, 'started')`。等整个 queryLoop **正常 return** 后，外层 `query()` 才统一发 `'completed'`。出错或被打断 → uuid 只有 'started' 没有 'completed'，UI 据此画"被中断的排队消息"。

##### 3. 8 个 immutable params（整个 query 生命周期不变）

| 字段 | 具体例子 |
|---|---|
| `systemPrompt` | 完整拼好的 system prompt（"You are Claude Code..." 加几千 token 工具说明） |
| `userContext` | 每条 user message **前置**的 meta，如 `<system-reminder>...</system-reminder>` |
| `systemContext` | 每条 system 消息**附加**的内容，如 git status、CLAUDE.md 摘录 |
| `canUseTool` | 决定 tool 是否被允许的函数（bypass 直接 allow / 否则弹窗） |
| `fallbackModel` | 主 `claude-opus-4-7`，fallback `claude-sonnet-4-6` |
| `querySource` | `'repl_main_thread'` / `'agent:Explore'` / `'compact'` / `'sdk'` |
| `maxTurns` | SDK 硬上限（如 50），超了 return `'max_turns'` |
| `skipCacheWrite` | 不污染父 agent 的 prompt cache |

为什么 immutable：这些由用户输入或 session config 决定，改了 systemPrompt 会破 cache、改了 canUseTool 会让权限语义跳变。

##### 4. 9 字段 mutable state（每轮可重写）

每个 `continue` 站点整体重写 `state = { ... }`，不是零散改字段。迭代顶部 destructure 时 `let { toolUseContext }`、其余全 `const` —— 明确信号"轮内只有 toolUseContext 会被原地改写"。

| 字段 | 演化 |
|---|---|
| `messages` | turn 1: `[user1]` → turn 2: `[user1, assistant1, tool_result1]` → ... 每轮递增 |
| `toolUseContext` | queryTracking / readFileState / abortController / options.tools — 轮内可能被 `{...spread}` 重写好几次 |
| `turnCount` | 1 → 2 → 3 ... 每完成一次 ping-pong 加 1 |
| `transition` | `{reason: 'next_turn' / 'max_output_tokens_recovery' / 'stop_hook_blocking' / ...}` — telemetry/debug |
| `autoCompactTracking` | undefined → `{compacted: true, turnId, turnCounter: 0}`，记录"距离上次 compact 多少轮" |
| `maxOutputTokensOverride` | undefined → `ESCALATED_MAX_TOKENS`（8k 默认上限被截 → 升 64k 重发） |
| `maxOutputTokensRecoveryCount` | 0 → 1 → 2，超 limit 放弃 |
| `hasAttemptedReactiveCompact` | false → true，turn 内一次性闸门 |
| `stopHookActive` | undefined → true，避免 stop hook 无限重试 |
| `pendingToolUseSummary` | haiku 异步生成的 tool batch 摘要 Promise，下一轮 await 并 yield |

##### 5. `using` + TC39 dispose

`using` 是 JS 新语法（TS 5.2+，Node 22+），声明带 `Symbol.dispose` 的变量，**离开 scope 自动 dispose**。在 generator 里覆盖 3 种退出方式：正常 return / 抛异常 / **消费者调 `.return()`**。第 3 种用 try/finally 很难写漂亮，using 把"退出时做什么"绑死在声明上。

具体到这里：`pendingMemoryPrefetch` 是个背景 Promise（去 sideQuery 问哪些 memory 跟当前对话相关），dispose 时记 telemetry 并 abort 没用完的请求。

##### 6. messages 改造主干（callModel 之前）

只看主干、忽略所有 `feature()` gate 后的分支：

```
messages
  → getMessagesAfterCompactBoundary    截掉 compact 边界前的旧消息
  → applyToolResultBudget              大 tool result → "[content removed to fit budget]" placeholder
  → microcompact                       微压缩
  → autocompact                        大压缩（必要时整段重写）
  → calculateTokenWarningState         硬阻断 → return 'blocking_limit'
messagesForQuery
```

`applyToolResultBudget` 是典型上下文管理策略：尾部新结果保留细节，头部老结果替换成 placeholder，替换记录写到 `contentReplacementState`。

**`blocking_limit` 是终点，不是恢复入口** —— 它的前提是上游所有自动恢复路径（snip / microcompact / autocompact / reactiveCompact / collapse）都被跳过或失败了。出现这个 return 后，用户得手动跑 `/compact`。

##### 7. `deps.callModel` 是 harness 与 API 唯一接触点

`callModel` 由 `productionDeps()` 注入。整个 harness 跟 Anthropic API 的所有交互都通过它。**做新 harness 时能不能干净替换 deps，就是抽象做得好不好的指标**。

外层 `while (attemptWithFallback)` 是为 fallback 模型准备的（主模型超载 → fallback 重发一次）。一次正常请求里只跑一遍。

##### 8. 流循环体的三个职责（最值得细读的 ~50 行）

代码里已经用 `⭐ START` / `⭐ END` 注释包住了。每条 stream message 进来做三件事：

1. **是否要 tombstone？** — fallback 触发过的话，之前累积的 assistant message 全作废，清空 `assistantMessages` / `toolResults` / `toolUseBlocks`，并给已经 yield 出去的 tool_use 补 tombstone 让 UI 划掉。

2. **是否要 clone & yield？** — `tool_use` 块要 backfill 派生字段（比如 file path 展开），**但只 clone 给"对外 yield 的那份"，原始 message 不动**。原因：原始 message 要回灌 API，mutate 会破 byte-level prompt cache。

3. **是否要 push & 记录 tool_use？** — `message.type === 'assistant'` 时 push 到 `assistantMessages`，扫 content 收集 `tool_use` 块，**任何 tool_use 出现就置 `needsFollowUp = true`**。这是"要不要再来一轮"的唯一信号。

##### 9. fallback = 主模型超载切备份

API 返回 529 overloaded → SDK 抛 `FallbackTriggeredError` → catch 接住做 5 件事：① 切 `currentModel`；② 清空 `assistantMessages` / `toolResults` / `toolUseBlocks`；③ 给已 yield 的 tool_use 补 tombstone（`yieldMissingToolResultBlocks`）；④ discard 乐观启动的 `streamingToolExecutor`、新建一个干净的；⑤ **剥 thinking 签名**（`stripSignatureBlocks`，opus 的签名 sonnet 不认，不剥会 API 400）。然后外层 while 转一圈重发。

##### 10. 流结束后的决策树

```
if (aborted) → return 'aborted_streaming'

if (!needsFollowUp):           ← 模型没要工具
   • 处理 withheld 错误
       prompt_too_long → 试 collapse drain / reactive compact → 成功 continue / 失败 return
       max_output_tokens → 升 cap / 注入恢复消息 → continue / 用完恢复次数 露出错误
   • 跑 stop hooks
       prevent → return 'stop_hook_prevented'
       inject error → continue
       pass → 落下来
   • return 'completed'   ◄── happy path 终点

else:                          ← needsFollowUp === true
   落到 runTools
```

恢复路径的共同模式：重新拼 `next: State`，`state = next; continue`。

##### 11. runTools + 末尾 state 重组

```ts
const toolUpdates = runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message              // 透传给消费者
    toolResults.push(...normalizeMessagesForAPI([update.message], ...))
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

工具跑完后还要 push 一连串 attachment（queued commands、memory prefetch、skill prefetch、file changes），refresh tools，检查 maxTurns，最后：

```ts
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  turnCount: nextTurnCount,
  ...
}
continue
```

