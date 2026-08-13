# Agent Note: DeepSeek adapter substitutes image markers instead of bricking sessions

Status: implemented

English | [中文](2026-08-13-deepseek-adapter-image-markers.zh.md)

## Problem

The DeepSeek chat-completions wire is text-only, and `serialize.ts` rejected any image block in the request with `UNSUPPORTED_CONTENT`, including blocks nested inside tool results. A block that entered durable session history made the adapter throw before network I/O on every later turn. The throw repeats forever because the block is durable history, so the session is permanently bricked: retrying, switching models, or reopening the conversation cannot recover it, and the log requires manual surgery to repair. The upstream gates did not make this impossible — the `read_image` route gate is bypassable by design, and a production session's dynamic plugin patched `resolveModelInfo` to admit exactly such a block.

## Decision

`packages/llm/llm-deepseek/src/serialize.ts` substitutes every image block with an explicit, deterministic text marker instead of throwing:

`[image-attachment] {"attachmentId":…,"mediaType":…,"width":…,"height":…,"bytes":…[,"name":…]}`

The substitution applies at every flattening site: top-level user, system, and assistant content, and recursively inside tool results (whose content contributes only to the role:tool wire message). The marker keeps the model informed — it carries the durable attachment id and metadata, so a vision-capable tool or subagent route can still act on the image — instead of silently erasing the block, which is the loss the previous rejection contract existed to prevent. Unknown declaration-merged block types keep the adapter's documented skip fallback. The image-specific `UNSUPPORTED_CONTENT` throw is removed from this adapter; its other error codes are unchanged.

## Alternatives considered

**Keep the throw and rely on the upstream gates.** The `read_image` execution gate and the Web upload admission both refuse images on text-only routes. Rejected: gates are bypassable by design (dynamic plugins can patch `ctx.llm.resolveModelInfo`, and one production session did exactly that), and one admitted block bricks the session permanently. Prevention alone cannot make the failure recoverable.

**Strip images at prompt assembly instead of the adapter.** Rejected: loop-level filtering would not cover compaction, session-title, or plugin one-shot `ctx.llm.stream` calls, and the adapter is the last common point where the text-only representation is decided.

**Auto-describe images through a vision-capable route.** Rejected: implicit model calls add cost, latency, and provider coupling inside serialization; the marker defers that choice to the model, which can call a vision tool when one is mounted.

**Retry without the offending message.** Rejected: rewriting or skipping history at retry time violates the log's authority and complicates replay and projections.

## Consequences

A session whose durable history contains an image block always progresses: the model sees the marker text and can route the attachment to a vision-capable tool, ask the user, or continue without it. The marker string is pinned verbatim by unit tests (`packages/llm/llm-deepseek/tests/serialize.spec.ts`) and documented in the package README; the `read-image` keyless snapshots, including the text-route variant, still pass.

The marker is derived deterministically from the logged block, so model-visible content remains reconstructable from the session log. `read_image`'s route gate and the Web upload admission remain the primary prevention (see the [read-image note](../feature/2026-08-10-minimal-read-image-tool.md)); the marker is the recovery net behind them. This note partially supersedes the DeepSeek rejection facts in the [multimodal feature note](../feature/2026-07-22-web-multimodal-image-input-and-durable-attachments.md); pi-ai routes keep their capability throws because they carry real image transport, leaving the analogous text-only brick there as deferred work.
