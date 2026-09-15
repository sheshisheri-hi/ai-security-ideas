## 2026-09-15

No strong new MCP SEP/protocol draft this window (2817/3140/3004/2643/2787 already queued; no post-09-08 security SEP with fresh sponsor heat). Soft mix: two papers + one recent MCP egress firewall + one monthly-momentum MCP context tool.

### 2026-09-15-a — ActGuard (pre-execution action auditing vs IPI)

- **Kind:** paper
- **Date:** submitted **14 Sep 2026** (arXiv:2609.14987)
- **Link:** [ActGuard: Pre-execution Action Auditing against Indirect Prompt Injection in LLM Agents](https://arxiv.org/abs/2609.14987)
- **Code:** [binzhwang/ActGuard](https://github.com/binzhwang/ActGuard) (paper artifact — **not** the unrelated ActGuard.org budget/SDK product)
- **Title:** ActGuard — audit the *action*, not the whole untrusted page
- **Track:** sample-ready
- **Status:** queued

#### What it is

Indirect prompt injection still wins when defenses either over-filter tool output (utility dies) or only harden prompts (ASR stays high). ActGuard flips the question: instead of “is this external content suspicious?”, it asks “does this content make the *upcoming* tool call deviate from a locally reasonable expectation?” At each step it builds a **local tool prior** (tools likely for the next action) without freezing a full plan, then before execution runs **tool-level contrast** plus **parameter-level evidence localization**. A verifier confirms which spans are malicious, masks only those, and regenerates the action from the sanitized context. Authors report ASR cut to SOTA-defense levels while task utility stays close to the no-attack baseline (treat numbers as author-reported until you re-run).

#### Why it is useful in the real world

MCP host/client stacks that already allowlist tools still execute poisoned *arguments* when tool results contain “please call X with Y.” Teams that tried blanket output scrubbing and lost agent usefulness need a **surgical** pre-dispatch gate. Pair with CapScope (who may call) and ROPE (origin of parameters): ActGuard is the “did this step go off-script?” layer.

#### Buildability

CPU weekend sample: mock a two-tool agent, inject a poisoned fetch result that steers a second call, implement a thin prior + parameter-span mask + regenerate stub (cite the paper; no claim of their full benchmarks). Full contrastive verifier needs LLM APIs — keep the LinkedIn slice on the **localize+mask** contract. No GPU required for the harness idea.

#### LinkedIn / demo

Strong: side-by-side “full scrub loses the task” vs “mask only the injection span, task completes.” Public sample + one-pager fit. Name the paper repo; disclaim the ActGuard.org SDK namesake.

#### Technical compare

- **CapScope (queued):** capability ceiling *before* untrusted reads; ActGuard audits the *candidate action* after context is already mixed.
- **ROPE (queued):** deterministic origin check on guarded parameters; ActGuard is deviation/evidence localization + selective mask.
- **OpenAPPA (queued):** destination information-flow policy; ActGuard is pre-exec *action* audit vs a local prior.
- **Content filters / SecAlign-class prompt defenses:** ActGuard’s stated delta is utility-preserving precision vs blanket sanitization.


### 2026-09-15-b — SkillSecurer (detect + patch skill prompt-injection)

- **Kind:** paper
- **Date:** submitted **12 Sep 2026** (arXiv:2609.14079)
- **Link:** [SkillSecurer: Detecting and Patching Prompt-Injection Vulnerabilities in AI Agent Skills](https://arxiv.org/abs/2609.14079)
- **Title:** SkillSecurer — red/blue agents that find *and* patch skill packs
- **Track:** sample-ready
- **Status:** queued

#### What it is

Agent skills (`SKILL.md` + scripts + config) are a new install surface: reusable instructions that can smuggle nine threat types of context-compatible injections. SkillSecurer is a fully agentic pipeline: a **red agent** generates injections and records exact edits; a **blue agent** analyses complete skill packages, grounds evidence, and proposes patches; a **verifier** scores findings against the recorded injection for controlled instances. Authors report **100%** injection detection with their best backend (only scanner in their compare to hit that), and latent vulns in **>17%** of popular skills examined on [skills.sh](https://skills.sh), including some that triggered real incidents in testing. Treat rates as author-reported.

#### Why it is useful in the real world

Same moment as npm for Cursor/Claude/Codex skill packs: developers install third-party skills into MCP host/client environments with almost no review. SkillSpector (yesterday) is the shippable pre-install scanner; SkillSecurer’s research wedge is **localization + remediation patches**, not only a score.

#### Buildability

CPU weekend: fixture skill pack + planted injection + a thin “blue” rule/LLM pass that emits a patch diff and a verifier that checks the planted span (cite SkillSecurer; don’t rebuild the full red/blue loop). Full nine-threat campaign needs API budget — LinkedIn slice stays on detect→patch→verify for one threat class.

#### LinkedIn / demo

Strong: “17%+ skills flagged — here is the red finding and the blue patch side-by-side.” Public sample + one-pager fit; link SkillSpector as the install gate and SkillSecurer as the research remediation path.

#### Technical compare

- **NVIDIA SkillSpector (queued, ~17.2k★):** deep pattern scanner + optional LLM stage for install CI; SkillSecurer is **agentic detect+patch+injection-level eval**.
- **OpenTrustBench (queued):** light Trust Card / badge CI; SkillSecurer is remediation-oriented research.
- **SIGIL / SkillGuard / SkillShift (queued papers):** audit–runtime gap, reachability, policy steering; SkillSecurer operationalizes **package-level red/blue**.
- **v4scan (queued):** deterministic install scanner; SkillSecurer adds grounded patches and a recorded-injection verifier.


### 2026-09-15-c — Pipelock (MCP/A2A egress firewall + signed receipts)

- **Kind:** tool / recent implementation
- **Date:** latest release **v3.5.0** published **1 Sep 2026**; active pushes through **15 Sep 2026**
- **Link:** [luckyPipewrench/pipelock](https://github.com/luckyPipewrench/pipelock)
- **Title:** Pipelock — open-source AI agent firewall for MCP / HTTP / A2A / WebSocket egress
- **Language / license:** Go; **Apache-2.0**
- **Stars:** **859** at briefing time (~101 forks); steady security-tool momentum + same-day commits (not the mega-viral lane)
- **Track:** sample-ready
- **Status:** queued

#### What it is

Pipelock sits **outside** the agent as an egress mediator: scans mediated HTTP, WebSocket, MCP, and A2A traffic (plus CONNECT contents when TLS interception is on) for secret exfil, prompt injection, SSRF, tool poisoning, and risky tool-call chains. Differentiator vs “another gateway README”: it emits **mediator-signed action receipts** — verifiable audit evidence from outside the agent runtime — with public [agent-egress-bench](https://github.com/luckyPipewrench/agent-egress-bench) and a scheduled Gauntlet workflow. v3.5.0 hardens fail-closed paths (unsafe trailers, opaque MCP args, immutable DLP floors) and ships verifier binaries alongside the proxy.

#### Why it is useful in the real world

MCP host/client deployments that only gate `tools/list` still lose to SSRF/exfil at call time. Compliance teams that need “prove what left the box” want receipts CI can verify, not chat logs the agent wrote about itself.

#### Buildability

CPU weekend LinkedIn path: **EgressReceiptCI** — thin Action/Python that verifies a Pipelock (or fixture) receipt chain and fails merge when high-risk egress lacks a valid mediator signature (wrap published `pipelock-verifier_*` or a mocked receipt schema). Full proxy + TLS intercept is ops-heavy — don’t clone the firewall for a sample.

#### LinkedIn / demo

Strong: “agent said OK / receipt says blocked” split-screen; public sample verifies receipts only. Cite Pipelock; keep your repo the CI gate.

#### Technical compare

- **Bifrost / mcp-visor / mcp-doorman class:** policy proxies; Pipelock’s delta is **signed outbound receipts + egress bench**.
- **ATSA / call-attestation samples (shipped):** MCP server admission / call labels; Pipelock attests **network-boundary decisions**.
- **OpenAPPA (queued):** in-process flow algebra; Pipelock is an **out-of-process** mediator.
- **SkillSpector / discovery sanitizer:** install/discovery time; Pipelock is **runtime egress**.


### 2026-09-15-d — Ripwire (MCP “ripgrep of AI context” + published loss)

- **Kind:** tool / monthly-trending
- **Date:** created **29 Jul 2026**; release **v0.6.1** published **15 Sep 2026** (same-day push)
- **Link:** [redhat-et/ripwire](https://github.com/redhat-et/ripwire)
- **Title:** Ripwire — zero-dependency C++23 CLI + MCP server; signatures first, every loss labelled
- **Language / license:** C++; **Apache-2.0**
- **Stars:** **2153** at briefing time (~128 forks)
- **Why stars are rising:** Red Hat Emerging Technologies backing; agents need map/blast-radius/tests-to-run without stuffing whole files into context; **v0.6.1** same-day release narrative (“every guess labelled, every loss published”; claimed **~74.7%** fewer bytes for signatures vs bodies — author/README figure); ships as MCP server tools agents can call.
- **Track:** sample-ready
- **Status:** queued

#### What it is

Ripwire is a coding-agent context engine exposed as CLI + **MCP server**: callers/callees, impact, tests-to-run, quality deltas, compact legends — oriented around **signatures and disclosures** rather than silent full-file reads. The security-relevant contract is honesty under compression: answers carry gauges like `unproven_defs=`, `merge_bombs_skipped=`, and compact legends that still define printed attributes. v0.6.1 tightens header→definition proof, agent-surface compact defaults, and scoped “what changed recently” windows.

#### Why it is useful in the real world

Agents that `cat` entire trees burn tokens and hide incomplete graph reads. Teams running MCP host/client coding agents want a **least-read** retrieval MCP server with explicit loss labels — useful for both cost and for catching “clean-looking” answers that dropped proof.

#### Buildability

CPU weekend: **LossPublishedGate** — harness/CI that consumes ripwire (or fixture) MCP traces and fails when an agent answer used body reads without a published loss label, or when signature→body promotion exceeds a budget. Don’t reimplement the C++ indexer; wrap the contract. No GPU.

#### LinkedIn / demo

Strong: “same question — full body dump vs signature+loss gauges; gate fails the silent promotion.” Public sample + one-pager fit; link the ripwire MCP server.

#### Technical compare

- **GraphBudget / context compressors (prior backlog lanes):** plan token budgets; Ripwire is a **shipped retrieval MCP server** with loss gauges.
- **zvec-grep / codebase-memory MCP class:** search/memory products; Ripwire’s delta is **published-loss / proof gauges** on answers.
- **CapScope (queued):** authorize tool calls; Ripwire reduces how much untrusted body text enters context in the first place.
- **SkillSpector (queued):** skill install scanner; Ripwire is **runtime context retrieval** for coding agents.
