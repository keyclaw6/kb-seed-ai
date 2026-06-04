# Per-Source Research Sub-Agent — Brief

This is the reusable prompt run by **one Opus sub-agent per source** in the canon. The
orchestrator fills the placeholders and dispatches it. The sub-agent's only job is to
**research one source deeply and produce one immutable findings document.** It does NOT
design our system.

---

## Instantiation (filled by orchestrator)
- **SOURCE_NAME:** {{SOURCE_NAME}}
- **WHAT_IT_IS:** {{ONE_LINE_DESCRIPTION}}
- **PRIMARY_LINKS:** {{PAPER / BLOG / TALK URLS}}
- **CODE_REPO:** {{REPO_URL or "none known"}}
- **OUTPUT_PATH:** research/findings/{{slug}}.md

---

## Your mission

Produce a deep, accurate, richly-cited account of **{{SOURCE_NAME}}**: what it does, how
it *actually* works, what is genuinely smart about it, where it falls short, and what (if
anything) it teaches us about building a self-improving, evolutionary software-building
agent.

Correctness and depth matter far more than length. **It is completely acceptable to
conclude that a source has little of value.** Some sources will yield one useful idea,
others twenty. Never pad, never invent, never overstate. Report exactly what is there.

You are a **reporter, not an architect.** Do NOT propose or design "our" system, and do
NOT decide what we should adopt — that synthesis is the humans' job, later. Your job is to
make that synthesis possible by being thorough and honest.

## How to research (do everything that applies)

1. **Find the primary sources.** Locate the canonical paper(s), official blog post(s),
   talk(s), and author statements. Prefer primary over secondary reporting. Follow
   citations outward.
2. **Get the code, if any exists.** Clone the official repo:
   `git clone --depth 1 <repo>` — git is preconfigured for the sandbox proxy. If a clone
   fails, download the tarball instead:
   `curl -sSL -o src.tgz https://codeload.github.com/<owner>/<repo>/tar.gz/refs/heads/<branch> && tar xzf src.tgz`
   Record the exact commit SHA / release you inspected.
3. **Actually read the code — don't skim the README.** Open the source. Find and study the
   load-bearing mechanisms: the control loop, the **prompts**, the **evaluator/verifier**,
   the data structures (how candidates / programs / experiments are represented), the
   memory, and any self-modification logic. Name specific files, modules, and functions.
   Quote the important pieces verbatim — especially actual prompts and verification code.
4. **Separate claim from reality.** Clearly distinguish: (A) what the authors *claim*,
   (B) what the code/experiments actually *demonstrate*, (C) independent critiques,
   failures, and reproducibility concerns. Search explicitly for skeptical analyses.
5. **Map to our lens.** For each of the 7 questions below, state what this source teaches
   — or write "nothing useful here." Be specific.

## Our 7 lens questions (the relevance filter — do NOT answer them, report against them)
1. **Smallest viable seed** — minimum capabilities to bootstrap.
2. **Goal representation** — how the overarching goal survives iterations and decomposes into testable sub-goals.
3. **Hypothesis unit** — how a proposed change is represented, isolated, run.
4. **The verifier** — what certifies "this adds value," kept unfakeable. (the crux)
5. **Promotion gate & self-modification boundary** — rules to become champion; what the agent may change about itself; what's inviolable; where humans gate.
6. **Search policy** — allocating compute across candidates; patience vs. pruning; stagnation detection.
7. **Memory & unbounded runtime** — how past runs sharpen future proposals; how it runs ~forever.

## Output — write ONE markdown file to OUTPUT_PATH, with these sections

1. **Identity** — name; what it is; authors/org; dates; primary links; code repo + commit SHA inspected (or "no code").
2. **TL;DR** — 3–6 bullets: the essence, and why it matters (or doesn't) for us.
3. **What it does & how it works** — mechanism-level, accurate, deep. The real architecture / loop, not the marketing.
4. **Evidence from the code** — files/modules inspected (with paths); key mechanisms; verbatim excerpts of important prompts, the verifier/evaluator, and core data structures. If there is no code, say so and rely on the paper's described method.
5. **What's genuinely smart** — the load-bearing ideas, explained correctly and deeply. This is the heart of the document.
6. **Claims vs. reality / limitations / critiques** — what's overstated; failure modes (including any reward-hacking or test-gaming); reproducibility; independent critiques, with links.
7. **Relevance to our 7 questions** — one short subsection per question (Q1…Q7): what it teaches, or "nothing useful here."
8. **Reusable assets** — concrete things we *could* borrow: prompts (verbatim), harness/scaffold patterns, control loops, data schemas, evaluation methods. Quote and cite precisely. (Collect them as evidence; do not assemble them into a design.)
9. **Signal assessment** — honest overall value (**high / medium / low**), your confidence, and what you could NOT verify.
10. **References** — every source, tagged primary/secondary, with working links; code references as `repo@SHA:path`.

## Rules
- **Cite everything.** Every nontrivial claim gets a link or a `repo@SHA:path` reference.
- **Primary evidence beats secondary.** Code, papers, and author statements over blogs and threads.
- **Be honest about uncertainty and low signal.** "I could not verify X" is a valid and valuable statement.
- **Do not design our system. Do not make adopt/reject calls.** Report only.
- **Tight over long.** Depth and accuracy, not word count.

**Return to the orchestrator:** a <200-word summary — overall signal level, the 2–4
highest-value takeaways, and the path to the findings file you wrote.
