# OpenSage Agent ADK — Findings

> Per-source research doc. Reporter, not architect. Judge by one test: would this help
> build a self-improving, evolutionary, software-building agent?

## 1. Identity

- **Name:** OpenSage (repo: `opensage-adk`; site brands it "Agent Development Kit / ADK").
- **What it is (one line):** A Python agent framework, built *on top of* Google's ADK
  (`google.adk`), that lets an LLM agent **create its own sub-agents and tools**, gives it
  a **hierarchical (file + Neo4j graph) memory**, **sandboxed execution**, and a suite of
  **software-engineering / security** domain tools + benchmarks (CyberGym, SWE-bench Pro,
  TerminalBench, SeCodePLT, PatchAgent).
- **Authors / org:** GitHub org `opensage-agent`. Sole committer on `main` is **"Wenbo"**
  (`henrygwb@gmail.com`, GitHub login **Henrygwb**). Site domain `opensage-agent.ai`,
  docs at `docs.opensage-agent.ai`. Discord community linked.
- **Dates:** Repo created **2026-03-23**; last push **2026-04-07**. So this is a *very
  young* project (≈2 weeks of public history at the inspected commit).
- **Popularity (signal proxy):** 90 stars, 20 forks at inspection.
- **License:** Apache 2.0.
- **Primary links:**
  - Site: https://www.opensage-agent.ai/adk.html
  - Docs: https://docs.opensage-agent.ai
  - Repo: https://github.com/opensage-agent/opensage-adk
- **Code inspected:** `opensage-agent/opensage-adk@481b4344f3d07de42082f367ecda4381f81c22c8`
  (branch `main`, commit msg "minor update", 2026-04-07). Obtained via codeload tarball
  (direct `git clone` blocked by sandbox proxy 407; tarball fallback used).

## 2. TL;DR

- **What it really is:** A thin-but-real **wrapper around Google ADK** (`google.adk`)
  that adds four things to a normal LLM agent: (1) tools that let the agent **spawn its own
  sub-agents** at runtime (`create_subagent`, `search_agent`, `call_subagent_as_tool`); (2)
  an **"AI-creates-tools"** scaffolding skill; (3) a **hierarchical memory** (file-based
  `/mem/...` + an optional Neo4j knowledge-graph long-term memory with an LLM-driven
  ADD/UPDATE/DELETE/NONE updater); and (4) a **sandboxed (Docker) execution** model plus a
  domain library of **security / SWE tooling** and benchmarks (CyberGym, SWE-bench Pro,
  SeCodePLT, TerminalBench, PatchAgent).
- **Marketing thesis ("Agent Development 2.0"):** agents that *dynamically build and manage
  their own structure/tools at runtime* instead of human-fixed topology (docs, verbatim:
  "transitioning from agents that execute predefined structures to agents that can
  dynamically build and manage their own systems at runtime").
- **It is NOT an evolutionary / self-improving optimizer.** There is **no candidate
  population, no fitness-guided selection, no "propose→test→keep-if-better" loop** in the
  framework itself. "Self-improvement" here = (a) the agent writing new tools/sub-agents
  in-context, and (b) an external **RL training** path (`rl/` + `evaluation/rl_adapters/`
  for slime & AReaL) where benchmark `reward_func`s fine-tune the policy model. Verification
  is **delegated to per-benchmark harnesses** (e.g. official SWE-bench Pro Docker eval), not
  built into the agent's own control loop.
- **`finish_task` is a one-liner** that just sets `state["task_finished"]=True`. The agent
  self-declares done; the framework does not independently verify task success at runtime.
- **Maturity:** very young (repo ≈2 weeks old at inspection, single author, 90 stars). Some
  load-bearing pieces are stubs (e.g. the memory-management sub-agent's instruction is a
  single sentence). Heavily security-research-flavored (fuzzing, GDB/PDB MCP debuggers,
  Joern/CodeQL). Treat self-improvement claims as aspirational, but several concrete
  *patterns* are reusable.
- **Highest-value transferable patterns:** the **explore-then-solve two-phase agent**
  (budgeted recon → memory → structured summary injected into the solver), the **LLM-as-judge
  memory-write gate** (`StorageDecider`), the **agentic-memory update pipeline**
  (entity-extract → similarity → ADD/UPDATE/DELETE/NONE), the **Skill (`SKILL.md`) tool
  format** with a hard "do not edit Skills in place; copy to `new_tools/`" rule, and the
  **dynamic sub-agent topology** tools.

## 3. What it does & how it works

OpenSage is a Python package (`src/opensage`) that subclasses ADK's `LlmAgent` as
`OpenSageAgent` and layers on five subsystems. The "agent" is a normal ADK LLM agent given
an enriched system prompt plus a set of OpenSage-specific tools; ADK's `Runner` drives the
turn loop (model call → tool calls → events), and OpenSage hooks behavior via ADK
**plugins** and a couple of **monkey-patches**.

**(a) The agent abstraction (`agents/opensage_agent.py`).** `OpenSageAgent.__init__`
normalizes tools, optionally injects a `memory_management_agent` tool (when
`memory_management=DATABASE`), and — crucially — **auto-generates a chunk of system prompt**
from discovered Skills. A `ToolLoader` scans `bash_tools/*/SKILL.md` (and
`~/.local/opensage/bash_tools`), parses YAML frontmatter + a `## Requires Sandbox` markdown
section, and appends to `self.instruction`: a list of available Skills (path + description +
`should_run_in_sandbox`), a generated **"Sandbox Environment"** description (mount points
`/shared`, `/bash_tools`, `/mem`, Neo4j DBs), and a **"Tool Usage Policy (MUST FOLLOW)"**.
`enabled_skills` (None | "all" | list-of-prefixes) controls which Skills load. There is also
`update_enabled_skills()` to rewrite the prompt at runtime.

**(b) Self-generated topology (`toolbox/general/dynamic_subagent.py` +
`session/opensage_dynamic_agent_manager.py`).** The agent is given tools to manage a live
pool of sub-agents: `create_subagent(agent_name, instruction, model_name, tools_list,
enabled_skills, description)` (validates requested Python tools/toolsets against the caller's
own tools, always injects baseline tools `run_terminal_command`,
`list_background_tasks`, `get_background_task_output`, `wait_for_background`, `complain`,
and supports `model_name="inherit"`), `list_active_agents`, `search_agent` (keyword scored:
name match=+2, desc match=+1), and `call_subagent_as_tool` (wraps the target in ADK's
`AgentTool`). Docs describe **vertical topology** (decompose into sequential specialists)
and **horizontal topology** (N parallel agents on the same task, merged via an "ensemble
manager"). Created agents can be **persisted and restored on demand** (manager
`_load_persisted_agents_on_demand`).

**(c) AI-creates-tools (`bash_tools/new_tool_creator/`).** A Skill whose script
`create_new_tool.sh` → `init_skill.py` **scaffolds a new Skill directory** under
`bash_tools/new_tools/<category>/<tool-name>/` (validates hyphen-case name ≤40 chars,
requires `--should_run_in_sandbox` and `--returns_json`). The policy prompt forbids editing
existing Skills in place and tells the agent to copy/adapt into `new_tools/`. So "the AI
writes its own tools" concretely = the model emits a bash/python script + a `SKILL.md` into
a conventionalized directory, which the `ToolLoader` then surfaces to (sub)agents.

**(d) Hierarchical memory (`memory/`, plus file layout in the prompt).** Two modes:
- **FILE memory**: a documented `/mem/<agent_name>/` layout — `planning.md` ("living
  plan/todo; read before work, update after major steps"), per-session trajectory dumps
  `session_<id>.json`, `/mem/topology.json` (cross-agent calls), and `/mem/shared/
  knowledge.jsonl` (a JSONL of reusable `{key,value,tags,source,updated_at}` knowledge the
  agent is told to curate and consult "before major decisions").
- **DATABASE memory (Neo4j)**: a full **agentic knowledge-graph** pipeline under
  `memory/update/` — `entity_extractor` → similarity lookup → `LLMOperationDecider`
  (ADD/UPDATE/DELETE/NONE) → `graph_operations`; plus `memory/search/` strategies
  (embedding / keyword / title-browse) behind a `search_controller`, and a
  `StorageDecider` (LLM-as-judge gate on whether a tool result is worth storing). A
  `memory_management_agent` (itself an `OpenSageAgent`) exposes these as a natural-language
  sub-tool. **Short-term memory** is separate: a monkey-patch (`patches/neo4j_logging.py`)
  records every agent run/event/tool-call into Neo4j (`AgentRun`/`Event` nodes,
  `AGENT_CALLS`/`HAS_EVENT` edges), with summarizer plugins that compact oversized tool
  responses and event history.

**(e) Sandboxed execution + domain tools (`sandbox/`, `toolbox/`, `bash_tools/`).** Tools
either run in-process (Python tools) or as scripts inside Docker sandboxes (`main`, `fuzz`,
`joern`, `neo4j`, `gdb_mcp`, `pdb_mcp`). The domain toolset is heavily security/SWE:
retrieval, static analysis (Joern/CodeQL), fuzzing (a simplified Python fuzzer + an AFL++
campaign), GDB/PDB MCP debuggers, coverage, and finish-task. Evaluation lives in
`benchmarks/` and `evaluation/`; an RL layer (`evaluation/rl_adapters/`, `rl/`) lets a
benchmark's `get_prompt`/`reward_func` drive policy training via slime or AReaL.

**Control loop reality:** the loop is **ADK's** (`Runner.run_async` streaming events to a
`max_llm_calls` budget). OpenSage adds: a `BuiltInPlanner` with high-effort thinking (in the
SWE-bench agent), the explore→solve two-phase pattern, the summarizer plugins for context
management, and the self-declared `finish_task`. There is **no autonomous verify-and-retry
or evolutionary selection loop** in the framework.

## 4. Evidence from the code

**Repo:** `opensage-agent/opensage-adk@481b4344f3d07de42082f367ecda4381f81c22c8` (main,
2026-04-07). ~45k LOC under `src/opensage` (largest: `sandbox/` ~7.8k, `toolbox/` ~6.3k,
`memory/` ~5k, `evaluation/` ~4k, `session/` ~3.3k).

### 4.1 Agent + Skill loader — `repo@481b4344:src/opensage/agents/opensage_agent.py`

The Skill-driven prompt and the **hard "don't edit Skills in place" policy** (lines
~657–671), injected into every agent's `instruction`:

```
Tool usage policy:
- When planning or describing how you will accomplish a task, prefer using the provided
  Skills under `/bash_tools/...` ...
- Only fall back to generic shell commands when there is **no** suitable `/bash_tools`
  Skill for the job.
- Before starting work, survey the tool ecosystem broadly:
  - Call `list_available_scripts to review relevant available Skill docs.
  - Then inspect and consider multiple relevant toolsets (e.g., retrieval +
    static_analysis + neo4j), not just one.
- If a workflow is repetitive, prefer writing a small wrapper script (or a new Skill) to
  automate it. You may compose existing `/bash_tools` Skills, and you may also adapt/extend
  them.
- Do NOT edit existing `/bash_tools/...` Skills in place. If you need changes, copy/adapt
  into a new Skill/script under `/bash_tools/new_tools/<tool_name>/` (with a `SKILL.md`).
  You can use `/bash_tools/new_tool_creator` to scaffold the initial directory structure.
```

The file-memory contract is also injected verbatim (lines ~513–561), including
`planning.md` semantics and the `/mem/shared/knowledge.jsonl` schema
(`key`, `value`, `tags`, `source`, `updated_at`) with the instruction: "When you discover
valuable reusable knowledge, proactively add/update entries ... and consult it via search
before major decisions."

### 4.2 Dynamic sub-agents — `repo@481b4344:src/opensage/toolbox/general/dynamic_subagent.py`

`create_subagent` signature + the always-injected baseline tools and the
`enabled_skills` guardrail it appends to each sub-agent's instruction:

```python
default_tools_by_name = {
    "run_terminal_command": run_terminal_command,
    "list_background_tasks": list_background_tasks,
    "get_background_task_output": get_background_task_output,
    "wait_for_background": wait_for_background,
    "complain": complain,
}
...
skills_guardrail = (
    "\n\n[Tooling policy]\n"
    "Bash tools availability is controlled by enabled_skills. "
    f"For this subagent, enabled_skills={enabled_skills_repr}.\n"
    "You must only use bash tools that are available under the enabled_skills "
    "selection. If a needed bash tool is not available, report the limitation "
    "and ask the caller to recreate the subagent with the correct enabled_skills.\n"
)
```

The docstring states the design intent: "A subagent's capabilities come from two sources:
1) Python tools/toolsets ... 2) Bash tools ... determined by `enabled_skills`."
`search_agent` scores matches by `name` (+2) and `description` (+1) and returns ranked
metadata — a lightweight retrieval over the live agent pool.

### 4.3 `finish_task` is trivial — `repo@481b4344:src/opensage/toolbox/finish_task/finish_task.py`

```python
def finish_task(tool_context: ToolContext) -> str:
    """Indicate that the task has been finished."""
    tool_context.state["task_finished"] = True
    return "Task finished"
```

No verification — the agent asserts completion; the runner stops on the flag (or on the
`max_llm_calls` budget). Any real grading happens later in the benchmark harness.

### 4.4 Memory write-gate (LLM-as-judge) — `repo@481b4344:src/opensage/memory/storage_decider.py`

`StorageDecider.should_store()` calls an LLM (default `gemini/gemini-2.5-flash-lite`,
temperature 1.0) with `STORAGE_DECISION_PROMPT` (verbatim, abridged):

```
You are a memory storage decision system. Analyze the following tool result and decide
whether it contains valuable information worth storing in long-term memory.
...
**STORE if the result contains:** File contents revealing project structure/architecture;
Bug locations, error messages, stack traces; Test patterns/results/coverage; Config
details; Important code snippets/functions/classes; Search results identifying code
locations; Build/deploy info.
**SKIP if the result is:** Routine navigation output (ls/pwd/cd); Redundant info; Very
short/uninformative (<50 chars useful); Tool acks; Temporary/transient state.
## Response Format  (JSON only)
{ "should_store": true/false, "content_type": "code|text|finding|error|search_result|
  config|test_result|other", "summary": "REQUIRED if should_store=true ...",
  "confidence": 0.0-1.0, "reason": "..." }
```
Failure/parse errors **default to not storing**.

### 4.5 Agentic memory update — `repo@481b4344:src/opensage/memory/update/operation_decider.py`

`LLMOperationDecider.decide_operation()` returns one of `ADD/UPDATE/DELETE/NONE`. Fast path:
empty `existing_nodes` → ADD. Otherwise it summarizes up to 3 similar nodes and prompts
(verbatim, abridged):

```
Decide what operation to perform for this entity.
New Entity: - Type: {label} - Content: {entity_content}
Existing similar entities in memory: {existing_summary}
Context: {intent}
Choose one operation:
- ADD: genuinely new information not covered by existing entries
- UPDATE: should replace/update an existing entry with better/more recent information
- DELETE: new information indicates an existing entry is outdated/incorrect
- NONE: information already exists and doesn't need changes
Respond with only the operation name (ADD, UPDATE, DELETE, or NONE).
```
`max_tokens=10`, temperature 1; unparseable/exception → defaults to ADD. This is the same
family as Mem0 / A-MEM "self-organizing" agentic memory.

### 4.6 Explore→solve two-phase agent — `repo@481b4344:benchmarks/swe_bench_pro/swe_bench_pro.py`

When `--use_explore_agent`, Phase 1 runs a separate **explore agent** (capped at
`explore_max_llm_calls`, default 40) on the same task, which writes findings to memory and
returns a structured summary; Phase 2 injects that summary into the main solver's first
message. If the explore agent hits the call limit, `_summarize_explore_session` manually
reconstructs a summary from the session events. The summary prompt (verbatim, abridged):

```
You are an expert code exploration assistant. ... The agent was interrupted before it could
provide a final summary. Based on the conversation history, please provide a structured
summary ...
## Required Summary Format:
## Exploration Summary
### Key Findings - [most important discoveries about the codebase]
### Relevant Files - [files examined/relevant, with brief descriptions]
### Code Structure - [how the relevant parts are organized]
### Suggested Approach - [how to approach the task]
```

Verification is the **official external harness**: `evaluate()` clones
`scaleapi/SWE-bench_Pro-os`, dumps predictions to `predictions.json`, and shells out to
`swe_bench_pro_eval.py --use_local_docker`. Solved instances are appended to
`successful_instances.txt` and skipped on re-runs (a crude "don't re-do solved work"
memory across runs).

### 4.7 RL / verification surface — `repo@481b4344:src/opensage/evaluation/rl_adapters/benchmark_interface.py`

A benchmark's registered `Evaluation` subclass must export `get_prompt(sample)->str` and
`reward_func(args, sample, **kwargs)->{"score":..,...}` (optional `preprocess_sample`,
`postprocess_response`). `BenchmarkInterface.load(name)` discovers these and the RL
adapters (`adapters/slime.py`, `adapters/areal.py`) use them as the reward signal to train
the policy LLM. This is the **only place "verifiably better" is operationalized**, and it
lives in the benchmark/RL layer, not the agent loop.

### 4.8 Provenance (paper + blog)

OpenSage is the public artifact of a **Berkeley RDI (Dawn Song's lab)** paper, "**OpenSage:
Self-programming Agent Generation Engine**" (arXiv:2602.16891, formatted for ICML). Authors:
Hongwei Li, Zhun Wang, Qinrun Dai, Yuzhou Nie, Jinjun Peng, Ruitong Liu, Jingyang Zhang,
Kaijie Zhu, Jingxuan He, Lun Wang, Yangruibo Ding, Yueqi Chen, **Wenbo Guo**, **Dawn Song**.
(Repo committer "Wenbo"/`Henrygwb` = Wenbo Guo.) The paper builds **"SageAgent"** on top of
the ADK and benchmarks it. The two-step topology-creation procedure ("first let the parent
agent create an agent configuration that specifies the metadata of the sub-agents it intends
to create … then parse … and create a sub-agent as a Python object … store it in a unified
sub-agent pool") is confirmed by `session/opensage_dynamic_agent_manager.py` (`AgentMetadata`
dataclass; `config` kept as a dict for persistence/restore; lifecycle
`AgentStatus.{CREATED,ACTIVE,PAUSED,STOPPED,ERROR,PENDING_TOOLS}`; persisted to
`~/.local/opensage/dynamic_agents`). Background/async tool execution (paper §3.3) is real:
`toolbox/general/bash_task_manager.py` (`BashTaskManager`/`TaskStatus`) + the baseline tools
`list_background_tasks`/`get_background_task_output`/`wait_for_background`. There is also a
`plugins/default/claude_code_hooks` directory (Claude-Code-style hook compatibility).

## 5. What's genuinely smart

The genuinely valuable ideas here are **engineering/scaffolding patterns**, not a new
optimization algorithm. Ranked by transfer value to a self-improving software agent:

1. **Two-phase "explore → solve" with a budgeted recon agent + structured handoff.**
   (`benchmarks/swe_bench_pro/swe_bench_pro.py`.) A separate explore agent runs first under a
   hard `max_llm_calls` budget, dumps findings into memory, and emits a fixed-format summary
   (Key Findings / Relevant Files / Code Structure / Suggested Approach) that is injected into
   the solver's first user message. Crucially, **if the explorer runs out of budget, they
   reconstruct the summary from session events** rather than losing the work. This is a clean,
   practical answer to long-horizon context management: spend a bounded budget building a
   compressed world-model, then hand it to the executor. Directly relevant to running agents
   reliably over long horizons.

2. **LLM-as-judge memory write-gate (`StorageDecider`) + agentic ADD/UPDATE/DELETE/NONE
   updater (`LLMOperationDecider`).** Instead of logging everything, a cheap model decides
   *per tool result* whether it's worth persisting and produces a summary; a second decider
   reconciles new facts against existing graph nodes (dedup / supersede / retract). This is a
   coherent, reusable recipe for a memory that stays **small and high-signal** over a long run
   — the paper's own ablation says this *management* (not the graph per se) is what helps.

3. **Skills as the unit of tool synthesis, with an explicit immutability rule.** Tools are
   `SKILL.md`-described directories (YAML frontmatter + `## Usage`/`## Requires Sandbox`
   markdown + optional `scripts/`), auto-discovered and turned into system-prompt entries. The
   agent is told **never to edit a Skill in place** — instead copy/adapt into
   `bash_tools/new_tools/<name>/` via a scaffolding Skill (`new_tool_creator`). That
   copy-don't-mutate discipline is exactly the kind of invariant a self-modifying agent needs
   to avoid corrupting its own working tools, and it gives a natural lineage of tool versions.

4. **A unified, searchable, persistable agent pool as the topology primitive.** Rather than a
   fixed graph, the agent gets CRUD-ish tools over a live pool (`create_subagent`,
   `list_active_agents`, `search_agent`, `call_subagent_as_tool`), each sub-agent defined by
   (model, instruction, Python tools, `enabled_skills`), with `model="inherit"` and on-demand
   restore from disk. Treating "the set of agents" as **searchable state the policy edits at
   runtime** is the right abstraction for orchestration/self-organization.

5. **Heterogeneous, per-tool sandboxing with a shared `/shared` workspace + async tasks.**
   Different tools declare which Docker sandbox they need (`main`/`fuzz`/`joern`/`neo4j`/
   `gdb_mcp`/`pdb_mcp`); long-running tools (compile, fuzz, static analysis) run as background
   tasks the agent polls. This is the unglamorous-but-essential substrate for "test a change
   in isolation" and for tools with conflicting dependencies.

6. **Multi-model horizontal ensemble with a message board + LLM aggregator.** N clones of an
   agent (optionally different models) run the same task in parallel, coordinate via a shared
   board (`post_to_board`; tool results carry a `_message_board_diff` the agent must read),
   and a final LLM "synthesize consensus & disagreement" prompt merges them. A pragmatic
   best-of-N / debate pattern (though it *merges* rather than *verifies-and-selects*).

7. **It is explicitly framed as a training environment, not just a runtime.** The paper's
   thesis is the ML analogy: provide a *scaffold* and let the model "learn how to organize
   topology, tools, and memory from experience and feedback." The `evaluation/rl_adapters/`
   (slime, AReaL) + benchmark `reward_func`s operationalize this as RL post-training. The
   ambition — "unifying agent construction and problem solving within a single, AI-centric
   loop" — is the same north star as a seed AI, even if the current code only delivers the
   scaffold + an RL hook.

## 6. Claims vs. reality / limitations / critiques

- **(A) Authors claim** the *first* AI-centric ADK where the model self-creates topology,
  tools, and memory; **SageAgent** beating Claude Code, Codex, OpenHands, SWE-Agent — CyberGym
  **60.2%** (vs 39.4% OpenHands), Terminal-Bench 2.0 **78.4%**, SWE-Bench Pro **59.0%** (vs
  40.2% SWE-Agent), DevOps-Gym **46.8%**.
- **(B) What the code actually shows:** a competent ADK *wrapper* with the four subsystems,
  plus benchmark harnesses. It does **not** contain SageAgent's full agent definitions/configs
  or the headline experiment scripts in a runnable form (the `examples/agents/` are modest:
  a PoC/dynamic-tools agent, ensemble/dynamic-subagent/summarization feature demos). The
  benchmark numbers are **not reproducible from this repo alone** (they depend on external
  datasets, Docker images, models like GPT-5.3-Codex / Gemini-3, and unshipped agent configs).
  I could **not verify any benchmark number** from the code.
- **"Self-improvement" is overstated for our purposes.** There is **no evolutionary loop, no
  fitness function over candidate programs, no automatic propose→verify→keep-if-better** inside
  the framework. "Self-improvement" = (i) in-context self-programming (write a tool / spawn a
  sub-agent), and (ii) offline RL via benchmark rewards. The agent does not measure whether its
  self-written tool actually *helped*, nor select among variants.
- **No built-in verifier in the control loop.** `finish_task` just flips a flag; correctness is
  judged only by external benchmark harnesses (e.g. SWE-bench Pro Docker eval). Nothing
  prevents the agent from declaring success on an unverified change — a real reward-hacking
  surface if you tried to close the loop naively.
- **Authors' own honesty (good signal):** the blog/site states current models "do not always
  use these advanced features optimally … models may forget to reuse existing agents or memory,
  create sub-agents with mismatched toolsets, or hallucinate tools." So the headline autonomy
  is aspirational and model-limited *today*.
- **Memory lift is modest.** Their ablation: **59.0%** (OpenSage memory) vs **56.4%** (Mem0g)
  vs **56.2%** (NoMem) on SWE-bench Pro — only ~+2.8 pts over no memory; they concede "simply
  adding a graph memory is not enough." Tool-creation is real but sparse: "agents created 39
  task[-specific tools]" across a **300-instance** CyberGym subset.
- **Maturity / code-smell flags:** repo ≈2 weeks old at inspection; single committer; many
  `TODO`s in core files (e.g. `opensage_agent.py` questions whether its own prompt is too
  long); the `memory_management_agent`'s entire instruction is one sentence ("You are a Memory
  Management Agent that manages the memory of the system.") — i.e. a load-bearing component is
  effectively unprompted; several sandbox backends documented as "under development." Default
  memory mode is FILE; the richer Neo4j graph memory requires extra infra.
- **Heavy security/SWE specialization.** The most differentiated tooling (Joern/CodeQL,
  AFL++/libFuzzer, GDB/PDB MCP) is vuln-research-oriented; a general software-building agent
  would use little of it directly.
- **Independent critique:** none found. As of inspection I found **no third-party reviews,
  reproductions, or skeptical analyses** — only first-party artifacts (site, docs, arXiv,
  Berkeley RDI blog, ResearchGate mirror). Treat all performance claims as unaudited.

## 7. Relevance to a self-improving, evolutionary agent

Mapped to the broad test (would it help build a self-improving, evolutionary,
software-building agent?):

- **Long-horizon running — HIGH.** The explore→solve two-phase pattern, the budget-then-
  summarize-on-overflow trick, and the summarizer plugins (compact oversized tool responses
  and event history into linked summary nodes with lineage) are directly useful for keeping a
  long autonomous run inside a context budget without losing recoverable detail.
- **Memory — MEDIUM/HIGH (patterns), LOW (their results).** The *write-gate + ADD/UPDATE/
  DELETE/NONE reconciliation + dual graph/embedding retrieval + read-only short-term, writable
  long-term* design is a solid template for an agent that must accumulate reusable knowledge
  across many build attempts. But the empirical lift was small, so borrow the *mechanism*, not
  the expectation of large gains.
- **Decision-making / orchestration — MEDIUM/HIGH.** A searchable, persistable **agent pool**
  edited at runtime (create/search/call/resume; per-agent model + toolset + skills) is a
  reusable way to represent and mutate "the team" — analogous to maintaining a population of
  workers. The ensemble/message-board gives a multi-model best-of-N substrate.
- **Tool self-extension — MEDIUM/HIGH.** The Skill format + the **copy-don't-edit-in-place**
  invariant + a scaffolding meta-tool is a clean, safe pattern for an agent that writes and
  versions its own tools — a core seed-AI capability. (What's missing: measuring whether a new
  tool improved outcomes.)
- **Verification / "keep if better" — LOW (the key gap).** This is exactly what OpenSage does
  *not* provide in-loop. Verification is delegated to external benchmark harnesses, and there
  is no candidate-vs-baseline comparison or selection. A seed AI would need to supply this
  layer itself; OpenSage offers the *substrate* (sandboxes, reward_func interface, success
  tracking file) but not the loop.
- **Self-improvement via training — MEDIUM (as an idea).** The "scaffold as RL training
  environment" framing and the `reward_func`/RL-adapter plumbing are a legitimate path to
  improving the *policy model* over time — complementary to in-context/evolutionary
  improvement, and worth noting as a design stance ("the agent's substrate is also its training
  env").

## 8. Reusable assets

Concrete, quotable artifacts (collected as evidence; not assembled into a design). All paths
are `repo@481b4344f3d07de42082f367ecda4381f81c22c8`.

**(1) Memory write-gate prompt** — `src/opensage/memory/storage_decider.py`
(`STORAGE_DECISION_PROMPT`): the STORE/SKIP criteria + JSON schema
(`should_store/content_type/summary/confidence/reason`), default-to-not-store on error.
Reusable as a generic "is this worth remembering?" gate.

**(2) Agentic memory-op prompt** — `src/opensage/memory/update/operation_decider.py`: the
ADD/UPDATE/DELETE/NONE selector (`max_tokens=10`, default-to-ADD on parse failure). Reusable
for self-organizing long-term memory.

**(3) Explore-summary prompt + two-phase harness** —
`benchmarks/swe_bench_pro/swe_bench_pro.py` (`_summarize_explore_session`, `_run_explore_agent`,
`_build_user_message`): the fixed summary format (Key Findings / Relevant Files / Code
Structure / Suggested Approach) and the budget-then-recover pattern.

**(4) Ensemble aggregation prompt + message-board coordination** —
`src/opensage/session/opensage_ensemble_manager.py` (`execute_agent_ensemble`): the
per-instance "you are one of N peers, coordinate via post_to_board, read `_message_board_diff`"
preamble and the "synthesize consensus & disagreement" aggregation prompt.

**(5) Tool-usage / Skill policy prompt** — `src/opensage/agents/opensage_agent.py`
(`OpenSageAgent.__init__`, ~lines 657–671): the "prefer Skills over raw shell; survey the
ecosystem first; **do NOT edit Skills in place — copy into `new_tools/`**" policy. Reusable as
guardrails for a self-extending tool system.

**(6) `SKILL.md` schema** — `docs/wiki/developer_guide/Adding-Tools.md` +
`bash_tools/new_tool_creator/SKILL.md`: YAML frontmatter (`name`, `description`,
`should_run_in_sandbox`, `returns_json`) + `## Usage` / `## Parameters` / `## Return Value` /
`## Requires Sandbox` / `## Timeout`; per-skill `deps/<sandbox>/install.sh` installers (run
once per session via a `/shared` marker). A compact, parseable tool-manifest format.

**(7) Sub-agent creation schema** — `src/opensage/toolbox/general/dynamic_subagent.py`
(`create_subagent`): inputs `(agent_name, instruction, model_name, tools_list, enabled_skills,
description)`; always-injected baseline tools; the per-subagent `[Tooling policy]` guardrail;
`search_agent` keyword scoring (name +2 / desc +1). Reusable schema for runtime agent spawning.

**(8) File-memory contract** — injected verbatim in `opensage_agent.py`
(`generate_sandbox_structure_description`): `/mem/<agent>/planning.md` (living plan; read
before, update after), per-session trajectory dumps, `/mem/topology.json`, and the
`/mem/shared/knowledge.jsonl` schema (`key/value/tags/source/updated_at`) with "curate and
consult before major decisions." A lightweight, file-only alternative to graph memory.

**(9) Short-term-memory graph schema** — `src/opensage/patches/neo4j_logging.py` (described in
README): `AgentRun`/`Event`/`RawToolResponse` nodes; `HAS_EVENT`/`AGENT_CALLS`/
`SUMMARIZES_EVENTS`/`SUMMARIZES_TOOL_RESPONSE` edges. A concrete data model for recording and
later traversing an agent's execution history (incl. parent→child agent calls).

**(10) RL reward interface** — `src/opensage/evaluation/rl_adapters/benchmark_interface.py`: a
benchmark exposes `get_prompt(sample)` + `reward_func(args, sample)->{"score":...}`
(+ optional pre/post-process); adapters for slime & AReaL. A clean contract if you want the
substrate to double as a training/eval environment.

## 9. Signal assessment

**Overall signal: MEDIUM.** (High as a *catalog of practical scaffolding patterns* for a
long-horizon, self-extending coding agent; Low as a *source of evolutionary / verifiable-
self-improvement machinery*, which it does not contain.)

- **What's solidly real and useful:** the explore→solve two-phase pattern; the LLM-judge
  memory write-gate + agentic memory updater; the Skill format with the copy-don't-mutate
  rule; the searchable/persistable agent pool; heterogeneous per-tool sandboxing + async
  tasks; the multi-model ensemble. These are credible, well-factored, and borrowable.
- **What's aspirational / unverified:** all headline benchmark numbers (not reproducible from
  this repo; depend on unshipped configs + external infra + frontier models); the "AI fully
  designs its own system" autonomy (authors admit models underuse it); the framing as
  "self-improving" (no in-loop verification or selection).
- **Confidence:** High on the *code-level mechanics* (I read the load-bearing modules at a
  pinned SHA). Medium on *intent/positioning* (corroborated by the arXiv paper + Berkeley RDI
  blog + site). Low/none on *performance claims* (no independent reproduction exists).
- **Could NOT verify:** (a) any benchmark result; (b) that the shipped repo == the exact code
  behind SageAgent's reported numbers; (c) real-world efficacy of the Neo4j long-term memory
  beyond the small ablation; (d) robustness of dynamic-agent persistence/restore across
  versions (legacy-snapshot caveats noted in README); (e) any third-party/skeptical review
  (none found).

## 10. References

**Primary — code** (all `repo@481b4344f3d07de42082f367ecda4381f81c22c8`):
- `repo@481b4344:README.md` — features, project structure, short-term memory, fuzzing arch.
- `repo@481b4344:src/opensage/agents/opensage_agent.py` — `OpenSageAgent`, `ToolLoader`,
  Skill→prompt generation, file-memory contract, tool-usage policy.
- `repo@481b4344:src/opensage/toolbox/general/dynamic_subagent.py` — `create_subagent`,
  `list_active_agents`, `search_agent`, `call_subagent_as_tool`.
- `repo@481b4344:src/opensage/session/opensage_dynamic_agent_manager.py` — agent pool,
  `AgentMetadata`, lifecycle, persistence.
- `repo@481b4344:src/opensage/session/opensage_ensemble_manager.py` — horizontal ensemble,
  message board, aggregation prompt.
- `repo@481b4344:src/opensage/toolbox/finish_task/finish_task.py` — trivial `finish_task`.
- `repo@481b4344:src/opensage/memory/storage_decider.py` — `StorageDecider` + prompt.
- `repo@481b4344:src/opensage/memory/update/operation_decider.py` — `LLMOperationDecider` +
  prompt.
- `repo@481b4344:src/opensage/util_agents/memory_management_agent/agent.py` — memory agent
  (one-line instruction).
- `repo@481b4344:src/opensage/evaluation/rl_adapters/benchmark_interface.py` — RL reward
  interface.
- `repo@481b4344:benchmarks/swe_bench_pro/swe_bench_pro.py` — explore→solve, official eval,
  success tracking.
- `repo@481b4344:src/opensage/bash_tools/new_tool_creator/{SKILL.md,scripts/create_new_tool.sh}`
  — AI-creates-tools scaffolding.
- `repo@481b4344:docs/wiki/user_guide/Introduction.md` — "Agent Development 2.0" framing.
- `repo@481b4344:docs/wiki/developer_guide/Adding-Tools.md` — Skill/tool docs.

**Primary — paper / first-party:**
- arXiv:2602.16891 — "OpenSage: Self-programming Agent Generation Engine," Li, Wang, Dai, Nie,
  Peng, Liu, Zhang, Zhu, He, Wang, Ding, Chen, **Guo**, **Song** (Berkeley RDI; ICML format).
  https://arxiv.org/abs/2602.16891 (HTML: https://arxiv.org/html/2602.16891v1)
- Berkeley RDI blog — https://rdi.berkeley.edu/blog/opensage/ (architecture overview, ablation
  numbers, "looking forward / single AI-centric loop" vision).
- Project site — https://www.opensage-agent.ai/ and https://www.opensage-agent.ai/adk.html
  (feature comparison table, SageAgent benchmark headline numbers, "model improvement"
  honesty note).
- Docs — https://docs.opensage-agent.ai / https://docs.adk.opensage-agent.ai (Introduction,
  Setup, Adding-Tools, Plugins, Sandboxes).
- GitHub repo — https://github.com/opensage-agent/opensage-adk (created 2026-03-23, pushed
  2026-04-07, 90★/20 forks, Apache-2.0).

**Secondary:**
- ResearchGate mirror of the paper —
  https://www.researchgate.net/publication/400970281_OpenSage_Self-programming_Agent_Generation_Engine
- Discord community — https://discord.gg/zbKe5ue8xc (linked from README).

*Independent / skeptical analyses: none found as of 2026-06-04.*
