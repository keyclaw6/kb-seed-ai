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
> That gap is exactly the organ our seed AI has to add. See `thinking/` for synthesis.

## Backlog (queued / from deep-search)

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| — | _(awaiting deep-search results + further sources)_ | | | | | |

> Row format: short name · type (paper / system / repo / blog / talk / critique) ·
> primary link(s) · `findings/<slug>.md` (once done) · signal · status.
