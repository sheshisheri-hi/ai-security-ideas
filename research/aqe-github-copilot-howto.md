# AQE + GitHub Copilot: what the code actually installs

**Date:** 2026-09-02 (overnight pass, from AQE source on `main` @ `38523b9`, v3.14.0)
**Question:** Does `aqe init --auto --with-copilot` give you QE **test creation** with Copilot Enterprise, or only a README row?

Verdict up front: **Copilot is a real AQE platform in code.** It is also the **thinnest** of the 11. You get an MCP server + a markdown instruction file. You do **not** get the 60 QE agents, slash commands, or a Copilot custom mode. Test **creation** is one MCP tool: `test_generate_enhanced`. Copilot may or may not call it.

This is not an implementation. Private notes only.

---

## What `--with-copilot` writes (from source)

Installer: [`src/init/copilot-installer.ts`](https://github.com/proffesor-for-testing/agentic-qe/blob/main/src/init/copilot-installer.ts)

Templates: [`src/init/platform-config-generator.ts`](https://github.com/proffesor-for-testing/agentic-qe/blob/main/src/init/platform-config-generator.ts)

Wired from: `src/init/phases/09-assets.ts` (also `aqe platform setup copilot`).

Copilot registry entry (quoted):

```
copilot: {
  configPath: '.vscode/mcp.json',
  configKey: 'servers',
  rulesPath: '.github/copilot-instructions.md',
  supportsAutoApprove: false,
  supportsCustomModes: false,
}
```

Exactly **two Copilot files**. Nothing else (no `.github/agents`, no prompt files, no VS `.mcp.json`, no custom Copilot agent).

**Important:** `--with-copilot` is **additive**. Default `aqe init --auto` still installs the Claude Code surface (`.claude/`, root `.mcp.json` with `mcpServers` + `aqe-mcp`, `CLAUDE.md`) unless you pass `--no-claude` (issue [#532](https://github.com/proffesor-for-testing/agentic-qe/issues/532)).

Auto-detect in `09-assets.ts`: if the repo already has a `.vscode/` folder, `--auto` writes the Copilot files **even without** `--with-copilot`.

### 1. `.vscode/mcp.json`

```json
{
  "servers": {
    "agentic-qe": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "agentic-qe@latest", "mcp"],
      "env": {
        "AQE_MEMORY_PATH": ".agentic-qe/memory.db",
        "AQE_V3_MODE": "true"
      }
    }
  }
}
```

Notes from code:

- Copilot is the only platform that uses the key `servers` plus `"type": "stdio"`. Everyone else uses `mcpServers`.
- Launch is `npx -y agentic-qe@latest mcp` (hits npm unless you already have it cached).
- `--no-database` / memory backend swaps env to `AQE_MEMORY_BACKEND=memory` and drops `AQE_MEMORY_PATH`.
- `supportsAutoApprove: false` so Copilot does **not** get Cline's `alwaysAllow` list. You must click Allow on tools.
- Merge on overwrite: other MCP servers in that file are kept.

### 2. `.github/copilot-instructions.md`

Same markdown blob used for Cursor/Windsurf (`AQE_RULES_CONTENT`). It tells Copilot:

- Always call `fleet_init` before other AQE tools
- Use `test_generate_enhanced` for test generation (unit / integration / e2e)
- Use `coverage_analyze_sublinear`, `quality_assess`, `security_scan_comprehensive`, `defect_predict`, `memory_*`
- Test pyramid 70/20/10, AAA, one assertion, mock at boundaries

That file is **behavioral rules**, not executable skills. Copilot reads it as repo instructions. It does not install 60 agents. Copilot is not required to call those tools.

---

## Copilot vs Claude vs Cline (the README table is uneven)

| What | Claude Code (`aqe init --auto`) | Cline / Kilo / Roo (`--with-cline` etc.) | GitHub Copilot (`--with-copilot`) |
| --- | --- | --- | --- |
| MCP server | Yes | Yes | Yes, same `aqe mcp` |
| 60 QE agents + skills | Yes (`.claude/agents/v3/`) | No | No |
| Slash commands `/aqe-generate` | Plugin only | No | No |
| Custom "QE Engineer" mode | Claude agents | Yes (`supportsCustomModes: true`) | **No** (`false` in registry) |
| Auto-approve QE tools | Claude policy | `alwaysAllow` includes `test_generate_enhanced` | **No** |
| Instruction file | CLAUDE.md | custom mode JSON | `.github/copilot-instructions.md` |

So: Copilot **does** support AQE QE **if** Agent mode is on and Copilot **chooses** to call MCP tools. It is not the Claude fleet with a Copilot skin.

VS Code Copilot is the only Copilot target AQE implements. Visual Studio is an accidental consumer of `.vscode/mcp.json`. The github.com coding agent is not configured (`no copilot-setup-steps.yml`, no repo Settings MCP JSON).

---

## Which MCP tools **create** tests

Live `tools/list` is **~86–88** tools (`registerAllTools` + `qe-tool-bridge.ts`). The 40 in `docs/tool-compatibility.md` is a stale OpenCode display doc (2026-02-24).

**Create / design**

- `test_generate_enhanced` — generate test **code** (the QE creation tool Copilot should call)
- `requirements_validate` — generate BDD scenarios from requirements
- `code_index` — index first so generation has context

**Not creation (run / score)**

- `test_execute_parallel` — run tests
- `coverage_analyze_sublinear` — find holes (then you still need generate)
- `quality_assess`, `security_scan_comprehensive`, `defect_predict`, `chaos_test`, `accessibility_test`

Exact `test_generate_enhanced` parameters from `src/mcp/protocol-server.ts`:

| Param | Notes |
| --- | --- |
| `sourceCode` | source text |
| `filePath` | import target in generated tests |
| `language` | enum: typescript, javascript, python, java, **csharp**, go, rust, swift, kotlin, dart. **No c, cpp, fortran.** |
| `testType` | unit, integration, e2e |
| `framework` | includes **xunit**, **nunit**. No GoogleTest, Catch2, MSTest, pFUnit |
| `coverageGoal` | default 80 |
| `aiEnhancement` | default true |
| `detectAntiPatterns` | default false |

Example in the tool description is JavaScript/jest. Copilot will copy that shape unless you name csharp/xunit.

**Schema vs typed path mismatch:** Copilot's flattened tool (`test_generate_enhanced` → `handleTestGenerate` → domain factory) advertises csharp. The older MCP class `src/mcp/tools/test-generation/generate.ts` (`qe/tests/generate`) types language as only `typescript | javascript | python | java | go` and framework as `jest | vitest | mocha | pytest | node-test`. If Copilot hits the bridged `qe/tests/generate` name, csharp can be dropped. Prefer the flattened name and say `language=csharp` in the prompt.

Issue [#535](https://github.com/proffesor-for-testing/agentic-qe/issues/535) (open): `test_generate_enhanced` can report success and still emit poor/incorrect tests, even on trivial JS.

C/C++/Fortran/WPF stay out. Copilot does not magically add them.

---

## How to use it with Copilot Enterprise (office)

Enterprise/Business: **MCP servers in Copilot is disabled by default.** Org admin must enable it (AI controls → MCP). If an allowlist exists, allow stdio `npx` args `-y agentic-qe@latest mcp`. Registry-only mode blocks AQE (not a GitHub gallery server). VS Code device policy `ChatMCP` can independently set `off` / `registryOnly` / `allowed`.

### A. VS Code (the path AQE actually generates)

1. Org Copilot MCP policy on (see above).
2. Node 18+ on PATH.
3. In the repo (skip the Claude dump):

```powershell
npm install -g agentic-qe
cd C:\path\to\repo
aqe init --auto --with-copilot --no-claude
aqe platform verify copilot
```

4. Open the folder in **VS Code**. Command Palette → MCP: List Servers → start `agentic-qe` → trust it.
5. Copilot Chat → **Agent** mode, not Ask. Enable `agentic-qe` tools (they start disabled).
6. First prompt (creation, not run):

```
Use the agentic-qe MCP tools. First call fleet_init.
Then call test_generate_enhanced with language=csharp, framework=xunit, testType=unit,
filePath set to this C# file, and sourceCode = the file contents.
Write the returned testCode to disk. Do not invent tests without that tool.
Do not execute tests.
```

7. Allow the tool when Copilot asks.

If Copilot writes tests **without** calling AQE, it is using Copilot's own generator. That is fine. AQE only ran if you see a tool call to `test_generate_enhanced`.

### B. Visual Studio 2022 17.14+ / VS 2026

AQE does **not** write `<solution>\.mcp.json` or `.vs\mcp.json`. It only writes `.vscode/mcp.json`. Visual Studio **does** read `.vscode/mcp.json` (4th discovery path in Microsoft docs), so the same init often works.

Do **not** point VS Copilot at Claude's root `.mcp.json` (`mcpServers` + `aqe-mcp`). That is a different schema.

Node/`npx` must be on **Visual Studio's** PATH (the VS Developer environment is not your user PATH).

Safer: copy the Copilot JSON to `<solution>\.mcp.json` (source-controlled) or `%USERPROFILE%\.mcp.json`.

Then: Copilot Chat → Agent → enable tools → same prompt as above.

`@Test` in VS 2026 is **Copilot's** C# generator, not AQE. You can use both. They are not the same pipeline.

### C. GitHub.com coding agent

`--with-copilot` does **not** write repo Settings → Copilot → MCP, `.github/agents/`, or `copilot-setup-steps.yml`. UNKNOWN whether that agent loads `.vscode/mcp.json`. Do not count on MCP there.

`.github/copilot-instructions.md` **does** apply as GitHub custom instructions, so the cloud agent may follow QE *prose* without AQE tools.

### D. Command line, no IDE

```powershell
aqe test generate --file path\to\Service.cs --framework xunit --type unit
```

That is AQE without Copilot. Copilot is not involved.

---

## Does this give you QE engineering?

**With Copilot as host, AQE can create tests** via `test_generate_enhanced`, plus coverage/quality/security tools if Copilot calls them. That is a toolbox, not the 60-agent fleet.

**What you do not get on Copilot:** the 60 named QE agents, `/aqe-generate`, a locked QE mode, auto-approved tools, github.com coding-agent MCP, or a Visual Studio installer.

**For this team's simulator:** Copilot+AQE can attempt **C# xUnit/NUnit-shaped** tests if Agent mode actually invokes `test_generate_enhanced`. Quality is unproven ([#535](https://github.com/proffesor-for-testing/agentic-qe/issues/535)). It will not create GoogleTest, Catch2, pFUnit, or WPF UI tests. Copilot `@Test` / `/tests` still creates C#/C++ tests without AQE, and that is the office-tomorrow path.

If org MCP policy is off (the Enterprise default), AQE's `.vscode/mcp.json` is inert.

---

## Copy-paste check after init

```powershell
aqe platform list
aqe platform verify copilot
# expect .vscode/mcp.json and .github/copilot-instructions.md present
Get-Content .vscode\mcp.json
Get-Content .github\copilot-instructions.md
```

In Copilot Agent, you should see an MCP server named `agentic-qe`. If you only see Copilot built-in tools, org MCP policy is off or VS/VS Code did not load `.vscode/mcp.json`.

---

## Known Copilot-path issues (AQE tracker)

| Issue | State | Why it matters |
| --- | --- | --- |
| [#532](https://github.com/proffesor-for-testing/agentic-qe/issues/532) | open | `--with-copilot` still dumps Claude unless `--no-claude` |
| [#533](https://github.com/proffesor-for-testing/agentic-qe/issues/533) | open | db-free env; Copilot path currently honors `memoryBackend` |
| [#528](https://github.com/proffesor-for-testing/agentic-qe/issues/528) | open | stdio MCP used to drop stdin (would kill Copilot). PR #543 claims a fix |
| [#535](https://github.com/proffesor-for-testing/agentic-qe/issues/535) | open | generator reports success with bad test code |
| PR [#313](https://github.com/proffesor-for-testing/agentic-qe/pull/313) | merged | original Copilot adapter (v3.7.4) |
| PR [#543](https://github.com/proffesor-for-testing/agentic-qe/pull/543) | merged | Copilot included in db-free env sweep |

No AQE issue tracks Visual Studio, github.com coding agent, or `.github/agents`.

---

## Sources (code)

- `src/init/copilot-installer.ts`
- `src/init/platform-config-generator.ts` (`PLATFORM_REGISTRY.copilot`, `AQE_RULES_CONTENT`, `getMcpServerEntry`)
- `src/init/phases/09-assets.ts` (auto-detect `.vscode/`)
- `src/cli/handlers/init-handler.ts` (`--with-copilot`, `--no-claude`)
- `src/mcp/protocol-server.ts` (`test_generate_enhanced` schema)
- `src/mcp/tools/test-generation/generate.ts` (narrower typed union; no csharp)
- `src/mcp/qe-tool-bridge.ts`
- `docs/platform-setup-guide.md`
- `docs/tool-compatibility.md` (stale 40-tool count)
- Microsoft: [MCP in Visual Studio](https://learn.microsoft.com/en-us/visualstudio/ide/mcp-servers?view=visualstudio)
