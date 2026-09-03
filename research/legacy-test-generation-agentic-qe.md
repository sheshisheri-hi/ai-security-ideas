# Legacy engineering test generation: can Agentic-QE help, and what to use tomorrow

**Date:** 2026-09-02
**Status:** ad-hoc research (not a daily idea candidate)
**Audience:** office trial on decades-old engineering simulation software
**Stack assumed:** C, C++, C#, Fortran numerical kernels, WPF desktop UI, Visual Studio / MSBuild / CMake mix, GitHub Copilot Enterprise
**Source evaluated:** [proffesor-for-testing/agentic-qe](https://github.com/proffesor-for-testing/agentic-qe) at commit `38523b9`, npm package `agentic-qe@3.14.0`, MIT license

This note stays in the private `ai-security-ideas` repo. It is not an implementation.

---

## Direct answer

**Do not make Agentic-QE (AQE) the primary test generator for this codebase.**

| Layer | Can AQE generate missing tests? | What to use instead tomorrow |
| --- | --- | --- |
| C# business / solver wrappers / ViewModels | Partial. C# is a first-class generation language (`xunit` / `nunit`). WPF UI is not. | GitHub Copilot testing for .NET (`@Test` in Visual Studio 2026 18.3+) |
| C / C++ kernels | No as a test-gen language. Files can be indexed for search. No GoogleTest / Catch2 / CTest generator. | Copilot Chat in VS + GoogleTest or Catch2, coverage with OpenCppCoverage / VS coverage |
| Fortran solvers | No. Zero real Fortran hits in the repo (false positives on the word "transform"). | Characterization tests (golden files) + [pFUnit](https://github.com/Goddard-Fortran-Ecosystem/pFUnit) + Copilot Chat |
| WPF desktop UI | No. No WPF, FlaUI, WinAppDriver, or coded-UI mention. | Unit-test ViewModels first. Later FlaUI / Appium Windows for UI. |
| Coverage-gap hunting across a mixed native/.NET solution | Weak. AQE's coverage story is built around JS/Python/Java-style reports, not `gcov` / VS coverage / Intel coverage. | VS Code Coverage / OpenCppCoverage / llvm-cov, then ask Copilot to fill listed gaps |

AQE **can** be wired into GitHub Copilot as an MCP server. That is an IDE plugin, not a replacement for Copilot Enterprise. AQE still needs its **own** LLM keys (Anthropic, OpenAI, Azure OpenAI, Gemini, Ollama). It does not call GitHub Copilot as the model.

If you try AQE at all, try it later on a **small isolated C# class library**, with `AQE_FREE_TIER=1` + local Ollama, never against the full solver tree on day one.

---

## What AQE actually is

AQE is a Node.js (>=18) CLI + MCP server marketed as an "Agentic Quality Engineering Fleet."

- Install: `npm install -g agentic-qe` then `aqe init --auto`
- Binaries: `aqe`, `agentic-qe`, `aqe-mcp`
- Claimed surface: 60 QE agents, coverage-gap analysis, flaky-test detection, pattern learning
- First-class host: Claude Code. Copilot, Cursor, and others are MCP clients of the same server.
- README test-framework list: Jest, Vitest, Playwright, Cypress, pytest, JUnit, Go, Rust, Swift, Flutter
- Windows: runs, but native deps (`hnswlib-node`, `@ruvector/gnn`) often fail to build. Fallback is a JS brute-force index, documented as unsuitable for large codebases. Native build needs Python 3 + VS 2022 C++ build tools.

It is a real npm package (MIT, active changelog, v3.14.0). It also grew up around JS/TS/Python/Java agent workflows, not around MSVC + Fortran + WPF.

---

## Language matrix (from AQE source, not marketing)

Canonical types live in [`src/shared/types/test-frameworks.ts`](https://github.com/proffesor-for-testing/agentic-qe/blob/main/src/shared/types/test-frameworks.ts).

**Supported for test generation (`SupportedLanguage`):**

`typescript`, `javascript`, `python`, `java`, `csharp`, `go`, `rust`, `swift`, `kotlin`, `dart`

**Not in that list:** `c`, `cpp`, `fortran`, WPF/XAML

MCP tool `test_generate_enhanced` language enum (issue #474, `src/mcp/protocol-server.ts`):

`typescript | javascript | python | java | csharp | go | rust | swift | kotlin | dart`

Framework enum includes `xunit` and `nunit` for C#. It does **not** include GoogleTest, Catch2, Boost.Test, CTest, pFUnit, or FRUIT. MSTest appears only as a filename suffix in a couple of maps, not as a first-class generator.

C# detector (`src/shared/language-detector.ts`) looks at `*.csproj` in the **project root only** (non-recursive) for the string `NUnit`, else defaults to xUnit. A typical Visual Studio solution with tests in a child folder will often miss NUnit and emit xUnit.

Their own plan doc (`docs/multi-language-goap-implementation-plan.md`) still lists a dedicated NUnit generator as **future / low**, "Dedicated generator instead of xUnit alias." So C# output is real but thinner than the README implies.

C/C++ file extensions (`.c`, `.h`, `.cpp`, `.hpp`, `.cc`) **are** scanned by code-intelligence indexing (`src/cli/utils/file-discovery.ts`, `code-intelligence-handlers.ts`). That is search/graph, not test generation.

Fortran extensions (`.f`, `.f90`, `.f95`, `.for`, `.F90`): not in the detector.

WPF / XAML / `System.Windows`: no hits.

---

## Fit to a decades-old engineering simulator

This kind of product usually has four test problems, not one:

1. **Numerical kernels** (C/C++/Fortran): golden-file / characterization tests beat "generated unit tests." A generated `EXPECT_EQ` on a double is worse than useless if it encodes today's drift.
2. **.NET wrappers and ViewModels** (C#): this is where AI test gen actually pays. Pure methods, input validation, unit conversion, project-file parsers.
3. **WPF shell**: click-paths, docking, command routing. Needs UI automation (FlaUI), not NUnit on `MainWindow.xaml.cs`.
4. **Build graph**: mixed CMake + `.vcxproj` + `.csproj` + Intel/oneAPI Fortran. AQE's `aqe init --auto` looks for `package.json`, `pom.xml`, `Cargo.toml`, `go.mod`. It will not understand this solution.

So "generate tests if missing" is the wrong first sentence. The first sentence is: **lock today's outputs of the solver as characterization tests, then generate unit tests only for the C# belt around them.**

---

## GitHub Copilot Enterprise: what you already paid for

You already have the better default for this stack.

### Privacy (why this beats AQE's default cloud keys)

GitHub states that **Copilot Business and Copilot Enterprise customer data is not used to train models**. Consumer Copilot Free/Pro/Pro+ interaction data may be used unless the user opts out (policy update April 24, 2026). Enterprise is on the Data Protection Agreement, not that consumer change.

Sources:

- [Updates to GitHub Copilot interaction data usage policy](https://github.blog/news-insights/company-news/updates-to-github-copilot-interaction-data-usage-policy/)
- [Hosting of models for GitHub Copilot](https://docs.github.com/en/copilot/reference/ai-models/model-hosting)

AQE's default path is `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` in `.env`. That sends proprietary solver source to a second vendor unless you force Ollama (`AQE_FREE_TIER=1` or `LLM_MODE=local-only`). For an engineering ISV, that is a procurement and ITAR/export conversation, not an `npm install`.

### C# tomorrow: Copilot testing for .NET (`@Test`)

Official feature, not a third-party fleet.

- Docs: [Overview of GitHub Copilot testing for .NET](https://learn.microsoft.com/en-us/visualstudio/test/github-copilot-test-dotnet-overview?view=visualstudio)
- Requires **Visual Studio 2026 18.3+**, a **C# project that builds**, and a **paid Copilot** seat (Business or Enterprise counts)
- Frameworks: xUnit, NUnit, MSTest. Reuses the framework already in the solution; MSTest if none exists
- Scope: member, class, file, project, solution, or git diff
- Flow: generates a test project if needed, writes tests, **builds, and runs them**
- Entry points:
  - Copilot Chat, start the prompt with `@Test`
  - Right-click, Copilot Actions, Generate Tests

This is the closest thing in the market to "fill missing tests" for your C# layer, and it already sits inside the IDE your desktop team uses.

WPF caveat (not stated as a blocker in the docs, but true in practice): `@Test` generates unit tests. It will struggle with `Window`, `DependencyObject`, STA threads, and visual-tree code. Point it at ViewModels, converters, services, and solver facades. Leave `.xaml.cs` alone on day one.

### C++ tomorrow: Copilot Chat, not `@Test`

There is no Copilot `@Test` agent for C++ as of the VS 2026 February 2026 notes. Those notes add `@BuildPerfCpp` and profiler-agent use of existing unit tests. Test generation for C++ is still a Chat/agent prompt against a GoogleTest (or Catch2) project you already have compiling.

A prompt that works in a native test project:

```
Generate GoogleTest cases for the public API in SolverKernels/src/mesh_refine.cpp.
Do not invent geometry. Use the fixtures in tests/data/coarse_pipe.mesh.
Cover the failure returns, not just the happy path.
Do not assert floating-point equality; use EXPECT_NEAR with the tolerances in tests/tolerances.h.
```

### Fortran tomorrow: characterization first

Copilot will draft pFUnit or even a C-wrapper test if you paste a subroutine, but it does not understand your compiler flags, COMMON blocks, or vendor extensions. The reliable pattern for a 20-year solver is:

1. Pick 5-10 customer input decks that already "look right."
2. Freeze stdout / key result files as golden artifacts in `tests/gold/`.
3. A tiny Fortran or C# harness runs the solver and diffs with a numeric tolerance.
4. Only then ask Copilot to add pFUnit tests around a **pure function** you extracted (unit conversion, input parse, checksum).

pFUnit: https://github.com/Goddard-Fortran-Ecosystem/pFUnit

### WPF tomorrow: do not generate UI tests first

Microsoft Coded UI is dead. AQE has Playwright/Cypress (browsers), not desktop.

Order of attack:

1. Extract logic out of code-behind into testable ViewModels (Copilot can help the extract).
2. `@Test` those ViewModels.
3. Later: [FlaUI](https://github.com/FlaUI/FlaUI) against a Debug build of the WPF exe for a handful of smoke paths (open project, run, export). That is a week-two project.

### Copilot MCP in Visual Studio (if you still want AQE as a sidecar)

Visual Studio 2026, or VS 2022 17.14+, can load MCP servers. Official: [Use MCP Servers to Extend GitHub Copilot](https://learn.microsoft.com/en-us/visualstudio/ide/mcp-servers?view=visualstudio)

Enterprise catch: **org policy "MCP servers in Copilot" must be on.** Admins can also set an MCP allow list. If your GitHub Enterprise Copilot dashboard has MCP off, AQE will never appear in Agent mode. Check that **before** installing Node.

VS looks for MCP config in this order:

1. `%USERPROFILE%\.mcp.json` (all solutions)
2. `<solution>\.vs\mcp.json`
3. `<solution>\.mcp.json` (good for source control)
4. `<solution>\.vscode\mcp.json` (what AQE's installer writes)
5. `<solution>\.cursor\mcp.json`

AQE's own Copilot recipe writes `.vscode/mcp.json`:

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

For Visual Studio, also drop the same `servers` block in `<solution>\.mcp.json` so VS Agent mode sees it. Tools start **disabled**; you enable them in the tool picker. First run will prompt to trust the server (VS 2026 18.7+).

---

## Tomorrow-morning runbook (office)

Do this in order. Stop if a step fails; do not jump to AQE.

### 0. Admin / IT (15 min, once)

- Confirm the seat is **Copilot Enterprise** (or Business), not a personal Pro account on a work machine.
- In github.com (or GHEC): org Copilot policies, Agent mode on, MCP servers on if you might try AQE later.
- Confirm Visual Studio version: **2026 18.3+** for `@Test`. If the office is still on VS 2022, `@Test` for .NET is not there; use Copilot Chat "generate tests" instead, or upgrade one box.
- Do **not** put solver source into a new Anthropic/OpenAI key without legal.

### 1. Pick a thin vertical (30 min)

Choose **one** C# project that already builds, with almost no native P/Invoke. Examples that usually exist in this kind of app:

- unit conversion
- project/file parser
- license/config reader
- a ViewModel for a settings dialog

Do not start with the Fortran CFD kernel or `MainWindow.xaml`.

### 2. Generate C# tests with Copilot (1-2 hours)

1. Open the `.sln` in Visual Studio.
2. Build Debug. If it does not build, stop. Copilot testing expects a compiling project.
3. Right-click the class, Copilot Actions, Generate Tests, **or** Chat: `@Test generate NUnit tests for class Foo, cover invalid inputs`.
4. Watch it create/reuse a test project, build, and run.
5. Delete tautologies (`Assert.True(true)`, tests that only check the mock).
6. Check the tests into a branch. That is the demo you show the team.

### 3. One C++ characterization test (1 hour)

If a GoogleTest (or Catch2) project already exists, add **one** test that loads a tiny fixture and `EXPECT_NEAR`s a known scalar. If no test project exists, do not ask an agent to invent the whole harness. Create an empty GoogleTest vcxproj by hand (or Copilot scaffolding under your review), then add one test.

### 4. Optional: AQE lab on a **copy** of that C# library (afternoon, skippable)

Only if Node 18+ is allowed on the workstation and legal is OK with local Ollama **or** Azure OpenAI already approved.

```powershell
# In an isolated clone of a small C# library, not the full simulator
node -v   # >= 18
npm install -g agentic-qe
cd path\to\small-csharp-lib
aqe init --auto --with-copilot
aqe health
```

On Windows, `aqe health` will likely say `HNSW backend: js`. Fine for a tiny lib, bad for the full tree.

Local models (recommended for proprietary code):

```powershell
# if Ollama is installed
ollama pull qwen3:8b
$env:AQE_FREE_TIER = "1"
```

Then in Copilot Agent mode (VS or VS Code):

```
Generate tests for src/UnitConversion.cs with 80% coverage target using nunit
```

Compare the AQE tests to the `@Test` tests from step 2. Keep whichever compiles and is less tautological. Expect `@Test` to win on C#.

If `npx agentic-qe mcp` does not stay up, stop. Do not debug native `node-gyp` on day one.

---

## How AQE would be configured with Copilot (reference, not day-one)

From [docs/platform-setup-guide.md](https://github.com/proffesor-for-testing/agentic-qe/blob/main/docs/platform-setup-guide.md):

```bash
npm install -g agentic-qe
cd your-project
aqe init --auto --with-copilot
aqe platform verify copilot
aqe-mcp          # smoke: MCP server on stdio
```

Installer writes:

- `.vscode/mcp.json` (Copilot / VS Code)
- `.github/copilot-instructions.md` (QE rules)

LLM keys (`.env`, never commit):

| Provider | Env | Use here |
| --- | --- | --- |
| Claude | `ANTHROPIC_API_KEY` | AQE default. Avoid for solver source unless approved |
| OpenAI | `OPENAI_API_KEY` | Same |
| Azure OpenAI | `AZURE_OPENAI_API_KEY` | Best cloud option if the company already has it |
| Ollama | local, `OLLAMA_URL` | Best default for this ISV |
| Gemini / OpenRouter | various | Don't |

Kill switch: `AQE_LLM_ROUTER_DISABLED=1` (deterministic only, no generation).

Useful CLI after init:

```bash
aqe agent list
aqe code index src/
aqe code search "mesh refinement"
aqe health
```

There is **no** `aqe generate-tests --lang=fortran` flag in the public CLI. Generation is "ask the coding agent to call MCP tools."

---

## Better options than AQE for this stack

Ranked for **your** office, not for a Node microservice team.

| Rank | Tool | Why it wins here | Start tomorrow? |
| --- | --- | --- | --- |
| 1 | Copilot Enterprise `@Test` in VS 2026 | Already licensed. Builds and runs C# tests. Same IDE as WPF. | Yes |
| 2 | Copilot Chat + existing GoogleTest/Catch2 | C++ kernels. You control fixtures and tolerances. | Yes |
| 3 | Characterization / golden files (ApprovalTests.NET, ApprovalTests.cpp, or a 50-line diff harness) | The right test type for numerical solvers. | Yes (C#/C++); Fortran harness by hand |
| 4 | Coverage report then Copilot "fill these lines" | VS coverage / OpenCppCoverage give a missing-test list AQE will not parse well | Yes, after 1-2 |
| 5 | IntelliTest / Pex (VS Enterprise, C#) | Systematic input generation for pure .NET methods. Poor on WPF and native interop. | Maybe later, if you have VS Enterprise |
| 6 | FlaUI | Real WPF UI smoke. Not AI. | Week 2 |
| 7 | pFUnit | Real Fortran units, once functions are extracted | Week 2-3 |
| 8 | AQE MCP sidecar | Useful extra coverage/flaky agents **if** the language is C#/Python/JS and legal accepts its LLM path | Lab only |
| skip | Diffblue Cover | Java. | No |
| skip | Playwright/Cypress via AQE | Web. Your UI is WPF. | No |
| skip | Coded UI | Retired. | No |

Qodo (Codium) can help C# in VS Code; it does not beat Copilot Enterprise when you already pay for Copilot and live in Visual Studio. Re-evaluate only if `@Test` is blocked by policy.

---

## Engineering-simulator specific pitfalls

- **P/Invoke / C++/CLI:** Copilot `@Test` will either skip or mock the native call. Prefer tests that hit a compiled native DLL with a known input deck over mocked `DllImport`.
- **Floating point:** ban `Assert.Equal` on doubles in the Copilot instructions file (`.github/copilot-instructions.md`). Give the team `tests/tolerances.h` / `Tolerances.cs`.
- **STA / Dispatcher:** WPF unit tests need `[StaFact]` (xUnit) or NUnit apartment attributes. `@Test` sometimes forgets this.
- **Secret sauce in comments and decks:** even with Enterprise no-train, prompts still **leave the machine** to GitHub/Azure to produce a completion. For ITAR / customer decks, keep gold files and customer models out of the prompt. Point Copilot at synthetic tiny meshes.
- **AQE learning DB** (`.agentic-qe/memory.db`) will store snippets of your code. Treat it as source. Do not copy it home.

Suggested Copilot instructions (drop in `.github/copilot-instructions.md` on the product repo):

```
This is a Windows engineering simulation product: C/C++/Fortran kernels, C#/WPF desktop.
Never assert exact equality on floating-point results. Use the project's tolerance helpers.
Prefer characterization tests with golden files in tests/gold/ for solvers.
Do not generate tests for WPF code-behind. Generate for ViewModels and services.
Do not invent mesh files. Use fixtures under tests/data/.
Do not add npm/jest tests. Native: GoogleTest. C#: NUnit or the framework already in the solution. Fortran: pFUnit.
```

---

## Verdict for the office

| Question | Answer |
| --- | --- |
| Can AQE generate tests for this legacy app? | Only the C# belt, and not as well as Copilot `@Test`. Not C/C++/Fortran/WPF UI. |
| Should we install it tomorrow morning? | No, not on the product tree. |
| What should we actually run tomorrow? | VS 2026 `@Test` on one C# ViewModel/service, plus one golden-file native test. |
| When is AQE worth a second look? | If we want MCP coverage/flaky agents on a **C#** service, with Ollama or approved Azure OpenAI, after Copilot `@Test` is in daily use. |
| Copilot Enterprise config | Seat + VS 2026 18.3 + Agent mode. MCP optional, needs org policy. |

---

## Sources

- AQE README, `.env.example`, `package.json` 3.14.0, `docs/platform-setup-guide.md`
- AQE `src/shared/types/test-frameworks.ts`, `src/shared/language-detector.ts`, `src/mcp/protocol-server.ts`
- AQE `docs/multi-language-goap-implementation-plan.md` (NUnit still an alias)
- [Copilot testing for .NET](https://learn.microsoft.com/en-us/visualstudio/test/github-copilot-test-dotnet-overview?view=visualstudio)
- [Copilot in Visual Studio February 2026 update](https://github.blog/changelog/2026-03-04-github-copilot-in-visual-studio-february-update/)
- [MCP servers in Visual Studio](https://learn.microsoft.com/en-us/visualstudio/ide/mcp-servers?view=visualstudio)
- [Copilot model hosting / no-train for Business and Enterprise](https://docs.github.com/en/copilot/reference/ai-models/model-hosting)
