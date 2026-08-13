# Agent Note: DeepSeek 适配器用图片标记替代卡死会话

Status: implemented

[English](2026-08-13-deepseek-adapter-image-markers.md) | 中文

## Problem

DeepSeek chat-completions 协议是纯文本的，`serialize.ts` 会对请求中的任何图片块抛出 `UNSUPPORTED_CONTENT`，包括嵌套在工具结果中的图片块。一旦某个块进入持久化会话历史，适配器就会在之后每一轮的网络 I/O 之前抛错。由于该块属于持久化历史，抛错会永远重复，会话被永久卡死：重试、切换模型或重新打开会话都无法恢复，日志必须经过手工修复才能挽救。上游门禁并不能杜绝这一点——`read_image` 的路由门禁在设计上就可被绕过，而一个生产会话的动态插件正是通过修补 `resolveModelInfo` 放行了这样一个块。

## Decision

`packages/llm/llm-deepseek/src/serialize.ts` 不再抛错，而是把每个图片块替换为显式、确定性的文本标记：

`[image-attachment] {"attachmentId":…,"mediaType":…,"width":…,"height":…,"bytes":…[,"name":…]}`

替换作用于每个展平位置：顶层的 user、system、assistant 内容，以及工具结果内部的递归位置（其内容只进入 role:tool 协议消息）。标记让模型保持知情——它携带持久化附件 id 与元数据，因此具备视觉能力的工具或子代理路由仍然可以处理该图片——而不是悄悄抹掉该块；悄悄抹掉正是此前的拒绝约定所要防止的损失。未知的声明合并块类型保留适配器已文档化的跳过回退。该适配器中针对图片的 `UNSUPPORTED_CONTENT` 抛错被移除；其他错误码不变。

## Alternatives considered

**保留抛错并依赖上游门禁。** `read_image` 执行门禁与 Web 上传准入都会在纯文本路由上拒绝图片。被否：门禁在设计上可被绕过（动态插件可以修补 `ctx.llm.resolveModelInfo`，一个生产会话正是这么做的），而一个被放行的块就会永久卡死会话。仅有预防无法让失败可恢复。

**在提示词组装阶段而不是适配器里剥离图片。** 被否：loop 层过滤无法覆盖压缩、会话标题或插件的单次 `ctx.llm.stream` 调用，而适配器是决定纯文本表示的最后一个公共点。

**经由具备视觉能力的路由自动描述图片。** 被否：在序列化内部做隐式模型调用会带来成本、延迟与提供方耦合；标记把这个选择推迟给模型，模型可以在挂载了视觉工具时调用它。

**重试时跳过那条出问题的消息。** 被否：在重试时改写或跳过历史违反日志权威，并使回放与投影复杂化。

## Consequences

持久化历史中包含图片块的会话总能继续推进：模型看到标记文本，可以把附件路由给具备视觉能力的工具、询问用户，或在不看图的情况下继续。标记字符串由单元测试逐字固定（`packages/llm/llm-deepseek/tests/serialize.spec.ts`），并记录在包 README 中；`read-image` 的无密钥快照（包括纯文本路由变体）仍然通过。

标记由已记录块确定性推导，因此模型可见内容仍可从会话日志重建。`read_image` 的路由门禁与 Web 上传准入仍是主要预防手段（见 [read-image note](../feature/2026-08-10-minimal-read-image-tool.md)）；标记是它们背后的恢复兜底。本 note 部分取代[多模态特性 note](../feature/2026-07-22-web-multimodal-image-input-and-durable-attachments.md) 中关于 DeepSeek 拒绝的事实；pi-ai 路由保留其能力抛错，因为它们承载真正的图片传输，其纯文本路径上类似的卡死风险留作暂缓工作。
