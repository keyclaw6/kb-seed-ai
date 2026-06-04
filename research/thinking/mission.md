# Mission — North Star (WIP)

> **One line: build an AI system that can build everything.**

## What we're building

Give the system a high-level goal — "build this project / this software" — and a harness
around the agent builds it **reliably**:

- no **false positives** (declaring success when it isn't),
- no **false negatives** (discarding work that was actually good),
- producing **clean code**,
- with **tokens treated as unlimited** — compute is not the constraint we optimize against.

And the seed AI we develop here should ultimately **develop itself**.

## Methodology

We build a system that **routes around the current limitations of language models** —
they decide poorly and lack fluid reasoning, but they are cheap and parallelizable — by
spending compute on **wide search + thorough verification** instead of trusting any
single model decision.

## Status

North star, not a spec. Editable by Kristian + the orchestrator. The concrete design
emerges from Phase 2 ingestion → Phase 3 synthesis.
