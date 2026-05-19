# cc.v

私人研究 fork,用于阅读 Claude Code v2.1.88 的内部实现。

## 来源

`@anthropic-ai/claude-code` 2.1.88 在 npm 上发布时附带了可还原源码的 source map (`cli.js.map`),其中 `sources` 与 `sourcesContent` 字段保留了原始 TypeScript 文件结构。本仓库 `src/` 即由那份 map 还原而来。

`vendor/` 是几个原生模块的 src 包(audio capture / image processor / modifiers napi / url handler)。

## 目录

```
src/             还原后的 TypeScript 源码(~1300 个 .ts 文件)
vendor/          原生模块 src
exploration.md   个人探索清单
```

## 用途

仅供个人研究、阅读、做笔记。不是可运行项目,也不打算二次发布。

## 安装(只是读源码不需要)

如果想跑起来,2.1.88 已从官方 npm 下架,可用腾讯镜像:

```
npm install -g https://mirrors.cloud.tencent.com/npm/@anthropic-ai/claude-code/-/claude-code-2.1.88.tgz
```

## 免责声明

非 Anthropic 官方仓库,与 Anthropic 无任何关联。原始代码版权、商标及相关权利归 Anthropic 所有。本仓库仅用于个人学习,不构成二次发布。如需衍生使用请自行评估法律风险。
