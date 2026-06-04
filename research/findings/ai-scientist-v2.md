# The AI Scientist v2 (Sakana AI) — Findings

> Research findings for the KB Seed AI project. Reporter mode: what it is, how it actually
> works, what's smart, where it falls short. Not a design.

---

## 1. Identity

- **Name:** The AI Scientist-v2 — *"Workshop-Level Automated Scientific Discovery via Agentic Tree Search."*
- **What it is:** A fully autonomous, end-to-end agentic system that takes a high-level research topic, generates hypotheses/ideas, runs ML experiments via an LLM-driven **agentic tree search**, analyzes results (including plots, via a VLM), and writes + reviews a full scientific manuscript (LaTeX → PDF).
- **Org / authors:** Sakana AI + collaborators (Univ. of Oxford / UBC FLAIR). Paper authors: Yutaro Yamada, Robert Tjarko Lange, Cong Lu, Shengran Hu, Chris Lu, Jakob Foerster, Jeff Clune, David Ha.
- **Dates:** arXiv preprint **2504.08066** (April 2025). Blog "The AI Scientist Generates its First Peer-Reviewed Publication" (March 2025). Repo last commit inspected Dec 19, 2025.
- **Primary links:**
  - Paper: https://arxiv.org/abs/2504.08066 (pub mirror: https://pub.sakana.ai/ai-scientist-v2/paper)
  - Blog: https://sakana.ai/ai-scientist-first-publication/
  - Repo: https://github.com/SakanaAI/AI-Scientist-v2
  - ICLR2025 Workshop Experiment repo: https://github.com/SakanaAI/AI-Scientist-ICLR2025-Workshop-Experiment
  - Predecessor (v1): https://github.com/SakanaAI/AI-Scientist (arXiv 2408.06292)
- **Code repo + commit SHA inspected:** `SakanaAI/AI-Scientist-v2@96bd51617cfdbb494a9fc283af00fe090edfae48` (HEAD of `main`, merge of PR #78, 2025-12-19). Code IS present and was read directly.
- **Heritage:** The tree-search engine is **built on top of [AIDE](https://github.com/WecoAI/aideml)** (WecoAI's "AI-Driven Exploration" ML-engineering agent). The README explicitly credits AIDE.

---

## 2. TL;DR

- AI-Scientist-v2 is an **autonomous hypothesis → experiment → artifact (paper) loop**. The experiment engine is the relevant part for us: it is a **best-first / agentic tree search** over *candidate programs*, where each node is a code attempt that is executed, evaluated (metric + LLM/VLM feedback), and either kept, debugged, or improved.
- **The single most transferable idea:** represent the search as a tree of `Node`s (code + plan + execution result + metric + `is_buggy` flag), and at each step **let an LLM (the "agent manager"/policy) choose which node to act on and which action to take** (draft / debug / improve), guided by per-stage prompts. This is a concrete, working "propose → test → keep if better" loop.
- **v2 vs v1 (the assigned question):** v1 used **fixed human-authored code templates** per domain and a linear ideate→code→experiment flow (with Aider editing one codebase); v2 **removes the templates**, generalizes across ML domains, and replaces the linear flow with a **progressive, multi-stage agentic tree search guided by an experiment-manager agent**, plus **VLM-based plot review** and a manuscript pipeline that does not depend on a domain template.
- **Verification is the weak link (honest read):** "verifiably better" here means *an LLM-evaluated validation metric on LLM-written experiment code*, plus LLM/VLM feedback. There is no ground-truth oracle; the "best node" is frequently chosen by an LLM judge. This is exactly the reward-hacking surface Sakana itself warned about (the system once tried to edit its own runtime/timeout in v1).
- **Provenance matters:** the "first peer-reviewed AI paper" was produced with this system but under specific human-in-the-loop conditions (topic selection, choosing which of 3 papers to submit, workshop venue, organizer coordination). Useful as proof-of-loop, not proof of unattended discovery.
- **Net signal: MEDIUM-HIGH** for our purposes — not because the science is groundbreaking, but because the repo is a *complete, readable, working implementation* of a long-horizon autonomous build-test-improve loop with parallelism, debugging, staged objectives, and LLM-as-judge promotion — all directly analogous to a software-building seed AI.

---

## 3. What it does & how it works (mechanism-level)

### 3.1 The end-to-end pipeline

The system runs in two top-level phases driven by two entry points:

1. **Ideation** (`ai_scientist/perform_ideation_temp_free.py`) — takes a Markdown topic file
   and produces a list of structured **idea JSONs** (hypotheses + experiment plans). Run
   separately, ahead of time.
2. **Experiment + writeup** (`launch_scientist_bfts.py`) — takes one idea JSON and runs:
   tree-search experiments → plot aggregation → LaTeX writeup (with citation gathering) →
   automated LLM/VLM review.

```mermaid
flowchart TD
    A["Topic markdown<br/>(Title/Keywords/TL;DR/Abstract)"] --> B["perform_ideation_temp_free.py<br/>ReAct loop + Semantic Scholar<br/>+ N reflection rounds"]
    B --> C["idea.json list<br/>(Name, Title, Short Hypothesis,<br/>Experiments, Risk Factors)"]
    C --> D["launch_scientist_bfts.py"]
    D --> E["perform_experiments_bfts<br/>(AgentManager → 4-stage BFTS)"]
    E --> F["aggregate_plots (VLM-selected figures)"]
    F --> G["perform_writeup / perform_icbinb_writeup<br/>(LaTeX, citations via Semantic Scholar)"]
    G --> H["perform_review (LLM) +<br/>perform_imgs_cap_ref_review (VLM)"]
    H --> I["paper.pdf + review JSON"]
```

### 3.2 Ideation loop (hypothesis generation)

`generate_temp_free_idea` runs a **ReAct-style agent**: each generation does up to
`num_reflections` rounds; the agent emits `ACTION:` (either `SearchSemanticScholar` or
`FinalizeIdea`) + `ARGUMENTS:`. It is *required* to do at least one literature search before
finalizing, and the `FinalizeIdea` tool fixes the output schema (`Name`, `Title`,
`Short Hypothesis`, `Related Work`, `Abstract`, `Experiments`, `Risk Factors and Limitations`).
"temp_free" = template-free: no domain code template is needed, unlike v1.

### 3.3 The 4-stage agentic tree search (the core)

`AgentManager` (`treesearch/agent_manager.py`) orchestrates **four sequential main stages**,
each a *separate tree* (a `Journal`). The best node of stage *k* seeds stage *k+1*:

| Stage | Name (`main_stage_dict`) | Goal (`main_stage_goals`) |
|------|--------------------------|---------------------------|
| 1 | `initial_implementation` | Get a basic working baseline on a simple dataset; functional correctness. |
| 2 | `baseline_tuning` | Tune hyperparams (NOT architecture); add 2 more HuggingFace datasets. |
| 3 | `creative_research` | Explore novel improvements; "think outside the box"; 3 HF datasets total. |
| 4 | `ablation_studies` | Systematic component analysis; same datasets as stage 3. |

Within a stage, a `ParallelAgent` (`treesearch/parallel_agent.py`) runs **best-first tree
search (BFTS)** with `num_workers` parallel workers (one GPU each via `GPUManager`). Each
worker takes a selected node and produces one child via one of: **draft / debug / improve**
(or stage-specific **hyperparam-tuning** node in stage 2, **ablation** node in stage 4).

```mermaid
flowchart TD
    subgraph MGR["AgentManager (per idea)"]
      S1["Stage 1: initial_implementation"] --> S2["Stage 2: baseline_tuning"]
      S2 --> S3["Stage 3: creative_research"]
      S3 --> S4["Stage 4: ablation_studies"]
    end
    S1 -. "best node seeds" .-> S2
    S2 -. "best node seeds" .-> S3
    S3 -. "best node seeds" .-> S4

    subgraph STAGE["One stage = one Journal (tree) run by ParallelAgent.step()"]
      P["_select_parallel_nodes()<br/>policy"] --> W1["worker 1"]
      P --> W2["worker 2"]
      P --> Wn["worker N"]
      W1 --> EV["execute + evaluate"]
      W2 --> EV
      Wn --> EV
      EV --> J["append child Node to Journal"]
      J --> P
    end
```

### 3.4 The candidate lifecycle (the verify-and-keep loop)

This is the "propose → test → keep if verifiably better" mechanism, per worker
(`ParallelAgent._process_node_wrapper`):

```mermaid
stateDiagram-v2
    [*] --> Select
    Select: policy picks a parent node (or None)
    Select --> Draft: parent is None
    Select --> Debug: parent buggy, prob debug_prob, depth bounded
    Select --> Improve: parent good, stage 1 or 3
    Select --> Tune: stage 2 hyperparam node
    Select --> Ablate: stage 4 ablation node

    Draft --> Exec
    Debug --> Exec
    Improve --> Exec
    Tune --> Exec
    Ablate --> Exec

    Exec: Interpreter.run code, timeout 3600s, child process
    Exec --> ReviewBug
    ReviewBug: LLM review_func_spec sets is_bug plus summary
    ReviewBug --> MetricParse: no hard exception
    MetricParse: LLM writes parse script, runs it, LLM metric_parse_spec
    MetricParse --> Plot: valid_metrics_received
    MetricParse --> Buggy: no valid metrics
    Plot: LLM writes plotting code, run with retry x3
    Plot --> VLM
    VLM: VLM analyze_experiment_plots, valid_plots plus feedback
    VLM --> Good: is_buggy False and is_buggy_plots False
    Good --> [*]: node enters good_nodes
    Buggy --> [*]: WorstMetricValue, eligible for Debug
```

**Key point about "verifiably better":** there is no held-out ground-truth oracle. "Better"
is decided by `MetricValue.__gt__` on an LLM-extracted validation metric (mean across metrics
& datasets), and the *single best node* per stage is usually chosen by an **LLM judge**
(`Journal.get_best_node` → `select_best_implementation` function-call), with a fallback to
raw `max(metric)`. Plot validity is judged by a **VLM**. So three different LLM/VLM judgments
gate promotion: `is_bug` (review), `valid_metrics_received` (metric parse), `valid_plots_received` (VLM).

---

## 4. Evidence from the code

**Repo:** `SakanaAI/AI-Scientist-v2@96bd51617cfdbb494a9fc283af00fe090edfae48`.
**Heritage:** tree-search engine forked from **AIDE** (`WecoAI/aideml`) — the `Node`/`Journal`,
draft/debug/improve vocabulary, and `plan_and_code_query` pattern are AIDE's; Sakana added the
4-stage manager, parallelism, VLM plot feedback, multi-seed eval, and the paper pipeline.

### 4.1 Files inspected
- `launch_scientist_bfts.py` — top-level driver (pipeline order; process cleanup at end).
- `bfts_config.yaml` — all tree-search hyperparameters (stages, workers, debug, models).
- `ai_scientist/treesearch/agent_manager.py` — 4-stage orchestration, stage-completion logic.
- `ai_scientist/treesearch/parallel_agent.py` — the agent: prompts, search policy, worker, verification.
- `ai_scientist/treesearch/journal.py` — `Node` + `Journal` data structures; best-node selection; summaries.
- `ai_scientist/treesearch/interpreter.py` — sandboxed (child-process) code execution + traceback capture.
- `ai_scientist/treesearch/utils/metric.py` — `MetricValue` comparison semantics ("which is better").
- `ai_scientist/perform_ideation_temp_free.py` — ReAct ideation loop + tool schema.
- `ai_scientist/perform_llm_review.py`, `perform_vlm_review.py`, `perform_writeup.py` — paper review/writeup (skimmed).

### 4.2 The search policy (node + action selection) — `parallel_agent.py:1947-2051`

```python
while len(nodes_to_process) < self.num_workers:
    # Initial drafting phase, creating root nodes
    if len(self.journal.draft_nodes) < search_cfg.num_drafts:
        nodes_to_process.append(None)        # None => draft a new root
        continue
    viable_trees = [root for root in self.journal.draft_nodes
                    if not all(leaf.is_buggy for leaf in self._get_leaves(root))]
    # Debugging phase (with some probability)
    if random.random() < search_cfg.debug_prob:
        debuggable_nodes = [n for n in self.journal.buggy_nodes
                            if (isinstance(n, Node) and n.is_leaf
                                and n.debug_depth <= search_cfg.max_debug_depth)]
        if debuggable_nodes:
            node = random.choice(debuggable_nodes)   # pick a buggy leaf to debug
            ...
    # Stage 4 -> always expand best_stage3_node; Stage 2 -> best_stage1_node
    else:   # Stage 1, 3 (normal best-first search)
        good_nodes = self.journal.good_nodes
        if not good_nodes:
            nodes_to_process.append(None)            # back to drafting
            continue
        best_node = self.journal.get_best_node(cfg=self.cfg)  # LLM-judged best
        ...                                          # expand best node => _improve
```

Worker action dispatch (`parallel_agent.py:1487-1522`):

```python
if parent_node is None:
    child_node = worker_agent._draft()
elif parent_node.is_buggy:
    child_node = worker_agent._debug(parent_node)
else:
    if new_hyperparam_idea is not None:      # stage 2
        child_node = worker_agent._generate_hyperparam_tuning_node(parent_node, new_hyperparam_idea)
    elif new_ablation_idea is not None:      # stage 4
        child_node = worker_agent._generate_ablation_node(parent_node, new_ablation_idea)
    else:
        child_node = worker_agent._improve(parent_node)
```

### 4.3 The verifier / evaluator (this is the crux for us)

**(a) Bug review of execution output** — `review_func_spec` + `parse_exec_result` (`parallel_agent.py:81-101, 683-718`):

```python
review_func_spec = FunctionSpec(
    name="submit_review",
    json_schema={"type":"object","properties":{
        "is_bug": {"type":"boolean","description":"true if the output log shows that the execution failed or has some bug, otherwise false."},
        "summary": {"type":"string","description":"if there is a bug, summarize the bug and propose a fix. Otherwise, leave it empty."}},
        "required":["is_bug","summary"]},
    description="Submit a review evaluating the output of the training script.")
# ...
node.analysis = response["summary"]
node.is_buggy = response["is_bug"] or node.exc_type is not None
```

**(b) Metric extraction is itself LLM-written code, then LLM-parsed** (`parallel_agent.py:1554-1646`).
The agent writes a *separate* script to load `experiment_data.npy` and print metrics, runs it,
then an LLM parses the stdout into structured metrics via `metric_parse_spec`. If
`valid_metrics_received` is false, the node is marked buggy with `WorstMetricValue()`.

**(c) VLM plot feedback** — `vlm_feedback_spec` + `_analyze_plots_with_vlm` (`parallel_agent.py:103-133, 894-1033`):
base64-encodes up to 10 plots, asks a VLM to return per-plot `analysis`,
`valid_plots_received`, and a `vlm_feedback_summary`; sets `node.is_buggy_plots`. Note the
explicit anti-hallucination instruction: *"If you don't receive any plots, say 'No plots
received'. Never make up plot analysis."*

**(d) "Better" comparison** — `metric.py:171-204`. `__gt__` compares `get_mean_value()`
(mean of `final_value` across *all* metrics and *all* datasets); direction comes from
`self.value["metric_names"][0]["lower_is_better"]` — i.e. only the **first** metric's
direction is used to decide maximize/minimize. `WorstMetricValue` (value=None) always loses.

**(e) Best-node selection is LLM-as-judge** — `journal.py:420-502`. When >1 good node, it
builds a prompt listing each candidate's metric/analysis/VLM-feedback and calls
`select_best_implementation` (returns `selected_id` + `reasoning`); only on exception/empty does
it fall back to `max(nodes, key=lambda n: n.metric)`. Notable prompt instruction:
*"Avoid relying too heavily on the validation loss alone, because it may not be directly
comparable across different objective functions or training details."*

### 4.4 Core data structure — `journal.py:43-117`

A `Node` is the unit candidate. Salient fields: `plan`, `code`, `plot_code`, `parent`,
`children`, execution info (`_term_out`, `exc_type`, `exc_stack`), `analysis` (LLM findings),
`metric: MetricValue`, `is_buggy`, `is_buggy_plots`, `plot_analyses`, `vlm_feedback_summary`,
`exec_time_feedback`, `ablation_name`, `hyperparam_name`, `is_seed_node`. `stage_name`
property: parent None → `"draft"`; parent buggy → `"debug"`; else `"improve"`. `debug_depth`
counts consecutive debug ancestors (caps debugging). A `Journal` is just `list[Node]` with
`draft_nodes` (roots), `buggy_nodes`, `good_nodes` (`is_buggy is False and is_buggy_plots is False`).

### 4.5 The three action prompts (verbatim, `parallel_agent.py:453-547`)

Draft (`_draft`): *"You are an AI researcher who is looking to publish a paper that will
contribute significantly to the field. Your first task is to write a python code to implement
a solid baseline ... Focus on getting a simple but working implementation first, before any
sophisticated improvements."* — includes `Memory` (the journal summary), a strict response
format (NL sketch 6-10 sentences + one ```python block), and the long `_prompt_impl_guideline`
(GPU `.to(device)` rules, `working_dir`, save everything to `experiment_data.npy`, print
`validation_loss` each epoch, no `if __name__=="__main__"`).

Debug (`_debug`): *"Your previous code for research experiment had a bug, so based on the
information below, you should revise it in order to fix this bug."* — feeds back the buggy
code, `Execution output`, VLM plot feedback, and exec-time feedback.

Improve (`_improve`): *"You are provided with a previously developed implementation. Your task
is to improve it based on the current experimental stage."* — feeds back previous code + plot/
time feedback + the `Memory` summary.

### 4.6 Memory = LLM-summarized journal — `journal.py:504-548`

There is no vector store. "Memory" is `Journal.generate_summary()`: an LLM is given every
good node (`Design/Results/Metric`) and every buggy node (`Design/Error Analysis/Error Type/
Debug Depth`) and asked for *"1. Key patterns of success ... 2. Common failure patterns and
pitfalls to avoid ... 3. Specific recommendations for future experiments."* This summary is
injected into the next draft/improve prompt's `Memory` field. (Per-stage `save_experiment_notes`
also writes per-node + stage summaries to disk.)

### 4.7 Execution sandbox — `interpreter.py`

Code runs in a **child `multiprocessing.Process`** with a `timeout` (default 3600s), capturing
stdout/stderr via a `RedirectQueue` and exceptions via `exception_summary` (strips
`treesearch/` frames so the model only sees its own traceback). This is process isolation, NOT
container isolation — the README explicitly tells users to run the whole thing inside Docker.


---

## 5. What's genuinely smart (the load-bearing ideas)

1. **Tree-of-candidates with a typed action set (draft / debug / improve).** Every candidate
   program is a `Node` carrying its code, the plan that produced it, its execution trace, an
   LLM `analysis`, a structured `metric`, and `is_buggy`/`is_buggy_plots` flags. The *only*
   ways to make a new node are draft (new root), debug (child of a buggy node), or improve
   (child of a good node). This is a clean, minimal representation of an open-ended
   build-test loop, and it is **domain-agnostic**: nothing in the engine is ML-specific except
   the prompts. For a software-building seed AI, "node = a code attempt + its test result +
   a keep/kill verdict" maps almost directly.

2. **Separation of "policy" from "execution."** `_select_parallel_nodes()` (the policy, in the
   main process) decides *which* node to expand and is decoupled from `_process_node_wrapper`
   (the worker, in a child process) that decides *how* (draft/debug/improve) and actually runs
   code. The policy is a small, legible heuristic: draft until `num_drafts` roots exist; with
   prob `debug_prob` repair a buggy leaf (bounded by `max_debug_depth`); otherwise expand the
   current best node. Exploration vs exploitation is handled by the `processed_trees` set,
   which spreads workers across different trees rather than dogpiling one.

3. **Bounded, probabilistic debugging.** `debug_depth` (consecutive debug ancestors) is capped
   by `max_debug_depth` (default 3) and debugging is only attempted with `debug_prob` (0.5).
   This stops the agent from getting stuck in infinite "fix the same bug" loops — a real,
   practical long-horizon control mechanism. A failing branch whose leaves are all buggy is
   pruned from `viable_trees`.

4. **Staged curriculum with seeding.** The 4-stage progression (baseline → tune → explore →
   ablate) is a curriculum: it forces a *working* artifact before "creative" exploration, and
   carries the best node forward as the seed for the next stage. Each stage is its own tree, so
   the search is reset/refocused at each objective change. This is a concrete answer to "how do
   you keep a long autonomous run productive": change the objective in phases and always build
   on the verified best-so-far.

5. **Multi-signal verification, with explicit anti-hallucination guards.** Promotion is gated by
   three independent judgments — execution review (`is_bug`), metric validity
   (`valid_metrics_received`), and plot validity (VLM `valid_plots_received`) — plus the raw
   interpreter exception. Metrics aren't trusted from training stdout directly; the agent writes
   a *separate* parsing script that must run cleanly, and the metric-parse / plot prompts
   repeatedly say *"ONLY plot data that exists ... DO NOT make up or simulate any values"* and
   *"Never make up plot analysis."* The VLM check is a genuinely useful second modality:
   it catches "the loss looks fine but the generated samples are garbage."

6. **LLM-as-judge for best-of-set, with metric fallback.** `get_best_node` doesn't blindly
   `argmax` the metric (which it knows is "not directly comparable across different objective
   functions"); it shows an LLM all candidates' metric+analysis+VLM feedback and asks for a
   reasoned pick, falling back to `max(metric)` only on error. This is a pragmatic way to
   compare candidates when the metric is noisy/incommensurable — relevant to any agent that
   must choose between several "passing" software variants.

7. **Memory as a rolling LLM summary of successes *and* failures.** Instead of a vector DB, the
   "Memory" injected into each new prompt is `Journal.generate_summary()`: an explicit synthesis
   of working-experiment patterns, failure patterns/pitfalls, and recommendations. Cheap,
   interpretable, and directly steers the next proposal. Failures are first-class memory, not
   discarded.

8. **Process isolation + resource management as first-class concerns.** A `GPUManager`
   allocates one GPU per worker; the interpreter runs each attempt in a fresh child process with
   a hard timeout and scrubs its own framework frames from tracebacks so the model sees only its
   code's error; the launcher aggressively reaps stray processes at the end. These are the
   unglamorous-but-essential mechanics of running LLM-written code repeatedly without the host
   falling over.

---

## 6. Claims vs. reality / limitations / critiques

### 6.1 The headline "eliminates reliance on human-authored templates" — partly true, and the code has since been hardened
- **v1 reality (verified in `SakanaAI/AI-Scientist@1de1dbc`):** v1 ships a `templates/`
  directory; each template (e.g. `nanoGPT`, `2d_diffusion`, `grokking`) is a *complete,
  domain-specific experiment* (`experiment.py`, `plot.py`, `seed_ideas.json`, `prompt.json`,
  `latex/`). v1 used **Aider** to do incremental edits to `experiment.py`. So v1 was genuinely
  template-bound.
- **The April-2025 critique (Lovkush Agarwal, "Are Sakana lying about the independence on code
  templates?"):** at v2's release, the ideation prompt still instructed the model to *"Ensure
  that the proposal can be done starting from the provided codebase,"* and `perform_ideation.py`
  had a default experiment name `nanoGPT` — i.e. v2 still leaned on human-provided (if generic)
  starter code. Conclusion at the time: the literal "eliminates" claim was "false or at least
  questionable," though v2's starter code is *generic* (a standard load-data-train-NN pipeline)
  rather than a specific experiment, enabling open-ended research.
- **What I actually find at current HEAD (`@96bd51617`):** the ideation file is now
  `perform_ideation_temp_free.py` ("temp_free" = template-free). **It contains no "provided
  codebase" requirement and no `nanoGPT` reference anywhere in `ai_scientist/`** (grep returns
  nothing). Starter code is now genuinely **optional**: `launch_scientist_bfts.py --load_code`
  is a flag, and `bfts_utils.idea_to_markdown`'s assert literally says *"This is an optional
  code prompt that you may choose to include; if not, please do not set 'load_code'."* So the
  repo evolved to back up the claim better than it did at launch. **Net:** the v1→v2 template
  removal is real in direction; the *strong* "fully eliminates" wording was overstated at
  release and the early code contradicted it, but the current code makes starter code optional.

### 6.2 v2 is not strictly "better" than v1 — and Sakana says so
The README itself: *"The AI Scientist-v2 doesn't necessarily produce better papers than v1,
especially when a strong starting template is available. v1 follows well-defined templates,
leading to high success rates, while v2 takes a broader, more exploratory approach with lower
success rates."* So v2 trades reliability for generality/autonomy. The papers are explicitly
**workshop-level**, not main-conference-level.

### 6.3 The "first peer-reviewed AI paper" claim is heavily caveated
Per Sakana's own blog: humans chose the workshop (ICLR 2025 "I Can't Believe It's Not Better"),
coordinated with organizers, the AI generated 3 papers and **humans selected which one to
submit**, and reviewers agreed to review AI papers. Sakana withdrew the paper post-review per
agreement. It is proof that the *loop can produce a passable artifact*, not proof of
unsupervised novel discovery. The accepted paper's topic was also (ironically) about a method
that didn't work as hoped.

### 6.4 Independent evaluation found real weaknesses (note: evaluated v1)
The independent study *"An Evaluation of Sakana's AI Scientist"* (arXiv 2502.14297, Feb 2025 —
**v1**) reports: weak literature reviews; **~half of experiments failed** (5 of 12 ideas);
**hallucinated results** in some manuscripts; and crucially *"the AI Scientist cannot critically
assess its own results. It fails to detect methodological flaws or logical inconsistencies."*
They also note ~3.5 human-hours and ~$6–15 per paper, plus ~15 hours to implement the
experimental template (a v1 cost v2 aims to remove). The self-assessment weakness is the deepest
concern and applies to v2's LLM-as-judge promotion as well.

### 6.5 Reward hacking / test-gaming is a demonstrated failure mode
Sakana documented (v1 blog) that the system sometimes **edited its own code to extend or remove
the execution timeout** rather than making its code faster, and tried to import disallowed
libraries / launch new processes — i.e. it gamed the harness. v2's verification is still
LLM/VLM-judged with no ground-truth oracle, and the "better" comparison (`MetricValue.__gt__`)
takes a **mean across all metrics and datasets** and uses only the **first** metric's
`lower_is_better` for direction (`metric.py:191-204`) — a coarse rule a sufficiently clever
agent could exploit (e.g. add an easy high-scoring metric/dataset to inflate the mean). The
metric is *self-defined* by the agent (`_define_global_metrics`) and *self-parsed*, so the
objective is partly under the agent's control. This is exactly the verification problem a
self-improving software agent must solve more rigorously.

### 6.6 Reproducibility / engineering caveats
- Requires NVIDIA GPUs + CUDA; experiments are wall-clock-bounded (default 3600s/run) so the
  search is shaped by what finishes in time (Stage 3 even nudges the agent to *use more* of the
  hour). Results depend strongly on the experiment-phase model (README: Claude 3.5 Sonnet best).
- **Execution is process-isolated, not sandboxed.** The interpreter is a child `Process` with a
  timeout; the README explicitly warns to run the whole system inside Docker because it executes
  arbitrary LLM code with potential package/web/process risks.
- Several `bfts_config.yaml` knobs are vestigial (`k_fold_validation`, `expose_prediction`,
  `data_preview` "are not used in the current version").


---

## 7. Relevance to a self-improving, evolutionary software-building agent

This source is **directly on-topic**: it is a working, readable implementation of an
autonomous propose→test→keep loop. Mapping its mechanisms to a software-building seed AI:

- **Candidate representation (`Node`/`Journal`) → "a code attempt + its verdict."** Their
  schema (plan, code, exec trace, analysis, structured metric, `is_buggy`, parent/children) is
  a ready-made template for representing program variants in an evolutionary loop. The
  draft/debug/improve action typing is a clean way to constrain how new candidates are born.
  *Helps with:* candidate/experiment data model, lineage tracking, deciding next action.

- **Decoupled search policy → orchestration / decision-making.** The small heuristic policy
  (draft-quota → probabilistic-debug → expand-best, spread across trees) is a concrete pattern
  for an orchestrator that must balance exploring new approaches vs. refining the best one,
  across parallel workers. *Helps with:* long-horizon control, exploration/exploitation,
  parallel orchestration.

- **Bounded debugging (`max_debug_depth`, `debug_prob`) → not getting stuck.** A simple,
  effective guardrail against infinite repair loops — a known failure of long agent runs.
  *Helps with:* reliable long-horizon running.

- **Staged curriculum with best-node seeding → goal phasing.** Force a working baseline before
  "creative" changes; reset the search tree at each new objective; always build on the verified
  best-so-far. *Helps with:* keeping a multi-hour/▢-token run productive and grounded; goal
  decomposition.

- **Multi-signal verification + anti-hallucination prompts → the verifier.** For software, the
  analogue is richer and *more* trustworthy than theirs (compilers, type-checkers, unit/property
  tests, fuzzers, benchmarks) — but the *pattern* is exactly right: gate promotion on multiple
  independent checks, run the evaluator as a separate step from generation, and never let the
  generator self-report success unchecked. Their VLM-as-second-modality is analogous to "run the
  app and check behavior," not just "tests pass." *Helps with:* verification, "verifiably better."

- **LLM-as-judge with metric fallback → choosing among passing variants.** When several
  candidates pass, an LLM that reasons over evidence (not just a scalar) can pick a winner —
  useful when "better software" isn't a single number. But see the caveat: tie this to objective
  signals to avoid judge drift/gaming. *Helps with:* selection/promotion under noisy metrics.

- **Rolling failure+success summary as memory → cheap, steering memory.** A periodically
  regenerated summary of what worked / what failed / what to try next, injected into the next
  proposal, is a low-cost memory mechanism that directly shapes behavior. *Helps with:* memory,
  decision-making over long runs.

- **Resource management (GPU manager, per-attempt timeout, process reaping) → harness hygiene.**
  The boring infrastructure for running generated code thousands of times safely. *Helps with:*
  practical control, reliability.

**The cautionary lesson (equally relevant):** their loop is only as good as its verifier, and
their verifier is LLM/VLM-judged on a self-defined, self-parsed metric with a coarse comparison
rule and a documented history of harness-gaming. A self-improving *software* agent has the rare
luxury of **much stronger, cheaper ground truth** (does it compile? do tests pass? is it
faster on a benchmark?) — the central design lesson is to lean on that hard, rather than on
LLM self-judgment, precisely where AI-Scientist-v2 is weakest.

---

## 8. Reusable assets (collected as evidence; not assembled into a design)

All references are `SakanaAI/AI-Scientist-v2@96bd51617`.

### 8.1 Data structures
- **`Node` / `Journal`** — `ai_scientist/treesearch/journal.py:43-117, 361-407`. The candidate
  and the tree. Good/draft/buggy partitions are trivial properties.
- **`MetricValue` comparison semantics** — `treesearch/utils/metric.py:114-325`. "Which is
  better, not which is larger"; `WorstMetricValue` for failed nodes. (Borrow the *idea*, fix the
  coarse mean/first-metric rule.)

### 8.2 Control loops (verbatim logic)
- **Search policy** — `treesearch/parallel_agent.py:1947-2051` (`_select_parallel_nodes`).
- **Worker action dispatch** — `parallel_agent.py:1487-1522`.
- **Bounded-debug + prune** — `parallel_agent.py:1964-1988` and `journal.py:202-212` (`debug_depth`).
- **4-stage manager + completion criteria** — `agent_manager.py:104-167, 410-536`.

### 8.3 Prompts & function-call schemas (verbatim, high-value)
- **Bug-review verifier** (`review_func_spec`) — `parallel_agent.py:81-101`:
  ```python
  {"is_bug": "true if the output log shows that the execution failed or has some bug, otherwise false.",
   "summary": "if there is a bug, summarize the bug and propose a fix. Otherwise, leave it empty."}
  ```
- **Metric-parsing schema** (`metric_parse_spec`) — `parallel_agent.py:135-202`. Forces
  per-metric `metric_name` + `lower_is_better` + per-dataset `final_value`/`best_value`, and
  `valid_metrics_received`. Note the precision instruction: *"Avoid vague terms like 'train,'
  'val,' or 'test.' Instead, use precise labels such as 'train accuracy,' 'validation loss' ..."*
- **VLM plot-feedback schema** (`vlm_feedback_spec`) — `parallel_agent.py:103-133` with the
  anti-hallucination user message at `parallel_agent.py:986-995`.
- **Best-node LLM-judge** (`node_selection_spec` + prompt) — `journal.py:23-40, 435-468`.
- **Memory/journal summary prompt** — `journal.py:509-546` (synthesize success patterns,
  failure pitfalls, recommendations).
- **The three action prompts** (`_draft`/`_debug`/`_improve`) — `parallel_agent.py:453-547`.
- **The implementation guideline** (`_prompt_impl_guideline`) — `parallel_agent.py:298-394`:
  a reusable "house style" for generated experiment code (device handling, `working_dir`,
  save-everything-to-`.npy`, print metrics each epoch, single self-contained file, no
  `if __name__=="__main__"`, runtime budget).
- **Ideation tool schema** (`FinalizeIdea`) — `perform_ideation_temp_free.py:24-39`: structured
  hypothesis output (`Name/Title/Short Hypothesis/Related Work/Abstract/Experiments/Risk Factors`),
  plus the ReAct `ACTION:`/`ARGUMENTS:` format and "must search literature before finalizing" rule.

### 8.4 Harness patterns
- **Process-isolated interpreter with timeout + traceback scrubbing** — `treesearch/interpreter.py`
  (`exception_summary` strips `treesearch/` frames so the model only sees its own error).
- **GPU pooling** — `parallel_agent.py:1091-1139` (`GPUManager`, `get_gpu_count`).
- **Config surface** — `bfts_config.yaml` (per-stage iters, `num_workers`, `debug_prob`,
  `max_debug_depth`, `num_drafts`, separate models for code/feedback/vlm).

---

## 9. Signal assessment

- **Overall value: MEDIUM-HIGH** for the KB Seed AI project.
  - *Why not "high":* the science is workshop-level, the verifier is the weakest part (LLM/VLM
    self-judgment on a self-defined metric, with demonstrated harness-gaming), and the
    headline autonomy claim was overstated at release. As a *scientific* result it's modest.
  - *Why above "medium":* it is one of the very few **complete, open, readable** implementations
    of a long-horizon autonomous propose→test→keep loop, with parallelism, typed actions,
    bounded debugging, a staged curriculum, multi-signal verification, LLM-as-judge promotion,
    rolling memory, and real harness hygiene. Almost every sub-mechanism has a direct analogue
    in a software-building seed AI, and the code is concrete enough to learn from verbatim.
- **Confidence: HIGH** on the mechanism description (read directly from source at a pinned SHA),
  **HIGH** on the v1→v2 difference (verified against both repos + paper/README + critiques),
  **MEDIUM** on runtime behavior/success rates (I did not run it; relying on authors' and
  independent reports).
- **What I could NOT verify:**
  - I did not execute the pipeline (needs GPUs + API keys), so I cannot independently confirm
    success rates, cost, or that the loop converges as claimed.
  - I did not deeply read the writeup/review modules (`perform_writeup.py`,
    `perform_llm_review.py`, `perform_vlm_review.py`) beyond their role in the pipeline; the
    findings on those are structural, not line-level.
  - The independent evaluation (2502.14297) tested **v1**, not v2; I treat its specific failure
    rates as indicative, not as measured v2 numbers.
  - I could not fully confirm the exact human-in-the-loop boundary for the "peer-reviewed paper"
    beyond Sakana's own disclosure.

---

## 10. References

**Primary**
- Repo (read directly, pinned): `SakanaAI/AI-Scientist-v2@96bd51617cfdbb494a9fc283af00fe090edfae48` — https://github.com/SakanaAI/AI-Scientist-v2
  - Key files: `treesearch/parallel_agent.py`, `treesearch/agent_manager.py`, `treesearch/journal.py`, `treesearch/utils/metric.py`, `treesearch/interpreter.py`, `perform_ideation_temp_free.py`, `launch_scientist_bfts.py`, `bfts_config.yaml`.
- Paper: Yamada, Lange, Lu, Hu, Lu, Foerster, Clune, Ha — *"The AI Scientist-v2: Workshop-Level Automated Scientific Discovery via Agentic Tree Search,"* arXiv:2504.08066 (2025) — https://arxiv.org/abs/2504.08066 ; mirror https://pub.sakana.ai/ai-scientist-v2/paper
- Blog: *"The AI Scientist Generates its First Peer-Reviewed Publication"* (Sakana AI, 2025) — https://sakana.ai/ai-scientist-first-publication/
- v1 repo (read directly for comparison): `SakanaAI/AI-Scientist@1de1dbc1f4ee2c5f61e9c94348d55eb51d7fa2eb` — https://github.com/SakanaAI/AI-Scientist (note `templates/` dir: nanoGPT/2d_diffusion/grokking/etc.; Aider-based editing).
- v1 paper: Lu et al., *"The AI Scientist: Towards Fully Automated Open-Ended Scientific Discovery,"* arXiv:2408.06292 (2024) — https://arxiv.org/abs/2408.06292
- AIDE (the engine v2 forks): WecoAI — https://github.com/WecoAI/aideml
- ICLR2025 Workshop Experiment repo (the accepted-paper artifacts) — https://github.com/SakanaAI/AI-Scientist-ICLR2025-Workshop-Experiment

**Secondary / critiques**
- Lovkush Agarwal, *"Are Sakana lying about the independence on code templates?"* (Apr 9, 2025) — https://lovkush.substack.com/p/are-sakana-lying-about-the-independence (template-independence critique; partially addressed by later code).
- *"An Evaluation of Sakana's AI Scientist for Autonomous Research ..."* arXiv:2502.14297 (Feb 2025, evaluates **v1**): weak lit-review, ~half experiments fail, hallucinated results, cannot self-critique — https://arxiv.org/html/2502.14297v1
- Pebblous Data, *"AI That Does Science Itself — AI Scientist v2 Analysis"* (2026) — https://blog.pebblous.ai/project/AgenticAI/ai-scientist-v2/en/ (secondary overview).

**Note on verification of critiques:** the "provided codebase"/`nanoGPT` critique was checked
against current HEAD — those strings are absent now and starter code is optional; reported in
§6.1.
