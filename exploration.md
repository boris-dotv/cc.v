# 探索清单

> 玩明白了再写博客。只记触发好奇的点，不写废话。

## 2026-05-19

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

## 之后再加
