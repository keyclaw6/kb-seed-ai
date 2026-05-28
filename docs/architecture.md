# Architecture Notes

This document will collect the ongoing design discussion for the KB Seed AI
agent. Diagrams live alongside in `../diagrams/` as D2 source.

## Open questions

- What's the smallest viable "seed" — what minimum capabilities must the agent
  start with to bootstrap self-improvement?
- How do we represent the overarching goal so it survives across iterations?
- What's the improvement loop? (e.g. reflect → propose change → test → commit)
- How do we sandbox self-modification safely?
- How do we evaluate whether an "improvement" actually improved anything?
- Memory: short-term scratchpad vs. long-term durable knowledge.
- Tool acquisition: can the agent write/install new tools for itself?
