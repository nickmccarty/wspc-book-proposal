
# Introduction

There is a class of engineering problem that becomes visible only after you have built enough of something to see the pattern in what breaks.

In software, the canonical example is concurrency. You can build a system that works perfectly under a single user, extend it to ten, extend it to a thousand, and never encounter the problem — until the day you do, and the failure has no stack trace, no error message, and no obvious cause. The root issue was always there. You just had not yet run the experiment that would reveal it.

Agentic AI systems have their own version of this problem, and it is not the one the field is currently debating.

The public conversation about AI agents is almost entirely about models: which foundation model is best, how large the context window is, whether chain-of-thought reasoning improves output quality. These are real questions. But they are the wrong level of abstraction for the practitioner who needs a system that works in production — reliably, auditably, and without silent failures that look like successes.

The practitioner's problem is the harness.

---

## What This Book Is About

A harness is the software layer that sits between a language model and the world: the code that decomposes a task into subtasks, routes each subtask to the right model, evaluates the result, decides whether to revise or accept, manages state across long-running runs, enforces security boundaries, and records enough telemetry to understand what happened after the fact.

Most practitioners build their harness the same way most people build their first concurrent system: incrementally, reactively, and without a vocabulary for the failure modes they have not yet encountered. They wire together a prompt template and an API call. They add memory when they realize the agent has none. They add an evaluation loop when they notice outputs are inconsistent. They add logging when something goes wrong and they cannot diagnose it.

This produces systems that work — until they do not. And when they fail, the failure is often of exactly the kind described above: silent, plausible-looking, and repeatable.

This book is a systematic treatment of the harness. It is organized as a pattern catalog in the tradition of Fowler's *Patterns of Enterprise Application Architecture* — twenty-seven named patterns across seven sections, each grounded in a real open-source codebase and validated against 1,500 logged production runs. The patterns are not theoretical. Each one was derived from observing a failure, diagnosing its root cause, and implementing a solution whose consequences were then measured over subsequent runs.

The empirical grounding matters. Pattern catalogs can become prescriptive without it — a collection of opinions dressed up as engineering. The patterns in this book each have a before-and-after: a failure class that motivated them, a metric by which their effect was measured, and a set of known uses drawn from the companion codebase's run log. Where a pattern has tradeoffs, those tradeoffs are stated explicitly, with the conditions under which they dominate.

---

## The Failure That Started This

The first concrete motivation for this book was a failure mode I call the *silent overwrite*.

When you run multiple research agents in parallel, each agent writes its output to a file. If two agents are working on related subtasks — variants of the same parent task — and both derive their output filename from a hash of the task string, the filenames can collide. The second writer overwrites the first. The run log shows two completions. The filesystem shows one file.

The assembly model reads the available outputs and produces a document that looks, on the surface, like a complete multi-perspective synthesis. It is not. The evaluation system scores it, and it passes — because the evaluator cannot detect the absence of content it never saw. The run goes into `runs.jsonl` as a PASS.

There is no crash. There is no warning. There is no indication that anything went wrong.

This failure occurred on the forty-third logged run of the first parallel orchestration implementation. It was not caught until run sixty-one, when a spot-check of the assembly model's output against its claimed source count revealed the discrepancy. Eighteen runs had passed evaluation with a hidden defect.

The fix — Pattern D2, the Worktree Context — is forty lines of code in `orchestrator.py`. It provisions each concurrent subtask with an isolated Git worktree before dispatch, giving each agent its own filesystem environment so that no two agents can write to the same path. The fix is not complicated. The problem is that you have to know to look for it, and to know to look for it, you have to have seen the failure.

This book is the accumulated knowledge of having seen the failures.

---

## How the Book Is Organized

The book is in two parts.

**Part I: The System** is four narrative chapters that establish the vocabulary and conceptual framework. Chapter 1 introduces the central claim of the book: that the quality of an agentic system's output is determined more by the harness than by the model, for any fixed task domain and model capability tier. Chapter 2 traces a single research task through the full pipeline — from entry point through planning, agent loop, evaluation, memory update, and telemetry — to make the system concrete before we begin decomposing it. Chapter 3 describes the experimental methodology used to generate the 1,500-run dataset that underpins the pattern validations: the six-dimension quality rubric, the evaluator pool rotation, the failure taxonomy. Chapter 4 is the failure taxonomy itself — six classes of failure derived from the run log, with representative records, frequency distributions, and pointers to the patterns that address each class.

**Part II: The Pattern Catalog** is thirty patterns organized into seven sections:

- **Section A — Inference** covers the substrate: the inference shim that routes calls to Ollama, vLLM, or llama.cpp through a single `chat()` interface; the model role separation that prevents the same model from serving as both producer and evaluator; the evaluator pool rotation that converts per-model scoring bias into diagnosable variance.

- **Section B — Context Engineering** covers what gets put into the model's context window: the planner-first architecture that separates task decomposition from execution; the novelty gate that prevents redundant retrieval; the dual-backend memory store that combines ChromaDB semantic search with SQLite FTS5 keyword search; the semantic chunker; the vision bridge.

- **Section C — Verification** covers evaluation loops: the Wiggum Loop (evaluate → revise → verify, three rounds maximum, 8.0 threshold); the dimensional rubric used to score outputs; the surgical compressor that reduces context before revision; the ReAct Comparator that benchmarks Wiggum against the standard ReAct pattern.

- **Section D — Orchestration** covers multi-agent coordination: the DAG orchestrator with Kahn's cycle detection; the Worktree Context (the sample chapter); the MCP dispatch router for cross-machine subtask routing; the skill registry.

- **Section E — Security** covers the four defense layers implemented in Python's standard library: AST Guard, Path Sandbox, Injection Scanner, CDP Guard.

- **Section F — Observability** covers the telemetry system: RunTrace, the dashboard, the failure classifier.

- **Section G — Self-Improvement** covers the data flywheel: the run-to-dataset pipeline, the RL rollout using the Wiggum loop as an online reward signal, the literature review pipeline.

Each pattern entry follows a consistent structure: a single-sentence intent, an "also known as" field that maps the pattern to equivalent terminology in adjacent literature, a how-it-works narrative, an applicability guide with explicit when-not-to-use conditions, a consequences table of benefits and tradeoffs, an implementation section with working code from the companion repository, and a set of cross-references to related patterns.

---

## What This Book Is Not

This book is not a tutorial on prompt engineering. Prompt engineering is a consequential practice, but it is the domain of the model, not the harness. The harness is what routes the prompt, evaluates the response, and decides what to do with it.

This book is not a survey of foundation models or a benchmark comparison. The companion codebase runs entirely on local hardware — Ollama, vLLM, llama.cpp — with no external API keys required, specifically because the claim of the book is that harness design matters more than model selection for a fixed capability tier. Any claim that depended on access to a specific API would be fragile and unverifiable.

This book is not an introduction to Python or TypeScript. The implementation chapters assume fluency with both. The patterns describe *what* to build and *why*; the code shows *how*, not how to set up a development environment.

This book is not a collection of opinions about what agentic systems should eventually look like. Every pattern in it is running in production. Every failure class in Chapter 4 is drawn from a real run log. The empirical grounding is not decoration — it is the point.

---

## Who This Book Is For

The primary audience is the software engineer who has already shipped something agentic and is now trying to understand why it behaves the way it does. The pattern vocabulary gives you language for problems you may have encountered without knowing what to call them.

The secondary audience is the researcher or architect who is designing an agentic system and wants to understand the failure modes in advance. Part I's failure taxonomy and the when-not-to-use sections of the pattern entries are written for this reader.

The tertiary audience is the technical leader who needs to evaluate an existing agentic implementation. The pattern catalog gives you a checklist: here are the thirty things a production harness should address, here is what to look for in the code, and here is what failure looks like when each is absent.

---

## The Companion Codebase

Every pattern in this book has a corresponding module in the companion repository. The codebase is the primary artifact; the book is its documentation and theoretical frame.

The repository runs on a single consumer GPU (minimum 8 GB VRAM for single-model deployment; 24 GB for simultaneous producer/evaluator/planner). All models are open-weight. All infrastructure dependencies — ChromaDB, SQLite, Git — are local. The system has been run on Linux, macOS, and Windows.

The run log — 1,500 entries in `runs.jsonl` — is included in the repository. Each entry records the task, the model configuration, the Wiggum scores across all evaluation rounds, the final output, the failure class (if any), and the Chrome Trace Events file for that run. The patterns in Part II are documented against this run log: the failure that motivated each pattern is traceable to specific entries.

---

## A Note on the Title

*Agentic harness engineering* is not a term of art in the current literature. It is a deliberate choice.

*Harness* is a word from the testing tradition — a test harness is the scaffolding around the code under test. Borrowing it for the agentic context makes explicit the relationship between the harness and the model: the model is under test, in the sense that its outputs are subject to evaluation, revision, and quality gating. The harness is the scaffolding that makes that evaluation possible and that routes the model's outputs to whatever comes next.

*Engineering* is equally deliberate. The patterns in this book are engineering artifacts, not theoretical proposals. They have implementations, failure modes, measurable consequences, and known uses. The book treats the harness as what it is: a production software system, subject to the same disciplines — empirical validation, failure analysis, pattern documentation — that software engineering has developed for production software systems in every other domain.

The patterns are waiting for the failures. This book is the map.

---

**Code.** The companion codebase is available at [github.com/upskilled-consulting/ollama-harness](https://github.com/upskilled-consulting/ollama-harness); install via `pip install ollama-harness`. **Running the examples** requires Python 3.11+, Git, and at least 8 GB of VRAM for a single-model deployment. Full environment setup is documented in `SETUP.md`. **The run log** is at `data/runs.jsonl`. The Chrome Trace Events files are in `data/traces/`.
