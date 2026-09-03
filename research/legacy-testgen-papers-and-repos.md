# Legacy test generation: papers and GitHub repos (not AQE)

**Date:** 2026-09-02
**Status:** ad-hoc research (not a daily idea candidate)
**Question:** Besides AQE, which white papers or GitHub repos actually **create** tests for this shop’s C / C++ / C# / Fortran / WPF simulator?
**Already paid:** GitHub Copilot Enterprise, Visual Studio

This note stays in the private `ai-security-ideas` repo. It is not an implementation.

---

## Direct answer

**No.** I had not done this survey until you asked. The earlier note ranked AQE against Copilot and a skip-list of products. This note is the paper/repo pass.

**Is there a better generator than Copilot for this stack tomorrow?** Not as a product you install and point at the `.sln`.

- **C#:** Copilot `@Test` (VS 2026 18.3+) is still the thing that creates tests in your IDE.
- **C++:** Copilot `/tests` (match Catch2/GTest already in the tree). Research (CITYWALK, CPP-UT-Bench) says LLMs are weaker here than on Java/Python; context + compile-in-VS is the mitigation, not a new tool.
- **C:** SPARC (arXiv:2602.16671, Feb 2026) is a paper, not a VS plugin.
- **Fortran:** FortranTestGenerator (2017 capture-and-replay) can **write a replay driver + dump files**. It does not invent assertions. pFUnit/Julienne are harnesses. No LLM Fortran generator of note.
- **WPF:** still no generator. ViewModels via Copilot. UI later with FlaUI.

Closest *new* GitHub find: Microsoft’s **code-testing-generator** (Aug 2026, MIT, inside `dotnet/skills`). It is a Copilot **skill/agent** that writes tests, then builds and repairs them. It lists C++ (GoogleTest/Catch2/doctest/Boost.Test) and C#. Visual Studio support is “working on”; Copilot CLI / VS Code plugin today. Worth a lab after `@Test` works, not instead of it.

---

## Papers that generate tests (or measure that)

| Paper | Year | Link | Languages | Creates tests? | Tomorrow? |
| --- | --- | --- | --- | --- | --- |
| CITYWALK | 2025 (TOSEM) | [arXiv:2501.16155](https://arxiv.org/abs/2501.16155) | C++ | Yes (LLM + dependency analysis, GPT-4o) | No. Research artifact on [Zenodo](https://zenodo.org/records/14022506), not a VS plugin. Sends code to GPT-4o. |
| CPP-UT-Bench | 2024 | [arXiv:2412.02735](https://arxiv.org/abs/2412.02735) | C++ | Benchmark, not a tool. Fine-tuned weights for research. | No. Use as a warning: vanilla LLMs are weak on compiled C++. |
| SPARC (scenario planning for C tests) | 2026 | [arXiv:2602.16671](https://arxiv.org/abs/2602.16671) | C | Yes (CFG + LLM + compiler repair). Beats vanilla prompting; competitive with KLEE on their 59 subjects. | No product. Do **not** confuse with SPARC-X DFT (`SPARC-X/SPARC`). GitHub for *this* SPARC: **UNKNOWN** (no verified product repo). |
| FortranTestGenerator | 2017 | [paper](https://www.wr.informatik.uni-hamburg.de/_media/research/publications/2017/dlfaafutgflh17-fortrantestgenerator_automatic_and_flexible_unit_test_generation_for_legacy_hpc_code.pdf) + [fortesg/fortrantestgenerator](https://github.com/fortesg/fortrantestgenerator) | Fortran 90+ | Yes: capture/replay **drivers** from a live run. Developer still writes checks. Needs gfortran assembler + Serialbox2. | Maybe week 2–3 on one kernel, not tomorrow. Intel Fortran / MSBuild path is extra work. |
| LLM unit-test survey | 2025 | [arXiv:2511.21382](https://arxiv.org/abs/2511.21382) | mostly Java/Python | Survey. Symbolic tools fail on floats, libraries, path explosion. | Read, don’t install. |
| Julienne (Fortran 2023 testing idioms) | 2025 | [OSTI](https://www.osti.gov/servlets/purl/3000200) | Fortran | Harness/assertions, not an LLM generator. | Later alternative to pFUnit. |
| Meta TestGen-LLM | 2024 | industry paper (Cover-Agent cites it) | Java/Kotlin at Meta | Creates tests in their CI. | Not your languages as a product. |

---

## GitHub repos that create tests

| Repo | Creates tests? | Languages that match | License / status | Windows / VS? | Fit |
| --- | --- | --- | --- | --- | --- |
| [dotnet/skills](https://github.com/dotnet/skills) `code-testing-generator` | **Yes.** Research-plan-implement agent: detect framework, write tests, build, repair. Polyglot list includes **C#** (MSTest/xUnit/NUnit) and **C++** (GoogleTest/Catch2/doctest/Boost.Test). | MIT. Open-sourced ~Aug 2026. | Copilot CLI + VS Code plugin now. Visual Studio: Microsoft said they are working on it. | **Best extra** after native `@Test` / `/tests`. Same Copilot seat. No extra LLM vendor. |
| [qodo-ai/qodo-cover](https://github.com/qodo-ai/qodo-cover) (Cover-Agent) | **Yes.** Iterates until coverage target. Integration fixtures exist for **C, C++, C#**. | AGPL-3.0. README: **no longer maintained** (fork if you want it). ~5.4k stars. Last release 0.3.10 (May 2025). | CLI + coverage XML (lcov/cobertura). Needs a working test command **and** an LLM key (not Copilot). | Lab only. Qodo the product is deprecating codegen. Do not put solvers on a third-party LLM. |
| [fortesg/fortrantestgenerator](https://github.com/fortesg/fortrantestgenerator) | **Yes** (capture + replay driver). Not LLM. | Research tool, gfortran-centric. | Linux/HPC oriented. | Fortran characterization path. Heavy setup. |
| [klee/klee](https://github.com/klee/klee) | **Yes** (symbolic execution → concrete tests) for C/C++ bitcode. | UIUC/NCSA-style. Mature. | Linux/LLVM. Floats and external libs are the classic failure mode (the 2025 survey says this too). | Skip for a numerical Windows simulator. |
| [approvals/ApprovalTests.cpp](https://github.com/approvals/ApprovalTests.cpp) / [ApprovalTests.Net](https://github.com/approvals/ApprovalTests.Net) | Writes **gold files** once you author a test that calls `Approvals.Verify`. Copilot can write that one-liner. | Apache-2.0. | Yes (C++ and .NET). | Use with Copilot for solvers. Not a generator by itself. |
| [Goddard-Fortran-Ecosystem/pFUnit](https://github.com/Goddard-Fortran-Ecosystem/pFUnit) | **No** (framework). Copilot writes `@test` procedures. | NASA, CMake. Intel ifx listed. | Possible with Intel Fortran. | Week 2–3 harness. |
| CITYWALK | Artifact zip on Zenodo, not a maintained product repo. | Research. | GPT-4o. | Skip as a shop tool. |

**Not generators for this shop (repeat skip-list):** Agentic-QE, Diffblue Cover (Java), IntelliTest (deprecated VS 2026), Qodo `/test` product, WinAppDriver, Coded UI, EvoSuite (Java), Playwright (web).

---

## Ranked shortlist for this office

1. **Copilot `@Test` (C#) and `/tests` (C++)** — already paid, in Visual Studio, creates tests. Tomorrow.
2. **Microsoft `code-testing-generator`** (`dotnet/skills`) — same Copilot, better loop (write/build/repair), C++ frameworks named. Lab in Copilot CLI or VS Code after (1) works. Wait for VS if you will not leave Visual Studio.
3. **Characterization tests** — freeze 5–10 solver decks as `tests/gold/`. Copilot writes the Catch2/`EXPECT_NEAR` or ApprovalTests harness. This is how Fortran and numerical C++ get tests without a fake oracle.
4. **FortranTestGenerator** — only if you can run the kernel under gfortran long enough to capture. Else skip and gold-file by hand.
5. **Cover-Agent fork** — only as a contained C#/C++ experiment with a **local** LLM, never the product tree. AGPL + unmaintained.

Do not start CITYWALK, SPARC-the-paper, KLEE, or AQE for this `.sln`.

---

## Gaps that no paper closes

- **Floating-point oracles.** Symbolic tools and LLM `ASSERT_EQ` both lie. Gold files + tolerances are still the method.
- **Fortran.** Capture/replay (2017) or a human-written pFUnit test. No 2025–2026 LLM Fortran generator with a real repo.
- **WPF.** Zero academic or GitHub generators for `MainWindow.xaml`. Test ViewModels. FlaUI smokes later.
- **Mixed MSBuild + CMake + Intel Fortran.** Every research tool assumes one build graph (CMake+g++, Maven, `dotnet test`). Yours is not that.

---

## Sources

- CITYWALK: https://arxiv.org/abs/2501.16155 — ACM TOSEM 2025, doi 10.1145/3763791
- CPP-UT-Bench: https://arxiv.org/abs/2412.02735
- SPARC (C test gen): https://arxiv.org/abs/2602.16671
- LLM unit-test survey: https://arxiv.org/abs/2511.21382
- FortranTestGenerator: https://github.com/fortesg/fortrantestgenerator
- Cover-Agent: https://github.com/qodo-ai/qodo-cover
- Microsoft polyglot test agent: https://devblogs.microsoft.com/dotnet/polyglot-unit-testing-agent/ and https://github.com/dotnet/skills
- Copilot C++ `/tests`: https://devblogs.microsoft.com/cppblog/document-build-instructions-and-more-with-enhanced-c-awareness-from-copilot-chat-in-visual-studio/
- Copilot testing for .NET: https://learn.microsoft.com/en-us/visualstudio/test/github-copilot-test-dotnet-overview
