# MCP C# servers: Entra JWT design-time auth checklist (official 2026-07-28)

**Date:** 2026-09-06
**Status:** ad-hoc research (not a daily idea candidate)
**Audience:** C# MCP server authors whose host already holds a Microsoft Entra ID access token
**Revision in scope:** [MCP specification 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/) only
**Out of scope:** SEP-2809 (ATSA / attested tool-server admission) and SEP-3004 (tamper-evident audit records). Both are non-Final and are not part of the official 2026-07-28 authorization surface.

This note stays in the private `ai-security-ideas` repo. It is a design-time checklist, not an implementation.

**ADR:** [ADR-001: MCP server OAuth with Microsoft Entra (HTTP + stdio)](adr/001-mcp-server-oauth-entra.md)

---

## Direct answer

If the host already has an Entra JWT, **every C# MCP server that will act on that token MUST validate it itself**. That is true for Streamable HTTP **and** for stdio. HTTP has a normative OAuth 2.1 resource-server profile. Stdio does not. Invent a stdio delivery profile (environment variable by default, official-leaning; optional custom `_meta.authorization` if you control host and server) and reuse the **same** JWT validation rules. Do not treat process launch, localhost binding, or "the host already signed in" as authentication.

The official 2026-07-28 authorization specification applies to HTTP-based transports. Implementations using stdio **SHOULD NOT** follow that HTTP OAuth flow and should retrieve credentials from the environment instead. Retrieval is not validation. Validation is still required if the server will honor the token.

---

## Problem

The host (IDE, agent, or desktop app) completes Entra sign-in and holds a JWT. It then talks to one or more C# MCP servers:

| Transport | How the host reaches the server | Official auth channel |
| --- | --- | --- |
| Streamable HTTP | `Authorization: Bearer <jwt>` on every request | OAuth 2.1 resource server + RFC 9728 PRM |
| stdio | Client-launched subprocess, JSON-RPC on stdin/stdout | Environment credentials. No `Authorization` header. No PRM. |

Failure mode this checklist is written to prevent: the HTTP server validates Entra JWTs while the stdio sibling trusts the parent process, or both servers accept any Entra token that happens to be signed for a different API.

---

## Official surface (2026-07-28 only)

Read these pages. Do not mix draft SEPs into this design.

| Topic | Official page | What it decides |
| --- | --- | --- |
| Authorization | [Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) | HTTP-only OAuth 2.1 resource-server profile. Optional to implement; when implemented, HTTP **SHOULD** conform and stdio **SHOULD NOT**. |
| Security considerations | [Authorization security considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations) | Normative MUST/SHOULD for audience binding, PKCE, HTTPS, mix-up, confused deputy, no token passthrough. |
| Security best practices | [Security best practices](https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices) | Why audience validation and token-passthrough bans exist. |
| Protected Resource Metadata | [RFC 9728](https://www.rfc-editor.org/rfc/rfc9728) | HTTP servers **MUST** publish PRM. Clients **MUST** use it to discover the authorization server. |
| Resource indicators | [RFC 8707](https://www.rfc-editor.org/rfc/rfc8707) | Clients **MUST** send `resource` = canonical MCP server URI on authorize and token requests. Servers **MUST** accept only tokens issued for themselves. |
| Streamable HTTP Origin | [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http) | Servers **MUST** validate `Origin` on all incoming connections (DNS rebinding). Invalid present `Origin` → HTTP 403. Local bind **SHOULD** be `127.0.0.1`. Servers **SHOULD** authenticate all connections. |
| stdio credentials | [Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) + [stdio](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio) | Stdio **SHOULD NOT** follow the HTTP OAuth spec. Retrieve credentials from the environment. The binding has no header layer. |

Related RFCs the authorization page selects: OAuth 2.1 draft, RFC 6750 (Bearer), RFC 8414 (AS metadata), RFC 7591 (DCR, deprecated in favor of Client ID Metadata Documents), RFC 9207 (`iss` in the authorization response), OpenID Connect Discovery.

---

## HTTP checklist (MUST / SHOULD)

Apply this when the C# server is reached over Streamable HTTP and authorization is enabled. The MCP server is an OAuth 2.1 **resource server**. It verifies tokens. It does not issue them. Entra is the authorization server.

### MUST

- **Publish RFC 9728 Protected Resource Metadata** at the well-known path and point unauthenticated clients at it with `WWW-Authenticate: Bearer resource_metadata="…"`.
- **Require `Authorization: Bearer <access-token>` on every HTTP request.** Tokens **MUST NOT** appear in the query string.
- **Validate the access token** as an OAuth 2.1 resource server (OAuth 2.1 §5.2): signature (Entra JWKS), issuer, audience, lifetime, and required claims. Invalid or expired tokens **MUST** produce HTTP 401.
- **Bind audience to this server.** Accept only tokens issued specifically for this MCP server (RFC 8707 §2). Reject tokens whose `aud` is another API, another MCP server, or a Graph/Microsoft resource.
- **Do not pass tokens through.** If this server calls an upstream API, it **MUST** use a separately issued upstream token. It **MUST NOT** accept or transit tokens that are not for its own resources.
- **Validate `Origin`** on Streamable HTTP. If `Origin` is present and not an allowed origin, respond HTTP 403. This is independent of JWT validation.
- **Use HTTPS** for authorization-server endpoints. Redirect URIs **MUST** be `localhost` or HTTPS.
- **Return the right status:** 401 (missing/invalid token), 403 (authenticated but insufficient scope), 400 (malformed authorization request).
- **Honor scope hierarchies** when deciding whether a token is sufficient for an operation.

Client-side requirements the host must already satisfy (the server should assume they happened, then still verify the resulting token):

- Authorization code flow **MUST** use **PKCE**. Prefer `S256`. Refuse to proceed if AS metadata lacks `code_challenge_methods_supported`.
- Clients **MUST** send RFC 8707 `resource` on both authorization and token requests, using the canonical MCP server URI (scheme + host, no fragment; prefer no trailing slash unless the slash is semantically significant).
- Clients **MUST** discover the AS via PRM, then RFC 8414 or OpenID Connect Discovery.
- Clients **MUST NOT** send this server a token issued for a different resource.

### SHOULD

- HTTP implementations **SHOULD** conform to the authorization specification when they support auth at all.
- Include a `scope` parameter on `WWW-Authenticate` (RFC 6750 §3) so clients request least privilege.
- On insufficient scope, prefer HTTP 403 with `error="insufficient_scope"`, the needed `scope`, and `resource_metadata`. Emit all scopes required for the current operation in one challenge.
- When running locally, bind only to `127.0.0.1`, not `0.0.0.0`.
- Authenticate all Streamable HTTP connections, not only tool calls.
- **SHOULD NOT** advertise `offline_access` in PRM `scopes_supported` or in the 401/403 `scope` challenge. Refresh tokens are not a resource-server requirement.
- Authorization servers **SHOULD** issue short-lived access tokens.

### Roles the C# server is not

The C# MCP server is not the Entra authorization endpoint, not a consent screen, and not a token issuer. Dynamic Client Registration is optional and deprecated relative to Client ID Metadata Documents. Do not re-implement `/authorize` or `/token` on the MCP host unless you are deliberately building a proxy AS (that path is where confused-deputy controls apply).

---

## Stdio invented profile (env delivery + same validation)

Official text: stdio implementations **SHOULD NOT** follow the HTTP authorization specification and should retrieve credentials from the environment. There is no normative JWT profile for stdio: no header name, no PRM, no 401/`WWW-Authenticate`, no `Origin`.

Invented profile for this design (not official MCP, but the only way both transports honor the same Entra JWT):

1. **Delivery (default / official-leaning).** The host places the raw Entra access token in a dedicated environment variable when it launches the subprocess (for example `MCP_ACCESS_TOKEN`). This is the default path and matches MCP guidance to retrieve credentials from the environment. Do not put the token on `argv`. Do not write it to a world-readable file. Do not log `stdout`/`stderr` that might echo it.
2. **Delivery (optional custom alternative).** `_meta.authorization` on each JSON-RPC message is an org-specific profile, not official MCP. There is no normative `_meta.authorization` for stdio. Use it only when you control host and server and need rotation or per-call identity. Documented in [Stdio JWT delivery: env vs `_meta.authorization`](#stdio-jwt-delivery-env-vs-_metaauthorization).
3. **Same validation.** The stdio server MUST run the same JWT checks as the HTTP resource server: signature against Entra JWKS, `iss`, `aud`, `exp`/`nbf`, token type (delegated `scp` vs app-only `roles`), and tenant if you pin one. Failure is a hard refuse: do not handle tools, prompts, or resources. Same non-negotiables apply whichever delivery path you pick.
4. **Identity into the SDK.** Official C# identity propagation copies `HttpContext.User` on HTTP. On stdio, `ClaimsPrincipal` is **null** unless a message filter sets `context.User`. After the JWT validates, the filter builds a `ClaimsPrincipal` from the token claims so tools can take `ClaimsPrincipal` the same way on both transports. Do not inject a synthetic "stdio-user" principal without validating a token.
5. **No PRM / no Origin on the pipe.** Do not fake `WWW-Authenticate` on stdout. Do not skip JWT validation because the process is local. Local launch is a trust boundary for *who can start the process*, not for *who the token is for*.

This profile is an engineering convention. Env delivery is the official-leaning default. `_meta.authorization` is an optional custom alternative. Call both out as invented in any implementation repo so they are not mistaken for 2026-07-28 text.

---

## Stdio JWT delivery: env vs `_meta.authorization`

Official MCP Authorization (2026-07-28): stdio **SHOULD NOT** follow the HTTP OAuth flow; retrieve credentials from the environment. There is no normative `_meta.authorization` for stdio.

### Env var at spawn (official-leaning)

**Pros**

- Matches MCP guidance (“credentials from the environment”)
- Works with stock hosts that can set `env` in MCP config
- Token stays off the JSON-RPC wire (less accidental log leakage of the protocol body)
- One validate-at-startup (or first use) path — simple in C#
- Child process isolation: only that process sees the env if you don’t inherit broadly

**Cons**

- Token lifetime ≈ process lifetime; refresh usually means restart
- Env can leak via `/proc`, crash dumps, process dumps, overly broad environment inheritance
- Harder to rotate mid-session or do step-up scopes without respawn
- Easy to misconfigure (token in shared parent env, argv fallback, config files)

### `_meta.authorization` on each message (custom profile)

**Pros**

- Per-request (or per-batch) token: rotation and step-up without restart
- Fits “validate every call” mental model same as HTTP Bearer
- Host can swap user/tenant tokens if one stdio process is multiplexed (advanced)
- Token not sitting in process env for the whole lifetime

**Cons**

- Not in the MCP Authorization spec — third-party hosts won’t send it unless you teach them
- Token rides in message metadata → higher risk of capture in protocol logs, OpenTelemetry attributes, debug traces
- Every call path must attach + server must reject missing/invalid `_meta` (easy to forget on notifications / `tools/list`)
- You invent schema (`_meta.authorization` vs Authorization string vs object) and version it yourself
- Slightly more host/server coupling and test surface

### Practical pick

- Interop / default host config → env
- You control host + server and care about rotation / per-call identity → `_meta` (document as org profile; still validate like HTTP)

Same non-negotiables either way: `iss`, JWKS/sig, `aud`, `exp`, scopes/roles; never log the raw JWT; fail closed.

---

## Entra mapping

Entra is a compatible authorization server if you force it onto the MCP resource-server shape.

| MCP 2026-07-28 expectation | Entra reality | Design choice |
| --- | --- | --- |
| OAuth 2.1 authorization code + PKCE | Entra interactive apps use auth code + PKCE | Use this grant for user-delegated hosts. Do not use implicit flow. App-only (`client_credentials`) is a different token shape (`idtyp=app`, `roles` instead of `scp`) and must be an explicit profile, not an accident. |
| RFC 8707 `resource` = canonical MCP URI (`https://mcp.example.com/mcp`) | Entra `aud` is usually the **Application ID URI** (`api://<app-id>`) or the application (client) ID GUID, depending on v1.0 vs v2.0 token | This is the main friction. Either set the Entra Application ID URI **equal** to the MCP canonical resource URI, or document that the server validates Entra `aud` (App ID URI / client ID) while PRM `resource` remains the HTTP URI. Do not silently accept both plus Graph (`00000003-0000-0000-c000-000000000000`). |
| PRM `authorization_servers` | `https://login.microsoftonline.com/{tenant}/v2.0` (or the tenant's OIDC issuer) | Publish that issuer in RFC 9728 metadata. Clients then hit OIDC discovery (`/.well-known/openid-configuration`). |
| Token audience binding | Entra will not emit `aud` = MCP URL unless you configured the API that way | Fail closed. One accepted `aud` (or a tight explicit list). No "any token from this tenant." |
| Scopes | Entra delegated scopes become the `scp` space-delimited claim; app roles become `roles` | HTTP 403 `insufficient_scope` should speak MCP/OAuth scope names. Map them onto Entra exposed-API scopes at design time. |
| PKCE metadata | Entra OIDC metadata includes `code_challenge_methods_supported` | Hosts **MUST** refuse if that field is missing. |
| Short-lived access tokens | Entra access tokens are typically ~60–90 minutes | Matches the SHOULD. Do not log or persist them on the MCP server beyond request handling. |

**Do not** configure the MCP server to accept a Microsoft Graph token and then call Graph with that same JWT. That is token passthrough, which 2026-07-28 forbids.

---

## Gaps (official spec)

1. **No stdio JWT profile.** Stdio is "get credentials from the environment." There is no MUST for JWT validation, claim set, env var name, rotation, or how to fail. There is also no normative `_meta.authorization`. The invented profile above fills that hole for Entra hosts (env as the official-leaning default; `_meta.authorization` as an optional custom alternative). It is not protocol text.
2. **No normative audit schema.** 2026-07-28 tells you to validate tokens and not leak data to unauthorized parties. It does not define an audit record, hash chain, or receipt format. SEP-3004 is out of scope here because it is not Final. If you need audit, invent a private log schema; do not claim MCP compliance for it.
3. **Audience identifier split.** RFC 8707 wants the MCP HTTP URI in `resource` / `aud`. Enterprise IdPs (Entra especially) want App ID URIs. The spec does not define a mapping profile.
4. **Authorization is OPTIONAL.** A C# server can ship with no auth and still be spec-valid. Enterprise hosts that already have Entra JWTs should treat validation as mandatory in *product* policy, not because the revision requires every server to implement OAuth.

SEP-2809 (attested admission) is also out of scope. This checklist is token validation at the server, not host-side attestation of the server binary.

---

## C# pointers

Use the official SDK as a resource server. Do not write a custom JWT parser.

| Piece | Where | Use |
| --- | --- | --- |
| `ModelContextProtocol.AspNetCore` | [modelcontextprotocol/csharp-sdk](https://github.com/modelcontextprotocol/csharp-sdk) | HTTP transport, `MapMcp` / `WithHttpTransport`, PRM helpers, `RequireAuthorization()`, `AddAuthorizationFilters()`. |
| JWT Bearer | `Microsoft.AspNetCore.Authentication.JwtBearer` | Authority = Entra tenant v2.0. `ValidateIssuer`, `ValidateAudience`, `ValidateLifetime`, JWKS signature. Tight `ValidAudience`. |
| `Microsoft.Identity.Web` | [Microsoft identity web](https://github.com/AzureAD/microsoft-identity-web) | `AddMicrosoftIdentityWebApi` for Entra resource-server defaults (`iss`/`tid`/`aud`/`scp`). Prefer this over hand-rolled `AddJwtBearer` if the host is already an Entra app. |
| Identity propagation | [Identity and role propagation](https://github.com/modelcontextprotocol/csharp-sdk/blob/main/docs/concepts/identity/identity.md) | HTTP: middleware fills `HttpContext.User`, SDK copies it to `ClaimsPrincipal`. Stdio: **null** until a message filter sets `context.User`. Inject `ClaimsPrincipal` into tools; do not use `IHttpContextAccessor` if you also serve stdio. |
| Sample | [ProtectedMcpServer](https://github.com/modelcontextprotocol/csharp-sdk/tree/main/samples/ProtectedMcpServer) | JWT bearer + resource metadata + protected tools. HTTP only. |

Design-time wiring (not a drop-in snippet to paste into production):

- HTTP: authentication middleware → `RequireAuthorization()` on the MCP endpoint → optional `[Authorize]` / `scp` policies on tools → `ClaimsPrincipal` in handlers.
- Stdio: read env token → same token-validation parameters as JWT Bearer → message filter sets `context.User` → same tool policies.
- Shared validation options object for both transports so `aud` / `iss` / clock skew cannot drift.

---

## Sources (official 2026-07-28)

- [MCP specification 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/)
- [Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [Authorization security considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations)
- [Security best practices](https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices)
- [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [stdio](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
- [Authorization tutorial (docs)](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/authorization)
- [RFC 9728 — OAuth 2.0 Protected Resource Metadata](https://www.rfc-editor.org/rfc/rfc9728)
- [RFC 8707 — Resource Indicators for OAuth 2.0](https://www.rfc-editor.org/rfc/rfc8707)
- [RFC 6750 — Bearer token usage](https://www.rfc-editor.org/rfc/rfc6750)
- [C# SDK identity](https://github.com/modelcontextprotocol/csharp-sdk/blob/main/docs/concepts/identity/identity.md)

---

## What this is not

- Not a daily `ideas/` candidate. Do not copy it into `ideas/2026-09.md`.
- Not an implementation repo and not a PoC.
- Not SEP-2809 or SEP-3004.
- Not permission to accept Graph tokens or skip JWT checks on stdio.
