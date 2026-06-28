
## Qwen Code Agent 实习 JD

> 来源:栗栗子老师 2026-05-25,投递 `busi.wp@alibaba-inc.com`

### Track 1 — Black-box Agent RL / Online Learning
- Black-box reward / value function 建模与优化
- Online RL 框架设计,支持 agent 在真实环境持续探索与策略迭代
- Long-horizon RL 下的 credit assignment 与 multi-step reasoning

### Track 2 — Computer-Use Agent (Claw/CLI Agent) 建设
- 海量真实用户交互数据 scale(办公、流程自动化、coding),构建部署-反馈-回流-训练完整链路
- 构建 code/cli centric 的 agent model,通过 bash / skills / mcp 执行各类数字任务
- AI agents 的自进化,探索 vibe research 等场景,覆盖 PaperBench、MLE-Bench 等

### Track 3 — CLI Anything
- 把任意网站 / 桌面应用 / 本地工具统一转化为标准化 CLI,为 agent 构建开放式工具环境
- 基于 Web 与 Desktop App 构成的 live environments 训练 open-ended agents

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

### tool 接力 / 隐式 pipeline

> 起因:别的 Claude session 拿到 arxiv PDF 后去装 poppler 跑 pdftotext，没走 Read → document block → API 服务端解析的正路。

- 一条"读 PDF"链路在 CC 里是怎么走的：WebFetch 落盘 → tool result 末尾埋 `persistedPath` 那行字 → 主模型读到提示 → 调 Read → `readPDF` base64 → document block 通过 `newMessages` 通道注入 user message → API 服务端原生解析。**整条链没有 hardcoded pipeline，全靠 prompt 暗示接力。**
- WebFetch 末尾那行 `[Binary content (...) also saved to ...]` 是个"next-step hint"。这种"一个 tool 的 output text 里埋 hint 让模型选下一个 tool"的模式，CC 里还有哪些？句式有没有规律？
- 失败模式：persistedPath 提示在场，模型仍然偏向去装本机工具（poppler / pdftotext）。是 hint 不够显眼、还是模型先验太强？什么样的措辞 / system-reminder 能 nudge 回正确路径？
- `document` block 必须通过 `newMessages` 注入新 user message，不能放 `tool_result` —— 这是 API 的硬限制。还有哪些 content type 受这个限制？harness 里所有走 newMessages 的注入点能不能列全？
- `isBinaryContentType` + `persistBinaryContent`（src/utils/mcpOutputStorage.ts）这套基础设施给 WebFetch 用了一次。MCP tool 的 binary output（`ReadMcpResourceTool`）走的是不是同一套？路径命名、扩展名映射有没有共享？
- 跨工具链路的"PDF 不可逆毒丸"：一旦把伪装 PDF 塞进 document block，后续 API 调用全 400。`pdf.ts:72-86` 用 magic byte 在入口卡死。还有哪些 content 一旦进 history 就让 session 不可恢复？harness 在哪些位置加了类似的入口校验？

### attribution header / 3P cache 真相

> 起因:知乎那篇骂 Anthropic"故意让 3P 变慢"的帖子,以及网友补的一长串 env var 配置。回头亲自把每个 var 在源码里 grep 一遍,看哪些真存在、哪些是编的。

源码核对结果(2026-06-03 时点 v2.1.88):

| 文章 / 网友给的 var | 源码 | 真实作用 |
|---|---|---|
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | `src/constants/system.ts:52-95` | 真。默认开,关掉去掉 `x-anthropic-billing-header: cc_version=...; cc_entrypoint=...;` 这块 prefix |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `src/utils/privacyLevel.ts:21` | 真。`essential-traffic` 级别,屏蔽 telemetry + auto-update + MCP registry + grove + release notes |
| `DISABLE_TELEMETRY` | `src/utils/privacyLevel.ts:24` | 真。`no-telemetry` 级别,只屏 Datadog/1P 事件/feedback survey |
| `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` | `src/utils/gitSettings.ts:14` | 真。砍 git 相关 sysprompt,纯省 token |
| `DISABLE_AUTOUPDATER` | `src/utils/config.ts:1739`, `src/migrations/migrateAutoUpdatesToSettings.ts:34` | 真 |
| `DISABLE_ERROR_REPORTING` | `src/utils/log.ts:173` | 真。关 Sentry |
| `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY` | `src/components/FeedbackSurvey/*.tsx` 三处 | 真 |
| `ENABLE_TOOL_SEARCH` | `src/utils/toolSearch.ts:84,185,300` | 真,**3P 用户最有价值的一条**。`toolSearch.ts:282-311` 注释明确写:3P 默认关掉 tool_reference 因为很多代理不支持,但 LiteLLM/CF AI Gateway 是支持的 —— 显式设 `true` / `auto` / `auto:N` 表示"我代理 OK",MCP tool 描述能 lazy load |
| `ENABLE_LSP_TOOL` | `src/tools.ts:224` | 真 |
| `ANTHROPIC_BETAS`(复数) | `src/utils/betas.ts:361` | 真。**注意名字** —— 网友写的 `ANTHROPIC_BETA_HEADER` 是错的,正确名字是 `ANTHROPIC_BETAS`,逗号分隔多 beta |
| `CLAUDE_CODE_ENABLE_FEEDBACK_SURVEY_FOR_OTEL` | 不存在 | 编的 |
| `DISABLE_NON_ESSENTIAL_MODEL_CALLS` | 不存在 | 编的 |
| `CLAUDE_CODE_MAX_PROMPT_CACHE_TTL` | 不存在 | 编的 |
| `ENABLE_PROMPT_CACHING_1H` | 不存在 | 只有 `ENABLE_PROMPT_CACHING_1H_BEDROCK`(`src/services/api/claude.ts:398`),**只对 Bedrock provider 生效**,deepseek / openai-compatible 代理设了无效 |
| `DO_NOT_TRACK` / `OTEL_SDK_DISABLED` / `SENTRY_DSN` / `ANTHROPIC_TELEMETRY` | src 里 0 references | CC 自己不读;OTEL_* 是 OpenTelemetry SDK 通用 var,CC 用到 otel 时**间接**生效不是 CC 主动检查 |

attribution header 内部机制(回头细读这几个文件):

- 生成: `src/constants/system.ts:73-95` `getAttributionHeader(fingerprint)`。header 字符串组成 = `cc_version=${MACRO.VERSION}.${fingerprint};` + `cc_entrypoint=${env.CLAUDE_CODE_ENTRYPOINT};` + 可选 `cch=00000;` + 可选 `cc_workload=cron;`
- fingerprint 算法: `src/utils/fingerprint.ts:50-63` `SHA256(SALT + msgText[4] + msgText[7] + msgText[20] + version).slice(0,3)`。**SALT 是硬编码 `'59cf53e54c78'`**。注释说"Do not change without 1P/3P (Bedrock/Vertex/Azure) coordination" —— 说明这是有意保持跨 provider 一致的协议
- 插入位置: `src/services/api/claude.ts:1358-1369`,放在 `systemPrompt[0]`,在 `getCLISyspromptPrefix()` 之前
- cache 处理: `src/utils/api.ts:296-433` `splitSysPromptPrefix`。attribution block 在三种模式下都被标 `cacheScope: null`,然后 `buildSystemPromptBlocks`(`src/services/api/claude.ts:3213-3236`)的逻辑是 `cacheScope === null` 就 **不输出 `cache_control` 字段**
- cch 占位符: `system.ts:64-82` 注释。`feature('NATIVE_CLIENT_ATTESTATION')` 开启时插 `cch=00000`,Bun 的原生 HTTP 栈在发包前用计算出的 attestation token 同长度替换这 5 个 0(避免改 Content-Length / buffer realloc)。这是 per-request 变的唯一字段
- workload tag: `src/utils/workloadContext.ts` 用 AsyncLocalStorage 传 `cron` 标记,只有 ScheduleCronTool 触发的 query 才有,正常交互 session 没有

回头要搞懂的具体问题:

- session 内 fingerprint 真的稳定吗?compact / autocompact 重写 first user message 后,fingerprint 会变 → attribution header 变 → cache miss?这条还没验证
- `cch=00000` 这条 attestation 路径,如果走 3P 代理,Bun 客户端到底替换了没?替换之后 3P 怎么处理(把 5 字符占位符当普通字节,所以每次请求 header 不同 → cache miss)?要 mitmproxy 抓一次看
- `buildSystemPromptBlocks` 三种 cache scope 模式(MCP 在场 / global cache + boundary / default)的设计取舍 —— 见 `src/utils/api.ts:296-433`。这块是工业级 prompt caching 的核心,值得当模板拆掉读
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 在 sysprompt 里的位置由谁决定?静态/动态边界放错会怎样?
- `should1hCacheTTL`(`src/services/api/claude.ts:393-414`):为什么只有 Bedrock 提供 env 开关,1P 的 1h cache 必须走 `isClaudeAISubscriber() && !isUsingOverage` + GrowthBook?这是合约/计费问题
- `PROMPT_CACHING_SCOPE_BETA_HEADER`(`prompt-caching-scope-2026-01-05`,`src/constants/betas.ts:18`)的语义到底是什么 —— scope=org / global 在 Anthropic 服务端怎么实现的?查 Anthropic 官方 prompt caching doc 对照

### stream 死连接 / circuit breaker 缺位

> 起因:遇到过 1-2 小时纯空挂 spinner 还在转的 session，定位下来不是模型在 think，是 SSE 流被中间网络 silent drop，客户端 5 层超时全部失效或被关。

5 层防线现状（src/services/api/claude.ts、client.ts）：

| 层 | 默认行为 | 关键源码 |
|---|---|---|
| SDK timeout (10 min) | 只覆盖 initial fetch()，流开始后失效 | `client.ts:144` + `claude.ts:1869-1873` 注释明确写了 |
| 底层 fetch (undici) body-read | 无 timeout | `client.ts:138-152` 无 keepAlive / socket config |
| Stream watchdog (90s) | ⭐ **默认 off**，靠 `CLAUDE_ENABLE_STREAM_WATCHDOG` env 启用 | `claude.ts:1874-1928` |
| Passive stall logger (30s) | 只 log 不 abort + 事件驱动（死流根本不触发） | `claude.ts:1936-1965` |
| useStalledAnimation (3s 变红) | 只控 UI 颜色，不 abort 请求 + thinking 期间会误红 | `useStalledAnimation.ts:42` |

- 唯一能真 abort 的 watchdog 默认 off —— 设计权衡是什么？误伤长 thinking vs hours-long hang，telemetry 有没有展示真实分布？把它默认 on 会破什么？
- passive stall logger 是 `for await (part of stream)` 内部检测：流彻底死了 `for-await` 永远阻塞，这个 if 永远不跑。为什么不用独立 setTimeout 心跳？watchdog 用的就是 setTimeout，为什么不复用？
- Anthropic SDK 的 `timeout` 字段对 streaming body 完全无效 —— 是 SDK design 还是 fetch / SSE spec 的限制？看 SDK 源码确认。
- watchdog abort 后会走 non-streaming fallback retry (`claude.ts:2305-2334`)。重试时上下文是哪个 snapshot？prompt cache 还能命中吗（毕竟流断的那次 cache write 完成了一半）？
- 跟 automode/长程自主开发的交集（已经在那一节漏了这条）：24h autonomous run 撞上一次 silent stream drop，用户不在场，整个 run 停在那里。harness 有没有"长跑模式自动开 watchdog / 缩短阈值"的逻辑？没有的话，proactive / KAIROS 模式下应不应该强制开？
- OS 层的 TCP keepalive 默认 `tcp_keepalive_time=7200` 才触发 —— 在应用层全失效的情况下，2 小时上限就是这个常数决定的。这是个不应该依赖的兜底。

### session 状态拓扑 / rewind / branch / fork / resume

> 起因:想改上一个 prompt、保留之前 context 再问一次。CC 里这条路径牵出来两个完全不同的机制 —— `/rewind`（销毁性 truncate）和 `/branch`（非销毁复制），背后是一整套 session/transcript/file checkpoint 的耦合。

- `/rewind`（`REPL.tsx:3661-3706` `rewindConversationTo`）是 `setMessages(prev.slice(0, idx))` 销毁性砍数组；`/branch`（`commands/branch/branch.ts:61-180` `createFork`）是把整个 transcript 复制到新 sessionId、写 `forkedFrom` 血缘指针。还有哪些操作会写 message array？compact / autocompact / contextCollapse 各自的写法跟这两种比是销毁还是非销毁？
- truncate message array 时必须级联清的派生缓存：`resetMicrocompactState`、`resetContextCollapse`、`setConversationId(randomUUID())`（`REPL.tsx:3673-3687`）。除此之外还有哪些 ref / cache 持有"message 顺序敏感"的状态？（pinned cache edit 引用 tool_use_id 这条注释提示了一类）
- `selectableUserMessagesFilter`（`MessageSelector.tsx:767-792`）过滤掉了 tool_result / synthetic / isMeta / compact summary / 命令 stdout / TICK / teammate message —— **数据层就把"真人输入"和"机器注入"分开了**。这种"is real user content" 的区分在 harness 其他位置还出现在哪？是统一的 `isMeta` 标志，还是各个消费方各写一套过滤？
- `/branch` 在每条 entry 上写 `forkedFrom: {sessionId, messageUuid}`（`branch.ts:128-133`）。有什么消费方在读这个？能不能基于它画一棵分支树？跨多次 fork 的链路能不能追溯到根？
- `claude -r <sessionId>` resume 跟 branch/fork 的关系：transcript 文件是按 sessionId 命名的（`getTranscriptPathForSession`），resume 时怎么找、isSidechain / sub-agent 的 transcript 怎么分离？sessionStorage.ts:3084 注释说"deduplicate branches in same session" —— branches 的判定逻辑是什么？
- compact / autocompact 跟 rewind 的耦合：rewind 到 `isCompactSummary` 边界**之前**会怎样？compact summary 是 selectable filter 排掉的，但底层 message 还在 —— rewind 选不到它，但 slice 索引能落在它之前吗？
- `file_checkpointing` 跟 rewind 的耦合（`fileHistory.ts:347` `fileHistoryRewind` + `supportedSettings.ts:72`）：文件回滚是 best-effort 还是事务？回滚一半失败时 message array 已经砍了文件却没回退，session 怎么收拾？`tengu_file_history_rewind_restore_file_failed` telemetry 有没有真实数据？
- 真正的"并行多分支同时跑" CC 不支持 —— `/branch` 是切过去，要并跑得开多个 terminal `claude -r`。这个限制是 UI 层还是底层 session model 限制？sessionStorage 能不能支撑同时 attach 多个 REPL？

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

