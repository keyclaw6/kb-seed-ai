# OpenSpace (HKUDS) — Research Findings

> Status: IN PROGRESS. Written incrementally during research.

## 1. Identity

- **Name:** OpenSpace ("OpenSpace: Make Your Agents: Smarter, Low-Cost, Self-Evolving")
- **Org/authors:** HKUDS (Data Intelligence Lab @ University of Hong Kong; the group behind LightRAG, MiniRAG, AutoAgent, etc.). Lead committer on `main`: **Xu Lingrui** (lrxu0818@gmail.com).
- **What it is (precise):** NOT a software-building seed-AI and NOT an autonomous coding agent. It is a **skill-management / "self-evolving skill" layer** that wraps *around* existing coding/computer-use agents (Claude Code, Codex, Cursor, OpenClaw, nanobot, etc.). It is delivered primarily as an **MCP server** plus a CLI. Its job: observe what a host agent does, distill reusable "skills" (Markdown SOP files in the Anthropic "Agent Skills" format), evolve/merge/refine them over time, and share them via a cloud marketplace. The pitch is token reduction + reusing past experience + "collective intelligence."
- **Dates:** Repo created **2026-03-24**; actively pushed (last push 2026-06-04). v0.1.0 released 2026-04-03.
- **Stars:** ~6,400 (as of 2026-06-04).
- **Primary links:** https://github.com/HKUDS/OpenSpace ; community site https://open-space.cloud/ ; cloud marketplace referenced in-repo.
- **Code inspected:** `HKUDS/OpenSpace@228f8f78073dc4ed0e63fff01c19596c50115d40` (main, commit "fix: Windows PID check", 2026-06-02). Downloaded via codeload tarball (direct `git clone` blocked by sandbox proxy 407). ~46.6K Python LOC / 160 `.py` files + a React frontend + `gdpval_bench/`.

## 2. TL;DR

- **OpenSpace is NOT a software-building seed-AI.** It is a **"self-evolving skill" memory layer** that bolts onto an *existing* host agent (Claude Code, Codex, Cursor, OpenClaw, nanobot…) via an **MCP server**. The host agent does the actual work; OpenSpace watches, distills reusable "skills," evolves them, and serves them back.
- **A "skill" = an Anthropic "Agent Skills"-format Markdown SOP** (a directory with `SKILL.md` + YAML frontmatter `name`/`description` + optional aux scripts). Skills are procedural memory / lessons-learned, not code modules.
- **The evolution engine is the genuinely interesting part for us.** After every task, an LLM "analyst" judges what happened and emits 0–N evolution suggestions of three operator types — **FIX** (in-place surgical patch), **DERIVED** (enhance/merge → new node), **CAPTURED** (mint a brand-new skill). Skills are stored in a **version DAG with full lineage** (parent IDs, generation depth, content diffs/snapshots) and carry **runtime fitness metrics** (selection / applied / completion / fallback rates).
- **Crucial caveat for our use case: "verifiably better" is NOT enforced.** The only check after an evolution is *structural* (`SKILL.md` exists, has frontmatter + `name`). There is **no behavioral re-test** that the new skill actually improves outcomes. Selection pressure is *retrospective and indirect* — bad skills are only weeded out later via accumulated metrics (Trigger-3 metric monitor), not via a held-out evaluator at edit time. This is the opposite of the brief's "propose → test in isolation → keep only if verifiably better" loop.
- **Headline numbers (46% fewer tokens, 4.2× more "money," $11K/6h) are self-reported and self-baselined.** They come from a 50-task GDPVal subset, scored by the *same lab's* ClawWork LLM-evaluator, with the baseline being the *same lab's* prior ClawWork agent on the same backbone (Qwen 3.5-Plus). The token win is a Phase-2-vs-Phase-1 re-run on the *identical task set* → strong overfitting/memorization risk. No independent reproduction found.
- **Net for us:** medium signal. The reusable gold is the **evolution-operator taxonomy + lineage DAG + per-skill fitness metrics + the actual analysis/evolution prompts + the apply-with-retry patch harness**. The cautionary lesson is its missing piece — it evolves memory without verifying improvements, exactly the gap our seed-AI must close.

## 3. What it does & how it works

### 3.1 Position in the stack
OpenSpace runs primarily as an **MCP server** (`openspace/mcp_server.py`, ~37KB; supports stdio, SSE, and streamable-HTTP). A host coding agent connects and gains a handful of OpenSpace tools (retrieve relevant skills, record/analyze an execution, etc.). It also ships its own CLI agent (`openspace --query <task>`, entrypoint `openspace/__main__.py`) and a grounding/computer-use agent (`openspace/agents/grounding_agent.py`, ~43KB) so it can run standalone. There is a React frontend + a cloud marketplace (`open-space.cloud`) for sharing skills ("collective intelligence").

### 3.2 The core loop (skill lifecycle)
1. **Retrieve** — at task start, relevant skills are surfaced via a **hybrid BM25 + embedding pre-filter, then an LLM selection step** (`registry.py` "Discovery, BM25+embedding pre-filter, LLM selection"; `skill_engine/skill_ranker.py` "BM25 + embedding hybrid ranking", `retrieve_tool.py`, `fuzzy_match.py`) and injected into the host agent's context.
2. **Execute** — the host agent does the task; OpenSpace records the trajectory (`openspace/recording/`, writing `traj.jsonl` + `conversations.jsonl`).
3. **Analyze** — `skill_engine/analyzer.py` (`ExecutionAnalyzer`) calls an LLM with the full trajectory and produces an `ExecutionAnalysis`: an *independent* judgment of `task_completed`, per-skill `skill_judgments` (was each selected skill actually *applied*?), `tool_issues`, and **`evolution_suggestions`**.
4. **Evolve** — `skill_engine/evolver.py` (`SkillEvolver`) turns each suggestion into a new skill version via an LLM "editor," applies it with retry, validates structure, and persists a new node in the DAG (`skill_engine/store.py`, SQLite-backed, `skill_engine/registry.py` on disk).
5. **Share (optional)** — public skills sync to the cloud registry (`openspace/cloud/`).

### 3.3 Three evolution operators (`EvolutionType`)
From `skill_engine/types.py`:
- **FIX** → `SkillOrigin.FIXED`: repair a broken/outdated skill **in place** (same `name`, same directory, new `skill_id`); exactly 1 parent (previous version). Old content preserved via `content_snapshot`. Only latest version `is_active=True`.
- **DERIVED** → `SkillOrigin.DERIVED`: enhance/specialize/compose into a **new** skill directory with a new name; 1+ parents (multi-parent = merge/fuse). `generation = max(parent.generation)+1`.
- **CAPTURED** → `SkillOrigin.CAPTURED`: mint a **brand-new** root skill (no parent) when the agent solved something reusable *without* skill guidance.
- (`IMPORTED` is the 4th origin: skills brought in from outside / the cloud.)

### 3.4 Three evolution triggers (`EvolutionTrigger`)
- **Trigger 1 — ANALYSIS** (`process_analysis`): the per-task analyzer directly suggested evolutions. Runs immediately (or as a background `asyncio.Task`).
- **Trigger 2 — TOOL_DEGRADATION** (`process_tool_degradation`): a `ToolQualityManager` flags tools whose reliability dropped; skills depending on those tools become FIX candidates. Has a state-driven **anti-loop** (`_addressed_degradations: Dict[tool_key, Set[skill_id]]`, pruned when a tool recovers).
- **Trigger 3 — METRIC_MONITOR** (`process_metric_check`): periodic scan of skill health metrics; rule-based screening (`_diagnose_skill_health`) proposes FIX/DERIVED candidates. Anti-loop is **data-driven**: freshly-evolved skills have `total_selections=0` and need `min_selections` (default 5) new data points before re-evaluation.

Triggers 2 & 3 use **relaxed thresholds + a mandatory LLM-confirmation gate** (`_llm_confirm_evolution` → `EVOLUTION_CONFIRM` prompt) before spending tokens on an actual edit — a cheap-screen-then-LLM-verify pattern to avoid wasteful evolution.

### 3.5 "Verification" — what it actually is
The apply path (`_apply_with_retry`) does: apply LLM edit → if apply fails (patch parse / anchor mismatch), **feed the error + current on-disk content back to the LLM and retry** (up to `_MAX_EVOLUTION_ATTEMPTS=3`) → on success, run `_validate_skill_dir`. **`_validate_skill_dir` checks only**: directory exists, `SKILL.md` non-empty, valid YAML frontmatter, has a `name` field. **No functional/behavioral test, no diff-quality gate, no A/B replay.** "Better" is decided only later, statistically, by runtime metrics. The editor LLM can self-abort by emitting `<EVOLUTION_FAILED>`, and self-approve with `<EVOLUTION_COMPLETE>` — but that is a self-judgment, not an external verifier.

## 4. Evidence from the code

**Inspected `HKUDS/OpenSpace@228f8f78073dc4ed0e63fff01c19596c50115d40`** (main, 2026-06-02). Key files:
- `openspace/skill_engine/types.py` — data model (below).
- `openspace/skill_engine/evolver.py` (1598 LOC) — the evolution control loop, triggers, apply-retry.
- `openspace/skill_engine/patch.py` (1001 LOC) — `fix_skill`/`derive_skill`/`create_skill`, the custom patch parser/applier (`*** Begin Patch` / `@@` anchors), diffing.
- `openspace/skill_engine/analyzer.py` (937 LOC) — post-execution `ExecutionAnalyzer`.
- `openspace/skill_engine/store.py` (1495 LOC) — SQLite store, atomic metric counters, lineage DAG.
- `openspace/prompts/skill_engine_prompts.py` (~29KB) — **all the prompts** (quoted below).
- `openspace/mcp_server.py`, `tool_layer.py` — host integration.
- `gdpval_bench/` — the benchmark harness + 203 evolved skills as artifacts.

### 4.1 Core data structures (`skill_engine/types.py`)
The **lineage / version-DAG node** (`SkillLineage`): `origin`, `generation` (distance from root), `parent_skill_ids` (list; multi-parent for merges), `source_task_id`, LLM-written `change_summary`, `content_diff` (combined unified git-style diff; empty for multi-parent merges), and a full `content_snapshot: Dict[relpath → text]`.

The **fitness record** (`SkillRecord`) tracks evolution + quality with explicit *selection-pressure* counters and derived rates:
```python
total_selections: int   # Times this skill was selected by the LLM
total_applied: int      # Times the skill was actually applied by the agent
total_completions: int  # Times task completed when skill was applied
total_fallbacks: int    # Times skill was not applied and task failed
# ...
@property
def applied_rate(self):     return self.total_applied / self.total_selections ...
@property
def completion_rate(self):  return self.total_completions / self.total_applied ...
@property
def effective_rate(self):   return self.total_completions / self.total_selections ...  # end-to-end
@property
def fallback_rate(self):    return self.total_fallbacks / self.total_selections ...
```
(`repo@228f8f7:openspace/skill_engine/types.py`)

### 4.2 The post-execution analysis prompt (the "judge")
`_EXECUTION_ANALYSIS_TEMPLATE` — note the **explicit instruction to distrust the agent's self-reported `<COMPLETE>` and judge ground-truth completion from evidence**:
> "This is the agent's **self-reported** status, not ground truth. `success` = agent output `<COMPLETE>` (may be wrong/premature) … You must independently judge actual task completion below."
> "### 2. Task completion assessment — Did the agent **actually** accomplish the user's request? Judge from conversation evidence … **not** the self-reported status. … Watch for mismatches: agent may claim `<COMPLETE>` after giving up or getting wrong results."

It also defines the evolution decision table verbatim (FIX / DERIVED / CAPTURED) and the JSON output schema (`task_completed`, `execution_note`, `tool_issues`, `skill_judgments`, `evolution_suggestions`). Guiding principle: *"Err on the side of capturing — a slightly redundant skill is better than lost knowledge."* (`repo@228f8f7:openspace/prompts/skill_engine_prompts.py:157-348`)

### 4.3 The evolution-editor prompts
- `_EVOLUTION_FIX_TEMPLATE`: "You are a skill editor … **fix** an existing skill … Be **surgical** — fix what's broken without unnecessary rewrites." Output = `CHANGE_SUMMARY: …` line + either **Format A: Patch** (`*** Begin Patch` / `@@ <anchor>` / `+`/`-`/space lines) — explicitly *"PREFERRED for fixes"* — or **Format B: Full rewrite** (`*** Begin Files`). Ends with self-assessment tokens `{evolution_complete}` / `{evolution_failed}`. (`…prompts…:351-484`)
- `_EVOLUTION_DERIVED_TEMPLATE`: create an enhanced version with a **new descriptive name** — and an explicit anti-pattern rule: *"Do NOT just append '-enhanced' or '-merged' to the parent name … pick a descriptive name that captures the NEW capability (e.g. `panel-circuit-breaker` instead of `panel-component-enhanced-enhanced`)."* (`…:487-631`)
- `_EVOLUTION_CAPTURED_TEMPLATE`: distill an observed pattern into a generalizable new skill. (`…:634-740`)
- `_EVOLUTION_CONFIRM_TEMPLATE`: the gate for Triggers 2/3 — asks the LLM "Is the signal real? Is the skill actually problematic? Is evolution worth the cost? Is the proposed direction correct?" → JSON `{proceed, reasoning, adjusted_direction}`. (`…:743-803`)

### 4.4 The patch format (`patch.py`)
A bespoke **OpenAI/Codex-style "apply_patch"** dialect with three accepted formats auto-detected by `detect_patch_type`: `PATCH` (`*** Begin Patch` + `@@` *content-anchor* hunks, not line-numbered), `FULL` (`*** Begin Files`/`*** File:` envelope or single-file), and `DIFF` (`<<<<<<< SEARCH` / `>>>>>>> REPLACE`). `@@` anchors match a verbatim existing line and apply changes after it (fuzzy via `seek_sequence` + `_normalize_unicode`). `fix_skill` snapshots before/after and computes a combined diff; `derive_skill` copies the parent dir then patches; `create_skill` writes fresh. (`repo@228f8f7:openspace/skill_engine/patch.py`)

### 4.5 Apply-with-retry (the closest thing to a "verifier")
`_apply_with_retry` (`evolver.py:1271`): up to 3 attempts; on failure it appends the error **and the current on-disk file contents** ("use this as the ground truth … so the LLM doesn't hallucinate what's on disk") to the message history and re-asks. On apply success it runs only `_validate_skill_dir` (structural). (`repo@228f8f7:openspace/skill_engine/evolver.py:1271-1424`)

### 4.6 Rule-based health diagnosis (`_diagnose_skill_health`, evolver.py:1564)
Pure-metric screening (relaxed thresholds, LLM confirms):
- `fallback_rate > 0.4` → **FIX** ("selected but not applied → instructions unclear/outdated").
- `applied_rate > 0.4 AND completion_rate < 0.35` → **FIX** ("applied but doesn't complete → instructions wrong/incomplete").
- `effective_rate < 0.55 AND applied_rate > 0.25` → **DERIVED** ("works sometimes, could be enhanced").

### 4.7 Safety moderation (`skill_utils.py`)
`check_skill_safety` runs regexes over skill text; only `blocked.malware` (matches a hardcoded `ClawdAuthenticatorTool`) blocks; `suspicious.*` (secrets, `curl … | sh`, url-shorteners, webhooks, crypto) are informational. Relevant because evolved/shared skills are executable instructions — a prompt-injection / supply-chain surface for the cloud marketplace.

### 4.8 Example evolved skills (artifacts in `gdpval_bench/skills/`, 203 dirs)
- `multi-deliverable-tracking/SKILL.md` is effectively a **learned goal-tracking / anti-premature-completion SOP**: "enumerate EVERY required output upfront," emit a `DELIVERABLES_IDENTIFIED:` block, maintain a visible `DELIVERABLE TRACKER`, and a "Filesystem Verification [MANDATORY BEFORE COMPLETION]" step. This is the system *discovering* a /goal-style discipline by itself.
- The directory zoo (`document-gen-fallback-enhanced-enhanced/`, `…-enhanced-enhanced-9f3b1f/`, `…-merged/`, `code-exec-fallback-266cba/`) is direct evidence of **skill proliferation / near-duplicate sprawl** — many DERIVED chains, hash-suffixed name collisions, and merges. (Inside, the LLM did rename `name:` to `document-gen-resilient-workflow`, but the *directory* kept the ugly lineage name — the dedup is imperfect.)

## 5. What's genuinely smart

1. **The evolution-operator taxonomy (FIX / DERIVED / CAPTURED) maps cleanly to the three ways an experiential memory should change**: repair-in-place, branch-an-improvement, and acquire-something-new. Encoding *which* operation as an explicit type (with different lineage/parenting rules) is cleaner than a single "edit memory" blob.
2. **Versioned lineage DAG with full content snapshots + diffs + generation depth.** Every change is an immutable new node; nothing is lost; you can trace why a skill looks the way it does (`change_summary`), and `generation` gives a natural notion of evolutionary distance. This is a solid, auditable substrate for an evolving knowledge base.
3. **Per-skill runtime fitness metrics as selection signal.** `selection → applied → completion → fallback` is a well-thought-out funnel: it separates "was it chosen," "was it actually used," and "did it work." `fallback_rate` and "applied-but-low-completion" are smart, *behavioral* failure signals — not just thumbs-up/down.
4. **Independent post-hoc judge that explicitly distrusts the agent's self-reported success.** The analyzer is told `<COMPLETE>` "may be wrong/premature" and must verify from trajectory evidence. This is exactly the discipline a verification layer needs (and a known failure mode of naive agent loops).
5. **Cheap-screen → LLM-confirm → edit, with anti-loop guards.** Rule-based thresholds are deliberately *relaxed* (catch more) and an LLM-confirmation gate kills false positives before spending edit tokens; both background triggers have explicit anti-thrash mechanisms (state-pruned for tool degradation, data-count-gated for metrics). Good cost-control engineering for an always-on self-improvement loop.
6. **Surgical diff-first editing with self-correcting apply-retry.** Preferring `@@`-anchored patches over full rewrites (token-efficient, low-blast-radius) and feeding *the real on-disk content + the parse error* back to the LLM on failure is a robust pattern for letting an LLM safely mutate its own artifacts.
7. **Self-abort token (`<EVOLUTION_FAILED>`).** The editor can decline ("the skill is actually correct; the issue is external") rather than being forced to produce a change — a useful guard against gratuitous mutation.

## 6. Claims vs. reality / limitations / critiques

### 6.1 The benchmark claims are self-baselined and overfitting-prone
- **Claim:** "4.2× more money, 46% fewer tokens, $11K in 6h." (`README.md:9,84,105`)
- **Reality (A vs B):** Measured on a **50-task subset of GDPVal** (`gdpval_bench/tasks_50.json`), scored by **ClawWork's own LLM-evaluator with a 0.6 "payment cliff"** (`gdpval_bench/README.md:8`). ClawWork is **HKUDS's own prior project**, used as the *baseline*. So this is OpenSpace-on-Qwen-3.5-Plus vs HKUDS-ClawWork-on-Qwen-3.5-Plus, judged by an HKUDS LLM rubric. It is internally consistent and the harness is genuinely reproducible (`run_benchmark.py`, `calc_subset_performance.py`, `token_tracker.py` are all present), but it is **not an independent benchmark**.
- **The 46% token reduction is Phase-2-vs-Phase-1 on the *same 50 tasks*** ("Cold Start → Warm Rerun," `gdpval_bench/README.md:5-6`). Re-running the *identical* tasks after the skill library has already seen them is close to measuring **memorization of the eval set**, not generalization. The honest read: "skills accumulated from these exact tasks make re-doing these exact tasks cheaper." How much survives distribution shift is untested here.
- The economic "$ earned" framing is a property of GDPVal's payment-rubric scoring, not literal revenue.

### 6.2 No verifier that an evolution is actually *better* (most important critique for us)
The edit-time gate is **purely structural** (`_validate_skill_dir`: file exists, frontmatter, `name`). The system **never re-runs the task with the evolved skill to confirm improvement before keeping it.** It commits the new version optimistically and relies on *future* runtime metrics to maybe-flag it later. Consequences:
- A FIX/DERIVE can silently make a skill worse; it stays active until enough new executions accumulate to trip `_diagnose_skill_health` — and a rarely-selected skill may never accumulate them.
- The "keep only if verifiably better" discipline the seed-AI needs is **absent**; OpenSpace is "mutate now, measure later, maybe."

### 6.3 Skill proliferation / near-duplicate sprawl (empirically visible)
In the shipped `gdpval_bench/skills/` (203 skills): **37** contain `enhanced`, **46** carry a 6-hex dedup suffix, **10** are `merged`. There are ~10 near-duplicate "fallback code execution" skills (`code-exec-fallback`, `code-execution-fallback`, `execute-code-fallback`, `fallback-code-execution`, `fallback-python-execution`, `fallback-python-shell`, `fallback-script-execution`, …) and chains like `document-gen-fallback → -enhanced → -enhanced-enhanced → -enhanced-enhanced-{6hex}` (six variants). The CAPTURED prompt explicitly says *"Err on the side of capturing — a slightly redundant skill is better than lost knowledge,"* and the DERIVED prompt's anti-`-enhanced` naming rule is **not enforced on directory names** — so the library bloats with redundant, lineage-named variants. This is a concrete instance of evolutionary search without effective deduplication/pruning pressure.

### 6.4 Environment-coupling of evolved skills (user-reported, Issue #71)
Evolved skills implicitly bake in the *host machine's* environment (OS, GUI/headless, installed tools, paths, MCP availability). A skill evolved on one worker can be "invalid or misleading on another," and the centralized cloud sharing makes this worse. The user proposes environment-metadata + compatibility filtering, which is **not yet in the system**. (Issue #71, 2026-04-09.) Directly relevant: an evolving software agent's "skills" must encode/verify their preconditions or they don't transfer.

### 6.5 Security / supply-chain surface
Skills are **executable instructions (and aux scripts) that run in your environment**; the cloud marketplace distributes them across users. Moderation is regex-only and mostly non-blocking (`check_skill_safety`; only a hardcoded `ClawdAuthenticatorTool` string blocks). Independent reviewers flag community skills as "random npm packages — audit before trusting." (starlog.is; opensourcedrop.com.)

### 6.6 Maturity / dependence on a strong reflection model
- Very young (created 2026-03-24), daily commits fixing crashes/platform bugs; reviewers warn of "nonexistent API stability." (starlog.is.)
- Quality of evolution is bounded by the *analyst/editor LLM's* reflection ability; weak models "hallucinate fixes or introduce subtle logic errors." Evolution itself costs tokens, so the savings only pay off for **high-frequency, repetitive** tasks — not novel/diverse ones. (starlog.is.)

### 6.6b "Zero human code" showcase claim
The repo ships `showcase/my-daily-monitor/` — described as a full app built with "zero human code" via 60+ evolved skills (`README.md` architecture block). I did not audit this end-to-end; it is a demo artifact, not a controlled study, but it is the closest the project comes to "an agent built software." Even so, the *building* is done by the host coding agent; OpenSpace's role is the skill memory, not an autonomous build loop.

### 6.7 A secondary-source claim I could NOT verify in code
Confirmed via code search: the only `replay`/`rerun` references in the skill engine concern **preserving the evolution dialogue for debugging** (`evolver.py:1108`), not re-executing the task to test an evolved skill. There is no behavioral re-test operator anywhere in `analyzer.py`/`evolver.py`.
starlog.is says OpenSpace uses **"differential evolution — comparing failed executions against successful ones from the same skill family to extract the delta."** I did **not** find a structured contrastive/diff-of-trajectories mechanism in `analyzer.py`/`evolver.py`. What exists is: recent `ExecutionAnalysis` records (which *include* `task_completed` flags) are formatted as free-text "failure context" / "execution insights" in the evolution prompt (`_format_analysis_context`). The LLM may informally contrast them, but there is no algorithmic differential-evolution operator. Treat that blog phrasing as **overstated/inaccurate**.

## 7. Relevance to a self-improving, evolutionary agent

Judged by "would this help build a self-improving, evolutionary, software-building agent?" — **medium, with specific transferable pieces and one big cautionary lesson.**

- **Procedural/experiential memory with selection pressure (HIGH relevance).** The `SkillRecord` funnel (`selection → applied → completion → fallback`, with `effective_rate`/`fallback_rate`) is a strong template for a *fitness-scored memory of "how to do X"*. Our seed-AI needs exactly this: a growing store of verified tactics, each carrying empirical evidence of whether it helps.
- **Explicit mutation operators on memory (HIGH).** FIX / DERIVED / CAPTURED is a clean ontology for *how an experience store should change* — repair, branch-improve, acquire-new — with different lineage semantics. Useful even if our "candidates" are code/prompts rather than SOPs.
- **Versioned lineage DAG (HIGH).** Immutable nodes, parent IDs, generation depth, `change_summary`, content snapshots+diffs → auditable evolutionary history and the ability to roll back or compare ancestors. This is the data-model backbone an evolutionary builder wants for both code candidates and learned heuristics.
- **Independent post-hoc judge that distrusts self-reported success (HIGH).** The analyzer's "do not trust `<COMPLETE>`; verify from evidence" stance is precisely the anti-self-deception guard a long-horizon builder needs (premature "done" is a top failure mode).
- **Cheap-screen → LLM-confirm → act, with anti-thrash guards (MEDIUM-HIGH).** Relaxed rule thresholds + an LLM-confirmation gate + state/data-driven anti-loop = a cost-aware controller for *when* to spend compute on self-modification. Transferable to deciding when to attempt a code change or prompt edit.
- **Self-correcting apply-with-retry over diffs (MEDIUM).** Feeding the apply error + real on-disk content back to the LLM, preferring anchored patches over rewrites, capping attempts — a robust harness for letting an agent edit its own artifacts/code.
- **Learned goal-tracking SOP (MEDIUM, and a nice validation of the brief).** The emergent `multi-deliverable-tracking` skill (enumerate deliverables → visible tracker → mandatory filesystem verification before completing) is essentially a self-discovered "/goal"-style control feature — evidence that such disciplines *can* be captured as reusable memory.
- **The cautionary lesson (HIGH value, negative).** OpenSpace shows what happens when you **evolve memory without an at-edit verifier and without dedup/pruning**: redundant sprawl, possible silent regressions, and benchmark wins that look like eval-set memorization. Our seed-AI's "test in isolation → keep only if verifiably better" loop is exactly the missing organ; OpenSpace is a useful blueprint of the *body* around that organ (memory, lineage, triggers, metrics, prompts) but not the organ itself.
- **Low/irrelevant parts:** the computer-use/grounding agent, GUI recording, WhatsApp/Feishu gateways, the React dashboard, and the cloud marketplace are product surface, not core to our question.

## 8. Reusable assets

All `repo@228f8f78073dc4ed0e63fff01c19596c50115d40`.

**A. Fitness-scored memory schema** — `openspace/skill_engine/types.py` (`SkillRecord`, `SkillLineage`). Borrow the `selections/applied/completions/fallbacks` counters + `applied_rate`/`completion_rate`/`effective_rate`/`fallback_rate`, and the DAG lineage fields (`origin`, `generation`, `parent_skill_ids`, `change_summary`, `content_diff`, `content_snapshot`).

**B. Evolution-operator enum + lineage rules** — `types.py:26-127`: FIX(1 parent, in-place)/DERIVED(N parents, new node)/CAPTURED(0 parents) with `generation` computed from parents. Clean to adapt.

**C. The post-execution analysis prompt** (the judge) — `openspace/prompts/skill_engine_prompts.py:157-348` (`_EXECUTION_ANALYSIS_TEMPLATE`). Especially the "self-reported status is NOT ground truth" framing and the structured JSON schema (`task_completed`, `skill_judgments[*].skill_applied`, `tool_issues`, `evolution_suggestions`). Quotable verbatim.

**D. The three evolution-editor prompts** — same file: `_EVOLUTION_FIX_TEMPLATE` (351-484), `_EVOLUTION_DERIVED_TEMPLATE` (487-631), `_EVOLUTION_CAPTURED_TEMPLATE` (634-740). Note the dual self-assessment tokens `<EVOLUTION_COMPLETE>` / `<EVOLUTION_FAILED>` for self-abort, and the patch-vs-rewrite format guidance.

**E. The evolution-confirmation gate prompt** — `_EVOLUTION_CONFIRM_TEMPLATE` (743-803): "Is the signal real? / actually problematic? / worth the cost? / direction correct?" → `{proceed, reasoning, adjusted_direction}`. A reusable "should I even attempt this self-modification?" check.

**F. Apply-with-retry harness** — `openspace/skill_engine/evolver.py:_apply_with_retry` (1271-1424). Pattern: apply → on failure feed back *error + current on-disk content* → retry ≤3 → structural-validate. Plus `_run_evolution_loop` (an info-gathering tool-use loop before editing).

**G. Custom apply-patch parser** — `openspace/skill_engine/patch.py`: `detect_patch_type`, `parse_patch`, `seek_sequence` (fuzzy anchor match w/ unicode normalization), `compute_skill_diff`, `collect_skill_snapshot`. A self-contained, dependency-light implementation of Codex-style `*** Begin Patch` editing for LLM-generated edits.

**H. Rule-based health diagnosis** — `evolver.py:_diagnose_skill_health` (1564) + threshold constants (1105-1110). Concrete metric→action mapping for retroactive pruning/repair triggers.

**I. The benchmark harness** — `gdpval_bench/` (`run_benchmark.py`, `calc_subset_performance.py`, `token_tracker.py`, `task_loader.py`, `tasks_50.json`): a working two-phase (cold/warm) evaluation + per-source token tracker (`set_call_source("evolver")` to attribute tokens to the self-improvement overhead). Useful as a model for measuring whether self-improvement actually pays for itself.

**J. The 203 evolved skills themselves** — `gdpval_bench/skills/**/SKILL.md`: a corpus of LLM-authored agent SOPs (goal-tracking, deliverable verification, ffmpeg/codec fallbacks, Excel/PDF generation workarounds) — useful as examples of what "captured experience" looks like, and as a dataset of failure-mode skill sprawl.

## 9. Signal assessment

- **Overall signal: MEDIUM.** Not what the name suggested and not a software-building seed-AI, but the **skill-evolution subsystem is a genuinely well-engineered, directly-relevant example of fitness-scored, lineage-tracked, LLM-mutated experiential memory** — with high-quality, reusable prompts and a clean data model. Its biggest contribution to us may be *negative*: a clear demonstration of the failure modes (no at-edit verification, redundant sprawl, eval-set memorization) that our "verify-before-keep" loop is specifically designed to avoid.
- **Confidence: HIGH on mechanism** (read the core code: `types.py`, `evolver.py`, `patch.py`, all skill-engine prompts, and inspected the 203 shipped skills). **MEDIUM on the benchmark's external validity** (harness is real and reproducible, but baseline and judge are the same lab's, and the headline token metric is a same-task re-run).
- **Could NOT verify:** (a) the headline numbers independently — no third-party reproduction exists; (b) the "differential evolution / contrastive delta" mechanism claimed by a secondary blog (not present in code as a structured operator — see §6.7); (c) end-to-end runtime behavior (did not execute the system — no API keys / sandboxed); (d) how well the dedup/retrieval layer (`skill_ranker.py`, `fuzzy_match.py`, cloud quality monitor) actually controls sprawl in practice (read partially; the shipped skill zoo suggests "not very well").

## 10. References

**Primary — code (`repo@228f8f78073dc4ed0e63fff01c19596c50115d40`):**
- `openspace/skill_engine/types.py` — data model (SkillRecord, SkillLineage, EvolutionType/Origin, ExecutionAnalysis).
- `openspace/skill_engine/evolver.py` — evolution loop, 3 triggers, apply-with-retry, health diagnosis.
- `openspace/skill_engine/patch.py` — fix/derive/create + custom apply-patch parser/applier.
- `openspace/skill_engine/analyzer.py` — ExecutionAnalyzer (post-execution judge).
- `openspace/skill_engine/store.py`, `registry.py`, `skill_ranker.py`, `fuzzy_match.py`, `skill_utils.py` — storage/DAG, ranking/retrieval, safety.
- `openspace/prompts/skill_engine_prompts.py` — all analysis/evolution/confirmation prompts (quoted in §4).
- `openspace/mcp_server.py`, `openspace/tool_layer.py`, `openspace/__main__.py` — host integration / CLI.
- `gdpval_bench/{README.md,run_benchmark.py,calc_subset_performance.py,token_tracker.py,tasks_50.json,skills/}` — benchmark harness + 203 evolved skills.

**Primary — repo docs & metadata:**
- README: https://github.com/HKUDS/OpenSpace/blob/main/README.md (headline claims §6.1; architecture overview).
- GDPVal benchmark dataset (third-party origin): https://huggingface.co/datasets/openai/gdpval
- ClawWork (the baseline + evaluator, same lab): https://github.com/HKUDS/ClawWork
- Repo metadata (GitHub API): created 2026-03-24, ~6.4K stars, default branch `main`, HEAD commit "fix: Windows PID check" 2026-06-02.
- Community site: https://open-space.cloud/

**Primary — user issue (limitation):**
- Issue #71 "Environment-aware skill sharing across heterogeneous workers" (2026-04-09): https://github.com/HKUDS/OpenSpace/issues/71

**Secondary — independent coverage / critiques (treat claims cautiously; some misdate to 2025):**
- opensourcedrop.com — concise critique (Darwinian framing; "self-evolving means behavior can change in ways you didn't approve"; sparse cloud library): https://www.opensourcedrop.com/tools/HKUDS/OpenSpace
- starlog.is — review with limitations (API instability, LLM-quality dependence, security/arbitrary-code, token-cost amortization; *also makes the unverified "differential evolution" claim, see §6.7*): https://starlog.is/articles/ai-agents/hkuds-openspace
- marktechpost.com — tutorial framing of the skill engine: https://www.marktechpost.com/2026/03/24/a-coding-implementation-to-design-self-evolving-skill-engine-with-openspace-for-skill-learning-token-efficiency-and-collective-intelligence/
- ngjoo.com, context7.com — overview entries (low-information, inferred architecture).

**Access note:** direct `git clone` was blocked by the sandbox proxy (HTTP 407); code obtained via the codeload tarball `https://codeload.github.com/HKUDS/OpenSpace/tar.gz/refs/heads/main` and extracted to `/agent/workspace/scratch/os_extract/OpenSpace-main`. Commit SHA confirmed via the GitHub commits API.
