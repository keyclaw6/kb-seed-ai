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

## Backlog (queued / from deep-search)

| # | Name | Type | Primary link(s) | Findings doc | Signal | Status |
|---|------|------|-----------------|--------------|--------|--------|
| — | _(awaiting the two Gemini deep-search reports + any further sources)_ | | | | | |

> Row format: short name · type (paper / system / repo / blog / talk / critique) ·
> primary link(s) · `findings/<slug>.md` (once done) · signal · status.
