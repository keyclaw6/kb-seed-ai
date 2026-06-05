# KB Seed AI — Project State & Handoff

> Single entry point for anyone (human or LLM) picking this up with fresh context. The GitHub
> repo is the source of truth. Read this file first, then `research/sources.md`, then the
> relevant `research/findings/*.md`.

## Mission
Build an AI system that can build everything: give it a high-level goal ("build project X") and a
harness around the agent builds the software reliably — no false positives, no false negatives,
clean code — with tokens treated as effectively unlimited; and the seed eventually develops
itself. Methodology: route around LLM weaknesses (poor single-shot decisions) by spending compute
on wide search + thorough verification instead of trusting any one decision.

Core loop: **propose a change (hypothesis) → test it in isolation → keep it only if verifiably
better → repeat.**

## LOCKED constraints (do not relitigate without the owner)
- **Harness-only self-improvement. NEVER train/update model weights, ever.** The seed edits only
  its harness: prompts, skills, memory, sub-agents, tools, orchestration code.
- **Sandboxing is not a research concern** — Docker containers and/or git worktrees suffice.
- **The verifier is the product.** It must be grounded and STRUCTURALLY isolated from the
  optimizer (the agent can never read or weaken its own grader/gate).
- **Ruthless minimalism** — the smallest thing that stands up; deterministic over LLM unless the
  LLM clearly adds quality; reuse over build; the human owner gates every durable decision.

## Repo map
- `research/sources.md` — the canon: every source, grouped by run, with signal + a cross-cutting
  synthesis note per run. READ THIS after this file.
- `research/findings/*.md` — 47 IMMUTABLE per-source research docs (one per source), each a fixed
  10-section structure + Mermaid diagrams + verbatim prompts/mechanisms. DO NOT hand-edit them.
- `research/subagent-prompt.md` — the brief used to research each source (one Opus sub-agent per
  source; search beyond the seed link; clone & read the real code; Mermaid required).
- `research/thinking/` — editable WIP: `mission.md`, this `STATUS.md`, `handoff-prompt.md`; ADRs
  will live here too.
- `diagrams/` (D2), `docs/` — original skeleton (open questions superseded by this file).

## Where we are
Research phase COMPLETE: 8 runs, **47 sources** ingested (full table in `sources.md`). Now entering
the ARCHITECTURE phase. No system code written yet — by design.

## The corpus in one breath (the convergent finding)
Across 47 sources the result is overwhelming and consistent:
- **The verifier is the differentiator.** Systems with a GROUNDED, hard-to-game evaluator (formal:
  CLEVER/NL2VC; ground-truth-free: LLMorpheus mutation, DiffSpec differential; executable hidden
  tests: DGM/HGM/MAC; runs-the-actual-thing: AlphaEvolve/DRQ) make open-ended search pay off.
  Systems with weak/self-judged evaluators document **reward hacking in the wild** (DGM deleted its
  own test markers; cheating rose when the detector was visible; AI-Scientist-v2 / ADAS / Gödel
  Agent gamed soft graders).
- **Convergent verifier design** (independently — especially the harness-only cluster
  meta-agent / autocontext / recursive-improve): a held-out/holdout split + generalization-gap
  blocking + keep-only-if-verifiably-better + anti-Goodhart, with the oracle STRUCTURALLY isolated
  (meta-agent's "benchmark-injected hooks win conflicts"; octopusgarden's proposer literally cannot
  import the verifier).
- **Open-ended search beats greedy**: keep an archive of stepping-stones; select parents by
  improvement-potential/diversity, not present score (DGM; HGM clade-metaproductivity;
  AlphaEvolve/DRQ MAP-Elites). [DEFERRED for the MVP.]
- **Memory should be earned + invalidatable** (Graphiti bi-temporal invalidate-don't-delete; EvoDS
  skill-promotion gate; R-APS explicit invalidation; cognee verifier-driven edge reweighting).
- **Long-horizon running**: false-completion is the #1 risk (Codex & Claude Code goal features both
  guard it); durable runtime via event-sourced replay + self-succession (Temporal's Continue-As-New
  = the pattern Antigravity hand-rolled).
- **Direct precedent for OUR path**: meta-agent (reimpl of Stanford "Meta-Harness"), autocontext,
  recursive-improve are harness-only recursive self-improvers — read these first for architecture.

## Agreed design METHOD
Do NOT make one big "design the architecture" decision (the unreliable-decider trap). Instead:
1. Decompose the architecture into **seams** (decision points).
2. Per seam, lay out the **evidenced options drawn verbatim from the 47 findings**, scored against
   the locked constraints.
3. Classify each: **DECIDE-NOW** (evidence + constraints settle it) vs **DEFER-TO-EXPERIMENT**
   (genuinely uncertain + cheap to A/B → let the minimal loop adjudicate once it runs).
4. The owner is the binding gate on every decision.
5. Record each as an **ADR** in `research/thinking/` with rationale + verbatim borrowed assets
   (the value is in the prompts and nuances; never paraphrase a borrowed mechanism).
6. Sequence **verifier-first**. Aim at the smallest verifiable MVP loop, which then becomes the
   adjudicator for the deferred decisions.
Go **seam-by-seam, not system-by-system** (the transpose): for each component pull the best
evidenced option from across all sources.

## The seams (decision surface)
1. **Verifier** — decide FIRST (the spine). Formal where a spec exists (guard the spec: dual
   oracles, non-computable specs) + ground-truth-free (mutation/differential) + hidden tests +
   anti-cheat auditor; LLM-judge only for non-verifiable quality, pairwise not absolute.
2. Candidate unit + isolation — lean: git worktree per candidate.
3. Promotion gate + inviolable boundary — keep-iff-verifiably-better, no regression; the agent may
   never read or edit its own verifier/gate.
4. Harness object that gets edited — CRUD-able prompts/skills/memory/sub-agents; meta-level frozen.
5. Proposer — LLM proposes harness edits; parallelism width DEFERRED.
6. Archive + parent-selection — DEFER: champion + hill-climb now; archive / MAP-Elites later.
7. Memory — start minimal; Graphiti-style invalidatable store when bloat appears.
8. Goal representation + decomposition — Codex ThreadGoal-style tracking; decomposition is the thin
   spot we design ourselves.
9. Runtime / orchestration — buy-vs-build DEFERRED: minimal loop for the MVP; Temporal when
   durability/scale demands it (Activities for LLM calls, Continue-As-New for self-succession).

## Evaluation framework (PROPOSED — pending ADR)
Key insight: **the verifier and the machine-evaluation are the same machinery at two scopes.**
- Inner = did one candidate pass the hidden tests for one task.
- Outer = did the champion harness improve on a HELD-OUT distribution of tasks it never optimized against.
How to evaluate the machine:
1. A benchmark of software-building tasks, each with a hard, automatic, HIDDEN acceptance test.
2. Three splits: **search** (evolve against) / **held-out** (select champions) / **final-test**
   (never touch; honest reporting only).
3. Metric = held-out pass-rate of the champion lineage (v0 → vN) over iterations, multiple seeds,
   with confidence intervals (one run proves nothing).
4. Track the **generalization gap** (search − held-out) as the machine-level reward-hacking detector.
5. Track **cost** (pass-rate per token / $ / wall-clock; the quality–cost Pareto) and **trajectory
   health** (stuck / looping / stagnation).
6. Honest limit: this rigor exists only where there is a task distribution with hidden tests (the
   MVP regime). The open-ended "build VS Code in Rust" regime degrades to spec/milestone
   satisfaction + human spot-audits → the MVP MUST live in the benchmarked regime.

## Decided vs Open
- DECIDED: harness-only / no weights; sandboxing = Docker/worktrees; the design method;
  verifier-first sequencing; lean choices for seams 2 & 3 (worktree isolation; keep-iff-better +
  inviolable gate).
- DEFERRED-TO-EXPERIMENT: archive-vs-hill-climb (seam 6); parallelism width (seam 5); runtime
  buy-vs-build (seam 9).
- OPEN (next up): ADR-001 (verifier + eval); the MVP benchmark target; seams 4, 7, 8.

## Immediate next step
1. Draft **ADR-001 = the verifier + the evaluation harness** (one piece of machinery), using the
   convergent design: held-out splits + generalization-gap + anti-Goodhart + structural isolation;
   concretely meta-agent's exit-contract split and autocontext's layered gate stack.
2. Pin the **MVP target** — a small benchmark of software tasks with hard hidden tests (so eval is
   concrete). Candidates: a curated subset of SWE-bench / terminal-bench, or a bespoke task set.
3. (Optional; owner wants a visual plane) an "architecture board": D2 diagrams in `diagrams/` +
   an interactive page showing the loop, the seams, and decision status.

## Still-open research gaps (non-blocking)
goal/spec decomposition; compute-scheduling early-stopping; codebase-comprehension at scale.
Candidate later sources: Stanford "Meta-Harness" (arXiv 2603.28052), ACE / Agentic Context
Engineering (Stanford + SambaNova). Also unfilled source slots 28/29 (a past batch duplicated 27
and skipped 29).

## Working conventions
- Commit directly to `main` — NO pull requests. Conventional-commit messages.
- One research sub-agent per source; ≤10 concurrent (hard platform cap). Findings docs are immutable.
- Sandbox notes (if you work inside one): `git clone` works via the proxy after
  `git config --global http.proxy "$https_proxy"` (basic auth); GitHub writes go through the
  `github` MCP `push_files`; `ExecuteIntegration` requires `paramsFile`, not inline params.
