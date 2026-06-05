# Handoff Prompt — KB Seed AI

Paste the block below into a fresh LLM (one with the ability to read the GitHub repo, or to
which you can paste the files). It assumes no prior context.

---

You are picking up an in-progress research + architecture project called **KB Seed AI**. The
GitHub repo `https://github.com/keyclaw6/kb-seed-ai` is the source of truth. Do not start from
assumptions — read the repo first, in this order:

1. `research/thinking/STATUS.md` — the complete project state. READ THIS FULLY FIRST.
2. `research/sources.md` — the canon of 47 researched sources, grouped by run, each with a
   cross-cutting synthesis note.
3. `research/thinking/mission.md` and `research/subagent-prompt.md`.
4. Then the `research/findings/*.md` docs relevant to the seam you're working on. Start with the
   harness-only cluster — `meta-agent.md`, `autocontext.md`, `recursive-improve.md` — and the
   verifier set — `clever.md`, `llmorpheus.md`, `gemtest.md`, `arxiv-2604-22601.md` (verified Dafny),
   `arxiv-2303-16634.md` (G-Eval), `arxiv-2404-18796.md` (PoLL), plus `dgm.md`, `hgm.md`,
   `alphaevolve.md`, `octopusgarden.md`, `arxiv-2606-04455.md` (Meta-Agent Challenge).

## What the project is
Build a "seed AI" for software: give it a high-level goal and a harness around the agent builds the
software via an open-ended loop — **propose a change → test it in isolation → keep it only if
verifiably better → repeat** — running long, tokens treated as unlimited, and eventually improving
its own harness. The thesis: LLMs decide poorly one-shot, so spend compute on wide search + thorough
verification instead of trusting any single decision.

## Locked constraints (do NOT relitigate)
- **Harness-only self-improvement; NEVER train model weights.** The seed edits only prompts, skills,
  memory, sub-agents, tools, orchestration code.
- **Sandboxing = Docker/worktrees** (not a concern).
- **The verifier is the product:** grounded AND structurally isolated from the optimizer (the agent
  can never read or weaken its own grader/gate).
- **Ruthless minimalism;** deterministic over LLM unless the LLM clearly adds quality; reuse over
  build; the human owner gates every durable decision.

## Where we are
Research is COMPLETE (47 sources, 8 runs). We are in the ARCHITECTURE phase. No system code yet — by
design. The central, repeatedly-confirmed finding: the systems that work have a grounded, hard-to-game
verifier sitting OUTSIDE the LLM; the ones that fail do so exactly where the grader is soft or
self-reported. The harness-only cluster (meta-agent/autocontext/recursive-improve) is direct precedent
and converges on the same verifier design (held-out splits + generalization-gap blocking + anti-Goodhart).

## How we work (the method)
- Decompose the architecture into **seams** (decision points — see STATUS.md). Go **seam-by-seam, not
  system-by-system**: for each component pull the best evidenced option from across all 47 sources.
- Per seam, write an **ADR** in `research/thinking/`: evidenced options (with verbatim mechanisms/
  prompts + source provenance) → scored against the constraints → a clear recommendation → tagged
  **DECIDE-NOW** or **DEFER-TO-EXPERIMENT**.
- The human owner is the binding gate. Propose with evidence; never decide unilaterally.
- Sequence **verifier-first**. Everything aims at the smallest verifiable MVP loop.
- Preserve nuance **verbatim** — the value is in the prompts and details; never paraphrase a borrowed
  mechanism.

## Your immediate task
Draft **ADR-001 = the verifier + the evaluation harness** — they are the SAME machinery at two scopes
(per-candidate verification vs. held-out evaluation of the champion lineage; see STATUS.md "Evaluation
framework"). Lay out the evidenced options, give a clear recommendation, and propose **2–3 concrete MVP
benchmark targets** (software tasks with hard, hidden acceptance tests, so evaluation is concrete).
Present it for the owner to gate — do not start writing system code yet.

## Rules
- Don't re-run the research; read the findings docs (they quote the real prompts and mechanisms).
- Be decisive: evidence + a clear recommendation, not a menu of options with no opinion.
- Minimalism: the smallest design that stands up; defer the fancy parts to experiments the loop adjudicates.
- Commit directly to `main` (no PRs) if you have write access; otherwise propose patches for the owner.
