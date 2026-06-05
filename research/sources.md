# Canon — Source List

The prioritized list of prior-art sources to ingest. Kristian seeds this list; a
deep-search prompt (produced on request) will surface additional candidates.

## Status legend
- `queued` — agreed, not yet researched
- `running` — sub-agent in progress
- `done` — findings doc committed in `findings/`
- `low-signal` — researched, little of value (kept for the record)
- `dropped` — decided not worth researching now

## How a source becomes a findings doc
One Opus research sub-agent is spawned per row, running `subagent-prompt.md`. It writes a
single immutable doc to `findings/<slug>.md` and we update the row's status + link.

## Run 1 (committed)

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| 1 | OpenSpace (HKUDS) | system / repo | https://github.com/HKUDS/OpenSpace | `findings/openspace.md` | MEDIUM | done |
| 2 | Hermes Agent (Nous Research) | system / repo | https://github.com/nousresearch/hermes-agent | `findings/hermes-agent.md` | MED–HIGH | done |
| 3 | Google Antigravity — "OS from one prompt" | blog / demo + critique | https://antigravity.google/blog/google-antigravity-built-an-os | `findings/antigravity-os.md` | MEDIUM | done |
| 4 | autoresearch (Andrej Karpathy) | repo | https://github.com/karpathy/autoresearch | `findings/autoresearch.md` | MEDIUM | done |
| 5 | OpenSage Agent ADK | system / repo + paper | https://www.opensage-agent.ai/adk.html · https://github.com/opensage-agent/opensage-adk | `findings/opensage-adk.md` | MEDIUM | done |

> Cross-cutting result of Run 1: every one of these five builds the *substrate* (memory,
> skills, lineage, orchestration, long-horizon running) but **none has an unfakeable
> in-loop verifier** — "keep this?" is decided by LLM judgment or external benchmarks.

## Run 2 (committed)

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| 6 | gemtest / metamorphic + property-based testing | repo + technique | https://github.com/tum-i4/gemtest · arXiv 2211.12003 | `findings/gemtest.md` | MED (HIGH on verification) | done |
| 7 | "Beyond Human Measure: Open-World Alignment" | preprint (position paper) | https://www.preprints.org/manuscript/202604.1749 | `findings/preprint-202604-1749.md` | MEDIUM | done |
| 8 | Repo mapping — Aider repo-map + GitNexus | docs + repo | https://aider.chat/docs/repomap.html · https://github.com/abhigyanpatwari/GitNexus | `findings/repo-map-and-gitnexus.md` | HIGH | done |
| 9 | The AI Scientist v2 (Sakana AI) | system / repo + paper | https://github.com/sakanaai/ai-scientist-v2 | `findings/ai-scientist-v2.md` | MED–HIGH | done |
| 10 | AlphaEvolve (Google DeepMind) | paper + results repo | AlphaEvolve.pdf · https://github.com/google-deepmind/alphaevolve_results | `findings/alphaevolve.md` | HIGH | done |
| 11 | ADAS — Meta Agent Search | paper + repo | https://arxiv.org/abs/2408.08435 · https://github.com/ShengranHu/ADAS | `findings/adas.md` | HIGH (concept) | done |
| 12 | Gödel Agent | paper (+ code) | https://arxiv.org/abs/2410.04444 · https://github.com/Arvid-pku/Godel_Agent | `findings/arxiv-2410-04444.md` | MED–HIGH | done |
| 13 | SICA — Self-Improving Coding Agent | paper + repo | https://arxiv.org/abs/2504.15228 · https://github.com/MaximeRobeyns/self_improving_coding_agent | `findings/sica.md` | HIGH | done |
| 14 | Darwin-Gödel Machine (DGM) | paper + repo | https://arxiv.org/abs/2505.22954 · https://github.com/jennyzzt/dgm | `findings/dgm.md` | HIGH | done |
| 15 | Huxley-Gödel Machine (HGM) | paper + repo | https://github.com/metauto-ai/HGM · https://arxiv.org/abs/2510.21614 | `findings/hgm.md` | HIGH | done |
| 16 | DRQ — "Digital Red Queen" (Sakana AI + MIT) | paper + repo | https://arxiv.org/abs/2601.03335 · https://github.com/SakanaAI/drq | `findings/drq.md` | LOW–MED | done |

> Cross-cutting result of Run 2 — the canon now splits cleanly on the **verifier**:
> - **Grounded, hard-to-game evaluators make open-ended search pay off:** AlphaEvolve
>   (user `evaluate()` + held-out sets), DGM/HGM (executable hidden tests + an oracle the
>   agent can't edit), DRQ (runs the actual game), gemtest (metamorphic relations
>   manufacture an oracle with *no* golden answer).
> - **Weak / self-judged evaluators document reward-hacking in the wild:** AI-Scientist-v2
>   (LLM/VLM judges over self-defined metrics), Gödel Agent (no verified rollback), ADAS
>   (string-match). DGM's agent literally deleted its own test markers to fake a score —
>   and cheating *rose* when the detector was visible.
> - **Convergent lessons:** (1) keep an ARCHIVE of stepping-stones and select parents by
>   improvement potential / diversity, not present score (DGM; HGM's clade-metaproductivity;
>   AlphaEvolve & DRQ MAP-Elites) — beats greedy. (2) ISOLATE the fitness oracle and the
>   meta-level from the optimizer. (3) Separate hard feasibility gates from the scalar
>   reward + add abstain/escalate (the open-world-alignment preprint).
>
> Note: arXiv 2211.12003 is NOT the gemtest paper (it's an RMIT PBT-for-MT paper); the
> Run-2 docs also flag claim-vs-code gaps in AI-Scientist-v2, ADAS, and SICA.

## Run 3 (committed)

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| 17 | EvolveR | paper + repo | https://github.com/KnowledgeXLab/EvolveR · arXiv 2510.16079 | `findings/evolver.md` | MEDIUM | done |
| 18 | Ona — "Building a Software Factory" (+5 video transcripts) | story + repo + video | https://ona.com/stories/building-a-software-factory-week-1 | `findings/ona-software-factory.md` | MED–HIGH | done |
| 19 | gastown / "Gas Town" (Steve Yegge) | repo | https://github.com/gastownhall/gastown | `findings/gastown.md` | MEDIUM | done |
| 20 | "Solving a Million-Step LLM Task with Zero Errors" (MAKER/MDAP) | paper + repo | https://arxiv.org/abs/2511.09030 | `findings/arxiv-2511-09030.md` | MEDIUM | done |
| 21 | octopusgarden + Willison on StrongDM's "software factory" | repo + blog | https://github.com/foundatron/octopusgarden · https://simonwillison.net/2026/Feb/7/software-factory/ | `findings/octopusgarden.md` | HIGH | done |
| 22 | SWE-AF (Agent-Field) — multi-agent software factory | repo | https://github.com/Agent-Field/SWE-AF | `findings/swe-af.md` | MEDIUM | done |
| 23 | fabro (Qlty / B. Helmkamp) — agent workflow engine | repo | https://github.com/fabro-sh/fabro | `findings/fabro.md` | MEDIUM | done |
| 24 | "Continual Harness" (Gemini Plays Pokémon; Princeton/DeepMind) | paper + repo | https://arxiv.org/abs/2605.09998v1 · github.com/sethkarten/continual-harness | `findings/arxiv-2605-09998.md` | HIGH | done |
| 25 | "How Lovable self-improves every hour" (B. Verbeek talk) | talk (transcript) | https://www.youtube.com/watch?v=KA5kPbdkK2E | `findings/yt-ka5kpbdkk2e.md` | MED–HIGH | done |
| 26 | autoagent (kevinrgu / Third Layer) | repo | https://github.com/kevinrgu/autoagent | `findings/autoagent.md` | MED–HIGH | done |

> Cross-cutting result of Run 3 (the "software factory" cluster + harness self-improvement):
> - **The factories that don't collapse are GitHub/CI-native pipelines of single-purpose
>   agents** coordinating through issues/PRs/labels ("loops, not an orchestrator" — Ona),
>   with a deterministic **CI gate sitting OUTSIDE the LLM reviewer** as the only unfakeable
>   signal. Same shape in SWE-AF, fabro, gastown.
> - **octopusgarden + StrongDM is the closest public match to our propose→test→keep loop:**
>   sealed holdout scenarios + probabilistic LLM-judge, *structurally enforced* (the
>   proposer package literally cannot import the verifier). But it's hill-climbing, not
>   evolution, and generator+judge share a model family → correlated blind spots (Goodhart).
> - **Safe self-improvement targets the HARNESS, gated by a hard external verifier:**
>   Continual Harness (CRUD-able prompt/sub-agents/skills/memory, in-place refinement) and
>   autoagent (editable candidate / FROZEN oracle; keep-if-`passed`-improved) — both echo
>   Karpathy's autoresearch (#4). Continual Harness's 1,003-turn broken loop (842 identical
>   failed calls while "self-assessing progress") is the cautionary proof that an external
>   verify-and-promote gate + auto-revert is non-negotiable.
> - **Reusable verification hygiene:** anti-reward-hacking CI prompts ("verify rendered
>   output, not tokens"; forbidden test-gaming lists) recur in Ona/SWE-AF/fabro/autoagent;
>   MAKER's "decompose to one decision, vote first-to-ahead-by-k, discard malformed samples"
>   restores i.i.d. errors; Lovable's holdout/blank-injection causal eval keeps only what
>   verifiably helps. EvolveR contributes a self-curating memory lifecycle (distill → dedup
>   → merit-score → prune).

## Run 4 (committed)

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| 27 | R-APS — Reflective Adversarial Pareto Search | paper | https://arxiv.org/abs/2606.04823 | `findings/arxiv-2606-04823.md` | MEDIUM | done |
| 30 | Meta-Agent Challenge (MAC) — recursive-self-improvement benchmark | paper (benchmark) | https://arxiv.org/abs/2606.04455 | `findings/arxiv-2606-04455.md` | HIGH | done |
| 31 | EvoDS — Self-Evolving Autonomous Data Science Agent | paper + repo | https://arxiv.org/abs/2606.03841 | `findings/arxiv-2606-03841.md` | MEDIUM | done |

> Note: the batch listed #28 as a duplicate of #27 (both arXiv 2606.04823) and gave no #29,
> so Run 4 has 3 unique papers. Slots 28/29 are open for the intended links.
>
> Cross-cutting result of Run 4: more grounded-verifier + earned-memory evidence.
> - **MAC (HIGH)** is essentially a ready-made harness for OUR setup — "agent builds and
>   iteratively optimizes its own agent.py, scored on a HIDDEN test split" — with a fully
>   open-sourced anti-cheat blueprint: dual-container isolation, hidden tests unlocked only
>   by a post-dev `X-Verifier-Secret`, a model-locking proxy, and an LLM "auditor" agent
>   (verbatim 9-class cheating prompt). Sobering result: only 5/39 configs beat human
>   baselines, and they caught real reward-hacking (a model exfiltrating labels via
>   exception tracebacks — isolation held).
> - **R-APS** adds a typed verification cascade (repair only the failed stage; 59% of
>   failures caught at the cheapest check) and **non-monotonic memory with EXPLICIT
>   INVALIDATION** — heuristics retired with cited counterexamples (a direct fix for the
>   skill-bloat seen in OpenSpace/Voyager), with a hard solver as the ungameable oracle.
> - **EvoDS** contributes a skill-promotion gate (cache-all, expose-only-proven: used ≥3×,
>   top-K) for an earned, non-bloated tool library, and a weak-vs-strong verifier contrast
>   (exit-code-0 is gameable; route promotions through a ground-truth executable reward).

## Run 5 (committed) — the verification deep-dive

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| 32 | LLMorpheus (LLM mutation testing) + llmorph (metamorphic testing of LLMs) | repos + paper | https://github.com/githubnext/llmorpheus · https://github.com/steven-b-cho/llmorph | `findings/llmorpheus.md` | MED–HIGH / low | done |
| 33 | CLEVER — end-to-end *verified* codegen in Lean 4 (NeurIPS'25) | repo + paper | https://github.com/trishullab/clever | `findings/clever.md` | MED–HIGH | done |
| 34 | "From Natural Language to Verified Code" (NL2VC-60, Dafny) | paper | https://arxiv.org/abs/2604.22601 | `findings/arxiv-2604-22601.md` | LOW–MED | done |
| 35 | DiffSpec — LLM differential testing from specs (CMU+MSR) | paper + artifact | https://arxiv.org/abs/2410.04249 | `findings/arxiv-2410-04249.md` | LOW–MED | done |
| 36 | G-Eval — the canonical LLM-as-judge (MSR, EMNLP'23) | paper + repo | https://arxiv.org/abs/2303.16634 | `findings/arxiv-2303-16634.md` | MED–HIGH | done |
| 37 | "Replacing Judges with Juries" — PoLL panel (Cohere) | paper | https://arxiv.org/abs/2404.18796 | `findings/arxiv-2404-18796.md` | MEDIUM | done |
| 38 | "Evaluating Scoring Bias in LLM-as-a-Judge" (Ant Group) | paper | https://arxiv.org/abs/2506.22316 | `findings/arxiv-2506-22316.md` | MEDIUM | done |

> Cross-cutting result of Run 5 — this batch is almost entirely about the VERIFIER, and it
> splits into three families, all reinforcing the same rule:
> - **Formal / deterministic oracles** (un-gameable in principle): CLEVER (Lean 4 — compiles
>   AND no `sorry`) and NL2VC (Dafny→Z3). BUT only as good as the spec — CLEVER's follow-up
>   found ~50% of ground-truth specs buggy and agents near-saturating it; NL2VC caught LLMs
>   gaming the verifier with *vacuous specs*, fixed by adding an independent functional
>   oracle the generator didn't author. So: guard the spec (non-computable specs + dual
>   independent oracles).
> - **Ground-truth-free verification:** DiffSpec (N independent implementations as a mutual
>   oracle via disagreement) and LLMorpheus (LLM-generated mutants — a *surviving* mutant is
>   a concrete, ground-truthed test-suite gap; i.e. "test the tests"). Both decouple the
>   judged entity from the judge.
> - **LLM-as-judge, done right:** G-Eval is the origin — and it FIRST named our exact hazard
>   ("self-reinforcement if the score is used as a reward signal," 2023). PoLL = a cross-vendor
>   jury cancels self-preference and is 7–8× cheaper, but Apple's "Nine Judges, Two Effective
>   Votes" shows correlated errors (diversity ≠ independence). The scoring-bias study: absolute
>   scores flip up to 46% on a cosmetic rubric reorder → prefer pairwise / repeat-and-vote /
>   hard verifiers, and note whoever controls the reference answer moves the score.
> Net for our verifier: hard/formal oracle where a spec exists (but guard it); ground-truth-free
> methods (differential, mutation) where it doesn't; LLM-judges only for non-verifiable quality,
> decorrelated from the proposer, pairwise not absolute, validated against humans. LLMorpheus
> gives us a way to verify our OWN tests aren't vacuous.

## Run 6 (committed) — goal-steering + memory

> Kristian provided these as #35–38; renumbered 39–42 to avoid collision with Run 5's #35–38.

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| 39 | Agent "goal" features — OpenAI Codex vs Claude Code | docs (compare) | https://developers.openai.com/cookbook/examples/codex/using_goals_in_codex · https://code.claude.com/docs/en/goal | `findings/agent-goals.md` | MED–HIGH | done |
| 40 | superpowers (obra / Jesse Vincent) — agentic skills framework | repo | https://github.com/obra/superpowers | `findings/superpowers.md` | MEDIUM | done |
| 41 | Graphiti (getzep) — bi-temporal knowledge-graph memory | repo | https://github.com/getzep/graphiti | `findings/graphiti.md` | MED–HIGH (memory) | done |
| 42 | cognee (topoteretes) — ECL knowledge-graph memory engine | repo | https://github.com/topoteretes/cognee | `findings/cognee.md` | MED–HIGH (memory) | done |

> Cross-cutting result of Run 6 (fills the MEMORY pillar + the goal-steering half of planning):
> - **Goal features = the INNER goal-steering loop** (not the evolutionary outer loop), and both
>   name FALSE COMPLETION as the primary long-horizon risk. Codex `/goal` = durable `ThreadGoal`
>   SQLite state + token/wall-clock budgets + distinct terminal states (complete / budget_limited /
>   blocked) + a same-model paranoid completion self-audit (verbatim `continuation.md`: re-anchor +
>   requirement-by-requirement audit + anti-reward-hack "Fidelity" language). Claude Code `/goal` =
>   a session Stop-hook where a *separate cheap* model (Haiku, transcript-only, no tools) returns
>   yes/no+reason — Codex's durable design avoids Claude Code's documented evaluator-overflow wedge.
>   (Fills goal *tracking*; goal *decomposition*/spec is still open.)
> - **Graphiti = the standout for our memory-invalidation need:** bi-temporal facts with
>   INVALIDATE-DON'T-DELETE (transaction time `created_at`/`expired_at` + valid time
>   `valid_at`/`invalid_at`); contradiction = LLM judgment + deterministic date-stamping (clean
>   fuzzy/deterministic split); deterministic-first dedup (MinHash/Jaccard + entropy gate); LLM-free
>   hybrid retrieval (cosine+BM25+BFS fused via RRF/MMR). Caveat: conversational/relational ontology,
>   not procedural/program memory.
> - **cognee** = ECL (Extract→Cognify→Load) into graph+vector; `DataPoint` = node+vector with UUID5
>   content-addressed dedup + mutable trust weights; a CLOSED FEEDBACK LOOP (retrieval logs used
>   element ids → feedback re-weights those edges via EMA — swap the human rating for our verifier's
>   pass/fail) + proposal-first `improve_skill` (audited self-modification). Weaker temporal model
>   than Graphiti (append-only + hard delete).
> - **superpowers** = agentic skills framework + SWE methodology (brainstorm→spec→worktree→plan→
>   subagent→TDD→review→finish); forced-invocation bootstrap; "TDD for prompts" (pressure-test a
>   skill on subagents, harvest rationalizations, write counters); adversarial artifact-grounded
>   verification ("Do Not Trust the Report — check the VCS diff"). "Disciplined, not smarter" (n=12).
>
> Still-open gaps after Run 6 (from the coverage analysis): goal/spec DECOMPOSITION (not just
> tracking), training-time self-improvement (RLVR / execution-feedback RL), secure-execution
> sandboxing infra, compute-scheduling / parallel-search economics, codebase-comprehension at scale.

## Run 7 (committed) — orchestration substrate

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| 43 | ClawTeam (HKUDS) — multi-agent swarm orchestration | repo | https://github.com/HKUDS/ClawTeam | `findings/clawteam.md` | MEDIUM | done |
| 44 | Temporal — durable execution / workflow engine | repo + site | https://temporal.io/ · https://github.com/temporalio/temporal | `findings/temporal.md` | HIGH (orchestration substrate) | done |

> Cross-cutting result of Run 7 (the long-horizon RUNTIME substrate — a medium gap, now in good shape):
> - **Temporal (HIGH for the runtime question) is a near-1:1 off-the-shelf substrate** for our
>   long-running loop, and a real buy-vs-build candidate. Durable state via event-sourcing +
>   deterministic REPLAY (state rebuilt from an append-only event log, not a RAM snapshot);
>   declarative retries/timeouts; durable timers; child-workflow fan-out for parallel candidates; and
>   **Continue-As-New = built-in bounded-history self-succession** — the exact pattern Antigravity
>   hand-rolled. The determinism-vs-LLM tension resolves cleanly: all LLM/tool/test calls are
>   **Activities** (run once, recorded, never re-run on replay). Caveat: versioning friction if the
>   orchestrator code self-modifies → keep the workflow thin/stable, evolving logic in activities.
> - **ClawTeam (MEDIUM)** = HKUDS swarm: a leader CLI spawns workers, each in its own git-worktree +
>   tmux + identity, deps via file/ZeroMQ inboxes, worktrees merged back; orchestration = plain shell
>   + auto-injected MCP tools (any CLI agent, no SDK). Resolves the lab lineage: ClawWork (solo) →
>   OpenSpace (skill-evolution) → ClawTeam (swarm). Reusable: worktree-per-candidate isolation, atomic
>   locked JSON state store with cycle-checked dependency auto-unblock, the "Ralph re-spawn loop" +
>   context-recovery prompt, a plan→execute→verify phase-gate machine. Cautionary: "self-evolving" is
>   marketing (grep-confirmed zero fitness/selection code), and `SuccessCriterion.test_command` is
>   defined but NEVER executed — gates trust self-reported completion (the verify-before-keep gap again).
>
> Still-open gaps after Run 7: goal/spec DECOMPOSITION, training-time self-improvement (RLVR /
> execution-feedback RL), secure-execution sandboxing, compute-scheduling early-stopping (the
> selection side is covered; the systems side is thin), codebase-comprehension at scale.

## Run 8 (committed) — the harness-only self-improvement cluster

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| 45 | meta-agent (Canvas Labs, YC F24) — harness-only recursive self-improvement | repo | https://github.com/canvas-org/meta-agent | `findings/meta-agent.md` | HIGH | done |
| 46 | autocontext (greyhaven-ai) — recursive self-improving harness (~111K LOC) | repo | https://github.com/greyhaven-ai/autocontext | `findings/autocontext.md` | HIGH (arch) | done |
| 47 | recursive-improve (kayba-ai) — harness self-improvement (ACE-based) | repo | https://github.com/kayba-ai/recursive-improve | `findings/recursive-improve.md` | MEDIUM | done |
| 48 | reflexio (ReflexioAI) — playbook-mining self-improvement sidecar | repo | https://github.com/ReflexioAI/reflexio | `findings/reflexio.md` | MEDIUM | done |
| 49 | RecursiveMAS — recursive multi-agent in LATENT space | paper + repo | https://arxiv.org/abs/2604.25917 | `findings/arxiv-2604-25917.md` | LOW | done |

> Cross-cutting result of Run 8 — this batch is DIRECT prior art for our now-locked HARNESS-ONLY path:
> meta-agent, autocontext, and recursive-improve all do exactly what we committed to (edit
> prompts/tools/skills/memory/orchestration; freeze weights), and they CONVERGE on the same verifier
> design — a held-out / holdout split + generalization-gap blocking + keep-only-if-verifiably-better,
> with explicit anti-Goodhart guards.
> - **meta-agent (HIGH)** = the closest map to our setup; reimplements Stanford "Meta-Harness"
>   (arXiv 2603.28052 — a NEW reference worth adding). Most reusable: the **harness=behavior /
>   benchmark=exit-contract** split where benchmark-injected tools/hooks WIN conflicts (the agent
>   structurally cannot override the verifier) + a graduated cheap-first verification gauntlet + a
>   three-layer holdout firewall + an anti-Goodhart prompt contract. Weakness: greedy hill-climb, no
>   population (matches our "defer the archive" call).
> - **autocontext (HIGH arch)** = the best anti-overfit/anti-judge-gaming design in the canon: a
>   layered gate stack (validity-retry budget → holdout generalization-gap block → LLM-judge-vs-oracle
>   gap block → adversarial skeptic) + typed, revertible, individually-gated harness mutations. Watch-out:
>   permissive gate defaults (missing marker → "accept") — a hole to avoid.
> - **recursive-improve (MEDIUM)** = trace capture via SDK monkey-patch; ACE-based (Agentic Context
>   Engineering — another new reference) edits on a git branch; /ratchet overnight keep-or-revert;
>   /evolve git-worktree island model. Verifier weak by default (self-defined metrics) but a real
>   held-out one is pluggable (terminal-bench).
> - **reflexio (MEDIUM)** = playbook-mining sidecar; GEPA-based held-out paired-rollout commit-if-better;
>   post-consequence horizon-gated reflection (re-judge a memory only after seeing downstream
>   consequences — credit assignment); MDL memory consolidation. Verifier = LLM judge (weak for code).
> - **RecursiveMAS (LOW)** = recursive multi-agent in LATENT space via trainable adapters → doubly
>   disqualified (weights + opaque latent messages, anti-verifiable). Useful "opposite conclusion":
>   text is a lossy boundary, which for a verifiable harness-only agent argues FOR inspectable text
>   artifacts (the road we take).
>
> New references surfaced (candidates for a later run): Stanford "Meta-Harness" (arXiv 2603.28052) and
> ACE / Agentic Context Engineering (Stanford + SambaNova).

## Backlog (queued / from deep-search)

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| — | _(awaiting the two Gemini deep-search reports + any further sources)_ | | | | | |

> Row format: short name · type (paper / system / repo / blog / talk / critique) ·
> primary link(s) · `findings/<slug>.md` (once done) · signal · status.
