# ADR-001: MCP server OAuth with Microsoft Entra (HTTP + stdio)

- Status: Proposed
- Date: 2026-09-06
- Deciders: sheshi sheri / platform (MCP C# servers)
- Related: [research/2026-09-06-mcp-csharp-entra-auth-design.md](../2026-09-06-mcp-csharp-entra-auth-design.md)
- Spec baseline: MCP Authorization 2026-07-28 (official only; SEP-2809/3004 out of scope)

## Context

We build C# MCP servers over Streamable HTTP and stdio. The host/agent authenticates users via Microsoft Entra ID (OIDC) and obtains a JWT. Product requirement: both transports must validate that JWT (`iss`, JWKS/sig, `aud`, `exp`, scopes/roles), including stdio even on the same machine/process tree. We need an ADR for how OAuth/credentials work at design time.

The official 2026-07-28 authorization specification applies to HTTP-based transports. Implementations using stdio **SHOULD NOT** follow that HTTP OAuth flow and should retrieve credentials from the environment (or third-party libraries embedded in the server). There is no normative `_meta.authorization`. Retrieval is not validation. Validation is still required if the server will honor the token.

Failure mode this ADR is written to prevent: the HTTP server validates Entra JWTs while the stdio sibling trusts the parent process, or both servers accept any Entra token signed for a different API.

## Decision

### Shared (both transports)

1. The MCP server is an OAuth 2.1 **resource server** for token *validation* semantics even when delivery differs.
2. Always validate Entra JWTs before tools, prompts, or resources. Fail closed. Never log raw tokens. No token passthrough to upstream APIs.
3. Pin audience to this MCP API (Entra Application ID URI / agreed `aud`). Reject Graph or other APIs’ tokens.
4. Prefer short-lived access tokens; refresh stays with the host (or with MSAL inside the server if that option is chosen).

Shared validation (identical regardless of delivery channel):

- Signature against Entra JWKS
- `iss` (tenant / v2.0 issuer)
- `aud` bound to this MCP API only
- `exp` / `nbf` (plus bounded clock skew)
- Token type and authorization: delegated `scp` vs app-only `roles`
- Optional tenant pin (`tid`) when product requires it

Do not treat process launch, localhost binding, or “the host already signed in” as authentication.

### Streamable HTTP

Follow official MCP Authorization for HTTP:

- `Authorization: Bearer <jwt>` on every request (tokens **MUST NOT** appear in the query string)
- RFC 9728 Protected Resource Metadata (PRM), advertised via `WWW-Authenticate`
- RFC 8707 resource indicators on the client (`resource` = canonical MCP server URI)
- Streamable HTTP `Origin` validation (invalid present `Origin` → HTTP 403)
- Status mapping: 401 (missing/invalid token), 403 (authenticated but insufficient scope), 400 (malformed authorization request)
- Use ASP.NET Core JwtBearer / `Microsoft.Identity.Web` + `ModelContextProtocol.AspNetCore`

The C# MCP server is not the Entra authorization endpoint, not a consent screen, and not a token issuer. Do not re-implement `/authorize` or `/token` on the MCP host unless deliberately building a proxy AS.

### stdio (delivery)

Official MCP: stdio **SHOULD NOT** follow the HTTP OAuth flow; retrieve credentials from the environment (or third-party libraries embedded in the server). There is no normative `_meta.authorization`.

**Default delivery (recommended for interop):** environment variable at process spawn (e.g. `MCP_ACCESS_TOKEN`), not argv.

**Allowed alternatives** (document as org profiles if chosen):

- Embedded MSAL / Azure.Identity in the server (server acquires token itself)
- OS secret store (Windows Credential Manager / DPAPI)
- Custom `_meta.authorization` on JSON-RPC messages (per-call / rotation)
- Bootstrap token on `initialize` `_meta` then process-lifetime cache
- Side-channel IPC (named pipe / socket) — only if already required by platform
- Avoid: world-readable temp files; OS-user-only with no JWT unless product explicitly drops Entra binding

**Validation is identical** regardless of delivery channel. After the JWT validates, a stdio message filter must set `context.User` to a `ClaimsPrincipal` built from the token claims. Official C# identity propagation copies `HttpContext.User` on HTTP; on stdio, `ClaimsPrincipal` is **null** unless that filter runs. Do not inject a synthetic “stdio-user” principal without validating a token.

## Alternatives considered (stdio delivery matrix)

| # | Delivery | Official leaning | Pros | Cons |
| --- | --- | --- | --- | --- |
| 1 | Env var at spawn | Yes — retrieve credentials from the environment | Simple; works with stock hosts that can set `env` in MCP config; token stays off the JSON-RPC wire; one validate-at-startup (or first use) path; child process isolation if env is not inherited broadly | Process-lifetime token; refresh usually means restart; env can leak via `/proc`, crash dumps, process dumps, overly broad inheritance; easy to misconfigure (shared parent env, argv fallback, config files) |
| 2 | Embedded MSAL / Azure.Identity | Yes — official tutorial “third-party libraries” | Server owns login/refresh; no host JWT handoff; short-lived tokens + refresh without respawn; matches Azure.Identity patterns already used in C# | Not “host hands JWT”; interactive/device-code or managed-identity setup lives in the server; host and server identities can diverge; extra dependencies and first-run UX |
| 3 | OS secret store / DPAPI | Custom (naming convention) | Cleaner than a raw env var in some Windows deployments; secret not sitting in the process environment table; OS ACLs / DPAPI bind to user or machine | Naming convention is custom; host and server must agree on credential name; not portable to every host; still typically process-lifetime unless polled |
| 4 | `_meta.authorization` per message | Custom — no normative `_meta.authorization` | Rotation and step-up without restart; per-call identity; fits “validate every call” like HTTP Bearer; host can swap user/tenant tokens if one stdio process is multiplexed | Non-interop with stock hosts; token rides in message metadata (protocol logs, OpenTelemetry, debug traces); every call path must attach it; invented schema to version; easy to forget on notifications / `tools/list` |
| 5 | Initialize bootstrap `_meta` then cache | Custom hybrid | One attachment at `initialize`; simpler than per-message `_meta`; still allows host-owned login | Still custom / non-interop; token lifetime ≈ process lifetime after bootstrap; initialize-only `_meta` is easy to miss on reconnect; cache must not be logged |
| 6 | Side-channel IPC (named pipe / socket) | Custom — only if platform already requires it | Powerful: rotation, back-channel revoke, no token on JSON-RPC or env; can reuse an existing platform IPC | Complexity (auth of the side channel itself, lifecycle, Windows vs Unix); fragments clients; overkill unless the platform already has the pipe/socket |
| 7 | Restricted file drop | Generally discouraged | Simple to reason about for some launchers; can be ACL’d to the OS user | TOCTOU and leftover files; world-readable temp files are unacceptable; easy to echo paths into logs; not official |
| 8 | OS identity only (no JWT) | Different trust model | No token handling; process identity / named-pipe ACL is the whole story | Only if product explicitly waives Entra audience binding; no `aud`/`iss`/`scp` check; “local process = trusted” is the failure mode this ADR rejects |

### Practical pick table

| Goal | Prefer |
| --- | --- |
| Interop / default host config | Env |
| Server owns login / refresh | MSAL inside server |
| Host owns login; per-call / rotate | `_meta.authorization` |
| Host owns login; cleaner than env | Credential Manager / DPAPI |

v1 recommendation: **env var at spawn** (`MCP_ACCESS_TOKEN`), with the same JWT validation as HTTP. Document any other row as an org profile, not as protocol-mandated MCP.

## Consequences

### Positive

- Clear split: HTTP = normative MCP OAuth; stdio = delivery profile + same validation
- Avoids false sense of security from “local process = trusted”
- Gives product a menu without pretending custom channels are protocol-mandated

### Negative / risks

- Env default needs careful inheritance and no argv/logging
- `_meta` or file/IPC profiles fragment client interoperability
- Entra `resource` URI vs Application ID URI `aud` friction remains a separate design decision (see [research note — Entra mapping](../2026-09-06-mcp-csharp-entra-auth-design.md#entra-mapping)): RFC 8707 wants the MCP HTTP URI in `resource` / `aud`; Entra `aud` is usually the Application ID URI (`api://<app-id>`) or the application (client) ID GUID. Either set the Entra Application ID URI **equal** to the MCP canonical resource URI, or document that the server validates Entra `aud` while PRM `resource` remains the HTTP URI. Do not silently accept both plus Graph (`00000003-0000-0000-c000-000000000000`).

### Follow-ups

- Pick one stdio delivery for v1 (recommend Env) and document env var name + `ClaimsPrincipal` filter for C# SDK
- Align Entra App ID URI with MCP canonical resource URI where possible
- Private audit schema (SEP-3004 not Final)

## References

- [MCP Authorization (2026-07-28)](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [MCP authorization tutorial (2026-07-28)](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/authorization)
- [Authorization security considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations)
- [Security best practices](https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices)
- [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [stdio](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
- [RFC 9728 — OAuth 2.0 Protected Resource Metadata](https://www.rfc-editor.org/rfc/rfc9728)
- [RFC 8707 — Resource Indicators for OAuth 2.0](https://www.rfc-editor.org/rfc/rfc8707)
- [research/2026-09-06-mcp-csharp-entra-auth-design.md](../2026-09-06-mcp-csharp-entra-auth-design.md)
- [C# SDK identity and role propagation](https://github.com/modelcontextprotocol/csharp-sdk/blob/main/docs/concepts/identity/identity.md)
- [ProtectedMcpServer sample](https://github.com/modelcontextprotocol/csharp-sdk/tree/main/samples/ProtectedMcpServer)
