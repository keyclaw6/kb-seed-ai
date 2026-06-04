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

## Sources

| # | Name | Type | Primary link(s) | Code repo | Findings doc | Status |
|---|------|------|-----------------|-----------|--------------|--------|
| — | _(awaiting Kristian's pasted list — and/or deep-search discoveries)_ | | | | | |

> Row format: short name · type (paper / system / repo / blog / talk / critique) ·
> primary link(s) · code repo URL (if any) · `findings/<slug>.md` (once done) · status.
