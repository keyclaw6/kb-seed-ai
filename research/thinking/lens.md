# The Lens — Design Questions (OPEN / UNANSWERED)

These are the questions the architecture must eventually answer. **We do not have the
answers yet, and that is expected at this stage** — our knowledge isn't deep enough to
answer them well, and guessing now would just bias the research.

Their job *right now* is to **focus the research**: every source is read asking
"what does this source teach us about each of these?" We answer them only later, after
ingestion (Phase 3 synthesis).

1. **Smallest viable seed** — the minimum capabilities the agent must start with to
   bootstrap self-improvement.
2. **Goal representation** — how the overarching goal survives across iterations and
   decomposes into testable sub-goals.
3. **Hypothesis unit** — how a proposed change is represented, isolated, and run
   (e.g. a git worktree / experiment object).
4. **The verifier** — what certifies "this actually works / adds value," and how that
   signal is kept *unfakeable*. *(the crux)*
5. **Promotion gate & self-modification boundary** — the rules for a candidate becoming
   the new champion; what the agent may change about itself; what is inviolable; where the
   human gates.
6. **Search policy** — how compute is allocated across candidates (explore/exploit,
   patience vs. pruning, stagnation detection).
7. **Memory & unbounded runtime** — how past runs make future proposals smarter, and how
   the system runs ~forever (context handoff / self-succession / durable state).

> These mirror and extend the open questions in `../../docs/architecture.md` and are the
> working set going forward. Refine freely.
