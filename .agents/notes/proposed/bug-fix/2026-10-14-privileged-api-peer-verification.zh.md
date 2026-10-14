# Agent Note: 为特权 API 方法验证传输对端

Status: proposed

[English](2026-10-14-privileged-api-peer-verification.md) | 中文

## Problem

静态审查基线：默认分支 master 的 `47f943859bef60e4160492346772ded9b24f765a`，2026-10-14 UTC 重新核对；完整递归树 `f904efab9ef435201d6ba4da88a34d6366568272` 共 8,617 项，未截断。本提案只修改文档，没有实施代码、部署、读取凭据、动态利用、构建或执行仓库测试。

[请求判定函数](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/client/connection/src/api-request-trust.ts#L96-L123) 检查 Host、Origin 和 Sec-Fetch-Site，没有连接对端信息。loopback Host 且无 Origin 的请求会通过。[特权路由策略](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/client/connection/src/index.ts#L69-L170) 承诺配置、凭据及其他宿主操作只能从本机访问，却仅给同一判定函数传入空 trusted-host 列表。非浏览器调用方可以自行设置这些头。[HTTP bridge](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/client/connection/src/http-bridge.ts#L69-L74) 没有在 Fetch 请求中保留 socket 对端；[RPC 拦截器](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/client/connection/src/rpc-host.ts#L71-L108) 同样使用头部判定。[API 处理器](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/host/apiproxy/src/api-proxy.ts#L3260-L3366) 包含配置修改和凭据设置/移除操作；本发现不声称 describe 返回明文秘密。

这是带前提的 P1 发现，经过静态确认，没有执行利用：非本机调用方须先通过对外监听、端口转发或无独立鉴权的代理到达处理器。默认隔离的 loopback 监听不会自动变成远端可达。本轮未检查实际部署或已安装的桌面构建。

## Proposal

在 HTTP 入口保留可信传输对端信息。两种注册路径中的特权方法都须在既有浏览器来源检查之外，验证真实 loopback 对端。有意远程开放特权方法的部署需要明确的鉴权与授权策略。任意 Host 或 X-Forwarded-* 值不得充当对端身份；代理信任需要限定范围并单独审查，特权方法缺失传输上下文时应拒绝访问。

[浏览器信任决策](../../implemented/architecture/2026-07-28-api-browser-trust-boundary.md) 继续有效：其中的 JSON/媒体类型、Host/Origin 检查针对浏览器 confused-deputy 和 DNS rebinding 请求，并明确不负责远程客户端鉴权。本提案为较窄的特权方法本机访问承诺增加对端验证，不替代或归档该决策，也不删除或改写任何既有 Agent Note。

## Alternatives considered

**仅保留空 trusted-host 列表这一额外检查。** 对特权方法不采用，因为调用方控制的头部不能证明网络对端。

**加入 socket 检查后移除浏览器检查。** 不采用，因为仅验证 loopback 对端不能阻止不可信浏览器来源访问本机服务。

**信任所有转发头。** 不采用，因为直连调用方可以自行提供这些字段，代理也可能抹去原始对端差异。远程访问需要明确的代理与鉴权策略。

## Acceptance criteria

- [ ] 非 loopback 传输对端携带 localhost Host 且无 Origin 时，必须在特权处理器产生副作用前返回 403；覆盖路由和 RPC 拦截器两种注册路径
- [ ] 有效 IPv4/IPv6 loopback 请求保留预期本机行为；错误 Origin、跨站元数据、畸形 authority 与缺失对端元数据按声明的策略拒绝
- [ ] 头部值不能覆盖可信传输元数据；未知或不可信代理上下文不能授予特权访问
- [ ] 既有 trustedHosts 策略允许的非特权远程客户端保留文档行为；有意开放的特权远程访问需要明确鉴权测试
- [ ] 使用合成请求/响应 fixture（测试前置数据）与副作用替身；关闭发现前，按实际实现提交记录针对性测试、类型检查、doc-sync（文档同步门禁）和组装应用证据。本次文档 PR（Pull Request）不操作凭据、外部程序或真实配置

## Risks

错误的代理处理可能阻断合法本机客户端，或把代理转发的远端请求视为本机。应明确处理兼容性决策，不能从请求头推断身份。本提案不是 OS 沙箱、完整远程鉴权系统，也不代表已审查全部 API 路由。运行时结果和仓库检查均未运行，文档 PR 合并后发现仍保持开放。
