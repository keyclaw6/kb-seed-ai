# KB Seed AI

An experimental project exploring a **self-improving agent** — a seed AI given an
overarching goal that it pursues continually, iterating on its own capabilities,
tools, and reasoning over time.

## Idea

- Give the agent a single, persistent, overarching goal.
- The agent loops indefinitely: plan → act → reflect → improve.
- Improvements may include its prompts, tools, memory, sub-agents, and code.
- Architecture and design decisions are diagrammed in [D2](https://d2lang.com/)
  so a human and an AI agent can collaborate visually.

## Status

Early architecture phase. We're sketching the system in D2 and iterating on the
design before writing the agent itself.

## Layout (tentative)

```
.
├── docs/         # Notes, design discussions
├── diagrams/     # D2 source files for architecture
└── src/          # Agent implementation (TBD)
```

## License

TBD.
