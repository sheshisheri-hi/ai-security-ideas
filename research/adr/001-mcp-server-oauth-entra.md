# ADR-001: MCP server OAuth with Microsoft Entra (HTTP + stdio)

- Status: Proposed
- Date: 2026-09-06
- Deciders: sheshi sheri / platform (MCP C# servers)
- Related: [research/2026-09-06-mcp-csharp-entra-auth-design.md](../2026-09-06-mcp-csharp-entra-auth-design.md)
- Spec baseline: MCP Authorization 2026-07-28 (official only; SEP-2809/3004 out of scope)
- Read next: [Flows](#flows-sequence-diagrams) · [Decision tree](#decision-tree-pick-one) · [Security best practices](#security-best-practices-role-mapped)

## Context

We build C# MCP servers over Streamable HTTP and stdio. The MCP host/client authenticates users via Microsoft Entra ID (OIDC) and obtains a JWT. Product requirement: both transports must validate that JWT (`iss`, JWKS/sig, `aud`, `exp`, scopes/roles), including stdio even on the same machine/process tree. We need an ADR for how OAuth/credentials work at design time.

The official 2026-07-28 authorization specification applies to HTTP-based transports. Implementations using stdio **SHOULD NOT** follow that HTTP OAuth flow and should retrieve credentials from the environment (or third-party libraries embedded in the MCP server). There is no normative `_meta.authorization`. Retrieval is not validation. Validation is still required if the MCP server will honor the token.

Failure mode this ADR is written to prevent: the HTTP MCP server validates Entra JWTs while the stdio sibling trusts the parent process, or both MCP servers accept any Entra token signed for a different API.

## Decision

### Shared (both transports)

1. The MCP server is an OAuth 2.1 **resource server** for token *validation* semantics even when delivery differs.
2. Always validate Entra JWTs before tools, prompts, or resources. Fail closed. Never log raw tokens. No token passthrough to upstream APIs.
3. Pin audience to this MCP API (Entra Application ID URI / agreed `aud`). Reject Graph or other APIs’ tokens.
4. Prefer short-lived access tokens; refresh stays with the MCP host/client (or with MSAL inside the MCP server if that option is chosen).

Shared validation (identical regardless of delivery channel):

- Signature against Entra JWKS
- `iss` (tenant / v2.0 issuer)
- `aud` bound to this MCP API only
- `exp` / `nbf` (plus bounded clock skew)
- Token type and authorization: delegated `scp` vs app-only `roles`
- Optional tenant pin (`tid`) when product requires it

Do not treat process launch, localhost binding, or “the MCP host/client already signed in” as authentication.

### Streamable HTTP

Follow official MCP Authorization for HTTP:

- `Authorization: Bearer <jwt>` on every request (tokens **MUST NOT** appear in the query string)
- RFC 9728 Protected Resource Metadata (PRM), advertised via `WWW-Authenticate`
- RFC 8707 resource indicators on the MCP host/client (`resource` = canonical MCP server URI)
- Streamable HTTP `Origin` validation (invalid present `Origin` → HTTP 403)
- Status mapping: 401 (missing/invalid token), 403 (authenticated but insufficient scope), 400 (malformed authorization request)
- Use ASP.NET Core JwtBearer / `Microsoft.Identity.Web` + `ModelContextProtocol.AspNetCore`

The C# MCP server is not the Entra authorization endpoint, not a consent screen, and not a token issuer. Do not re-implement `/authorize` or `/token` on the MCP server unless deliberately building a proxy authorization server.

### stdio (delivery)

Official MCP: stdio **SHOULD NOT** follow the HTTP OAuth flow; retrieve credentials from the environment (or third-party libraries embedded in the MCP server). There is no normative `_meta.authorization`.

**Default delivery (recommended for interop):** environment variable at process spawn (e.g. `MCP_ACCESS_TOKEN`), not argv.

**Allowed alternatives** (document as org profiles if chosen):

- Embedded MSAL / Azure.Identity in the MCP server (the MCP server acquires the token itself)
- OS secret store (Windows Credential Manager / DPAPI)
- Custom `_meta.authorization` on JSON-RPC messages (per-call / rotation)
- Bootstrap token on `initialize` `_meta` then process-lifetime cache
- Side-channel IPC (named pipe / socket) — only if already required by platform
- Avoid: world-readable temp files; OS-user-only with no JWT unless product explicitly drops Entra binding

**Validation is identical** regardless of delivery channel. After the JWT validates, a stdio message filter must set `context.User` to a `ClaimsPrincipal` built from the token claims. Official C# identity propagation copies `HttpContext.User` on HTTP; on stdio, `ClaimsPrincipal` is **null** unless that filter runs. Do not inject a synthetic “stdio-user” principal without validating a token.

## Alternatives considered (stdio delivery matrix)

| # | Delivery | Official leaning | Pros | Cons |
| --- | --- | --- | --- | --- |
| 1 | Env var at spawn | Yes — retrieve credentials from the environment | Simple; works with stock MCP host/clients that can set `env` in MCP config; token stays off the JSON-RPC wire; one validate-at-startup (or first use) path; child process isolation if env is not inherited broadly | Process-lifetime token; refresh usually means restart; env can leak via `/proc`, crash dumps, process dumps, overly broad inheritance; easy to misconfigure (shared parent env, argv fallback, config files) |
| 2 | Embedded MSAL / Azure.Identity | Yes — official tutorial “third-party libraries” | MCP server owns login/refresh; no MCP host/client JWT handoff; short-lived tokens + refresh without respawn; matches Azure.Identity patterns already used in C# | Not “MCP host/client hands JWT”; interactive/device-code or managed-identity setup lives in the MCP server; MCP host/client and MCP server identities can diverge; extra dependencies and first-run UX |
| 3 | OS secret store / DPAPI | Custom (naming convention) | Cleaner than a raw env var in some Windows deployments; secret not sitting in the process environment table; OS ACLs / DPAPI bind to user or machine | Naming convention is custom; MCP host/client and MCP server must agree on credential name; not portable to every MCP host/client; still typically process-lifetime unless polled |
| 4 | `_meta.authorization` per message | Custom — no normative `_meta.authorization` | Rotation and step-up without restart; per-call identity; fits “validate every call” like HTTP Bearer; MCP host/client can swap user/tenant tokens if one stdio process is multiplexed | Non-interop with stock MCP host/clients; token rides in message metadata (protocol logs, OpenTelemetry, debug traces); every call path must attach it; invented schema to version; easy to forget on notifications / `tools/list` |
| 5 | Initialize bootstrap `_meta` then cache | Custom hybrid | One attachment at `initialize`; simpler than per-message `_meta`; still allows MCP host/client-owned login | Still custom / non-interop; token lifetime ≈ process lifetime after bootstrap; initialize-only `_meta` is easy to miss on reconnect; cache must not be logged |
| 6 | Side-channel IPC (named pipe / socket) | Custom — only if platform already requires it | Powerful: rotation, back-channel revoke, no token on JSON-RPC or env; can reuse an existing platform IPC | Complexity (auth of the side channel itself, lifecycle, Windows vs Unix); fragments MCP host/clients; overkill unless the platform already has the pipe/socket |
| 7 | Restricted file drop | Generally discouraged | Simple to reason about for some launchers; can be ACL’d to the OS user | TOCTOU and leftover files; world-readable temp files are unacceptable; easy to echo paths into logs; not official |
| 8 | OS identity only (no JWT) | Different trust model | No token handling; process identity / named-pipe ACL is the whole story | Only if product explicitly waives Entra audience binding; no `aud`/`iss`/`scp` check; “local process = trusted” is the failure mode this ADR rejects |

### Practical pick table

| Goal | Prefer |
| --- | --- |
| Interop / default MCP host/client config | Env |
| MCP server owns login / refresh | MSAL inside the MCP server |
| MCP host/client owns login; per-call / rotate | `_meta.authorization` |
| MCP host/client owns login; cleaner than env | Credential Manager / DPAPI |

v1 recommendation: **env var at spawn** (`MCP_ACCESS_TOKEN`), with the same JWT validation as HTTP. Document any other row as an org profile, not as protocol-mandated MCP.

## Flows (sequence diagrams)

These diagrams are the option analysis a reader would otherwise have to reconstruct from chat. **Validation is identical in every flow** (Entra JWKS signature, `iss`, `aud` pinned to this MCP API, `exp`/`nbf`, `scp` vs `roles`, optional `tid`). Only *delivery* and *who is the Entra client* change.

How to read them:

- The MCP server is always a **resource server**. It never issues tokens.
- HTTP follows official MCP Authorization (2026-07-28): 401 + PRM, the MCP host/client does auth code + PKCE + `resource`, then `Authorization: Bearer`.
- stdio **SHOULD NOT** follow that HTTP OAuth dance. The MCP host/client (or a library embedded in the MCP server) retrieves a credential; the MCP server still validates the JWT and binds `ClaimsPrincipal`.
- Side-channel IPC, world-readable files, and “OS user only / no JWT” are in the alternatives table and are **not** drawn here.

Shared C# bind step after a JWT validates: message filter sets `context.User` from token claims. On stdio this is required; `ClaimsPrincipal` is otherwise null.

### 1. HTTP — MCP host/client Entra + Bearer

Normative MCP HTTP path. The MCP host/client is the Entra public client application. The MCP server publishes RFC 9728 PRM and validates with ASP.NET Core JwtBearer / `Microsoft.Identity.Web`.

```mermaid
sequenceDiagram
    actor User
    participant Host as MCP host/client
    participant Browser as Browser/Entra
    participant Server as MCP HTTP server

    User->>Host: Open remote MCP session
    Host->>Server: Streamable HTTP request (no Authorization)
    Server-->>Host: 401 WWW-Authenticate Bearer resource_metadata
    Host->>Server: GET PRM well-known
    Server-->>Host: authorization_servers, resource, scopes_supported
    Host->>Browser: Discover AS, then authorize code+PKCE+resource
    User->>Browser: Sign in and consent
    Browser-->>Host: Authorization code
    Host->>Browser: POST /token (code, PKCE verifier, resource)
    Browser-->>Host: Access token JWT aud=this MCP API
    Host->>Server: MCP request Authorization Bearer JWT
    Server->>Server: JwtBearer JWKS iss aud exp scp
    alt Missing or invalid token
        Server-->>Host: 401
    else Authenticated but insufficient scope
        Server-->>Host: 403 insufficient_scope
    else Valid
        Server-->>Host: MCP response
        Host->>Server: tools/call Authorization Bearer JWT
        Server->>Server: JwtBearer again, copy HttpContext.User
        Server-->>Host: Tool result
    end
```

**When to use / when not / failure modes**

- Use when the MCP server is reached over Streamable HTTP (local bind or remote). This is the only official OAuth path.
- Do not invent a parallel HTTP header or query-string token. Do not put the JWT in the URL.
- Do not implement `/authorize` or `/token` on the MCP server unless the product is deliberately a proxy authorization server (confused-deputy controls then apply).
- Failures: missing PRM / wrong `resource` → the MCP host/client cannot obtain a token with the right `aud`; Graph or other-API `aud` → 401; invalid `Origin` → 403 independent of JWT; clock skew / expired `exp` → 401; insufficient `scp` → 403.
- Entra friction: PRM `resource` is the canonical MCP URI; Entra `aud` is usually the Application ID URI. Align them or document the split. Never silently accept both plus Graph.

### 2. stdio — Env var handoff

Official-leaning default. The MCP host/client stays the Entra client application. Token is delivered at spawn in `MCP_ACCESS_TOKEN` (not argv). The MCP server validates once (startup or first use) and binds identity for the process.

```mermaid
sequenceDiagram
    actor User
    participant Host as MCP host/client
    participant Entra as Entra (MCP host/client login)
    participant Server as MCP stdio server

    User->>Host: Sign in
    Host->>Entra: Auth code + PKCE (MCP host/client is IdP client)
    Entra-->>Host: Access token JWT
    Host->>Server: Spawn env MCP_ACCESS_TOKEN (not argv)
    Server->>Server: Read env, validate JWT, set ClaimsPrincipal
    alt Missing env or JWT invalid
        Server-->>Host: Fail closed (no tools/prompts/resources)
    else Valid
        Host->>Server: JSON-RPC initialize
        Server-->>Host: initialize result
        Host->>Server: tools/call
        Server->>Server: Use bound identity, re-check exp
        Server-->>Host: Tool result
    end
```

**When to use / when not / failure modes**

- Use for interop and stock MCP host/client config (`env` in the MCP server launch block). v1 recommendation.
- Do not use when the token must rotate or step-up without respawn, or when one stdio process must multiplex users.
- Do not put the JWT on `argv`, in a world-readable file, or in a shared parent environment that children inherit broadly.
- Failures: empty/malformed env → refuse; expired token mid-session → refuse until the MCP host/client restarts the process with a fresh JWT; `/proc`, crash dumps, and process dumps leak the env; logging `Environment.GetEnvironmentVariable` or echoing config dumps the token.
- Validation still runs. Spawning the MCP server is not authentication.

### 3. stdio — Embedded MSAL interactive + token cache

Official-leaning “third-party libraries” path. The MCP server is the Entra client application. The MCP host/client launches with **no** token. Prefer **first-need** or an explicit **login tool**; do **not** block `initialize` forever waiting on a popup.

```mermaid
sequenceDiagram
    actor User
    participant Host as MCP host/client
    participant Server as MCP stdio server
    participant MSAL
    participant Browser as Browser/Broker/Entra

    Host->>Server: Spawn (no MCP_ACCESS_TOKEN)
    Host->>Server: initialize
    Note over Server: Do not block initialize on UI
    Server-->>Host: initialize OK (auth deferred)
    Host->>Server: tools/call or login tool (first need)
    Server->>MSAL: Silent acquire
    MSAL-->>Server: Cache miss
    alt Interactive UI or broker available
        Server->>MSAL: Interactive / WAM broker
        MSAL->>Browser: Login popup or broker
        User->>Browser: Sign in and consent
        Browser-->>MSAL: Tokens
        MSAL->>MSAL: Write token cache
        MSAL-->>Server: Access token JWT
        Server->>Server: Validate JWT, bind ClaimsPrincipal
        Server-->>Host: Tool result
    else Headless / no UI
        Server-->>Host: Error: interactive login required
        Note over Server: Device code only if product chose it; else fail
    end
    Host->>Server: Later tools/call
    Server->>MSAL: Silent acquire from cache
    alt Cache has valid access token
        MSAL-->>Server: Access token
    else Access token expired, refresh possible
        MSAL->>Browser: Refresh
        Browser-->>MSAL: New access token
        MSAL-->>Server: Access token
    end
    Server->>Server: Validate JWT, bind identity
    Server-->>Host: Tool result
```

**When to use / when not / failure modes**

- Use when the MCP server should own login and refresh (desktop IDE with UI, WAM/broker, or an explicit login tool). Matches Azure.Identity / MSAL patterns already used in C#.
- Do not use as the silent default on headless CI, SSH, or service machines. Interactive MSAL will hang or fail; use env / a token from the MCP host/client or (if product accepts) device code.
- Do not block `initialize` on the first interactive prompt. MCP host/clients time out; prefer deferred auth on first privileged call or a `login` tool the MCP host/client can invoke when a human is present.
- Failures: cache ACL too open → token theft; cache miss after restart with no UI → hard fail (do not retry-loop); MCP host/client identity and MCP server identity diverge if both sign in; refresh-token persistence needs DPAPI/broker protection, not a plaintext file next to the binary.
- Still validate the access token the same way. MSAL acquisition is delivery, not a substitute for `aud`/`iss` checks.

### 4. stdio — `_meta.authorization` per message

Custom org profile (not official MCP). The MCP host/client stays the Entra client application and holds the JWT. Every JSON-RPC message carries `_meta.authorization`. The MCP server validates **each** time and rejects if the field is missing.

```mermaid
sequenceDiagram
    actor User
    participant Host as MCP host/client
    participant Entra
    participant Server as MCP stdio server

    User->>Host: Sign in
    Host->>Entra: MCP host/client login
    Entra-->>Host: JWT
    Host->>Server: Spawn (no env token required)
    Host->>Server: JSON-RPC initialize plus _meta.authorization
    Server->>Server: Validate JWT on this message
    Server-->>Host: initialize result
    Host->>Server: tools/list plus _meta.authorization
    Server->>Server: Validate again
    Host->>Server: tools/call plus _meta.authorization
    Server->>Server: Validate each time, bind ClaimsPrincipal
    alt _meta.authorization missing
        Server-->>Host: Reject JSON-RPC error
    else JWT invalid or wrong aud
        Server-->>Host: Reject
    else Valid
        Server-->>Host: Tool result
    end
```

**When to use / when not / failure modes**

- Use when the same team controls the MCP host/client **and** the MCP server and needs rotation, step-up, or per-call / multiplexed identity without respawn.
- Do not use with stock MCP host/clients. There is no normative `_meta.authorization`; third-party MCP host/clients will not send it.
- Do not attach the token only on `tools/call`. `tools/list`, prompts, resources, and notifications that authorize work must carry it or be rejected.
- Failures: forgotten `_meta` on one method → fail closed (good) or accidental anonymous path (bad if the filter is method-scoped); protocol logs, OpenTelemetry attributes, and debug traces capture the JWT; invented schema drift (`string` vs object, `Bearer ` prefix) between MCP host/client and MCP server.
- Version the field in the org profile. Still never log the raw token.

### 5. stdio — OS secret store / DPAPI

Shorter custom profile. The MCP host/client stores the JWT under an agreed name; the MCP server reads that credential, validates, binds. Cleaner than a raw env var on Windows.

```mermaid
sequenceDiagram
    participant Host as MCP host/client
    participant Store as OS secret store / DPAPI
    participant Server as MCP stdio server

    Host->>Store: Write named credential (JWT)
    Host->>Server: Spawn (name/hint, not raw token)
    Server->>Store: Read named credential
    Server->>Server: Validate JWT, set ClaimsPrincipal
    alt Missing name, ACL deny, or invalid JWT
        Server-->>Host: Fail closed
    else Valid
        Host->>Server: tools/call
        Server-->>Host: Tool result
    end
```

**When to use / when not / failure modes**

- Use on Windows when the MCP host/client owns login and the secret should stay out of the process environment table (Credential Manager / DPAPI, user- or machine-bound).
- Do not use as a portable interop default. macOS Keychain / libsecret variants need their own naming contract; stock MCP host/clients will not know the name.
- Still typically process-lifetime unless the MCP server re-reads on a schedule. Refresh usually means the MCP host/client overwrites the credential and the MCP server re-reads, or the process restarts.
- Failures: MCP host/client and MCP server disagree on credential name; ACL too broad (other local processes read it); leftover credentials after logout; treating DPAPI unwrap as validation (still run JWKS/`aud`/`exp`).
- Do not fall back to a plaintext file if the store is unavailable.

### 6. stdio — initialize bootstrap `_meta` then cache

Shorter custom hybrid. Token is attached once on `initialize`. The MCP server validates, caches the identity in memory, and uses it for later calls until expiry or process restart.

```mermaid
sequenceDiagram
    participant Host as MCP host/client
    participant Server as MCP stdio server

    Host->>Server: initialize plus _meta.authorization JWT
    Server->>Server: Validate once, cache ClaimsPrincipal
    alt Missing or invalid bootstrap token
        Server-->>Host: Reject initialize
    else Valid
        Server-->>Host: initialize result
        Host->>Server: tools/call (no token on later messages)
        Server->>Server: In-memory identity until exp or restart
        Server-->>Host: Tool result
        Note over Server: Expiry or reconnect requires a new initialize token
    end
```

**When to use / when not / failure modes**

- Use when the same team controls the MCP host/client and the MCP server, wants MCP host/client-owned login, and wants one attachment instead of per-message `_meta`.
- Do not use when tokens must rotate mid-process without a new `initialize`, or when a reconnect can skip `_meta` (easy to miss).
- Not interop. Same invented-schema cost as per-message `_meta`, with a longer cached lifetime.
- Failures: cached principal used after `exp` if the filter does not re-check lifetime; reconnect without `_meta` silently reuses a dead cache (must not); initialize payload logged → token leak; process dump contains the in-memory JWT.
- Cache must not be written to disk or logs. On expiry, fail closed until the MCP host/client re-bootstraps.

## Decision tree (pick one)

Use this to lock a v1 profile. Do not mix two deliveries in the same MCP server process unless there is an explicit migration story.

```mermaid
flowchart TD
    start["Need Entra JWT validated on the MCP server"]
    start --> httpQ{"Is MCP remote HTTP?"}
    httpQ -->|Yes| httpPath["HTTP Bearer: 401 plus PRM, auth code PKCE resource, JwtBearer"]
    httpQ -->|No - stdio| ownerQ{"Who owns Entra login?"}
    ownerQ -->|MCP host/client already has JWT and stays Entra client| hostNeed{"What does delivery need?"}
    hostNeed -->|Interop / stock MCP host/client config| envPath["stdio env MCP_ACCESS_TOKEN"]
    hostNeed -->|Rotation or per-call identity| metaPath["stdio _meta.authorization per message"]
    hostNeed -->|Cleaner than env on Windows| dpapiPath["stdio OS secret store / DPAPI"]
    ownerQ -->|MCP server should own login| uiQ{"UI or broker available?"}
    uiQ -->|Yes| msalPath["stdio embedded MSAL plus token cache"]
    uiQ -->|Headless / no UI| headless["No interactive MSAL: env or MCP host/client token, or device code if product accepts"]
```

Reading the tree:

- Remote HTTP is not a choice among stdio profiles. It is the official Bearer path.
- If the MCP host/client is already the Entra client application, keep it that way: env (interop), `_meta` (rotation), or DPAPI (Windows secret store). Bootstrap-`initialize`-then-cache is a hybrid of `_meta` if one attachment only is enough.
- If the MCP server should own login, embed MSAL. That requires a human or broker. Headless CI/SSH must not sit on an interactive prompt; take a token issued to the MCP host/client (env/DPAPI/`_meta`) or an explicit device-code profile.
- v1 default when the MCP host/client already has a JWT: **env var at spawn**.

<a id="security-best-practices-role-mapped"></a>

## Security best practices (2026-07-28) — role-mapped

Source: [MCP Security Best Practices (2026-07-28 tutorial)](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices). That page sits **beside** Authorization: Authorization is how tokens are obtained and checked; the best-practices page is the attack catalog and the matching mitigations.

Actors used below (defined once):

| Actor | Role here |
| --- | --- |
| **MCP host/client** | The IDE / agent that speaks MCP and (usually) talks to Entra |
| **MCP server** | The C# resource this ADR designs (HTTP and/or stdio) |
| **Entra** | The authorization server that issues the JWT |
| **Third-party API** | Some other resource (Graph, an internal API) that is **not** this MCP server |

Terminology in this section is strict: **MCP server** or **MCP host/client** only. Entra may be called Entra or the authorization server.

### Confused deputy

**Attack.** An MCP server that is also an OAuth proxy to a third-party API uses one static client ID at that third-party authorization server. After a real user consents once, a cookie says “already approved.” A malicious MCP host/client then dynamically registers, sends the user a crafted link, the third-party authorization server skips consent, and the authorization code is redirected to the attacker. The attacker ends up with tokens as that user.

**Mitigation.** The MCP server must show its own per-MCP-host/client consent **before** forwarding to the third-party authorization server; store that consent; exact-match `redirect_uri`; CSRF / `state` only after consent; no skip.

**Applies to us.** **MCP server only if** that MCP server is an OAuth proxy to a third-party API. **N/A** for this ADR’s default: a plain Entra JWT resource server that does not implement `/authorize` or `/token`.

### Token passthrough

**Attack.** The MCP server accepts a token minted for some other API (Graph, another MCP server) and either honors it locally or forwards it upstream. Stolen or “wrong audience” tokens work. Logs lie about who called whom. Downstream rate limits and audit assume the token was issued for that API, not laundered through this MCP server.

**Mitigation.** The MCP server **MUST** accept only tokens issued for this MCP server (`aud` pin). If a third-party API must be called, obtain a **separately issued** upstream token. Never transit a Graph JWT (or any other-API JWT).

**Applies to us.** **MCP server MUST.** This is the core of this ADR for HTTP Bearer and every stdio delivery. Reject Graph / other-API `aud`; do not pass the inbound Entra JWT to an upstream API.

### Server-Side Request Forgery (SSRF)

**Attack.** While discovering OAuth metadata, the MCP host/client fetches URLs the MCP server (or a CIMD document) chose: `resource_metadata` from `WWW-Authenticate`, `authorization_servers` from PRM, then token/authorize endpoints from authorization-server metadata. A malicious MCP server points those at cloud metadata (`169.254.169.254`), RFC1918, localhost services, or a redirect chain. The MCP host/client becomes an egress proxy past the perimeter.

**Mitigation.** The MCP host/client must treat those URLs as untrusted: HTTPS in production (http only on loopback in development), block private / link-local IPs (do not hand-roll parsers), do not blindly follow redirects, prefer an egress proxy. The same rules apply if Entra fetches CIMD URLs.

**Applies to us.** **MCP host/client** (and Entra if CIMD). Not an MCP server JWT-validation duty.

### State-handle hijacking

**Attack.** MCP has no protocol-level sessions. The MCP server mints a cart / workflow handle and later takes it as a tool argument. An attacker who guesses or steals the handle calls the same tool. If the MCP server does not check that the handle belongs to the authenticated principal, the attacker operates on someone else’s state.

**Mitigation.** Possession of a handle is not authentication. Bind handles on the MCP server to the user id from the **verified** JWT; use unguessable, expiring values; reject a handle presented by any other principal.

**Applies to us.** **MCP server if** tools keep stateful handles. **N/A** if tools are stateless.

### Local MCP server compromise

**Attack.** A local MCP server runs on the same machine as the MCP host/client. A malicious startup command, a malicious binary, or DNS rebinding to a localhost HTTP MCP server gives the attacker the MCP host/client’s privileges (exfil, arbitrary code, data loss).

**Mitigation.** The MCP host/client must show the **exact** command (no truncation) and get consent before one-click install; sandbox; least privilege. An MCP server meant to run locally should prefer stdio (only that MCP host/client is on the pipe). If the MCP server speaks HTTP on localhost, still require a token and restrict bind / IPC.

**Applies to us.** **MCP host/client** owns consent and sandbox. **MCP server:** prefer stdio; if local HTTP, still require the Entra JWT (already this ADR).

### OAuth authorization URL validation

**Attack.** A malicious MCP server returns an authorization URL with a `javascript:` or `data:` scheme, or a string that a shell will interpret as extra commands. The MCP host/client opens it with `window.open` or `cmd.exe` / `sh` / PowerShell and gets XSS or RCE. Combined with a stdio gateway, that can escalate to full machine compromise.

**Mitigation.** The MCP host/client **MUST** allow only `http` / `https` (http only on loopback in development); **MUST** reject `javascript:`, `data:`, `file:`, `vbscript:`; **MUST NOT** open URLs via a shell; sanitize. Web MCP host/clients SHOULD use a tight CSP.

**Applies to us.** **MCP host/client.** Not the C# MCP server resource-server path.

### stdio transport in proxy scenarios

**Attack.** The stdio transport itself is not the bug. The attack applies **only** when a local gateway / proxy spawns stdio MCP servers as child processes. XSS or a stolen proxy token lets an attacker ask that gateway to spawn arbitrary commands with the user’s privileges.

**Mitigation.** Stop the XSS / authorization-URL class first. Then sandbox spawned processes, least-privilege the gateway, and log stdio spawn.

**Applies to us.** **Only if** a local gateway spawns stdio MCP servers. Direct MCP host/client → stdio MCP server is out of scope for this attack.

### Mix-up attacks

**Attack.** The MCP host/client talks to many authorization servers over its lifetime. A malicious one tricks the MCP host/client into sending an honest Entra authorization code (or token) to the attacker’s token endpoint. PKCE does not stop this: the `code_verifier` is sent to the attacker. Resource indicators do not help if the attacker intercepts before honest Entra.

**Mitigation.** The MCP host/client records which authorization server it started with and validates `iss` in the authorization response (RFC 9207). Honest authorization servers must emit `iss`. This mitigation does nothing against an honest authorization server that omits `iss`.

**Applies to us.** **MCP host/client + `iss` validation.** Entra v2.0 typically identifies the issuer; the MCP server still independently checks the JWT `iss`.

### Localhost redirect URI impersonation

**Attack.** A native MCP host/client uses a localhost redirect. An attacker presents the legitimate CIMD `client_id` (the consent screen looks like the real app) and binds a different localhost port. The user approves the familiar name; the authorization code lands on the attacker.

**Mitigation.** Entra / the authorization server should warn on localhost-only redirect URIs and show the redirect hostname. MCP host/client registration must use exact redirect URIs.

**Applies to us.** **Entra + MCP host/client registration.** The MCP server as a resource server does not see this redirect.

### CIMD trust policies

**Attack.** If Entra (or another authorization server) accepts any HTTPS Client ID Metadata Document, a phishing MCP host/client can look legitimate. Fetching a CIMD URL is also an SSRF input at the authorization server.

**Mitigation.** Domain allowlists, reputation, age / cert checks; show CIMD hostnames in the consent UI. The authorization server keeps full control of that policy.

**Applies to us.** **Only if using CIMD.** The C# MCP server as a resource server typically does not consume CIMD. This is an Entra / platform registration choice.

### Scope minimization

**Attack.** The MCP server advertises every scope in `scopes_supported`; the MCP host/client asks for all of them up front. A stolen token with omnibus scopes (`files:*`, `admin:*`) unlocks the whole surface. Revoke is painful; audit is noisy; users abandon huge consent dialogs.

**Mitigation.** Progressive, least-privilege scopes. The MCP server emits a precise 403 `scope=` challenge, not the full catalog; accepts down-scoped tokens; authorizes the **operation**, not merely “a scope claim exists.” The MCP host/client starts with a baseline and steps up; unions previously granted scopes on re-auth. Do not publish wildcard / omnibus scopes.

**Applies to us.** **MCP server + MCP host/client.** This ADR already maps 401 vs 403 vs `scp` / `roles`; do not publish omnibus scopes in PRM.

### Cheat sheet

| Topic | MCP server | MCP host/client |
| --- | --- | --- |
| Confused deputy | Only if OAuth proxy to a third-party API; **N/A** as a plain Entra JWT resource server | — |
| Token passthrough | **MUST**; core of this ADR (`aud` pin; no Graph transit) | Do not send a token issued for another API |
| SSRF | — (unless this process fetches CIMD as an authorization server) | Treat discovery URLs as untrusted (HTTPS, no private IPs, no blind redirects) |
| State-handle hijacking | If stateful tools: bind handle to the JWT principal | — |
| Local MCP server compromise | Prefer stdio; if local HTTP, still require a token | Consent + sandbox before launch |
| OAuth authorization URL validation | — | Allow only `http`/`https`; never open via a shell |
| stdio in proxy scenarios | Only if this process is a gateway that spawns stdio MCP servers | Isolate / sandbox that gateway |
| Mix-up attacks | Still validate JWT `iss` | Validate `iss` on the authorization response (RFC 9207) |
| Localhost redirect impersonation | — | Exact redirect URI registration; Entra must show the redirect hostname |
| CIMD trust policies | N/A as a resource server | Relevant if the platform uses CIMD at Entra |
| Scope minimization | Precise 403 challenges; no omnibus scopes; authorize operations | Baseline + step-up; do not request the full catalog first |

Authorization is how tokens work. Best practices are how attackers abuse that. This ADR owns **MCP server** duties and notes **MCP host/client** duties for the platform.

## Consequences

### Positive

- Clear split: HTTP = normative MCP OAuth; stdio = delivery profile + same validation
- Avoids false sense of security from “local process = trusted”
- Gives product a menu without pretending custom channels are protocol-mandated
- Flows + decision tree are self-contained: an implementer can pick a delivery and code the bind/`ClaimsPrincipal` path without another design thread

### Negative / risks

- Env default needs careful inheritance and no argv/logging
- `_meta` or file/IPC profiles fragment MCP host/client interoperability
- Interactive MSAL will stall MCP host/clients if `initialize` waits on a popup; headless must fail fast instead
- Entra `resource` URI vs Application ID URI `aud` friction remains a separate design decision (see [research note — Entra mapping](../2026-09-06-mcp-csharp-entra-auth-design.md#entra-mapping)): RFC 8707 wants the MCP HTTP URI in `resource` / `aud`; Entra `aud` is usually the Application ID URI (`api://<app-id>`) or the application (client) ID GUID. Either set the Entra Application ID URI **equal** to the MCP canonical resource URI, or document that the MCP server validates Entra `aud` while PRM `resource` remains the HTTP URI. Do not silently accept both plus Graph (`00000003-0000-0000-c000-000000000000`).

### Follow-ups

- Lock v1 with the [decision tree](#decision-tree-pick-one) (still recommend env `MCP_ACCESS_TOKEN`) and document the C# `ClaimsPrincipal` filter
- If the MSAL profile is chosen: login tool or first-need auth; never block `initialize` indefinitely; headless → device code or a token from the MCP host/client
- If `_meta.authorization` is chosen: publish the org schema and require it on every JSON-RPC method, not only `tools/call`
- Align Entra App ID URI with MCP canonical resource URI where possible
- Private audit schema (SEP-3004 not Final)

## References

- [MCP Authorization (2026-07-28)](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [MCP authorization tutorial (2026-07-28)](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/authorization)
- [Authorization security considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations)
- [Security best practices (specification)](https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices)
- [Security best practices tutorial (2026-07-28)](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)
- [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [stdio](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio)
- [RFC 9728 — OAuth 2.0 Protected Resource Metadata](https://www.rfc-editor.org/rfc/rfc9728)
- [RFC 8707 — Resource Indicators for OAuth 2.0](https://www.rfc-editor.org/rfc/rfc8707)
- [research/2026-09-06-mcp-csharp-entra-auth-design.md](../2026-09-06-mcp-csharp-entra-auth-design.md)
- [C# SDK identity and role propagation](https://github.com/modelcontextprotocol/csharp-sdk/blob/main/docs/concepts/identity/identity.md)
- [ProtectedMcpServer sample](https://github.com/modelcontextprotocol/csharp-sdk/tree/main/samples/ProtectedMcpServer)
