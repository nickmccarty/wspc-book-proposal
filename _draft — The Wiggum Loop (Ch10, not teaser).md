
# Chapter 10: The Wiggum Loop

---

> *"He's not the sharpest knife in the drawer, Lou."*
> — Chief Clancy Wiggum, *The Simpsons*

---

The first time you run a research agent without an evaluator, you will probably be impressed. The output will be long. It will be fluent. It will have headings and bullet points and citations and a confident, authoritative tone that reads like something a competent analyst spent an afternoon on.

It will also, in a meaningful fraction of cases, be wrong.

Not wrong in the way a hallucination is wrong — where a model fabricates a fact that is obviously false on inspection. Wrong in the way that a first draft is wrong: structurally correct but thin in the places that matter, confident about things it should hedge, and silent about the gaps a domain expert would immediately notice. The model is not lying to you. It simply does not know that it does not know.

The problem compounds when you ask it to evaluate itself. In controlled experiments using the harness pipeline described in this book, outputs rated by the same model that produced them average 0.9 points higher on the six-dimension quality rubric than outputs rated by a separate evaluator model using the same rubric. The gap is largest on the *groundedness* dimension — the one that asks whether claims in the output are traceable to retrieved sources — because a model that invented a claim is also the most likely to believe it is grounded.

This is the problem the Wiggum Loop is designed to solve. It is named, affectionately, after the reliably imperfect Chief Wiggum of Springfield: a figure who catches most problems, misses some, and is constitutionally unable to evaluate his own blind spots. That description is an honest account of what this pattern does. It is not a guarantee. It is an improvement — and a measurable one.

---

## The Pattern

**Intent.** Use a second language model instance, never the same as the producer, to score the agent's output on multiple quality dimensions, then route the dimensional feedback back to the producer for targeted revision. Repeat until the composite score exceeds a threshold or the maximum number of rounds is reached.

**Also known as.** Evaluate–Revise Loop; Cross-Model Verification; Producer–Evaluator Separation.

**Motivation.** A single-pass research pipeline has no mechanism to distinguish a high-quality output from a fluent-but-shallow one. Adding an evaluator after synthesis creates a feedback signal, but self-evaluation is biased. Producer–evaluator separation removes the bias and enables the most important capability the loop provides: not a score, but a diagnosis.

**When to use it.** Use the Wiggum Loop whenever:

- The output will be read and acted on by a human (as opposed to being fed immediately into a downstream automated process)
- The task requires synthesis across multiple retrieved sources, where the risk of selective reading or missed coverage is high
- You are running in a domain where confident-sounding errors are harder to catch than obvious ones (research, medicine, law, finance)
- You have a second model available — even a smaller one — that was not used to produce the output

**When not to use it.** Do not use the loop when latency is the primary constraint and a single-pass output is acceptable. The loop adds at minimum one full inference round — typically 15 to 40 seconds on local hardware depending on the evaluator model — and up to three rounds in the worst case. For real-time applications or lightweight tasks (email drafts, quick lookups, simple summaries), the overhead is not justified. The *leverage* framing introduced in Chapter 20 is useful here: the loop is worth its cost for high-leverage tasks and wasteful for low-leverage ones.

---

## Structure

The Wiggum Loop involves four components:

**Producer.** The model that generated the research output being evaluated. Never used as the evaluator in the same run.

**Evaluator.** A separate model instance that scores the output on six dimensions and generates structured textual feedback. Selected from the evaluator pool using a seeding mechanism that distributes load and mitigates evaluator drift.

**Rubric.** The six-dimension scoring framework applied by the evaluator. Each dimension is scored at 0.5-point resolution on a 0–10 scale. The composite score is the arithmetic mean. A composite of 8.0 or above constitutes a pass. The rubric is described in full in Chapter 11; this chapter treats it as a black box.

**Revision Prompt.** A prompt that presents the producer model with its original output, the evaluator's dimensional scores, and the evaluator's free-text feedback, and instructs it to revise the output with specific attention to the dimensions that fell below threshold.

---

## Interaction Diagram

```
Task
  │
  ▼
producer.synthesize(task, context)
  │
  ▼
[draft output]
  │
  ▼
evaluator.score(draft, rubric)  ◄──────────────────────┐
  │                                                     │
  ▼                                                     │
composite ≥ 8.0?                                        │
  │                                                     │
  ├─ YES → persist(output) → DONE                       │
  │                                                     │
  └─ NO, rounds < 3                                     │
       │                                                │
       ▼                                                │
  producer.revise(output, scores, feedback)             │
       │                                                │
       ▼                                                │
  [revised output] ─────────────────────────────────────┘
```

At most three evaluation rounds are executed. The choice of three is not arbitrary: analysis of 1,500 logged runs in the harness shows that 75% of achievable quality improvement occurs within the first round, another 18% within the second, and the remainder within the third. Beyond three rounds, the marginal gain falls below measurement noise, and the probability of the loop destabilizing — the producer overwriting good content to satisfy a dimension that the evaluator scores inconsistently — rises above the probability of further improvement.

---

## Implementation

The loop is implemented in `harness/wiggum.py`. The entry point is the `loop()` function:

```python
def loop(
    task: str,
    output_path: str,
    models: ModelConfig,
    parent_trace: RunTrace,
) -> WiggumResult:
    """
    Evaluate and conditionally revise the output at output_path.
    Returns the final score, all round scores, and whether the run passed.
    """
    content = Path(output_path).read_text(encoding="utf-8")

    # Normalize HTML to markdown if needed (Playwright-based)
    if content.strip().startswith("<!DOCTYPE") or "<html" in content[:500]:
        content = _normalize(output_path)

    round_scores: list[float] = []
    evaluator = select_evaluator(seed=parent_trace.run_id)

    for round_num in range(1, MAX_ROUNDS + 1):
        parent_trace.enter_stage(f"wiggum_round_{round_num}")

        result = _evaluate(content, task, evaluator, models)
        round_scores.append(result["score"])

        parent_trace.log_wiggum_round(
            round=round_num,
            score=result["score"],
            dimensions=result["dimensions"],
            feedback=result["feedback"],
            evaluator=evaluator,
        )

        if result["score"] >= PASS_THRESHOLD:
            Path(output_path).write_text(content, encoding="utf-8")
            return WiggumResult(
                passed=True,
                final_score=result["score"],
                round_scores=round_scores,
                rounds_taken=round_num,
            )

        if round_num < MAX_ROUNDS:
            content = _revise(content, task, result, models.producer)

    # Exhausted all rounds without passing
    Path(output_path).write_text(content, encoding="utf-8")
    return WiggumResult(
        passed=False,
        final_score=round_scores[-1],
        round_scores=round_scores,
        rounds_taken=MAX_ROUNDS,
    )
```

Three constants govern the loop's behavior:

```python
PASS_THRESHOLD = 8.0    # composite score required to pass
MAX_ROUNDS     = 3      # maximum evaluation–revision cycles
```

These values are not configuration parameters intended for routine adjustment. They were calibrated against 1,500 labeled runs and represent the point at which the tradeoff between improvement probability and overshoot risk is optimal for the task distribution the harness was designed for. Lowering `PASS_THRESHOLD` accepts more mediocre outputs; raising it increases the proportion of runs that exhaust all rounds without passing. Both adjustments should be made with logged experimental data, not intuition.

---

## The Evaluation Step

The `_evaluate()` function is where the rubric is applied. It constructs a structured prompt, calls the evaluator model, and parses the dimensional scores out of the response:

```python
def _evaluate(
    content: str,
    task: str,
    evaluator: str,
    models: ModelConfig,
) -> dict:
    """
    Score content against the six-dimension rubric.
    Returns {score, dimensions, feedback}.
    """
    # Compress the content if it exceeds the evaluator's useful window
    compressed = summarize_for_eval(content, task) \
        if len(content) > EVAL_COMPRESS_THRESHOLD \
        else content

    prompt = EVAL_PROMPT_TEMPLATE.format(
        task=task,
        content=compressed,
    )

    response = inference.chat(
        model=evaluator,
        messages=[{"role": "user", "content": prompt}],
        options={"temperature": 0.0},   # deterministic scoring
    )

    return _parse_rubric_response(response.message.content)
```

Three implementation details deserve attention.

**Temperature zero.** Scoring prompts use `temperature=0.0` to make the evaluator's scores as deterministic as possible. Stochastic evaluation introduces noise that makes it difficult to tell whether a score difference between two runs reflects a real quality difference or a sampling accident.

**Pre-evaluation compression.** When the content exceeds `EVAL_COMPRESS_THRESHOLD` (6,000 characters by default), it is passed through `summarizer.summarize_for_eval()` before scoring. This compression is section-preserving: all second-level headings (`## ...`) are retained verbatim, and section bodies are condensed. The evaluator can therefore assess structural coverage — whether the output addresses all the main subtopics the task requires — without having to read a document too long for its context window. The alternative — truncating the document at the context limit — produces systematically wrong *Completeness* scores because the evaluator only sees the first N characters of a document that continues beyond them.

**Deterministic model selection.** The `select_evaluator()` function chooses the evaluator from the pool using the run ID as a seed:

```python
EVALUATOR_POOL = os.getenv("HARNESS_EVALUATOR_POOL", "").split(",")

def select_evaluator(seed: str) -> str:
    if not EVALUATOR_POOL or EVALUATOR_POOL == [""]:
        return DEFAULT_EVALUATOR
    idx = int(hashlib.md5(seed.encode()).hexdigest(), 16) % len(EVALUATOR_POOL)
    return EVALUATOR_POOL[idx].strip()
```

The hash-based selection ensures that the same run always uses the same evaluator (reproducibility), while different runs are distributed across the pool (drift mitigation). This matters because evaluator models are not perfectly calibrated against each other: a model that consistently scores 0.5 points higher than average on *Groundedness* will inflate the pass rate if used exclusively. Rotating across the pool converts this systematic bias into variance, which is more honest and easier to detect in the analytics.

---

## The Insight: Dimensions Beat Scores

The Wiggum Loop's most important output is not the composite score. It is the dimensional breakdown.

Here is what a passing evaluation looks like in `runs.jsonl`:

```json
{
  "wiggum_rounds": 2,
  "wiggum_scores": [6.4, 8.3],
  "wiggum_dimensions": {
    "round_1": {
      "relevance":     8.0,
      "completeness":  5.5,
      "depth":         6.0,
      "specificity":   6.5,
      "structure":     7.5,
      "groundedness":  5.0
    },
    "round_2": {
      "relevance":     8.5,
      "completeness":  8.0,
      "depth":         8.0,
      "specificity":   8.5,
      "structure":     8.5,
      "groundedness":  8.0
    }
  }
}
```

The composite scores (6.4 → 8.3) tell you that revision worked. The dimensional breakdown tells you *what* revision fixed. In this run, the first-round output was structurally solid (Structure: 7.5, Relevance: 8.0) but failed on coverage and sourcing (Completeness: 5.5, Groundedness: 5.0). The revision prompt would have highlighted exactly those two dimensions:

```python
REVISION_PROMPT_TEMPLATE = """
You wrote the following research output for the task below.
An independent evaluator has reviewed it and identified specific weaknesses.
Revise the output to address those weaknesses directly.

TASK:
{task}

EVALUATOR FEEDBACK:
{feedback}

DIMENSION SCORES:
{dimension_scores}

DIMENSIONS BELOW THRESHOLD (focus your revision here):
{failing_dimensions}

ORIGINAL OUTPUT:
{content}

Write the full revised output. Do not explain your changes — just write the improved version.
"""
```

The `failing_dimensions` field contains only the dimensions that scored below 7.0 — not all six. This is deliberate: presenting the producer with a six-item checklist reliably causes it to make superficial changes across all dimensions while missing the depth needed to fix the ones that actually failed. Presenting only the failing dimensions forces a focused revision.

This pattern — specific dimensional feedback producing targeted revision — accounts for most of the loop's quality lift. The evaluator's free-text `feedback` field amplifies it:

```
EVALUATOR FEEDBACK (Round 1):
The output covers the topic at a surface level but does not cite the 
primary sources that established the benchmark numbers it references 
(lines 34–38). The "Completeness" score reflects that the comparison 
with competing approaches (mentioned in the task) is entirely absent. 
The "Groundedness" score reflects that three of the five quantitative 
claims in the output are not traceable to any retrieved source — they 
appear to have been generated rather than retrieved.
```

A producer model given this feedback knows exactly what to do. A producer model given only a score of 6.4 does not.

---

## The Revision Step

The revision step is the simplest component of the loop:

```python
def _revise(
    content: str,
    task: str,
    eval_result: dict,
    producer: str,
) -> str:
    """
    Ask the producer model to revise its output based on evaluator feedback.
    Returns the revised content string.
    """
    failing = {
        dim: score
        for dim, score in eval_result["dimensions"].items()
        if score < REVISION_FOCUS_THRESHOLD   # 7.0
    }

    compressed = summarize_for_revision(content, eval_result["feedback"]) \
        if len(content) > REVISE_COMPRESS_THRESHOLD \
        else content

    prompt = REVISION_PROMPT_TEMPLATE.format(
        task=task,
        feedback=eval_result["feedback"],
        dimension_scores=format_dimensions(eval_result["dimensions"]),
        failing_dimensions=format_failing(failing),
        content=compressed,
    )

    response = inference.chat(
        model=producer,
        messages=[{"role": "user", "content": prompt}],
        options={"temperature": 0.3},   # slight variation to escape local minima
    )

    return response.message.content
```

The revision compression mode is different from the evaluation compression mode. In `summarize_for_revision()`, sections mentioned in the evaluator's feedback are kept verbatim (so the producer can see exactly what it wrote in the criticized passage), while all other sections are condensed. This surgical approach ensures the producer model can see what needs fixing without the revision prompt exceeding its own context window.

The temperature of 0.3 during revision is intentional. A temperature of 0.0 causes the producer to make minimal changes — it finds the local minimum it can reach deterministically. A temperature of 0.3 introduces enough variation that the producer can make meaningful structural changes while remaining coherent.

---

## The Normalization Step

Research outputs are not always markdown. The harness's browser skill produces HTML. Some outputs contain mixed content. The `_normalize()` function handles this:

```python
def _normalize(output_path: str) -> str:
    """
    Convert HTML output to markdown using Playwright + markitdown.
    Called only when the content appears to be HTML.
    """
    from playwright.sync_api import sync_playwright
    from markitdown import MarkItDown

    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        page.goto(f"file://{output_path}")
        html = page.content()
        browser.close()

    md = MarkItDown()
    return md.convert_string(html, file_extension=".html").text_content
```

Normalizing to markdown before evaluation matters because the rubric dimensions include *Structure* — which asks whether the output uses appropriate organizational devices like headings and lists — and these are more reliably scored against consistent markup than against raw HTML.

---

## Observability

Every round of the loop appends structured data to the run's trace. You can reconstruct the full evaluation history for any run from `runs.jsonl`:

```python
import pandas as pd

df = pd.read_json("data/runs.jsonl", lines=True)

# Explode wiggum_scores into one row per round
rounds = df[["run_id", "task", "wiggum_scores", "final"]].explode("wiggum_scores")
rounds["round"] = rounds.groupby("run_id").cumcount() + 1

# Score trajectory: how much do scores improve across rounds?
pivot = rounds.pivot_table(
    index="round", 
    values="wiggum_scores", 
    aggfunc=["mean", "std"]
)
print(pivot)
```

Typical output from a production harness over 200 runs:

```
        mean    std
round               
1       6.87   1.12
2       8.04   0.73
3       8.31   0.61
```

Three things are visible in this data. First, the average first-round score of 6.87 confirms that single-pass synthesis reliably falls below the 8.0 threshold — the loop is not a safety net for an occasional bad run; it is a necessary stage of the pipeline for the majority of runs. Second, the jump from round 1 to round 2 (6.87 → 8.04) represents the bulk of achievable improvement. Third, the jump from round 2 to round 3 (8.04 → 8.31) is real but smaller — and the declining standard deviation tells you that the runs still in round 3 are increasingly those where revision is difficult, not those where improvement is easy.

The Chrome Trace file for each run — loadable in `ui.perfetto.dev` — shows this in timeline form. A run that passes in round 2 shows two evaluation blocks of roughly equal width. A run that exhausts all three rounds shows three blocks with the third one visibly wider: the producer model is generating longer revisions as it struggles against dimensions that are harder to fix than the evaluator's feedback implies.

---

## Failure Modes of the Loop Itself

The Wiggum Loop catches most quality problems. It does not catch all of them, and there are specific classes of failure where it makes things worse.

**Evaluator drift.** An evaluator model's scoring calibration changes over Ollama version updates, quantization changes, or system prompt variations. A model that was well-calibrated at deployment may inflate or deflate its scores after an update. The solution is the evaluator pool with rotation, combined with periodic recalibration runs using a held-out set of human-rated documents.

**Revision regression.** When the producer model revises an output to improve a failing dimension, it sometimes overwrites content that was correct. A document that had strong *Depth* but weak *Groundedness* may emerge from revision with improved *Groundedness* and degraded *Depth*: the producer added citations but condensed the analytical sections to make room. The loop's score trajectory will show this as a non-monotonic pattern: round 1 scores 6.8, round 2 scores 7.2, round 3 scores 7.6 — with the final score still below threshold, and the document materially different from the round-1 draft in ways that are not all improvements.

**Evaluator hallucination.** Evaluator models, like all language models, can hallucinate. An evaluator that claims "the output does not address X" when X is clearly present in the output will cause the producer to add redundant coverage of X in its revision — wasting tokens and sometimes degrading document coherence. This failure mode is most common on the *Completeness* dimension, where the evaluator must reason about what is absent rather than what is present.

**Context starvation.** When the pre-evaluation compression removes too much of the document, the evaluator scores structural completeness based on a skeleton that misrepresents the actual content. The compressed section headings are present, but the evaluator cannot tell whether the bodies contain substantive analysis or placeholder filler. The 6,000-character compression threshold was chosen to be conservative on this axis; lowering it risks starvation.

---

## Tuning the Loop

Three parameters are available for adjustment:

| Parameter | Default | Effect of raising | Effect of lowering |
|-----------|---------|-------------------|--------------------|
| `PASS_THRESHOLD` | 8.0 | Stricter quality gate; more runs exhaust all rounds | More outputs pass in round 1; lower average output quality |
| `MAX_ROUNDS` | 3 | More recovery attempts for difficult tasks; higher latency | Faster pipeline; more runs fail without passing |
| `REVISION_FOCUS_THRESHOLD` | 7.0 | Fewer dimensions highlighted for revision; tighter focus | More dimensions in the revision prompt; broader but shallower changes |

Tune `PASS_THRESHOLD` last, after establishing a baseline with the default value. The right threshold depends on your task domain, your user's quality expectations, and your latency budget — not on a universal standard. The 8.0 default was calibrated for general research synthesis. A medical literature review pipeline might want 8.5. A lightweight summarization pipeline might want 7.0.

Tune `MAX_ROUNDS` when you observe the loop frequently exhausting all three rounds without passing — which indicates either that `PASS_THRESHOLD` is too high for your task distribution, or that the evaluation–revision cycle is not improving the dimensions that matter. Before raising `MAX_ROUNDS`, examine the dimensional score trajectories in `runs.jsonl` to distinguish these two cases.

Do not touch `REVISION_FOCUS_THRESHOLD` without A/B evidence. It is the parameter most likely to introduce subtle regressions.

---

## A Note on the Name

Inspector Wiggum is not a good police chief. He is well-meaning, occasionally perceptive, and constitutionally limited in ways he cannot see himself. He would be the first to give a suspect a clean bill of health while standing in front of the evidence that convicts him.

The Wiggum Loop was named for this reason deliberately. It is not called the "Quality Assurance Loop" or the "Verification Cycle" or the "Excellence Gate." It is named for a known-imperfect inspector because that is an accurate description of what it is. Any practitioner who deploys it expecting it to catch *all* quality problems will be disappointed and, worse, will fail to build the additional safeguards that a realistic expectation would have prompted.

What the loop provides is reliable, substantial improvement for the majority of outputs, implemented in approximately four hundred lines of Python, with no external dependencies beyond the models already in the pipeline. That is not nothing. In the 1,500 runs analyzed to calibrate the harness, the loop raised the mean Wiggum score from 6.87 to 8.12 — a gain of 1.25 points — and converted a 61% first-pass rate into an 89% final-pass rate. The 11% of runs that exhausted all three rounds without passing are the most interesting part of the dataset.

They are also the subject of the next two chapters.

Chapter 11 dissects the rubric — how each of the six dimensions is scored, how the 8.0 threshold was calibrated, and what the dimensional breakdown reveals about the failure modes that the loop routinely surfaces. Chapter 12 maps the complete failure taxonomy from those 1,500 runs: the six classes of harness failure, their observed frequencies, the specific runs that exemplify each class, and — most importantly — the pipeline changes that reduced each class's failure rate. That chapter will challenge something you probably believe about where quality problems originate.

The answer is rarely where you expect.

---

## Summary

The Wiggum Loop is a producer–evaluator separation pattern that iterates evaluate → revise → verify until the output passes a multi-dimensional quality rubric or the maximum round count is reached. Its key design decisions are:

- The evaluator must be a different model from the producer
- Temperature-zero scoring for determinism; temperature-0.3 revision for creative flexibility  
- Dimensional feedback, not composite score, drives the revision prompt
- Pre-evaluation and pre-revision compression modes are distinct and serve different goals
- Three rounds captures 75% + 18% = 93% of achievable improvement; beyond three, risk exceeds reward
- Evaluator pool rotation distributes load and converts systematic bias into measurable variance

The loop is not a correctness guarantee. It is a reliable quality lift for the runs it handles, and a structured failure record for the runs it does not. Both are valuable — but only if you read the records.

---

*Next: Chapter 11 — The Evaluation Rubric: how each dimension is defined, scored, and calibrated.*

---

**Code.** All code in this chapter is from `harness/wiggum.py`. Constants are defined in `harness/config.py`. The evaluator prompt template is in `harness/prompts/eval.py`. The full source is available at [github.com/upskilled-consulting/harness-refactor].
