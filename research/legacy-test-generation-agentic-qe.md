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
| C / C++ kernels | No as a test-gen language. Files can be indexed for search. No GoogleTest / Catch2 / CTest generator. | Copilot Chat **`/tests`** in Visual Studio + GoogleTest or Catch2 |
| Fortran solvers | No. Zero real Fortran hits in the repo. | Characterization tests (golden files) + [pFUnit](https://github.com/Goddard-Fortran-Ecosystem/pFUnit) + Copilot Chat |
| WPF desktop UI | No. | Unit-test ViewModels first. Later **FlaUI** only. Do not start WinAppDriver. |
| Coverage-gap hunting | Weak vs `gcov` / VS coverage. C# **dotCover XML** is the one AQE format that fits. | VS coverage (2026: Community+) or OpenCppCoverage, then Copilot fills listed gaps |

AQE **can** be wired into GitHub Copilot as an MCP server. That is an IDE plugin, not a replacement for Copilot Enterprise. AQE still needs its **own** LLM keys. It does not call GitHub Copilot as the model.

If you try AQE at all, try it later on a **small isolated C# class library**, with `AQE_FREE_TIER=1` + local Ollama, never against the full solver tree on day one.

---

## What AQE actually is

AQE is a Node.js (>=18) CLI + MCP server marketed as an "Agentic Quality Engineering Fleet." Not a VS Code extension of its own. npm `agentic-qe@3.14.0` (MIT, ~5k weekly downloads, last push 1 Sep 2026). Real package, wrong stack.

- Install: `npm install -g agentic-qe` then `aqe init --auto`
- Binaries: `aqe`, `agentic-qe`, `aqe-mcp`
- First-class host: Claude Code. Copilot is an MCP client of the same server.
- Windows: native deps (`hnswlib-node`) often fail to build. JS fallback is documented as unsuitable for large indexes.

Canonical test-gen languages (`src/shared/types/test-frameworks.ts`): `typescript`, `javascript`, `python`, `java`, `csharp`, `go`, `rust`, `swift`, `kotlin`, `dart`. **Not** C, C++, Fortran, WPF.

Shipped C# generator is **xUnit**. NUnit is a factory **alias** to that generator; `[TestCase]` is explicitly not supported. Playwright/Cypress are in the type enum but the factory **throws** `not yet implemented`. Tree-sitter grammars on disk include `tree-sitter-c_sharp.wasm`. No C, C++, or Fortran grammar.

Coverage parser formats (`coverage-parser.ts`): lcov, cobertura, json, jacoco, **dotcover**, tarpaulin, gocover, kover, xcresult. **No gcov, llvm-cov, or OpenCppCoverage.**

---

## Fit to a decades-old engineering simulator

Four test problems, not one:

1. **Numerical kernels** (C/C++/Fortran): golden-file / characterization tests. A generated `EXPECT_EQ` on a double locks in today's drift.
2. **.NET wrappers and ViewModels** (C#): where AI test gen pays.
3. **WPF shell**: FlaUI smokes later, not NUnit on `MainWindow.xaml.cs`.
4. **Build graph**: mixed CMake + `.vcxproj` + `.csproj` + Intel Fortran. AQE `init --auto` looks for `package.json` / `pom.xml`. It will not understand this solution.

**Lock today's solver outputs as characterization tests, then generate unit tests only for the C# belt around them.** Oracles (analytic solutions, published benchmarks, gold files, tolerances) must come from you. Copilot writes the harness.

---

## GitHub Copilot Enterprise: what you already paid for

### Privacy

GitHub does **not** use Copilot Business/Enterprise customer data to train models. Prompts still leave the machine to GitHub/Microsoft for inference. Optional: GHEC data residency + "restrict Copilot to data residency compliant models" (US/EU, +10% credits).

IP indemnity for **unmodified** suggestions requires the public-code / duplication filter **on**.

### C# : `@Test` (VS 2026 18.3+)

[Copilot testing for .NET](https://learn.microsoft.com/en-us/visualstudio/test/github-copilot-test-dotnet-overview?view=visualstudio): xUnit/NUnit/MSTest, generate/build/run. Point it at ViewModels and services, not `.xaml.cs`.

On **VS 2022**, `@Test` is not there. Use Chat `/tests` instead.

### C++ : `/tests` in Visual Studio (not `@Test`)

No C++ equivalent of the generate/build/repair `@Test` loop. There **is** a documented `/tests` slash command that copies **existing C++ test style** (Catch2 `SCENARIO`/`GIVEN`, GoogleTest, etc.) if a test file is open.

Source: [C++ Team Blog, 30 Jan 2025](https://devblogs.microsoft.com/cppblog/document-build-instructions-and-more-with-enhanced-c-awareness-from-copilot-chat-in-visual-studio/)

Tomorrow: open a sibling Catch2/GTest file, select the kernel, type `/tests`. Then:

```
Match test_ship.cpp style. Use Approx / EXPECT_NEAR with the tolerances in tests/tolerances.h.
Do not invent geometry. Use tests/data/coarse_pipe.mesh.
Cover NaN, empty mesh, and failure returns.
```

### Fortran: characterization first, then pFUnit

Fortran is **not** in Copilot's core-language list. Chat can still draft pFUnit. Quality expected weaker. Freeze 5-10 input decks as `tests/gold/`, then pFUnit only on extracted pure functions.

### WPF: FlaUI later, not tomorrow

- Coded UI: dead (VS 2019 last recorder).
- **WinAppDriver 1.2.1 (Nov 2020): unmaintained. Do not start new work.**
- Default for new WPF UI: [FlaUI](https://github.com/FlaUI/FlaUI) (UIA3).
- Plots/OpenGL viewports are often invisible to UI Automation.

90% effort on solvers. 10% FlaUI smokes (open case, run, see Complete).

### Cloud coding agent: later, not tomorrow

Copilot Workspace sunset 30 May 2025. Successor is the cloud coding agent. Default runner is **Ubuntu**. Windows via `.github/workflows/copilot-setup-steps.yml` since 18 Feb 2026; firewall is incompatible with Windows runners. Do not assign it a WPF/.NET Framework solution until a self-hosted Windows runner can **build the real .sln**.

Knowledge bases retired 1 Nov 2025 (use Spaces). Copilot Extensions (GitHub Apps) disabled 10 Nov 2025 (use MCP).

### MCP for AQE (optional sidecar)

VS 2026 or VS 2022 17.14+. Org policy **MCP servers in Copilot** must be on. Config: `<solution>\.mcp.json` plus what AQE writes to `.vscode/mcp.json`.

Copilot Enterprise is the **host that calls AQE tools**. It is **not** an AQE HybridRouter provider. There is no `copilot` / GitHub Models provider id. AQE's own LLM still wants Anthropic, OpenAI, Azure OpenAI, Bedrock, Gemini, OpenRouter, or Ollama, or `AQE_LLM_ROUTER_DISABLED=1`.

Even with AQE on Ollama, Copilot Chat as host still sends prompts to GitHub/Microsoft. Two data paths, not one.

---

## Tomorrow-morning runbook (office)

Do this in order. Stop if a step fails; do not jump to AQE.

### 0. Admin (15 min)

- Copilot Enterprise/Business seat, not personal Pro.
- Public-code filter **on** (indemnity).
- Ask IT if data-residency model restriction is on.
- VS 2026 18.3+ for `@Test`. VS 2022: `/tests` only.
- Commit `.github/copilot-instructions.md` (template at the bottom of this note).
- Do **not** put solver source into a new Anthropic/OpenAI key.

### 1. Thin vertical (30 min)

One C# project that already builds, almost no P/Invoke: unit conversion, file parser, settings ViewModel. Not the Fortran kernel. Not `MainWindow.xaml`.

### 2. C# tests (1-2 hours)

Build Debug. `@Test` or right-click Generate Tests. Delete tautologies. Branch it. That is the demo.

### 3. C++ `/tests` + one golden file (1 hour)

Open existing Catch2/GTest file. `/tests` on one kernel. Add one characterization test: tiny fixture, `EXPECT_NEAR`, gold CSV. If no native test project exists, scaffold GoogleTest by hand first.

### 4. Coverage if the SKU allows (30 min)

VS 2022 coverage: **Enterprise only**. **VS 2026: Community / Professional / Enterprise.** Else OpenCppCoverage. Paste 3 uncovered functions into Chat.

### 5. Optional AQE lab: skip unless legal + Node + Ollama/Azure OpenAI already approved. Isolated tiny C# lib only.

If you do the lab, prefer a global install (not `npx @latest` on every start) and:

```powershell
aqe test generate --file path\to\Service.cs --framework xunit --type unit
```

Ask for `nunit` and you still get xUnit-shaped tests.

**Do not tomorrow:** npm-install AQE on the product tree; IntelliTest; WinAppDriver; Diffblue/Qodo/Testsigma; cloud coding agent against the WPF .sln.

---

## Better options (ranked for this office)

| Rank | Tool | Start tomorrow? |
| --- | --- | --- |
| 1 | Copilot `@Test` (C#, VS 2026 18.3+) or `/tests` (C++/C# in VS 2022+) | Yes |
| 2 | Characterization / [ApprovalTests.cpp](https://github.com/approvals/ApprovalTests.cpp) / [ApprovalTests.Net](https://github.com/approvals/ApprovalTests.Net) | Yes |
| 3 | Catch2 generators or GoogleTest parameterized tests | Yes |
| 4 | VS coverage or OpenCppCoverage, then Copilot fills holes | This week |
| 5 | [pFUnit](https://github.com/Goddard-Fortran-Ecosystem/pFUnit) | Week 2-3 |
| 6 | FlaUI WPF smokes | Week 2 |
| 7 | Copilot custom instructions / prompt files / skills in `.github/` | Yes (10 min) |
| 8 | Copilot cloud coding agent on a Windows runner | Later |
| skip | **IntelliTest / Pex** | **Deprecated in VS 2026.** .NET Framework C# only, no unmanaged, no x64 in GA |
| skip | Diffblue Cover | Java |
| skip | Qodo `/test` | C++/C# plugin exists, but Qodo is **deprecating code generation** (Apr 2026) |
| skip | Symflower, Functionize | Wrong stack / web only |
| skip | WinAppDriver / Coded UI | Unmaintained / dead |
| skip | mutmut / Hypothesis | Python only |
| lab only | AQE MCP sidecar | C# xUnit only, own LLM keys |

---

## Copilot instructions to commit tomorrow

`.github/copilot-instructions.md`:

```
This is a Windows engineering simulation product: C/C++/Fortran kernels, C#/WPF desktop.
Never assert exact equality on floating-point results. Use the project's tolerance helpers.
Prefer characterization tests with golden files in tests/gold/ for solvers.
Do not generate tests for WPF code-behind. Generate for ViewModels and services.
Do not invent mesh files. Use fixtures under tests/data/.
Do not add npm/jest tests. Native: Catch2 or GoogleTest (match the file already open). C#: NUnit or the framework already in the solution. Fortran: pFUnit.
Do not mock physics kernels. Do not "fix" solver output to make a test pass.
```

---

## Addendum: AQE internals checked against source (same evening)

Does not change the verdict. Tightens the C# lab path.

| Claim in README | Runtime fact |
| --- | --- |
| NUnit support | Factory aliases NUnit to the **xUnit** generator. `[TestCase]` not supported. |
| Playwright / Cypress | Type exists; factory **throws** `Generator for '${framework}' not yet implemented`. |
| C/C++ | Indexing claimed. **No** generator, **no** tree-sitter grammar in `assets/grammars/`. |
| Fortran / WPF / GoogleTest / Catch2 / CTest / MSTest / SpecFlow / pFUnit | Absent. |
| Coverage for this shop | **dotCover XML** is the C# format AQE parses. Not gcov / OpenCppCoverage. |
| Copilot Enterprise as the model | **No.** MCP host only. No `copilot` provider id. |
| Compile-check generated C# | `compileValidation` defaults **false**. Opt-in `dotnet build` exists in a plan, not the default. |

For proprietary code on an AQE lab: `AQE_LLM_ROUTER_DISABLED=1` or Ollama (`AQE_FREE_TIER=1`), do not set Anthropic/OpenAI keys, prefer `npm install -g` over `npx @latest`. Copilot Chat remains a separate cloud path.

---

## Addendum: extra Copilot / skip-list sources

1. **C++ `/tests` is documented.** [C++ Team Blog](https://devblogs.microsoft.com/cppblog/document-build-instructions-and-more-with-enhanced-c-awareness-from-copilot-chat-in-visual-studio/)
2. **Writing tests tutorial:** https://docs.github.com/en/enterprise-cloud@latest/copilot/tutorials/write-tests
3. **IntelliTest is a dead end** in VS 2026. [IntelliTest](https://learn.microsoft.com/en-us/visualstudio/test/generate-unit-tests-for-your-code-with-intellitest?view=vs-2022)
4. **VS 2026 coverage** is Community/Pro/Enterprise. VS 2022 coverage is Enterprise-only.
5. **WinAppDriver is unmaintained.** New WPF UI: FlaUI.
6. **Qodo is deprecating code generation** (Apr 2026).
7. CITYWALK / CPP-UT-Bench (2025): LLMs are weaker on compiled C++ than Python/Java. Mitigation is project context + running tests in VS.

---

## Sources

- AQE `package.json` 3.14.0, `src/shared/types/test-frameworks.ts`, `src/domains/test-generation/factories/test-generator-factory.ts`, `src/domains/coverage-analysis/services/coverage-parser.ts`, `docs/platform-setup-guide.md`
- [Copilot testing for .NET](https://learn.microsoft.com/en-us/visualstudio/test/github-copilot-test-dotnet-overview?view=visualstudio)
- [C++ `/tests`](https://devblogs.microsoft.com/cppblog/document-build-instructions-and-more-with-enhanced-c-awareness-from-copilot-chat-in-visual-studio/)
- [Copilot write tests](https://docs.github.com/en/enterprise-cloud@latest/copilot/tutorials/write-tests)
- [MCP in Visual Studio](https://learn.microsoft.com/en-us/visualstudio/ide/mcp-servers?view=visualstudio)
- [Copilot model hosting](https://docs.github.com/en/copilot/reference/ai-models/model-hosting)
