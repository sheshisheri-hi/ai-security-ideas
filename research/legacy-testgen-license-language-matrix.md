# License + language audit: can these tools actually generate tests for this shop?

**Date:** 2026-09-02
**Method:** LICENSE files + source (language lists, agents, OS requirements). Not README marketing.
**Stack:** proprietary C / C++ / C# / Fortran / WPF simulator, Visual Studio, Copilot Enterprise
**Legal:** this is an engineering map, not counsel. AGPL/GPL rows need your lawyer before any install.

This is the AQE-style check: license first, then “does the code emit tests in *our* languages,” then Copilot host.

---

## Direct answer

| Language | What actually generates tests tomorrow, with a license you can live with |
| --- | --- |
| **C#** | **Copilot `@Test`** (VS 2026 18.3+, xUnit/NUnit/MSTest). Already paid. |
| **C++** | **Copilot `/tests`** in Visual Studio (Catch2/GTest if a test file is open). |
| **C** | Same `/tests` path if you have a C test project. No dedicated C generator in Copilot. |
| **Fortran** | **Nothing** that both generates tests *and* is Copilot + Visual Studio. pFUnit is Apache-2.0 but it is a **harness**. FortranTestGenerator is GPL-3.0, 2019, gfortran. |
| **WPF UI** | **Nothing.** Copilot can test ViewModels (C#). FlaUI later. |

There is **no** OSS or paper tool that is “AQE but it actually does C/C++/C#/Fortran with Copilot in Visual Studio.” The closest extra is Microsoft `dotnet-test` skills (**MIT**), which **does** tell Copilot to write C# and C++ tests — but it installs in **Copilot CLI / VS Code preview**, and the README does **not** list Visual Studio.

---

## Matrix (source-checked)

Legend: **Gen** = writes test *source files*. **Harness** = you/Copilot still write the tests.

| Tool | License (file) | Closed-source product risk | Gen? | C | C++ | C# | Fortran | WPF | Copilot? | VS tomorrow? | AQE-style trap? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **GitHub Copilot Enterprise** (already have) | Commercial GitHub/MS | None extra | **Yes** | `/tests` if a C test exists | **Yes** `/tests` Catch2/GTest | **Yes** `@Test` xUnit/NUnit/MSTest | **No** first-class | ViewModels only | It *is* Copilot | **Yes** | No |
| **dotnet/skills `dotnet-test`** (`code-testing-generator`) | **MIT** (`LICENSE`) | Low (permissive) | **Yes** (Copilot writes files, then build/repair) | Not listed | **Yes as markdown skills** GoogleTest/Catch2/doctest/Boost.Test (`extensions/cpp.md`). Detects `CMakeLists.txt` **and** `*.vcxproj` | **Yes** MSTest/xUnit/NUnit/TUnit; classic `packages.config` called out | **Zero hits** for Fortran in the repo | WPF only in *upgrade* plugins, not test-gen | **Yes:** Copilot **CLI** `/plugin install dotnet-test@dotnet-agent-skills`. VS Code preview (`chat.plugins.enabled`). Cursor. Codex. **Visual Studio not in install list** (blog: “working on it”) | **C#/C++ only if you leave VS for CLI/VS Code** | Soft trap: polyglot list looks like AQE’s 11 platforms. C++ is prompt skills, not a C++ parser. Fortran/WPF absent. |
| **Agentic-QE** | **MIT** (`LICENSE`) | Low | Partial | **No** generator | **No** | **Advertised csharp**; factory is xUnit-shaped; older `qe/tests/generate` types omit csharp | **No** | **No** | MCP host only; `--with-copilot` writes `.vscode/mcp.json` + instructions. No 60 agents | C# lab only, Enterprise MCP off by default | **The trap you already hit** |
| **UTBotCpp** | **Apache-2.0** (`LICENSE`) | Low for *using* the binary; tests it writes are your code | **Yes** Google Test regression/error suites | **Yes** | **Yes** | **No** | **No** | **No** | **No.** Own VS Code VSIX, not Copilot | **No.** README: “only use UTBot under Ubuntu 20.04 and above” | Language OK for C/C++, OS/IDE trap |
| **qodo-cover (Cover-Agent)** | **AGPL-3.0** (`LICENSE`) | **High.** Copyleft + network clause. Unmaintained. Do not put on the product tree. Counsel if anyone insists. | **Yes** (appends to existing test files; C/C++/C# docker fixtures exist) | Fixtures yes | Fixtures yes | Fixtures yes | **No** | **No** | **No** (own LLM keys) | No | License trap + dead project |
| **FortranTestGenerator** | **GPL-3.0** (`LICENSE`) | **High** if you ship/link it. Generated capture/replay code: ask counsel. | **Yes** (capture instrumentation + replay *driver*). You still write asserts. | No | No | No | **Yes** (F90+, gfortran assembler, Serialbox2) | No | **No** | **No** (2019, HPC/gfortran) | License + compiler trap |
| **pFUnit** | **Apache-2.0** (`LICENSE`, NASA) | Low | **No** (framework) | No | No | No | Harness only | No | Copilot can *author* `@test` into it | Week 2–3 if Intel Fortran + CMake | Not a generator |
| **ApprovalTests.cpp** | **Apache-2.0** (`LICENSE`) | Low | **No** (oracle files after you write a test) | via C++ tests | **Yes** | No | No | No | Copilot writes the one-liner | Yes as a library | Not a generator |
| **ApprovalTests.Net** | **Apache-2.0** (`LICENSE.md`) | Low | **No** | No | No | **Yes** (library) | No | screenshot package exists; still not a generator | Copilot writes `Approvals.Verify` | Yes as a NuGet | Not a generator |
| **KLEE** | UIUC-style (LLVM family) | Generally OK to run; not a VS product | Inputs, not Catch2/GTest files | LLVM C | limited | No | No | No | No | Linux | Wrong shape |
| **VectorCAST ATG / Parasoft C/C++test / Coyote C++** | Proprietary paid | Contract | Vendor claims yes for C/C++ | Yes (paid) | Yes (paid) | Parasoft dotTEST for C# (overlap with `@Test`) | **No** | **No** | Parasoft 2026.1 mentions AI/MCP — **UNKNOWN** vs Copilot | Not tomorrow (sales + install) | Don’t confuse with Copilot |

---

## What “with Copilot” actually means (hosts)

| Host | Copilot `@Test` / `/tests` | AQE MCP | `dotnet-test` plugin |
| --- | --- | --- | --- |
| Visual Studio 2022 17.14+ / 2026 | **Yes** (this is the office path) | Maybe if `.vscode/mcp.json` loads and org MCP policy is on | **Not listed** in `dotnet/skills` README |
| VS Code Copilot Chat | `/tests` style | Yes (installer target) | Preview: `chat.plugins.enabled` |
| Copilot CLI | N/A as `@Test` | If you wire MCP yourself | **Yes** `/plugin marketplace add dotnet/skills` then `/plugin install dotnet-test@dotnet-agent-skills` |
| GitHub.com coding agent | Custom instructions only | Not configured by AQE | UNKNOWN |

---

## AQE-style traps, named

1. **AQE** — MIT, Copilot row in README, no C/C++/Fortran generator, Copilot is MCP not the model.
2. **dotnet-test “polyglot”** — MIT, real C# pipeline, C++ is `extensions/cpp.md` guidance (Catch2/GTest examples). Fortran not in the repo. Install path is CLI/VS Code, not Visual Studio.
3. **UTBotCpp** — really does emit Google Test for C/C++, Apache-2.0, **Ubuntu only**, not Copilot.
4. **Cover-Agent** — C/C++/C# examples exist, **AGPL-3.0**, unmaintained, not Copilot.
5. **FortranTestGenerator** — really does generate Fortran *drivers*, **GPL-3.0**, 2019, gfortran, not Copilot.

---

## Office rule

Use **permissive or already-paid** only: Copilot Enterprise, MIT `dotnet-test` (CLI lab), Apache-2.0 pFUnit / ApprovalTests as libraries.

Do **not** install AGPL Cover-Agent or GPL FortranTestGenerator on the product tree without counsel.

Tomorrow: Visual Studio + Copilot `@Test` (C#) and `/tests` (C++). Fortran = gold files. WPF = ViewModels only.

---

## Sources (files)

- `dotnet/skills` `LICENSE` (MIT); `README.md` install (Copilot CLI / VS Code / Cursor / Codex); `plugins/dotnet-test/plugin.json`; `plugins/dotnet-test/README.md`; `plugins/dotnet-test/skills/code-testing-extensions/extensions/cpp.md`
- `proffesor-for-testing/agentic-qe` `LICENSE` (MIT); Copilot installer (earlier note)
- `UnitTestBot/UTBotCpp` `LICENSE` (Apache-2.0); `README.md` Ubuntu-only
- `qodo-ai/qodo-cover` `LICENSE` (AGPL-3.0)
- `fortesg/fortrantestgenerator` `LICENSE` (GPL-3.0)
- `Goddard-Fortran-Ecosystem/pFUnit` `LICENSE` (Apache-2.0)
- `approvals/ApprovalTests.cpp` `LICENSE` (Apache-2.0); `approvals/ApprovalTests.Net` `LICENSE.md` (Apache-2.0)
