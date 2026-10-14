# Agent Note: Verify the transport peer for privileged API methods

Status: proposed

English | [中文](2026-10-14-privileged-api-peer-verification.zh.md)

## Problem

Static review baseline: default branch master at `47f943859bef60e4160492346772ded9b24f765a`, rechecked on 2026-10-14 UTC; complete recursive tree `f904efab9ef435201d6ba4da88a34d6366568272` contains 8,617 entries and is not truncated. This proposal changes documentation only. No implementation, deployment, credential access, exploit, build or repository test was performed.

The [request predicate](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/client/connection/src/api-request-trust.ts#L96-L123) checks Host, Origin and Sec-Fetch-Site without connection-peer information. A loopback Host with no Origin passes. The [privileged route policy](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/client/connection/src/index.ts#L69-L170) promises loopback-only access for configuration, credential and other host operations but applies the same predicate with an empty trusted-host list. A non-browser caller can choose these headers. The [HTTP bridge](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/client/connection/src/http-bridge.ts#L69-L74) does not preserve the socket peer in its Fetch request; the [RPC interceptor](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/client/connection/src/rpc-host.ts#L71-L108) uses the same header-based decision. The [API handlers](https://github.com/nico2656-ui/deepseek-harness/blob/47f943859bef60e4160492346772ded9b24f765a/packages/host/apiproxy/src/api-proxy.ts#L3260-L3366) include settings changes and credential set/unset operations; this finding does not assert that describe returns plaintext secrets.

This is a conditional P1 finding, confirmed statically rather than exploited: a non-local caller must first reach the handler through non-loopback listening, forwarding or a proxy without independent authentication. The default isolated loopback listener is not automatically remotely reachable. Neither a deployed instance nor an installed desktop build was inspected.

## Proposal

Preserve trusted transport-peer information at the HTTP entry point. Require a real loopback peer for privileged methods in both registration paths, in addition to the existing browser-origin checks. A deployment that intentionally exposes privileged methods remotely needs an explicit authenticated authorization policy. Arbitrary Host or X-Forwarded-* values must not supply peer identity; proxy trust needs a bounded, separately reviewed design and missing transport context must fail closed for these methods.

The [browser-trust decision](../../implemented/architecture/2026-07-28-api-browser-trust-boundary.md) remains active: its JSON/media-type and Host/Origin checks defend against browser confused-deputy and DNS-rebinding requests and explicitly do not authenticate remote clients. This proposal adds peer verification for the narrower privileged-method promise; it does not supersede or archive that decision. No existing note is deleted or rewritten.

## Alternatives considered

**Retain the empty trusted-host list as the only extra check.** Rejected for privileged methods because caller-controlled headers cannot establish the network peer.

**Remove browser checks after adding a socket check.** Rejected because a loopback peer alone does not stop an untrusted browser origin from reaching local services.

**Trust every forwarding header.** Rejected because direct callers can supply those fields and proxies may erase the original peer distinction. Remote access needs explicit proxy and authentication policy.

## Acceptance criteria

- [ ] A non-loopback transport peer with localhost Host and no Origin receives 403 before any privileged handler effect; cover both route and RPC-interceptor registration paths.
- [ ] Valid IPv4/IPv6 loopback requests retain intended local behavior, while wrong Origin, cross-site metadata, malformed authority and missing peer metadata fail according to the declared policy.
- [ ] Header values cannot overwrite trusted transport metadata; unknown or untrusted proxy context cannot grant privileged access.
- [ ] A non-privileged remote client allowed by the existing trustedHosts policy keeps its documented behavior; intentional privileged remote access requires explicit authentication tests.
- [ ] Use synthetic request/response fixtures and stub side effects. Record focused tests, type checks, doc-sync and assembled-application evidence at the exact implementation commit before closing the finding. No credentials, external programs or real settings are touched by this documentation PR.

## Risks

Incorrect proxy treatment can block legitimate local clients or treat forwarded remote requests as local. Handle that compatibility decision explicitly rather than inferring identity from request headers. This proposal is not an OS sandbox, a complete remote-authentication system or a claim that all API routes have been audited. Runtime results and repository checks remain not run; the finding stays open after this documentation PR merges.
