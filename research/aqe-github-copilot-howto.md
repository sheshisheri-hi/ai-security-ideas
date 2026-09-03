# AQE + GitHub Copilot: what the code actually installs

**Date:** 2026-09-02 (overnight pass, from AQE source on `main` @ `38523b9`, v3.14.0)
**Question:** Does `aqe init --auto --with-copilot` give you QE **test creation** with Copilot Enterprise, or only a README row?

Verdict up front: **Copilot is a real AQE platform in code.** It is also the **thinnest** of the 11. You get an MCP server + a markdown instruction file. You do **not** get the 60 QE agents, slash commands, or a Copilot custom mode. Test **creation** is one MCP tool: `test_generate_enhanced`. Copilot may or may not call it.

This is not an implementation. Private notes only.

---

## What `--with-copilot` writes (from source)

Installer: [`src/init/copilot-installer.ts`](https://github.com/proffesor-for-testing/agentic-qe/blob/main/src/init/copilot-installer.ts)

Templates: [`src/init/platform-config-generator.ts`](https://github.com/proffesor-for-testing/agentic-qe/blob/main/src/init/platform-config-generator.ts)

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

Exactly **two files**. Nothing else (no `.github/agents`, no prompt files, no VS `.mcp.json`, no custom Copilot agent).

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

### 2. `.github/copilot-instructions.md`

Same markdown blob used for Cursor/Windsurf (`AQE_RULES_CONTENT`). It tells Copilot:

- Always call `fleet_init` before other AQE tools
- Use `test_generate_enhanced` for test generation (unit / integration / e2e)
- Use `coverage_analyze_sublinear`, `quality_assess`, `security_scan_comprehensive`, `defect_predict`, `memory_*`
- Test pyramid 70/20/10, AAA, one assertion, mock at boundaries

That file is **behavioral rules**, not executable skills. Copilot reads it as repo instructions. It does not install 60 agents.

---

## Copilot vs Claude vs Cline (the README table is uneven)

| What | Claude Code (`aqe init --auto`) | Cline / Kilo / Roo (`--with-cline` etc.) | GitHub Copilot (`--with-copilot`) |
| --- | --- | --- | --- |
| MCP server | Yes | Yes | Yes, same `aqe mcp` |
| 60 QE agents + skills | Yes (`.claude/`) | No | No |
| Slash commands `/aqe-generate` | Plugin only | No | No |
| Custom "QE Engineer" mode | Claude agents | Yes (`supportsCustomModes: true`) | **No** (`false` in registry) |
| Auto-approve QE tools | Claude policy | `alwaysAllow` includes `test_generate_enhanced` | **No** |
| Instruction file | CLAUDE.md | custom mode JSON | `.github/copilot-instructions.md` |

So: Copilot **does** support AQE QE **if** Agent mode is on and Copilot **chooses** to call MCP tools. It is not the Claude fleet with a Copilot skin.

---

## Which MCP tools **create** tests

From [`docs/tool-compatibility.md`](https://github.com/proffesor-for-testing/agentic-qe/blob/main/docs/tool-compatibility.md) and [`src/mcp/protocol-server.ts`](https://github.com/proffesor-for-testing/agentic-qe/blob/main/src/mcp/protocol-server.ts) (40+ tools, same server Copilot attaches to):

**Create / design**

- `test_generate_enhanced` — generate test **code** (the QE creation tool)
- `requirements_validate` — generate BDD scenarios from requirements
- `code_index` — index first so generation has context

**Not creation (run / score)**

- `test_execute_parallel` — run tests
- `coverage_analyze_sublinear` — find holes (then you still need generate)
- `quality_assess`, `security_scan_comprehensive`, `defect_predict`, `chaos_test`, `accessibility_test`

Exact `test_generate_enhanced` parameters Copilot will see:

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

`test_generate_enhanced` still only emits those languages. C/C++/Fortran/WPF stay out. Copilot does not magically add them.

---

## How to use it with Copilot Enterprise (office)

### A. VS Code (what AQE's installer targets)

1. Org Copilot policy: **MCP servers in Copilot = on**. If this is off, Agent mode never sees AQE.
2. Node 18+ on PATH.
3. In the repo:

```powershell
npm install -g agentic-qe
cd C:\path\to\repo
aqe init --auto --with-copilot
aqe platform verify copilot
```

4. Open the folder in **VS Code** (or Cursor). Copilot Chat → **Agent** mode, not Ask.
5. Tools picker: enable `agentic-qe` tools. They start disabled.
6. First prompt (creation, not run):

```
Use AQE. Call fleet_init, then test_generate_enhanced for src/UnitConversion.cs.
Language csharp, framework xunit, testType unit.
Do not execute tests. Write the files.
```

7. Allow the tool when Copilot asks.

If Copilot writes tests **without** calling AQE, it is using Copilot's own generator. That is fine. AQE only ran if you see a tool call to `test_generate_enhanced`.

### B. Visual Studio 2022 17.14+ / VS 2026

AQE does **not** write `<solution>\\.mcp.json`. It only writes `.vscode/mcp.json`. Visual Studio **does** read `.vscode/mcp.json` (4th discovery path in Microsoft docs), so the same init often works.

Safer for VS: copy the same JSON to `<solution>\\.mcp.json` (source-controlled) or `%USERPROFILE%\\.mcp.json` (all solutions).

Then: Copilot Chat → Agent → enable tools → same prompt as above.

`@Test` in VS 2026 is **Copilot's** C# generator, not AQE. You can use both. They are not the same pipeline.

### C. GitHub.com coding agent

`--with-copilot` does **not** configure the cloud coding agent. No `copilot-setup-steps.yml`. UNKNOWN whether that agent loads `.vscode/mcp.json`. Do not count on it.

### D. Command line, no IDE

```powershell
aqe test generate --file path\\to\\Service.cs --framework xunit --type unit
```

That is AQE without Copilot. Copilot is not involved.

---

## Does this give you QE engineering?

**With Copilot as host, AQE can create tests** via `test_generate_enhanced`, plus coverage/quality/security tools if Copilot calls them. That is real QE *tooling*.

**What you do not get on Copilot:** the 60 named QE agents, `/aqe-generate`, a locked QE mode, auto-approved tools, or language support beyond AQE's generator list.

**For this team's simulator:** Copilot+AQE can create **C# xUnit-shaped** tests if Agent mode actually invokes the tool. It will not create GoogleTest, Catch2, pFUnit, or WPF UI tests. Copilot `@Test` / `/tests` still creates C#/C++ tests without AQE.

---

## Copy-paste check after init

```powershell
aqe platform list
aqe platform verify copilot
# expect .vscode/mcp.json and .github/copilot-instructions.md present
Get-Content .vscode\\mcp.json
Get-Content .github\\copilot-instructions.md
```

In Copilot Agent, you should see an MCP server named `agentic-qe`. If you only see Copilot built-in tools, org MCP policy is off or VS/VS Code did not load `.vscode/mcp.json`.

---

## Sources (code)

- `src/init/copilot-installer.ts`
- `src/init/platform-config-generator.ts` (`PLATFORM_REGISTRY.copilot`, `AQE_RULES_CONTENT`, `getMcpServerEntry`)
- `src/cli/handlers/init-handler.ts` (`--with-copilot`)
- `src/mcp/protocol-server.ts` (`test_generate_enhanced` schema)
- `docs/platform-setup-guide.md`
- `docs/tool-compatibility.md`
- Microsoft: [MCP in Visual Studio](https://learn.microsoft.com/en-us/visualstudio/ide/mcp-servers?view=visualstudio)
