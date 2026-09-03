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
| **GitHub Copilot Enterprise** (already have) | Commercial GitHub/MS | None extra | **Yes** | `/tests` if a C test exists | **Yes** `/tests` Catch2/GTest | **Yes** `@Test` xUnit/NUnit/MSTest | **No** first-class | ViewModels only | It *is* Copilot | **Yes** | Mild: `@Test` product page is C# only. Don’t read Fortran/WPF into “any language.” |
| **dotnet/skills `dotnet-test`** (`code-testing-generator`) | **MIT** (`LICENSE`) | Low (permissive) | **Yes** (Copilot writes files, then build/repair) | **No `c.md`** | **Yes `extensions/cpp.md`** GoogleTest/Catch2/doctest/Boost.Test. Detects `CMakeLists.txt` **and** `*.vcxproj` | **Yes `extensions/dotnet.md`** MSTest/xUnit/NUnit/TUnit; classic `packages.config` | **No `fortran.md`** | No WPF test extension | **Yes:** Copilot **CLI** `/plugin install dotnet-test@dotnet-agent-skills`. VS Code preview. Cursor. Codex. **Visual Studio not in README install list** | **C#/C++ only if you leave VS for CLI/VS Code** | Agent says “any language”; extension table is the truth. Fortran/C/WPF missing. |
| **Agentic-QE** | **MIT** (`LICENSE`) | Low | Partial | **No** generator | **No** | **Advertised csharp**; factory is xUnit-shaped; older `qe/tests/generate` types omit csharp | **No** | **No** | MCP host only; `--with-copilot` writes `.vscode/mcp.json` + instructions. No 60 agents | C# lab only, Enterprise MCP off by default | **The trap you already hit** |
| **UTBotCpp** | **Apache-2.0** (`LICENSE`) | Low for *using* the binary | **Yes** Google Test only (`TestsPrinter`, `GTestLogger`). **No Catch2 printer.** | **Yes** (`Language.h`: `C, CXX, ANY, UNKNOWN`) | **Yes** | **No** | **No** | **No** | **Nothing.** Own VS Code VSIX. | **No.** Ubuntu 20.04+ only | Don’t infer Catch2/Fortran. OS trap. |
| **qodo-cover (Cover-Agent)** | **AGPL-3.0** (`LICENSE`) | **High.** Copyleft + AGPL §13 network clause. Unmaintained. Counsel before any use. | Appends to `--test-file-path`. Full-repo mode **Python only** (`NotImplementedError` otherwise). | Template `templated_tests/c_cli` | Template `cpp_cli` | Template `csharp_webservice` | **No template.** `language_extensions.toml` lists FORTRAN because it is a **GitHub-linguist dump** (ABAP…Zig), not a generator list | **No** | **Nothing** | No | **Yes — same class as AQE.** Linguist map ≠ generator. |
| **FortranTestGenerator** | **GPL-3.0** (`LICENSE`) | **High** if you ship/link it. | Capture + replay `.f90` driver (`DEFAULT_SUFFIX = '.f90'`). You still write asserts. | No | No | No | **Yes** (gfortran `-S`, Serialbox2, 2019) | No | **Nothing** | **No** | GPL + compiler trap |
| **pFUnit** | **Apache-2.0** (`LICENSE`, NASA) | Low | **No.** `funitproc` converts `.pf` → `.F90`. You author tests. | No | No | No | Harness only | No | Copilot can *author* `@test` into it | Week 2–3 if Intel Fortran + CMake | “Unit testing” ≠ auto-generation |
| **ApprovalTests.cpp** | **Apache-2.0** (`LICENSE`) | Low | **No** (`.approved.*` oracles) | via C++ tests | Library | No | No | No | Copilot writes the one-liner | Yes as a library | Snapshot oracle, not a generator |
| **ApprovalTests.Net** | **Apache-2.0** (`LICENSE.md`) | Low | **No.** Readme: not actively maintained. | No | No | Library | No | `ApprovalTests.Net.Wpf` is UI snapshot verify, not codegen | Copilot writes `Approvals.Verify` | Yes as a NuGet | WPF bullet ≠ WPF generator |
| **KLEE** | NCSA/UIUC (`LICENSE.TXT`) | Low (attribution) | **No test source.** Emits `.ktest` inputs. | Clang bitcode | optional libc++ | No | No | No | Nothing | Linux/macOS/FreeBSD. No Windows. | “Test generation” = path inputs |
| **VectorCAST ATG** | Proprietary | Contract | Yes (C/C++/Ada; GoogleTest-syntax option) | Yes | Yes | No | **UNKNOWN** (not listed) | No | VS Code Test Explorer. No Copilot claim in unpaid docs. | After purchase | Fortran UNKNOWN |
| **Parasoft C/C++test 2026.1** | Proprietary | Contract | Vendor: yes. 2025.2 unpaid MCP table is **coverage/docs**, not ATG. 2026.1 marketing says test gen. | Yes | Yes | No | UNKNOWN | No | MCP stdio; Copilot mentioned in 2025.2 docs | After license | Confirm 2026.1 MCP **tool list** with vendor |
| **Parasoft dotTEST 2026.1** | Proprietary | Contract | Claims AI unit tests for uncovered .NET lines | No | No | **Yes** | No | **UNKNOWN** | Copilot **CLI** `dottestmcp.bat` + skill | After license | Don’t assume WPF from “.NET” |
| **Codemind Coyote C++** | Proprietary | Contract | Auto harness + concolic cases | Yes | Yes | No | No | No | No Copilot/MCP in unpaid FAQ | After purchase | Not Microsoft’s .NET “Coyote” |

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
2. **Cover-Agent `language_extensions.toml`** — FORTRAN appears next to ABAP because it is GitHub Linguist. Templates: no Fortran. Full-repo: Python or `NotImplementedError`. **AGPL-3.0.**
3. **dotnet-test “you are polyglot / any language”** — MIT, real `cpp.md` + `dotnet.md`. No `fortran.md`, no `c.md`. CLI/VS Code, not Visual Studio.
4. **UTBotCpp** — `Language { C, CXX }` and GoogleTest printers only. Ubuntu. Not Copilot. Not Catch2.
5. **KLEE / ApprovalTests / pFUnit** — “tests” means inputs, oracles, or a preprocessor, not ATG.
6. **Parasoft MCP** — unpaid 2025.2 tool list is coverage/search; 2026.1 banner says generation. Get the tool names in writing.

---

## Office rule

Use **permissive or already-paid** only: Copilot Enterprise, MIT `dotnet-test` (CLI lab), Apache-2.0 pFUnit / ApprovalTests as libraries.

Do **not** install AGPL Cover-Agent or GPL FortranTestGenerator on the product tree without counsel.

Tomorrow: Visual Studio + Copilot `@Test` (C#) and `/tests` (C++). Fortran = gold files. WPF = ViewModels only.

---

## Sources (files)

- `dotnet/skills` `LICENSE` (MIT); `plugins/dotnet-test/skills/code-testing-extensions/SKILL.md` (language table); `extensions/cpp.md`; `code-testing-generator.agent.md`
- `proffesor-for-testing/agentic-qe` `LICENSE` (MIT)
- `UnitTestBot/UTBotCpp` `LICENSE`; `server/src/Language.h`; `server/src/printers/`
- `qodo-ai/qodo-cover` `LICENSE` (AGPL-3.0); `templated_tests/README.md`; `cover_agent/main_full_repo.py`; `language_extensions.toml`
- `fortesg/fortrantestgenerator` `LICENSE` (GPL-3.0); `generator.py`
- `klee/klee` `LICENSE.TXT` (NCSA)
- `Goddard-Fortran-Ecosystem/pFUnit` `LICENSE`; `bin/funitproc`
- `approvals/ApprovalTests.cpp` `LICENSE`; `approvals/ApprovalTests.Net` `LICENSE.md`
- Microsoft Learn: Copilot testing for .NET (C# only); C/C++ `/tests`
