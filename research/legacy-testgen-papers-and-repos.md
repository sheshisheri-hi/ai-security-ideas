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

A 2026 follow-up paper ([arXiv:2601.09695](https://arxiv.org/abs/2601.09695)) found that **plain newer LLMs often beat the 2023–24 specialist test-gen tools**. That supports staying on Copilot rather than buying a Java-era generator.

---

## Papers that generate tests (or measure that)

| Paper | Year | Link | Languages | Creates tests? | Tomorrow? |
| --- | --- | --- | --- | --- | --- |
| CITYWALK | 2025 (TOSEM) | [arXiv:2501.16155](https://arxiv.org/abs/2501.16155) | C++ | Yes (LLM + dependency analysis, GPT-4o) | No. Research artifact on [Zenodo](https://zenodo.org/records/14022506), not a VS plugin. Sends code to GPT-4o. **Steal the workflow:** dump include/lib graph, paste C++ rules, compile-repair. |
| CPP-UT-Bench | 2024 | [arXiv:2412.02735](https://arxiv.org/abs/2412.02735) | C++ | Benchmark, not a tool. Fine-tuned weights for research. | No. Use as a warning: vanilla LLMs are weak on compiled C++. Few-shot examples inside Copilot are the only office use. |
| SPARC (scenario planning for C tests) | 2026 | [arXiv:2602.16671](https://arxiv.org/abs/2602.16671) | C | Yes (CFG + LLM + compiler repair). Beats vanilla prompting; competitive with KLEE on their 59 subjects. | No product. Do **not** confuse with SPARC-X DFT (`SPARC-X/SPARC`). GitHub for *this* SPARC: **UNKNOWN** (no verified product repo). |
| How well LLM test-gen techniques perform with newer models? | 2026 | [arXiv:2601.09695](https://arxiv.org/abs/2601.09695) | Java-heavy eval of HITS/CoverUp/TestSpark | Evaluates, does not ship a tool | Read. Newer base models often beat specialized 2023–24 tools. |
| FortranTestGenerator | 2017 | [paper](https://www.wr.informatik.uni-hamburg.de/_media/research/publications/2017/dlfaafutgflh17-fortrantestgenerator_automatic_and_flexible_unit_test_generation_for_legacy_hpc_code.pdf) + [fortesg/fortrantestgenerator](https://github.com/fortesg/fortrantestgenerator) | Fortran 90+ | Yes: capture/replay **drivers** from a live run. Developer still writes checks. Needs gfortran assembler + Serialbox2. Last GitHub push **2019-11-04**, GPL-3.0, 24★. | Maybe week 2–3 on one kernel, not tomorrow. Intel Fortran / MSBuild path is extra work. |
| LLM unit-test survey | 2025 | [arXiv:2511.21382](https://arxiv.org/abs/2511.21382) | mostly Java/Python | Survey. Symbolic tools fail on floats, libraries, path explosion. | Read, don’t install. |
| Julienne (Fortran 2023 testing idioms) | 2025 | [OSTI](https://www.osti.gov/servlets/purl/3000200) | Fortran | Harness/assertions, not an LLM generator. | Later alternative to pFUnit. |
| Coyote C++ (Codemind) | 2023–24 | [arXiv:2310.14500](https://arxiv.org/abs/2310.14500) | C, C++ | Yes (industrial concolic: harness + cases + asserts) | **Procurement**, not OSS. Windows + VS per vendor FAQ. No Fortran/C#/WPF. |
| Meta TestGen-LLM | 2024 | industry paper (Cover-Agent cites it) | Java/Kotlin at Meta | Creates tests in their CI. | Not your languages as a product. |
| CoverUp, TestPilot, ChatTester, ChatUniTest, HITS, MuTAP, CAT-LM | 2023–25 | see sources | Python / Java / JS | Yes | **Skip.** Wrong language. ChatTester’s compile-repair loop is what Copilot `@Test` already does for C#. |

---

## GitHub repos that create tests

| Repo | Creates tests? | Languages that match | License / status | Windows / VS? | Fit |
| --- | --- | --- | --- | --- | --- |
| [dotnet/skills](https://github.com/dotnet/skills) `code-testing-generator` | **Yes.** Research-plan-implement agent: detect framework, write tests, build, repair. Polyglot list includes **C#** (MSTest/xUnit/NUnit) and **C++** (GoogleTest/Catch2/doctest/Boost.Test). | MIT. Open-sourced ~Aug 2026. 5330★ (2026-09-02). | Copilot CLI + VS Code plugin now. Visual Studio: Microsoft said they are working on it. | **Best extra** after native `@Test` / `/tests`. Same Copilot seat. No extra LLM vendor. |
| [UnitTestBot/UTBotCpp](https://github.com/UnitTestBot/UTBotCpp) | **Yes.** Writes **Google Test** files from KLEE-style analysis (regression + error suites). | Apache-2.0. 185★. Last push **2024-10-18**. | **No.** README: Ubuntu 20.04+ server. VS Code client can be Windows; analysis box is Linux. | Best OSS C/C++ *file writer* found. Not tomorrow (Linux VM later, if ever). |
| [qodo-ai/qodo-cover](https://github.com/qodo-ai/qodo-cover) (Cover-Agent) | **Yes.** Iterates until coverage target. Integration fixtures exist for **C, C++, C#**. | AGPL-3.0. README: **no longer maintained** (2025-06-15). ~5.6k stars. Last release 0.3.10 (May 2025). Successor `qodo-ci` languages: Python/PHP/Java/Go/Kotlin/JS/TS — **no C/C++/C#/Fortran**. | CLI + coverage XML. Needs an LLM key (not Copilot). | Do not adopt. |
| [fortesg/fortrantestgenerator](https://github.com/fortesg/fortrantestgenerator) | **Yes** (capture + replay driver). Not LLM. | GPL-3.0. Last push 2019. 24★. | Linux/HPC, gfortran assembler + Serialbox2. | Fortran characterization ancestor. Too stale to install tomorrow. |
| [klee/klee](https://github.com/klee/klee) | Inputs / path tests for LLVM bitcode, not GTest files. | Alive (push 2026-08). ~3k★. | Linux/LLVM. Floats and external libs fail. | Skip as QE product. |
| [approvals/ApprovalTests.cpp](https://github.com/approvals/ApprovalTests.cpp) / [ApprovalTests.Net](https://github.com/approvals/ApprovalTests.Net) | Writes **gold files** once you author a test that calls `Approvals.Verify`. Copilot can write that one-liner. | Apache-2.0. | Yes (C++ and .NET). | Use with Copilot for solvers. Not a generator by itself. |
| [Goddard-Fortran-Ecosystem/pFUnit](https://github.com/Goddard-Fortran-Ecosystem/pFUnit) | **No** (framework). Copilot writes `@test` procedures. | NASA, CMake. Intel ifx listed. | Possible with Intel Fortran. | Week 2–3 harness. |
| CITYWALK | Artifact zip on Zenodo, not a maintained product repo. | Research. | GPT-4o. | Skip the zip. Copy the prompt style. |

**Not generators for this shop:** Agentic-QE, Diffblue Cover (Java), IntelliTest (deprecated VS 2026), Qodo `/test` product, WinAppDriver, Coded UI, EvoSuite, Pynguin, TestSpark, ChatUniTest, CoverUp, TestPilot (archived), Playwright, AFL++/libFuzzer (inputs, not unit-test files).

---

## Ranked shortlist for this office

1. **Copilot `@Test` (C#) and `/tests` (C++)** — already paid, in Visual Studio, creates tests. Tomorrow.
2. **Microsoft `code-testing-generator`** (`dotnet/skills`) — same Copilot, better loop (write/build/repair), C++ frameworks named. Lab in Copilot CLI or VS Code after (1) works. Wait for VS if you will not leave Visual Studio.
3. **CITYWALK *as a Copilot prompt***, not the Zenodo prototype — dump MSBuild/CMake includes, name Catch2/GTest, compile-repair. That is the paper’s contribution.
4. **Characterization tests** — freeze 5–10 solver decks as `tests/gold/`. Copilot writes the Catch2/`EXPECT_NEAR` or ApprovalTests harness. This is how Fortran and numerical C++ get tests without a fake oracle.
5. **Parking lot (not tomorrow):** VectorCAST ATG, Parasoft C/C++test, or Codemind Coyote C++ if a later budget exists for C/C++ structural coverage. UTBotCpp on a Linux VM only if you want symbolic GTest without an LLM. None cover Fortran or WPF. None beat Copilot on C# in VS 2026.

Do not start CITYWALK-the-zip, SPARC-the-paper, KLEE, Cover-Agent, or AQE for this `.sln`.

---

## Gaps that no paper closes

- **Floating-point oracles.** Symbolic tools and LLM `ASSERT_EQ` both lie. Gold files + tolerances are still the method. Excvate / KLEE-Float / NuMFUZZ produce **inputs**, not oracles.
- **Fortran.** Capture/replay (2017, stale) or a human-written pFUnit test. Fortran2CPP (arXiv:2412.19770) generates asserts for *translation*, not a shop suite.
- **WPF.** Zero academic or GitHub generators for `MainWindow.xaml`. Test ViewModels. FlaUI smokes later.
- **Mixed MSBuild + CMake + Intel Fortran.** Every research tool assumes one build graph (CMake+g++, Maven, `dotnet test`). Yours is not that.
- **Privacy.** Almost every 2024–26 LLM paper uses GPT-4o. Do not ship solver sources to CoverUp/ChatUniTest/ConcoLLMic.

---

## Sources

- CITYWALK: https://arxiv.org/abs/2501.16155 — ACM TOSEM 2025, doi 10.1145/3763791
- CPP-UT-Bench: https://arxiv.org/abs/2412.02735
- SPARC (C test gen): https://arxiv.org/abs/2602.16671
- Newer-LLM eval: https://arxiv.org/abs/2601.09695
- LLM unit-test survey: https://arxiv.org/abs/2511.21382
- Coyote C++: https://arxiv.org/abs/2310.14500
- FortranTestGenerator: https://github.com/fortesg/fortrantestgenerator
- UTBotCpp: https://github.com/UnitTestBot/UTBotCpp
- Cover-Agent: https://github.com/qodo-ai/qodo-cover
- Microsoft polyglot test agent: https://devblogs.microsoft.com/dotnet/polyglot-unit-testing-agent/ and https://github.com/dotnet/skills
- Copilot C++ `/tests`: https://devblogs.microsoft.com/cppblog/document-build-instructions-and-more-with-enhanced-c-awareness-from-copilot-chat-in-visual-studio/
- Copilot testing for .NET: https://learn.microsoft.com/en-us/visualstudio/test/github-copilot-test-dotnet-overview
- VectorCAST C: https://www.vector.com/en/product/vectorcast/vectorcast-c/
- Parasoft C/C++test: https://www.parasoft.com/products/parasoft-c-ctest/
