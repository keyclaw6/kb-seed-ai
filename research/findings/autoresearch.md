# Findings: `autoresearch` (Andrej Karpathy)

> Per-source research findings doc. Reporter, not architect. Cited throughout.
> Code inspected from the GitHub tarball of branch `master`.

---

## 1. Identity

- **Name:** `autoresearch` (repo `karpathy/autoresearch`; pyproject `description = "Autonomous pretraining research swarm"`).
- **What it is:** A deliberately tiny scaffold that lets a coding agent (Claude Code / Codex / etc.) do **autonomous LLM-pretraining research overnight**. You give it a single-GPU, single-file LLM training script; the agent edits the code, trains for a fixed 5-minute budget, checks whether a single scalar metric improved, **keeps or discards** the change via git, and loops indefinitely. It is the propose→test-in-isolation→keep-if-verifiably-better loop applied to ML research, with the loop itself written in plain-English Markdown (`program.md`) rather than code.
- **It is NOT** (correcting the brief's prior speculation): it is not a generic meta-agent framework, not a hypothesis-DB system, and not (in the committed code) a git-**worktree** sandbox. The committed `program.md` uses git **branches + commits + `git reset`** for isolation. (Worktrees appear only as a `.gitignore` entry — see §3/§6 — i.e. an anticipated launcher pattern, not implemented in this repo.)
- **Author / org:** Andrej Karpathy (solo; repo shows ~7–8 contributors total per OSS Insight, but the core is his). Derived/"cherry-picked and simplified from nanochat" (his own repo).
- **Dates:** Pushed to GitHub on the **night of March 6–7, 2026** (per The New Stack); repo files dated `2026-03-26` in the tarball (last touch). Karpathy framed it with a tongue-in-cheek "March 2026" note and two X posts (links below).
- **Primary links:**
  - Repo: https://github.com/karpathy/autoresearch
  - Karpathy tweets (cited in README, currently 403 to crawlers): https://x.com/karpathy/status/2029701092347630069 and https://x.com/karpathy/status/2031135152349524125
- **Code repo + commit SHA inspected:** `github.com/karpathy/autoresearch` @ **`228791fb499afffb54b46200aca536f79142f117`** (= `refs/heads/master` HEAD, confirmed via `git ls-remote`). Note `master`, not `main`; codeload `main` tarball 404s. Other live branches seen via ls-remote: `agenthub` (`7004de0…`), `exp/H100/mar8` (`6c087cb…`) — not inspected in depth here.

---

## 2. TL;DR

- **The whole "agent" is ~114 lines of Markdown.** `program.md` is the control loop, the verification rule, and the stopping criteria, all in prose. There is **no orchestration code** — the loop is executed by a general coding agent reading the Markdown. Karpathy calls `program.md` "essentially a super lightweight 'skill'."
- **The load-bearing idea is the experimental harness, not the model.** Three primitives make it work: (1) a **single editable file** (`train.py`) so every hypothesis is a reviewable git diff; (2) a **single scalar metric** computed without human judgment (`val_bpb`, vocab-size-independent so architecture changes compare fairly); (3) a **fixed wall-clock budget** (5 min training, excluding compile) so all experiments are directly comparable regardless of what changed or what hardware runs it.
- **Isolation + keep/discard = hill-climbing on git.** Each experiment is a commit on a dedicated branch; if `val_bpb` improves you "advance" the branch, else `git reset` back. The git log becomes a legible experiment journal; `results.tsv` is an explicit lab notebook (kept **untracked**).
- **A hard "NEVER STOP" long-horizon directive.** The prompt explicitly forbids the agent from pausing to ask "should I keep going?" — "The human might be asleep… You are autonomous… until you are manually stopped, period." This is a concrete, copyable long-horizon-autonomy mechanism.
- **Honest scope (per Karpathy + critics):** it automates the *technician*, not the *scientist*. The human still owns the metric, the eval, the data, the time budget, and the simplicity criterion. No cross-experiment memory, no automated hypothesis generation, no meta-strategy in the baseline. "Research becomes search" inside a human-defined sandbox.
- **It demonstrably works (modestly) — and the "less is more" origin is itself a key lesson.** Karpathy's own 2-day run reportedly did **~700 experiments → ~20 genuine improvements → 11% faster "time-to-GPT-2" (2.02h→1.80h)**, and the agent **caught a real attention-scaling bug he'd missed for months** (secondary-reported, §6). Crucially, this minimal design came *after* a richer multi-agent attempt (8 agents, chief-scientist/junior role-play, tmux grids) that **"didn't work"** — he "constrained his way to a better result" (§6). For a project designing such a loop, that failure→simplification arc is as instructive as the loop itself.
- **Relevance to us: HIGH as a reference pattern.** This is almost exactly a "seed AI for software" loop, minus the self-improvement of the loop itself — and it shows the minimal scaffold that makes such a loop actually run unattended.

---

## 3. What it does & how it works (mechanism-level)

**The repo is intentionally three files that matter** (README §"How it works"):

- `prepare.py` — *fixed*, do-not-modify. One-time data prep (downloads `karpathy/climbmix-400b-shuffle` shards, trains an 8192-vocab rustbpe tokenizer) **and the runtime utilities the agent must not touch**: the dataloader (`make_dataloader`) and, critically, **the evaluator `evaluate_bpb` — the ground-truth metric**. Also pins the constants `MAX_SEQ_LEN=2048`, `TIME_BUDGET=300` (seconds), `EVAL_TOKENS=40*524288`, and a **pinned validation shard** (`shard_06542`, the last shard) held out from training.
- `train.py` — *the only file the agent edits*. Full GPT model, the `MuonAdamW` optimizer, the hyperparameter block, and the training loop. "Everything is fair game: architecture, hyperparameters, optimizer, batch size, etc."
- `program.md` — *the only file the human edits*. The agent's instructions = the entire research-org "code." README: "you are not touching any of the Python files… Instead, you are programming the `program.md` Markdown files that provide context to the AI agents and set up your autonomous research org."

**The loop (verbatim from `program.md` §"The experiment loop"):**

```
LOOP FOREVER:
1. Look at the git state: the current branch/commit we're on
2. Tune `train.py` with an experimental idea by directly hacking the code.
3. git commit
4. Run the experiment: `uv run train.py > run.log 2>&1` (redirect everything — do NOT use tee or let output flood your context)
5. Read out the results: `grep "^val_bpb:\|^peak_vram_mb:" run.log`
6. If the grep output is empty, the run crashed. Run `tail -n 50 run.log` to read the Python stack trace and attempt a fix. If you can't get things to work after more than a few attempts, give up.
7. Record the results in the tsv (NOTE: do not commit the results.tsv file, leave it untracked by git)
8. If val_bpb improved (lower), you "advance" the branch, keeping the git commit
9. If val_bpb is equal or worse, you git reset back to where you started
```

**Setup phase** (`program.md` §Setup): agree on a run tag (e.g. `mar5`), create branch `autoresearch/<tag>` from master (must not already exist — "this is a fresh run"), read the three in-scope files, verify `~/.cache/autoresearch/` has data + tokenizer, initialize `results.tsv` with just a header, confirm, go.

**The verification metric** (`program.md` §Experimentation): "**The goal is simple: get the lowest val_bpb.**" Because the time budget is fixed, the agent need not reason about training time. The only constraint is "the code runs without crashing and finishes within the time budget." VRAM is a **soft** constraint ("some increase is acceptable for meaningful val_bpb gains, but it should not blow up dramatically").

**A "simplicity criterion" (a secondary objective baked into the prompt)** — this is unusually thoughtful and worth quoting in full (`program.md`):

> "**Simplicity criterion**: All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Conversely, removing something and getting equal or better results is a great outcome — that's a simplification win. When evaluating whether to keep a change, weigh the complexity cost against the improvement magnitude. A 0.001 val_bpb improvement that adds 20 lines of hacky code? Probably not worth it. A 0.001 val_bpb improvement from deleting code? Definitely keep. An improvement of ~0 but much simpler code? Keep."

This is a prose **regularizer against reward-hacking/complexity-creep** — it gives the agent a tie-breaker the scalar metric alone can't express.

**Output contract** (`program.md` §"Output format"): the script prints a fixed key:value block (`val_bpb`, `training_seconds`, `total_seconds`, `peak_vram_mb`, `mfu_percent`, `total_tokens_M`, `num_steps`, `num_params_M`, `depth`) so the agent extracts the metric with a single `grep "^val_bpb:" run.log` instead of reading the whole log (context hygiene is explicitly engineered).

**Lab notebook** (`program.md` §"Logging results"): `results.tsv`, **tab-separated** ("commas break in descriptions"), 5 columns: `commit  val_bpb  memory_gb  status  description`, where `status ∈ {keep, discard, crash}`, crashes logged as `0.000000 / 0.0 / crash`. Kept **untracked** by git (also in `.gitignore`).

**Time/crash guardrails** (`program.md`): each experiment ~5 min + overhead; "**If a run exceeds 10 minutes, kill it and treat it as a failure** (discard and revert)." Crashes: fix trivial bugs (typos/imports) and re-run, else skip and log `crash`.

**Multi-agent / parallelism intent (not in baseline code).** README hints "how you'd add more agents to the mix"; `program.md` branch examples include `autoresearch/mar5-gpu0` (per-GPU branches). The `.gitignore` reveals the **anticipated launcher machinery**: it ignores `worktrees/`, `results/`, `queue/`, and per-session `CLAUDE.md` / `AGENTS.md` ("Agent prompt files (generated per-session by launchers)"). So the *intended* scaled design uses git worktrees + a queue + per-session generated agent-prompt files — but **none of that launcher code ships in this repo**; the committed baseline is single-agent, branch-based.

**How you actually run it** (README §"Running the agent"): "Simply spin up your Claude/Codex or whatever you want in this repo (and **disable all permissions**), then you can prompt something like: *'Hi have a look at program.md and let's kick off a new experiment! let's do the setup first.'*"

---

## 4. Evidence from the code

Files inspected (all at `autoresearch@228791fb`):

- **`program.md`** (114 lines) — the agent loop/prompt. Quoted extensively in §3.
- **`README.md`** (92 lines) — project framing, design choices, smaller-compute tuning guide.
- **`prepare.py`** (389 lines) — fixed harness; contains the verifier.
- **`train.py`** (630 lines) — the editable model+optimizer+loop.
- **`pyproject.toml`** (27 lines) — deps pinned: `torch==2.9.1` (cu128), `kernels`, `rustbpe`, `tiktoken`, etc. `program.md` forbids adding deps.
- **`.gitignore`** — reveals launcher/worktree/queue intent (§3).
- **`analysis.ipynb`** — reads `results.tsv` and plots `val_bpb` vs experiment progress (the `progress.png` teaser). "Analysis of autonomous hyperparameter tuning results from `results.tsv`."

### 4a. The verifier / evaluator (the ground truth) — `autoresearch@228791fb:prepare.py`

Marked "**Evaluation (DO NOT CHANGE — this is the fixed metric)**". This is the single most load-bearing piece:

```python
@torch.no_grad()
def evaluate_bpb(model, tokenizer, batch_size):
    """
    Bits per byte (BPB): vocab size-independent evaluation metric.
    Sums per-token cross-entropy (in nats), sums target byte lengths,
    then converts nats/byte to bits/byte. Special tokens (byte length 0)
    are excluded from both sums.
    Uses fixed MAX_SEQ_LEN so results are comparable across configs.
    """
    token_bytes = get_token_bytes(device="cuda")
    val_loader = make_dataloader(tokenizer, batch_size, MAX_SEQ_LEN, "val")
    steps = EVAL_TOKENS // (batch_size * MAX_SEQ_LEN)
    total_nats = 0.0
    total_bytes = 0
    for _ in range(steps):
        x, y, _ = next(val_loader)
        loss_flat = model(x, y, reduction='none').view(-1)
        y_flat = y.view(-1)
        nbytes = token_bytes[y_flat]
        mask = nbytes > 0
        total_nats += (loss_flat * mask).sum().item()
        total_bytes += nbytes.sum().item()
    return total_nats / (math.log(2) * total_bytes)
```

Why BPB and not loss/perplexity: cross-entropy in nats is per-*token*, so it shrinks just by enlarging the vocabulary — an agent could "improve" by gaming tokenization. **BPB normalizes by target UTF-8 byte length**, making the score invariant to vocab/tokenizer choices and **architecture changes fairly comparable**. The validation set is a **pinned held-out shard** (`VAL_SHARD = MAX_SHARD = shard_06542`), explicitly excluded from the training split in `_document_batches`/`text_iterator`. This is a clean defense against the agent "winning" by changing the measuring stick.

### 4b. The fixed budget & time accounting — `autoresearch@228791fb:train.py` (loop)

The budget is enforced in the training loop, and crucially it **excludes compilation/warmup** so the comparison is steady-state:

```python
if step > 10:
    total_training_time += dt
...
# Time's up — but only stop after warmup steps so we don't count compilation
if step > 10 and total_training_time >= TIME_BUDGET:
    break
```

Plus a **fast-fail** on divergence (so a bad idea dies fast instead of burning the budget):

```python
# Fast fail: abort if loss is exploding or NaN
if math.isnan(train_loss_f) or train_loss_f > 100:
    print("FAIL")
    exit(1)
```

Final summary block (the agent's machine-readable result surface):

```python
print("---")
print(f"val_bpb:          {val_bpb:.6f}")
print(f"training_seconds: {total_training_time:.1f}")
print(f"total_seconds:    {t_end - t_start:.1f}")
print(f"peak_vram_mb:     {peak_vram_mb:.1f}")
print(f"mfu_percent:      {steady_state_mfu:.2f}")
print(f"total_tokens_M:   {total_tokens / 1e6:.1f}")
print(f"num_steps:        {step}")
print(f"num_params_M:     {num_params / 1e6:.1f}")
print(f"depth:            {DEPTH}")
```

### 4c. The editable surface — `autoresearch@228791fb:train.py` (hyperparameter block)

The agent is pointed at a clearly delimited, comment-annotated knob block (`# Hyperparameters (edit these directly, no CLI flags needed)`), e.g.:

```python
ASPECT_RATIO = 64       # model_dim = depth * ASPECT_RATIO
WINDOW_PATTERN = "SSSL" # sliding window pattern: L=full, S=half context
TOTAL_BATCH_SIZE = 2**19 # ~524K tokens per optimizer step
EMBEDDING_LR = 0.6; UNEMBEDDING_LR = 0.004; MATRIX_LR = 0.04; SCALAR_LR = 0.5
WEIGHT_DECAY = 0.2; ADAM_BETAS = (0.8, 0.95)
WARMUP_RATIO = 0.0; WARMDOWN_RATIO = 0.5; FINAL_LR_FRAC = 0.0
DEPTH = 8; DEVICE_BATCH_SIZE = 128
```

But the prompt says architecture/optimizer/loop are *all* fair game — so the search space is the whole 630-line file, not just these constants. The model itself is a strong modern baseline (nanochat lineage): RMSNorm, RoPE, ReLU²-MLP, GQA-capable attention, **value embeddings / value residual (ResFormer-style) with input-dependent per-head gates**, sliding-window attention via Flash-Attention-3 kernels (`kernels.get_kernel`), logit softcap (tanh, cap=15), per-layer learnable residual scalars (`resid_lambdas`, `x0_lambdas`), and a **`MuonAdamW`** optimizer combining AdamW (embeddings/head/scalars) with **Muon** for 2D matrices (Nesterov momentum, Polar-Express orthogonalization, NorMuon variance reduction, cautious weight decay). LR is scaled ∝ 1/√d_model. This matters for us only as context: the baseline is already near-SOTA, so the agent is doing *fine-grained* hill-climbing, not finding low-hanging fruit.

### 4d. No orchestration code exists

There is no Python "agent loop," scheduler, or evaluator-of-agents in the repo. The "agent" is whatever coding tool the user runs, *driven entirely by `program.md`*. This is the single most important architectural fact: **the scaffold is prose + a clean metric + git, nothing more.**

---

## 5. What's genuinely smart (the heart)

1. **The metric is the contract, and it's locked.** Putting `evaluate_bpb` in the do-not-touch file and choosing a **vocab/architecture-invariant scalar (BPB)** on a **pinned held-out shard** is the crux. It (a) makes heterogeneous experiments comparable, and (b) closes the most obvious reward-hacking door (you can't win by changing the ruler). For any self-improving loop, "what is the unfakeable, cheap-to-compute, direction-unambiguous fitness signal?" is *the* question, and this repo answers it crisply for one domain.

2. **Fixed wall-clock budget as the great equalizer.** Because every run is exactly 5 minutes of steady-state training, "double the model" and "halve the LR" cost the same and are judged on equal footing — and the loop **auto-discovers the best model *for your specific hardware* in that budget** (README §Design choices). It also makes throughput predictable (~12 experiments/hr ⇒ ~100 overnight) — turning "research" into a known-rate search process. Excluding compile time from the budget (`step > 10`) is a small but crucial fairness detail.

3. **Single editable file ⇒ every hypothesis is a reviewable diff; git *is* the experiment store.** Keep = advance the branch; reject = `git reset`. The git log becomes a legible, auditable journal of validated decisions; `results.tsv` is the parallel human-readable notebook. No bespoke experiment-tracking DB needed. This is "version control as evolutionary memory."

4. **The prompt encodes long-horizon autonomy explicitly.** The "**NEVER STOP**" section (quoted verbatim in §8) is a direct, transferable answer to the #1 failure mode of long-running agents: stopping to ask permission. It also tells the agent *what to do when it runs out of ideas* ("think harder — read papers referenced in the code, re-read the in-scope files, combine previous near-misses, try more radical architectural changes"), and authorizes rewinds "very very sparingly (if ever)."

5. **A prose "simplicity regularizer" as a second objective.** The simplicity criterion (verbatim §3) is a clever way to express a multi-objective trade-off the scalar metric can't, and it directly counters complexity-creep / overfitting-to-the-metric. It even *rewards deletions*, which most metric-driven loops never do.

6. **Context-window hygiene engineered into the protocol.** "`> run.log 2>&1` (do NOT use tee or let output flood your context)" + "extract with `grep`" + "`tail -n 50` only on crash." These are concrete, copyable habits for keeping a long-running agent's context clean — a real operational concern for long-horizon agents.

7. **Crash/timeout handling as first-class loop states.** OOM/NaN/timeout are not exceptions that halt the loop; they're normal `crash`/`discard` outcomes that get logged and stepped over. The fast-fail (`loss > 100 or NaN → exit(1)`) prevents a diverging run from wasting the budget.

8. **"Program the org in Markdown."** Framing `program.md` as a lightweight, human-edited **skill** that defines the *research organization* (and is iterated on over time, with more agents added) is a clean separation: humans evolve the *process* (prose), the agent executes and the code/model evolves. Karpathy's stated long-term vision is to find "the 'research org code' that achieves the fastest research progress."

---

## 6. Claims vs. reality / limitations / critiques

**(A) What Karpathy claims:** an agent can autonomously run a real (if small) LLM-pretraining research loop overnight, modify code, keep/discard via a metric, and you "wake up to a better model." He is explicit that the default `program.md` is "intentionally kept as a bare bones baseline." The framing tweet/README is semi-satirical ("10,205th generation… self-modifying binary beyond human comprehension") — clearly aspirational, not a claim about this repo.

**(B) What the code actually demonstrates:** a *correct, minimal harness* that makes the loop runnable and the comparisons fair. The repo ships **no logged experiment results** (`results.tsv` is gitignored; `progress.png` is the only artifact, showing val_bpb declining over a run). So the repo *itself* is evidence of a working scaffold, not of discoveries — the efficacy evidence is all **secondary-reported** (treat accordingly):
- **Karpathy's own larger run (the strongest signal):** a **2-day** autonomous run did **~700 experiments**, found **~20 genuine improvements** stacking to an **11% reduction in "time-to-GPT-2" on the nanochat leaderboard (≈2.02h → ≈1.80h)**, and the agent **discovered a bug in his attention scaling that he'd missed for months.** Some kept changes were **new PyTorch code, not just hyperparameters** (e.g. a "smear gate," a "backout skip connection" — found after Karpathy edited `program.md` to "look at modded-nanogpt for inspiration"). (Towards AI: https://pub.towardsai.net/karpathy-left-his-gpu-running-overnight-the-agent-found-a-bug-everyone-missed-for-months-eae7e351104d ; Karpathy quoted on HN: https://news.ycombinator.com/item?id=47442435 ; SkyPilot blog: https://blog.skypilot.co/scaling-autoresearch/)
- The New Stack: on Mar 7 the agent "ran 50 experiments, discovered a better learning rate, and committed the proof to git" with no human in between; a community RTX-4090 MobileNetV3 run went 95.6%→improved via an autonomous LR sweep.
- Independent replications of the *pattern*: **Red Hat** ran it on OpenShift AI / H100 for 24h → 198 experiments, 29 kept, **2.3% val-loss improvement**, zero human intervention; **Shopify CEO Tobi Lütke** ran 37 overnight and got a **0.8B model beating his hand-tuned 1.6B** (both via Towards AI). Magnitudes are modest and mostly fine-grained, but the "staircase" improvement pattern reproduces across independent infra.

**(B′) Design history — the breakthrough was *removing* structure.** Per Garry Tan / LinkedIn quoting Karpathy (Feb 27, 2026, pre-release): his first attempt used **8 agents on 1 GPU each, role-playing "chief scientist" + "junior researchers," with git branches and tmux grids — "a whole research org in code. And it didn't work."** He noted "the agents' ideas are just pretty bad out of the box, even at highest intelligence… an agent 'discovered' that increasing the hidden size improves val loss, which is a totally spurious result… it's not clear why I had to come in and point that out." autoresearch is the *constrained* successor: "one file, one metric, one 5-minute budget." Lesson for builders: **scarcity/constraint design beat multi-agent elaboration** here. (https://www.linkedin.com/posts/garrytan_karpathy-just-turned-one-gpu-into-a-research-activity-7436229926485204992-ERWH)

**(B″) Scaling the loop (parallelism changes search behavior — with caveats).** SkyPilot wired autoresearch to **16 GPUs via a `experiment.yaml` + `instructions.md` + a SkyPilot "skill"**, letting one Claude Code agent provision clusters and run **factorial grids (~10–13 experiments/wave)** instead of greedy serial hill-climbing — ~90 experiments/hr, ~910 in 8h, reaching the same best val-loss **~9× faster in wall-clock** than simulated serial. Notably, **the agent emergently discovered H200 > H100 (more steps in the fixed budget) and invented a two-tier "screen cheap on H100s, promote winners to H200" strategy with no prompt to do so.** *Skeptical caveat (HN):* counted in **GPU-hours the parallel run is ~2× *less* efficient** than serial (16 GPUs for 8h ≈ 128 GPU-h vs ~44 GPU-h serial for similar loss) — "the blog oversells itself"; parallel buys wall-clock at an efficiency cost, and a serial adaptive agent can in principle match it on sample-efficiency. (https://blog.skypilot.co/scaling-autoresearch/ , https://news.ycombinator.com/item?id=47442435)

**(C) Independent critiques (the important ones):**
- **It's search, not science — the bottleneck didn't move.** OSS Insight: "The agent can change everything about the model. But it cannot change the game. The human still defines the metric, the evaluation function, the data, the time budget, and the aesthetic preference for simplicity… The scientist hasn't been automated. The technician has." "Most ML research is… bottlenecked by knowing which experiments are worth running. autoresearch removes the former but not the latter." (https://ossinsight.io/blog/autoresearch-overnight-ai-scientist)
- **No memory, no meta-strategy in the baseline.** OSS Insight: "A fixed environment, a mutable state, a scalar reward, and a loop… No planning across experiments, no memory of what worked three hours ago, no meta-strategy." It is explicitly "Stage 1" / "only has the loop" of a larger "memory + loop + abstraction" stack. (Same source.)
- **Narrow applicability.** "It works when you have a clear scalar metric, a fast feedback cycle, and safe reversibility. Most real research problems satisfy none of these. That's a powerful subset — but it's a subset." (Same source.)
- **Forks are islands / no cross-run learning.** ~7,600 forks vs ~8 contributors; each person's `program.md` + findings stay private. "Cross-fork knowledge transfer" is named as the missing primitive. (Same source.)

**Failure modes / reward-hacking surface (my analysis of the code):**
- The metric defense is good but **not airtight**: the agent edits `train.py`, which *imports* `evaluate_bpb` and constructs the val loader. The prompt forbids touching `prepare.py` and "the evaluation harness," but nothing *mechanically* prevents an agent from, e.g., shadowing `MAX_SEQ_LEN`, calling eval on fewer tokens, leaking val data into training, or otherwise gaming the comparison from within `train.py`. Enforcement is **prompt-level, not sandbox-level.** (The intended `worktrees/`+launcher design and "disable all permissions" run mode make this *worse* on the safety axis — full permissions, prose-only guardrails.)
- **Local-optimum / greedy hill-climbing (independently confirmed):** strict keep-iff-improved with only sparing rewinds can get stuck; no population, no diversity mechanism, no simulated-annealing-style acceptance of worse steps (contrast the Darwin Gödel Machine, listed by OSS Insight as the natural next step). Towards AI states it plainly: "A change that sets up a better change two steps later **will get rolled back** if it doesn't immediately improve val_bpb. The ratchet loop is greedy by design. That is also its constraint." (The SkyPilot 16-GPU factorial-grid approach is partly a workaround — parallel breadth makes local optima "much harder to get stuck in.")
- **Noise:** a single 5-min run's `val_bpb` has variance; the protocol keeps/discards on a single sample (seed is fixed at 42, which controls some but not all nondeterminism), so it can keep a lucky run or discard an unlucky good idea.
- **Reproducibility across people is *intentionally* broken:** fixed-time-budget means results are non-comparable across hardware (README §Design choices explicitly notes this as a deliberate trade-off).

**Reproducibility of the repo itself:** requires a single NVIDIA GPU (tested H100); CPU/MPS unsupported in-tree (forks exist for macOS/MLX/Windows-RTX/AMD, linked in README). The harness code I read is self-consistent and complete; I did **not** execute it (no GPU in this sandbox), so I cannot independently verify the reported val_bpb numbers or the "50 experiments overnight" claim — those rest on Karpathy's tweets and The New Stack's reporting.

---

## 7. Relevance to a self-improving, evolutionary, software-building agent

This source is **directly on-target** for our project — it is essentially a working, minimal instance of "give the agent a goal, let it propose→test-in-isolation→keep-if-verifiably-better, overnight." Specific transferable mechanisms, each tied to what it helps:

- **Locked, cheap, unfakeable fitness signal (helps: verification).** The core lesson: define ONE scalar that is (a) computable without human judgment, (b) direction-unambiguous, (c) invariant to the things the agent is allowed to change, and (d) **physically out of the agent's editable scope.** For us (software), the analog is a test/benchmark suite the agent cannot edit, scored on held-out cases. The BPB design (normalize so the agent can't win by changing the representation) is the key abstraction to port.
- **Fixed-budget comparability (helps: decision-making, fair selection).** Bound each candidate's evaluation by a fixed resource (wall-clock, or # of test runs) so heterogeneous changes are judged equally and throughput is predictable. Excluding warmup/compile from the budget = measure steady-state, not setup.
- **Git-as-evolutionary-memory + explicit notebook (helps: long-horizon memory, auditability).** Keep=commit/advance, reject=`git reset`; a tab-separated `results.tsv` of `commit/score/cost/status/description` is a dead-simple persistent memory of every attempt (including crashes). This is a ready-made schema for our experiment ledger. (And it exposes the gap we'd need to fill: the baseline has **no cross-experiment reasoning** over this ledger — that's exactly where a "seed AI" should add a memory/strategy layer.)
- **The "NEVER STOP" long-horizon directive (helps: running reliably over long horizons).** A concrete, battle-tested prompt clause that prevents the most common long-run failure (asking permission / declaring false "good stopping points"), plus an explicit "what to do when out of ideas" playbook. Near-verbatim reusable (§8).
- **Crash/timeout/divergence as normal loop states (helps: robustness/control).** Treat OOM/NaN/timeout as logged `discard`/`crash` outcomes with a fast-fail abort and a hard kill at 2× budget — don't let one bad candidate stall the loop. Directly portable to "build attempt fails to compile / tests hang."
- **Context-window hygiene protocol (helps: long-horizon reliability).** Redirect all subprocess output to a log; extract only the metric via `grep`; read the tail only on failure. Prevents context flooding over hundreds of iterations — a real operational must for our loop.
- **Process-in-prose / "program.md as skill" (helps: self-improvement of the loop, orchestration).** The separation "humans iterate the *process* (Markdown), agent iterates the *artifact* (code)" is the seed of self-improvement: a self-improving agent would edit its *own* `program.md` (its strategy/skill), then verify that the new process produces better outcomes. Karpathy names this explicitly ("iterate on it over time to find the 'research org code' that achieves the fastest research progress, how you'd add more agents").
- **Multi-agent/parallel intent via worktrees+queue (helps: orchestration/scaling).** Even though unshipped, the `.gitignore` (`worktrees/`, `queue/`, per-session `CLAUDE.md`/`AGENTS.md`) sketches how to fan out many isolated agents over branches/worktrees with generated per-session prompts — a useful pattern for parallel candidate evaluation. The SkyPilot scale-out (16 GPUs, `experiment.yaml` + a teachable "skill", `sky launch`/`sky exec` pipelining) shows a concrete realization, and a key payoff for our purposes: **parallel breadth (factorial grids / waves) directly mitigates the greedy-local-optimum failure mode** and surfaces parameter *interaction* effects a serial ratchet misses — at a GPU-hour efficiency cost (helps: decision-making, escaping local optima, but watch the cost trade-off).
- **Emergent resource-aware strategy as evidence of capability (helps: orchestration).** Given heterogeneous hardware and *no* prompt about it, the agent inferred H200>H100 from step-counts and self-organized a "screen-cheap-then-promote-winners" tiering. Suggests that with the right cheap signal + budget framing, an agent will invent sensible meta-strategy on its own — relevant to letting a seed AI manage its own compute/verification budget.
- **"Less is more" as a design prior (helps: how we architect the loop itself).** Karpathy's richer 8-agent role-play org *failed*; the constrained single-file/single-metric/single-budget design *worked*. Strong prior to start minimal and add structure only when a verified gain demands it.
- **Simplicity-as-secondary-objective (helps: anti-reward-hacking, code quality).** A prose tie-breaker that penalizes complexity creep and *rewards deletions* — exactly the kind of guardrail a code-building agent needs so "make tests pass" doesn't degenerate into accreting hacks.

What it does **not** give us (be plain): automated hypothesis generation, any memory/learning across runs, population/diversity-based evolution, escaping local optima, sandbox-level (vs prompt-level) protection of the evaluator, or anything about multi-file/large-codebase software building (it's deliberately single-file). These are precisely the gaps a "seed AI for software" would need to fill on top of this loop.

---

## 8. Reusable assets (verbatim, cited)

All from `autoresearch@228791fb`.

**(i) The keep/discard control loop** — `program.md` §"The experiment loop" (full text quoted in §3 above).

**(ii) The long-horizon autonomy clause** — `program.md`, verbatim:

> "**NEVER STOP**: Once the experiment loop has begun (after the initial setup), do NOT pause to ask the human if you should continue. Do NOT ask 'should I keep going?' or 'is this a good stopping point?'. The human might be asleep, or gone from a computer and expects you to continue working *indefinitely* until you are manually stopped. You are autonomous. If you run out of ideas, think harder — read papers referenced in the code, re-read the in-scope files for new angles, try combining previous near-misses, try more radical architectural changes. The loop runs until the human interrupts you, period."

**(iii) The simplicity criterion (secondary objective / anti-complexity regularizer)** — `program.md`, verbatim (quoted in full in §3).

**(iv) Crash & timeout policy** — `program.md`, verbatim:

> "**Timeout**: Each experiment should take ~5 minutes total (+ a few seconds for startup and eval overhead). If a run exceeds 10 minutes, kill it and treat it as a failure (discard and revert).
> **Crashes**: If a run crashes (OOM, or a bug, or etc.), use your judgment: If it's something dumb and easy to fix (e.g. a typo, a missing import), fix it and re-run. If the idea itself is fundamentally broken, just skip it, log 'crash' as the status in the tsv, and move on."

**(v) The experiment-ledger schema** — `program.md` §"Logging results": TSV with header `commit  val_bpb  memory_gb  status  description`; `status ∈ {keep, discard, crash}`; crashes recorded as `0.000000 / 0.0 / crash`; kept untracked. Example rows:

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
```

**(vi) The verifier pattern** — `prepare.py:evaluate_bpb` (full code quoted in §4a). Reusable idea: representation-invariant scalar metric, on a pinned held-out split, in a do-not-edit module.

**(vii) The machine-readable result surface + extraction** — fixed `key: value` summary block (`train.py`, §4b) + `grep "^val_bpb:" run.log` extraction (`program.md`). Reusable as the "how the agent reads its own result without flooding context" pattern.

**(viii) The launch incantation** — README: run any coding agent in the repo with permissions disabled, prompt: *"Hi have a look at program.md and let's kick off a new experiment! let's do the setup first."* `program.md` is "essentially a super lightweight 'skill'."

(Collected as evidence only — not assembled into a design, per brief.)

---

## 9. Signal assessment

- **Overall value: HIGH** — but specifically as a **reference pattern / minimal-scaffold blueprint**, not as a turnkey self-improving system. It is the closest public artifact to "the simplest thing that makes an autonomous propose→verify→keep loop actually run unattended," and it nails the two hardest design questions (the unfakeable metric, and the fair budget) plus several real operational details (context hygiene, crash states, long-horizon "never stop", git-as-memory, simplicity regularizer). For a project building a seed AI for software, the harness lessons transfer almost directly; the domain (single-file ML pretraining) does not.
- **Confidence:** High on *what the code/prompt is and how the loop works* (I read all the load-bearing files verbatim at a known SHA). Medium on *real-world efficacy*: the repo ships no result logs, but efficacy is now corroborated by *multiple independent* secondary sources reporting concrete numbers (Karpathy's own ~700-run/11%/bug-find; Red Hat 198 runs/2.3%; Shopify 0.8B>1.6B; SkyPilot 16-GPU scaling) — consistent and modest. Still secondary, and I did not run the code.
- **What I could NOT verify:**
  - The actual experimental results / val_bpb trajectories (results.tsv is gitignored; I only have the `progress.png` teaser and reported anecdotes). I did not run the code (no GPU).
  - Karpathy's two X posts (403 to crawlers) — relied on README context + The New Stack's quotes of them.
  - The `agenthub` and `exp/H100/mar8` branches (seen via `git ls-remote` but not inspected); these may contain the multi-agent launcher / worktree machinery hinted at by `.gitignore`. **Flagging for possible follow-up** if the orchestrator wants the scaled/parallel design.
  - The community fork results (macOS/MLX/Windows/AMD) — not inspected.

---

## 10. References

**Primary:**
- `karpathy/autoresearch` @ `228791fb499afffb54b46200aca536f79142f117` (branch `master`). Files cited: `program.md`, `README.md`, `prepare.py`, `train.py`, `pyproject.toml`, `.gitignore`, `analysis.ipynb`. Repo: https://github.com/karpathy/autoresearch — code refs given inline as `autoresearch@228791fb:<path>`.
- Andrej Karpathy, X posts (cited by the README; not directly retrievable — 403): https://x.com/karpathy/status/2029701092347630069 , https://x.com/karpathy/status/2031135152349524125
- `karpathy/nanochat` (the parent project autoresearch is "cherry-picked and simplified from"): https://github.com/karpathy/nanochat

**Secondary:**
- The New Stack, "Andrej Karpathy's 630-line Python script ran 50 experiments overnight without any human input" (2026-03-14): https://thenewstack.io/karpathy-autonomous-experiment-loop/ — source for the "three primitives" framing, the Mar 7 50-experiment anecdote, and the RTX-4090 MobileNetV3 example.
- OSS Insight Docs, "54,000 Stars in 19 Days: What karpathy/autoresearch Tells Us About the Next Frontier" (2026-03-25): https://ossinsight.io/blog/autoresearch-overnight-ai-scientist — source for the "search not science / bottleneck didn't move," "no memory/meta-strategy," "Stage 1 of memory+loop+abstraction," fork-island, and adjacency to DGM / facebookresearch HyperAgents critiques.
- (Noted, not fetched) buttondown.com/verified "The Dawn of Autonomous ML…" (2026-03-26): https://buttondown.com/verified/archive/the-dawn-of-autonomous-ml-how-andrej-karpathys/ — additional secondary commentary.

**Comparanda named by critics (not this source; for context only):** `jennyzzt/dgm` (Darwin Gödel Machine — evolutionary self-improving agents), `facebookresearch/HyperAgents` (self-referential optimization). Both flagged by OSS Insight as the "next primitives" (hypothesis generation, evolution, memory) that autoresearch deliberately omits.
