# Research Workspace

This directory holds the Phase 1–2 research effort for **KB Seed AI**: ingesting prior
art on self-improving / evolutionary software-building agents, one source at a time, and
distilling each into a per-source findings document we can later synthesize into an MVP
design.

## The process (recap)

1. **Canon** — the prioritized source list. See `sources.md`.
2. **Ingest** — one Opus research sub-agent per source produces an immutable findings doc
   in `findings/`, using `subagent-prompt.md`. Relevance filter: *anything that would help
   build a self-improving, evolutionary, software-building agent* — memory systems,
   long-horizon running, decision-making, verification, orchestration, agent control
   features, and so on.
3. **Synthesize** — humans read all findings and *subtractively* design the MVP; notes
   live in `thinking/`.
4. **MVP** — spec + build (later).

## Folder rules

- `sources.md` — the canon: every source with provenance + status.
- `subagent-prompt.md` — the comprehensive research brief each per-source sub-agent runs.
  Review/tune this before we spawn agents.
- `findings/` — **IMMUTABLE.** One markdown doc per source, written verbatim by a research
  sub-agent. **Do not hand-edit.** Our reactions, picks, and rejections live in
  `thinking/`, never here.
- `thinking/` — shared, freely-editable WIP for Kristian + the orchestrator: the mission,
  synthesis notes, decisions.

## Why the separation

Agent-generated findings are **evidence**. We keep them untouched so the record of "what
each source actually said" stays trustworthy and auditable. Our opinions about that
evidence are a separate layer, in `thinking/`. Mixing the two would let our later
preferences quietly rewrite the source record.
