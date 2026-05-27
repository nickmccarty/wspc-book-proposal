
# Table of Contents with Chapter Summaries

## *Agentic Harness Engineering: Full-Stack Open Source Scaffolding Using Python and TypeScript*

Nicholas McCarty · Upskilled Consulting

---

### Introduction *(~5 pp.)*

Why this book was written, who it is for, and how to use it. The introduction situates the book against the current moment in AI tooling — a period in which foundation models have become commodities while the infrastructure surrounding them remains largely artisanal and undocumented. It describes the Upskilled harness as the running artifact throughout the text: every pattern in Part II is grounded in a real, publicly available codebase that readers can clone, instrument, and modify. It explains the two-part structure (narrative foundation, then pattern catalog), defines the notation used in pattern entries, and provides three suggested reading paths for different practitioner profiles.

---

## Part I — Foundations *(~80 pp.)*

*Four narrative chapters that build the argument, establish the measurement framework, and catalog the failure modes that motivated the patterns in Part II. Read these before engaging the catalog.*

### Chapter 1: The Harness Thesis *(~20 pp.)*

The book's central claim: for a fixed task domain, the quality of the scaffolding surrounding a language model matters more than the choice of model. This chapter presents the experimental evidence — controlled A/B runs across five open-source models of varying parameter counts, held constant except for the harness configuration — and shows that a well-designed pipeline lifts a 7B model above the unscaffolded baseline of a 70B model. The chapter frames the rest of the book around a single design question: *what does the harness owe the model, and what does it owe the user?* Key concepts introduced: the five-stage pipeline (decompose → research → synthesize → evaluate → persist), the producer/evaluator separation principle, and the distinction between model capability and pipeline capability.

### Chapter 2: The Pipeline in Motion *(~20 pp.)*

A complete walkthrough of the harness pipeline as a software system. The chapter covers each of the eleven subsystems — Entry Points, Orchestration, Core Agent Loop, Inference, Research Tools, Skills, Memory, Security, API Routes, Observability, and Storage — by tracing a single research task from CLI invocation to vetted, persisted markdown output. This is the reader's first encounter with the system-overview diagram; each subsystem is introduced in functional terms before its implementing pattern is encountered in Part II. The chapter closes with a map from Part I chapters to Part II patterns, so readers know which patterns address which pipeline stages.

### Chapter 3: Experimental Methodology *(~20 pp.)*

How to measure whether a harness change actually helps. This chapter introduces the TREATMENT/CONTROL naming convention used in the harness's output files, the `runs.jsonl` audit trail, and the six-dimension decimalized rubric that produces a scalar quality score for every output. It covers the sample size problem (how many runs are needed before a difference in mean score is statistically reliable?), the risk of evaluator drift when the same model scores thousands of outputs over months, and the use of the Evaluator Pool pattern (Part II, Section C) to counteract it. Working Python snippets load `runs.jsonl` into pandas, run Mann–Whitney U tests on score distributions, and generate the treatment-effect plots used throughout the book.

### Chapter 4: Failure Taxonomy *(~20 pp.)*

Before the catalog of solutions, a catalog of problems. This chapter documents the six classes of harness failure observed across 1,500 logged runs: Retrieval Failures (search returned insufficient or misleading content), Planning Failures (the planner generated queries that missed the task's core requirement), Synthesis Failures (the producer hallucinated, hedged excessively, or lost the thread), Evaluation Failures (the evaluator scored a low-quality output above threshold), Revision Failures (the producer failed to act on dimensional feedback), and Infrastructure Failures (model timeouts, VRAM exhaustion, context overflow). For each class, the chapter gives the observed frequency, a representative `runs.jsonl` record, and a pointer to the Part II pattern that reduced its rate.

Retrieval Failures include a memory contamination sub-variant that is distinct from search failures: low-quality observations re-injected from the memory store crowd out diverse context. In one documented case, two copies of a 6.2-scoring run occupied two of the top-four retrieval slots, anchoring synthesis near the same quality ceiling. The fix deduplicates retrieved observations by title and applies a 0.5× quality weight to any observation scoring below 7.0 at retrieval time, so low-quality prior runs cannot displace higher-quality ones.

Revision Failures include two shipped safeguards documented in this chapter. Cycling — where the producer revises but the evaluator returns identical dimensional scores, indicating zero measurable progress — is detected by comparing the full six-dimension score vector between consecutive rounds; the loop exits early rather than burning remaining revision budget. On failed-loop exit, the implementation writes the highest-scoring round's output to disk rather than the final round's, preventing regression in the 21% of multi-round runs where round-over-round scores were non-monotonic.

This chapter is the connective tissue between the argument in Part I and the solutions in Part II.

---

## Part II — The Pattern Catalog *(~350 pp.)*

*Thirty patterns organized into seven sections. Each entry follows the standard structure: a one-line intent, an "also known as" field, a how-it-works narrative, a when-to-use section, a consequences table, implementation notes with code, and cross-references to related patterns.*

---

### Section A: Inference Patterns *(~60 pp.)*

*Patterns governing how LLM calls are issued, routed, and kept warm — the substrate on which every other pattern depends.*

---

#### Pattern A1: The Inference Shim *(~15 pp.)*

**Intent.** Route all LLM calls through a single `chat(model, messages)` interface regardless of which backend is serving the model, so that callers are insulated from backend-specific APIs.

**Also known as.** Backend Abstraction Layer; Unified Inference Endpoint; OllamaLike Shim.

**How it works.** `harness/inference.py` reads the model name, looks it up in `HARNESS_ENDPOINTS` (a JSON configuration mapping model tags to backend URLs and types), and routes the call to the appropriate backend: Ollama, vLLM, llama.cpp, or any OpenAI-compatible endpoint. The response is normalized into an `OllamaLike` object — a minimal shim that exposes `.message.content` and `.message.role` — so all calling code receives the same interface regardless of backend. Backend selection is invisible to the agent, planner, evaluator, and every other pattern in the catalog.

**Consequences.** Callers never import backend SDKs directly. Switching a model from Ollama to vLLM requires only a one-line change in `HARNESS_ENDPOINTS`. The shim adds one function call of overhead — negligible against inference latency. The cost is that error messages from backends must be normalized, which requires maintaining per-backend error handling in one place. See also: *The Keep-Alive Budget* (A4), *The Model Role Separation* (A2).

---

#### Pattern A2: The Model Role Separation *(~12 pp.)*

**Intent.** Assign language model calls to one of three non-overlapping roles — producer, evaluator, planner — and never use the same model instance for two different roles within the same run.

**Also known as.** Producer–Evaluator Separation; Role-Typed Inference; Three-Role Architecture.

**How it works.** The `ModelConfig` dataclass carries three fields — `producer`, `evaluator`, `planner` — that are populated at run start and passed to every component that issues inference calls. The agent loop uses only the producer; the Wiggum Loop uses only the evaluator; the planner module uses only the planner model. The evaluator field is always set to a different model than the producer; the harness enforces this at `ModelConfig` construction time and raises `ConfigurationError` if the two are the same. In practice, the producer is the largest available model optimized for long-form synthesis; the evaluator is a different family model (to avoid shared biases); the planner is the smallest fast model available, since planning latency directly affects perceived responsiveness.

**Consequences.** Eliminates the self-evaluation bias documented in Chapter 4. Requires at minimum two distinct model instances to be loaded; see *The Keep-Alive Budget* (A4) for VRAM management. Cross-family selection of evaluator (e.g., GLM4 evaluating Qwen3 output) reduces shared-distribution bias at the cost of calibration variance across families. See also: *The Evaluator Pool* (A3), *The Wiggum Loop* (C1).

---

#### Pattern A3: The Evaluator Pool *(~12 pp.)*

**Intent.** Rotate the evaluator model across runs using a deterministic hash of the run ID, converting systematic per-model scoring bias into measurable across-model variance.

**Also known as.** Evaluator Rotation; Bias Dilution; Seeded Evaluator Selection.

**How it works.** `HARNESS_EVALUATOR_POOL` is a comma-separated list of model names. `select_evaluator(seed)` hashes the seed (typically the run ID) using MD5 and takes the result modulo the pool size. The same run ID always selects the same evaluator — reproducibility is preserved — but different runs are distributed across the pool. A model that consistently over-scores *Groundedness* by 0.5 points will inflate the pass rate if used exclusively; distributed across a pool of four evaluators, the bias becomes variance that is visible in score distribution analytics and can be diagnosed and corrected.

**Consequences.** Requires multiple evaluator models to be installed. Pool selection is deterministic, so reruns of the same task will use the same evaluator and produce comparable scores. Calibration drift across pool members shows up as high variance in per-evaluator score distributions in the `runs.jsonl` analytics; the remedy is periodic recalibration runs against a held-out human-rated set. See also: *The Model Role Separation* (A2), *The RunTrace* (F1).

---

#### Pattern A4: The Keep-Alive Budget *(~12 pp.)*

**Intent.** Pre-allocate VRAM residency across producer, evaluator, and planner models before any run begins, eliminating cold-start latency between pipeline stages.

**Also known as.** Model Warm Cache; VRAM Residency Management; Keep-Alive Tuning.

**How it works.** Ollama's `keep_alive` parameter controls how long a model remains loaded in VRAM after its last inference call. Setting `keep_alive=-1` keeps a model loaded indefinitely. The `_estimate_keep_alive()` function in `harness/agent.py` estimates the time between consecutive calls to each model (based on task type and the skill being run) and sets `keep_alive` accordingly. For a standard research run, the producer and evaluator are loaded before the first call and held warm until the run completes. The planner, which finishes before synthesis begins, is released earlier to free VRAM for the producer's context window. On constrained hardware, the three-model budget is managed sequentially — planner loads, plans, unloads; producer loads, synthesizes; evaluator loads, evaluates — with each model loading taking 2–5 seconds rather than 30–60 seconds on a cold start.

**Consequences.** Correct warm-cache management eliminates 60–90% of the latency that appears in Chrome Trace timelines as gaps between stages. Incorrect management — forgetting to release the planner — causes VRAM pressure during synthesis that manifests as generation slowdown rather than OOM errors. See the Perfetto timeline patterns in *The RunTrace* (F1) for diagnosis. See also: *The Inference Shim* (A1).

---

### Section B: Context Engineering Patterns *(~80 pp.)*

*Patterns governing what information reaches the synthesis model — the research half of the pipeline.*

---

#### Pattern B1: The Planner-First *(~15 pp.)*

**Intent.** Intercept every task before any search query is issued, producing a structured plan that replaces vague auto-generated queries with targeted ones and identifies what is already known.

**Also known as.** Pre-Search Planning; Query Generation; Plan-Then-Search.

**How it works.** `harness/planner.py` receives the task string and the agent's current memory context and calls a fast planner model (typically 9B parameters, targeting under 15 seconds latency). The model returns a `Plan` dataclass with four fields: `search_queries` (concrete, targeted queries replacing the task string itself), `known_facts` (facts already in memory that do not need re-fetching), `knowledge_gaps` (what the pipeline does not yet know and must find), and `subtasks` (a decomposition hint for the orchestrator). The agent loop uses `search_queries` exclusively; it never issues the raw task string as a search query.

**Consequences.** Logged run comparisons show that planner-guided search reduces rounds needed to pass *The Wiggum Loop* (C1) by an average of 0.8 while cutting token consumption by ~22%. The cost is 12–18 seconds of planning latency before the first search. Planning latency is dominated by model load time on cold starts; see *The Keep-Alive Budget* (A4). The planner's quality depends on the richness of the memory context injected; see *The Dual-Backend Memory Store* (B3). See also: *The DAG Orchestrator* (D1), *The Novelty Gate* (B2).

---

#### Pattern B2: The Novelty Gate *(~15 pp.)*

**Intent.** Score each batch of search results against the agent's accumulated knowledge state and discard batches that add no new information before they reach the synthesis prompt.

**Also known as.** Saturation Gate; Information Saturation Gating; Novelty-Gated Search.

**How it works.** `agent.gather_research()` runs multiple search rounds, each producing a batch of results (title, snippet, URL). After each batch is retrieved, `memory.assess_novelty(results, knowledge_state)` scores it on a 0–10 scale by asking the planner model whether the batch contains information not already represented in the knowledge state. Batches scoring below a configurable threshold (default: 3.0) are discarded. The knowledge state is updated after each passing batch, so the threshold applies incrementally — early rounds retrieve foundational content, later rounds are penalized more heavily for overlap. `RESEARCH_CACHE=1` caches results keyed by query string so that Wiggum revision rounds do not re-fetch already-retrieved content.

**Consequences.** Prevents the synthesis prompt from being diluted by redundant content, which is the primary cause of *Completeness* score inflation (the evaluator scores "comprehensive coverage" when the document repeats three sources sixteen times). The gate also reduces inference cost on revision rounds by reusing cached results. The main risk is an overly aggressive threshold that gates out novel content on the grounds that it overlaps with a general background fact. Tune the threshold using the `novelty_score` field in `runs.jsonl` tool calls. See also: *The Planner-First* (B1), *The Dual-Backend Memory Store* (B3).

---

#### Pattern B3: The Dual-Backend Memory Store *(~20 pp.)*

**Intent.** Maintain a persistent observation store backed by two complementary retrieval engines — a vector database for semantic similarity and a full-text search index for exact keyword overlap — and query both at retrieval time, using each to compensate for the other's blind spots.

**Also known as.** Hybrid Memory Retrieval; ChromaDB + FTS5; Two-Stage Memory.

**How it works.** Every completed run writes a compressed observation (narrative + structured fact list) to both ChromaDB (embedding it with `all-MiniLM-L6-v2` into a 384-dimensional vector) and a SQLite FTS5 index on the `task`, `title`, and `narrative` fields. At retrieval time, `memory.get_context(task)` queries ChromaDB first (cosine similarity finds semantically related observations even when no words overlap) and then re-ranks the results using FTS5 (exact keyword matches score higher). The top four observations by re-ranked score are injected into the synthesis prompt as memory context. A novelty check at write time prevents storing observations that add no new information to the store.

**Consequences.** Two backends are necessary because neither alone is sufficient: vector search finds "gradient descent optimization" when the task is "neural network training" but misses exact proper nouns and model names; FTS5 finds exact matches but treats "speculative decoding" and "token prediction acceleration" as unrelated. Together they achieve recall that neither can match alone, at the cost of maintaining two synchronized stores and a more complex retrieval pipeline. Prompt injection scanning gates all writes; see *The Injection Scanner* (E3). See also: *The Planner-First* (B1), *The Novelty Gate* (B2), *The JSONL Audit Log* (F2).

---

#### Pattern B4: The Semantic Chunker *(~15 pp.)*

**Intent.** Extract the most relevant sections of a document too large for the synthesis context window, using section-priority rules for structured documents and embedding-based retrieval for unstructured ones.

**Also known as.** Context Budget Extraction; Relevance-Guided Chunking; Hybrid Document Extractor.

**How it works.** `harness/chunker.py` auto-detects the document type. Documents with three or more markdown headings are treated as structured: sections are extracted in priority order (Abstract > Conclusion > Introduction > Results > Methods > Other) until the 12,000-character budget is exhausted. Documents without headings are split into overlapping 600-character chunks with 80-character overlap, embedded with `sentence-transformers`, and the top-K chunks ranked by cosine similarity to the task string are retained. Both modes operate within a configurable character budget that prevents context overflow.

**Consequences.** Section-priority extraction is deterministic and fast but assumes the document structure follows a known template (academic papers, technical reports). Semantic chunking handles arbitrary documents but requires the sentence-transformers model to be loaded, adding ~1 second on first call. The 12,000-character default budget was chosen to leave room for the task string, memory context, and synthesis instructions within a 32K context window. Tables, code blocks, and numbered lists require special-case handling documented in the implementation notes. See also: *The Dual-Backend Memory Store* (B3), *The Surgical Compressor* (C3).

---

#### Pattern B5: The Vision Bridge *(~10 pp.)*

**Intent.** Extract a structured text description from an image found in the task string and inject it into the pipeline context before any other processing, making visual content available to text-only synthesis models.

**Also known as.** Image-to-Context Extraction; Multimodal Injection; Pre-Pipeline Vision.

**How it works.** `harness/vision.py` scans the task string for file paths with image extensions (PNG, JPG, JPEG, GIF, BMP, WebP). If found, it routes each image through the vision model (`llama3.2-vision`, 11B parameters) with a structured extraction prompt that requests: all visible text, data points from charts or tables, a layout description, and identified objects. The resulting text description replaces the image path in the task context. Downstream pipeline stages see only text; they are unaware that the task originally contained an image path.

**Consequences.** Enables the research pipeline to reason about visual artifacts (benchmark plots, architecture diagrams, scanned documents) without modifying any downstream pattern. The vision model must be loaded separately from the producer and evaluator, adding to the VRAM budget; see *The Keep-Alive Budget* (A4). Vision extraction adds 15–40 seconds per image depending on complexity. The pattern is not triggered for remote URLs — only local file paths — to prevent accidental disclosure of credentials embedded in URLs. See also: *The Inference Shim* (A1).

---

### Section C: Verification Patterns *(~80 pp.)*

*Patterns governing how output quality is measured and improved after synthesis.*

---

#### Pattern C1: The Wiggum Loop *(~20 pp.)*

**Intent.** Use a second model instance to evaluate synthesis output on multiple quality dimensions, route the dimensional feedback back to the producer for targeted revision, and repeat until the composite score exceeds a threshold or the maximum round count is reached.

**Also known as.** Evaluate–Revise Loop; Cross-Model Verification; Producer–Evaluator Cycle.

**How it works.** `harness/wiggum.py` runs up to three evaluate–revise cycles. In each cycle, the evaluator model (never the same as the producer; see *The Model Role Separation*, A2) scores the output on six dimensions at 0.5-point resolution, producing a composite score and free-text dimensional feedback. If the composite is below 8.0, the feedback is routed to the producer as a revision prompt that highlights only the failing dimensions — not all six. Analysis of 1,500 logged runs shows that 75% of achievable quality improvement occurs in round 1, 18% in round 2, and the remainder in round 3; beyond three rounds, marginal gain falls below measurement noise. Named for the reliably imperfect Chief Wiggum of *The Simpsons*: it catches most problems, but not all.

**Consequences.** Raises mean Wiggum score from 6.87 (single-pass) to 8.12 (post-loop) across the reference run set, converting 61% first-pass rate to 89% final-pass rate. Adds 15–80 seconds per evaluation round depending on output length and evaluator model. The loop's most common failure mode is revision regression — the producer improves a failing dimension while degrading a passing one; 21% of multi-round runs showed this pattern across a 57-run analysis. Score trajectories in `runs.jsonl` detect this as non-monotonic round-over-round scores. A second failure mode is cycling: the producer revises but the evaluator returns identical scores across all six dimensions, indicating the revision had no measurable effect. The loop detects cycling by comparing the full dimensional score vector between consecutive rounds and exits early rather than burning remaining revision budget on zero-progress iterations. On loop exit without a pass, the implementation writes the highest-scoring round's output to disk — not the final round's — guarding against regressions in the last round. See Chapter 4 (Evaluation Failures, Revision Failures) for frequency data. See also: *The Dimensional Rubric* (C2), *The Surgical Compressor* (C3), *The ReAct Comparator* (C4).

---

#### Pattern C2: The Dimensional Rubric *(~15 pp.)*

**Intent.** Score output quality on six named dimensions — Relevance, Completeness, Depth, Specificity, Structure, Groundedness — at 0.5-point resolution, using the per-dimension breakdown rather than the composite score to guide revision.

**Also known as.** Six-Dimension Quality Rubric; Decimalized Scoring; LLM-as-a-Judge Rubric.

**How it works.** The rubric is a structured prompt template in `harness/prompts/eval.py` that instructs the evaluator to score each dimension independently before computing a composite. Equal weighting was chosen after A/B experiments with alternative weightings failed to produce statistically significant quality improvements over equal weights on the reference task distribution. The 8.0 composite threshold was calibrated against a human-rated holdout set of 200 outputs spanning the harness's full task type distribution; it corresponds to the point at which human reviewers rate outputs as "good enough for intended use" at a 90% agreement rate. Dimension definitions, scoring anchors at 0, 5, and 10, and common scoring errors by dimension are documented in the implementation notes.

**Consequences.** The dimensional breakdown is more valuable than the composite score for revision guidance; see *The Wiggum Loop* (C1). The rubric is not universal — it was calibrated for long-form research synthesis and performs poorly for code generation, short-form Q&A, and creative writing. Domain-specific adaptations (replacing *Groundedness* with *Correctness* for factual Q&A, replacing *Depth* with *Parsimony* for code) are straightforward modifications to the prompt template. See also: *The Wiggum Loop* (C1), *The ReAct Comparator* (C4).

---

#### Pattern C3: The Surgical Compressor *(~15 pp.)*

**Intent.** Compress documents before evaluation and before revision using two distinct strategies: section-preserving compression for the evaluator (to support structural scoring) and section-targeted compression for the producer (to preserve verbatim the passages that need fixing).

**Also known as.** Pre-Evaluation Summarization; Differential Compression; Context-Mode Compression.

**How it works.** `harness/summarizer.py` exposes two functions with different compression strategies. `summarize_for_eval(content, task)` retains all second-level headings verbatim and condenses section bodies, enabling the evaluator to score *Completeness* and *Structure* without reading a document too long for its context window. `summarize_for_revision(content, feedback)` parses the evaluator's feedback for section references and retains those sections verbatim, condensing everything else — so the producer can see exactly what it wrote in the passages the evaluator criticized. Triggering thresholds are distinct: 6,000 characters for evaluation, 5,000 for revision.

**Consequences.** Without section-preserving evaluation compression, the evaluator scores *Completeness* on a skeleton and produces systematically biased feedback. Without section-targeted revision compression, the producer cannot locate the passages it needs to fix. The two-threshold design prevents unnecessary compression overhead on short outputs. The 20,000-character input cap prevents OOM errors on pathologically long outputs. See also: *The Wiggum Loop* (C1), *The Semantic Chunker* (B4).

---

#### Pattern C4: The ReAct Comparator *(~12 pp.)*

**Intent.** Use the ReAct (Reason+Act) evaluation loop as a baseline against which the Wiggum Loop's performance is measured, and decide which loop to deploy based on task type, latency budget, and the availability of a distinct evaluator model.

**Also known as.** ReAct vs. Wiggum; Evaluation Loop Selection; Loop-Mode Decision.

**How it works.** The ReAct loop interleaves reasoning and action steps within a single model context: the model reasons about what to verify, issues a search or lookup action, observes the result, and updates its assessment. This is the standard evaluation approach in most published agentic frameworks. The Wiggum Loop replaces this with an external cross-model evaluation: a separate model scores a completed synthesis against a fixed rubric. The two approaches have complementary failure modes: ReAct loops fail when the evaluating model's reasoning about what to check is itself incorrect (reasoning errors compound); Wiggum loops fail when the rubric does not capture what matters for the task (rubric specification errors compound). The pattern documents the decision criteria — task type, model availability, latency constraints — and provides a diagnostic for identifying which failure mode is occurring in practice.

**Consequences.** For general research synthesis with a distinct evaluator model available, the Wiggum Loop outperforms ReAct on all six rubric dimensions in the reference dataset. For code verification tasks, ReAct — where the evaluating model can execute code snippets and observe errors — outperforms rubric-based scoring. For creative tasks, neither loop is well-suited and human evaluation is necessary. This pattern is reference material; it does not introduce new code but documents the trade-off analysis that should precede any deployment decision. See also: *The Wiggum Loop* (C1), *The Dimensional Rubric* (C2).

---

#### Pattern C5: The Judge Calibration Guard *(~15 pp.)*

**Intent.** Detect and compensate for systematic LLM evaluator failure modes — position bias, preference leakage, calibration gap, and score inflation — before they corrupt the Wiggum Loop's quality signal and produce false pass rates.

**Also known as.** Evaluator Bias Checker; Calibration Anchor System; Judge Health Monitor.

**How it works.** The empirical literature on LLM-as-a-Judge reliability documents four failure modes that affect production evaluation pipelines. *Calibration gap*: even the best judge models lag inter-human agreement by up to 5 points, and smaller models produce unreliable absolute scores (though usable ranking signals). *Position bias*: 8 of 9 judges in a broad study show degenerate "always-A" behavior in pairwise comparison tasks regardless of output quality; model scale does not predict stability. *Preference leakage*: when generator and judge share a model family, parent–child relationship, or training lineage, the judge systematically favors its own architecture's style (same-model, inheritance, and same-family relatedness all introduce correlated scoring priors). *Score inflation*: automated judges cannot reliably resolve disagreements between strong models — in one mathematics benchmark, the leading judge was wrong in 96.4% of expert-audited disagreements.

The Calibration Guard addresses these through four complementary mechanisms. First, *cross-family evaluator selection* is enforced at `ModelConfig` construction time: the evaluator must not share a model family string with the producer (e.g., `qwen` and `qwen3` are treated as the same family). This is a hard constraint, not advisory. Second, *periodic anchor runs* execute the baseline instruction against a 50-run held-out human-rated set every 200 production runs (configurable via `HARNESS_CALIBRATION_INTERVAL`); if the evaluator's mean score on that set drifts more than 0.3 points from its initial calibration score, a recalibration alert is emitted to `data/security_events.jsonl`. Third, *conformal prediction wrappers* around Wiggum composite scores convert point estimates into calibrated score intervals — a composite of 8.2 is reported as [7.9, 8.5] at 90% coverage — making it explicit when a borderline PASS decision falls within the margin of error. Fourth, *score variance monitoring* in `runs.jsonl` analytics flags any evaluator whose per-dimension scores show zero variance over 20 consecutive runs, indicating the evaluator has frozen and is no longer providing diagnostic information.

**Consequences.** The cross-family constraint reduces the evaluator pool by eliminating same-family candidates; on constrained hardware with limited model diversity, this may be difficult to satisfy. The conformal intervals are conservative — early in a deployment they are wide because few calibration runs exist — and narrow as the held-out set accumulates. The anchor run cadence adds ~2 hours of compute every 200 runs at standard throughput; this is adjustable but cannot be eliminated without losing the calibration guarantee. The pattern does not eliminate judge error; it makes the error measurable and visible. See also: *The Model Role Separation* (A2), *The Evaluator Pool* (A3), *The Wiggum Loop* (C1), *The Dimensional Rubric* (C2).

---

### Section D: Orchestration Patterns *(~70 pp.)*

*Patterns governing how the harness coordinates multiple agents, routes tasks across instances, and exposes its capabilities to external systems.*

---

#### Pattern D1: The DAG Orchestrator *(~18 pp.)*

**Intent.** Decompose a task too broad for a single agent run into a dependency graph of subtasks, verify the graph is acyclic, execute all independent subtasks concurrently in a thread pool, and assemble a cross-reference synthesis from the outputs.

**Also known as.** Multi-Subtask Orchestrator; Parallel Research Decomposition; Kahn's-Algorithm Coordinator.

**How it works.** `harness/orchestrator.py` receives a task and calls the planner for a `subtasks` decomposition. Before any execution begins, `_check_dag_cycles()` runs Kahn's topological sort algorithm and raises `OrchestratorError` if a cycle is detected — preventing deadlocks before threads are launched. Independent subtasks are dispatched to a `ThreadPoolExecutor` (default: 4 workers, bounded by VRAM). Subtasks whose task strings match a configured `HARNESS_MCP_ENDPOINTS` keyword are routed to remote harness instances via *The MCP Dispatch Router* (D3) rather than executed locally. After all futures complete, `_assemble()` passes all outputs to an assembly model that produces a unified cross-reference synthesis.

**Consequences.** Effective parallelism is bounded by `SUBTASK_MAX_WORKERS`, which should not exceed the number of models that can remain warm simultaneously given the VRAM budget; see *The Keep-Alive Budget* (A4). The assembly model is the most underspecified component: it must read N outputs and synthesize coherent cross-references, a task that degrades with N > 8 on models below 30B parameters. Subtask state isolation requires *The Worktree Context* (D2). See also: *The Worktree Context* (D2), *The MCP Dispatch Router* (D3), *The Planner-First* (B1).

---

#### Pattern D2: The Worktree Context *(~20 pp.)*  **← SAMPLE CHAPTER**

**Intent.** Provision each concurrent subtask with an isolated, checkpointable Git worktree before dispatch, eliminating shared-state race conditions between parallel agent loops without container overhead or process isolation.

**Also known as.** Worktree-per-Agent; Branched Execution Contexts; Git-Native State Isolation.

**How it works.** Before the orchestrator dispatches any subtask, it calls `git worktree add -b subtask/<id> .worktrees/<id> HEAD` to create a linked working tree on its own branch. Each subtask's agent loop runs with `cwd` set to its worktree root, so all file operations — output writes, memory database access, research cache reads — resolve into the isolated environment. Worktrees are removed in a `finally` block that runs regardless of what else goes wrong, with `--force` to handle uncommitted changes. Long-running subtasks can commit intermediate progress (`git commit -m "checkpoint: ..."`) inside their worktree; these commits persist in the shared object store even if the worktree is subsequently removed, enabling recovery from interrupted runs. Cleanup calls `git worktree prune` and `git branch -D subtask/<id>` to prevent branch accumulation.

**Consequences.** Eliminates the silent failure mode documented in Chapter 4 (shared output paths overwrite each other; assembly model receives fewer outputs than expected; no error is raised). Worktree creation is instantaneous; no additional software dependencies are required. Disk usage equals one working tree copy per concurrent subtask — acceptable for typical repository sizes, expensive for large-binary repositories. The pattern solves filesystem isolation; it does not solve coordination — subtasks cannot observe each other's intermediate findings. That problem motivates *The MCP Dispatch Router* (D3) and Chapter 22 on multi-instance orchestration. See also: *The DAG Orchestrator* (D1), *The RunTrace* (F1).

---

#### Pattern D3: The MCP Dispatch Router *(~15 pp.)*

**Intent.** Route subtasks to remote harness instances over HTTP using keyword-pattern matching, enabling cross-team and cross-machine orchestration without modifying the orchestrator's local execution model.

**Also known as.** Keyword-Routed Remote Dispatch; MCP Client Router; Cross-Instance Subtask Routing.

**How it works.** `harness/mcp_dispatch.py` reads `HARNESS_MCP_ENDPOINTS` — a JSON map of keyword patterns to HTTP endpoints — and performs case-insensitive substring matching of each subtask's task string against all configured keywords. The first match wins; the subtask is sent to that endpoint using the MCP protocol's `run_task` tool. The result (content string, `ok` flag, elapsed time) is returned to the orchestrator in the same format as a locally executed subtask, making remote and local execution interchangeable from the assembly step's perspective. A subtask with no matching keyword falls through to local execution.

**Consequences.** Enables horizontal scaling: security-focused subtasks route to a security team's harness instance with specialized models; financial analysis subtasks route to a finance team's instance. Matching is first-match-wins and case-insensitive; keyword conflicts between teams must be resolved by ordering entries in `HARNESS_MCP_ENDPOINTS`. Remote dispatch adds network round-trip latency (typically 100–500ms for LAN, 500ms–5s for WAN) on top of the remote execution time. Error handling is best-effort: a failed remote subtask is recorded in the `errors` dict passed to `_assemble()`, which notes the gap in its synthesis. See also: *The DAG Orchestrator* (D1), *The MCP Server* (D4).

---

#### Pattern D4: The Skill Registry *(~12 pp.)*

**Intent.** Extend the agent with callable slash-command skills that inject additional context, replace the standard agent loop, or add post-processing steps, registered in a central dict that maps `/command` strings to handler functions.

**Also known as.** Slash-Command Registry; Plugin Hook System; Pre/Post-Hook Skills.

**How it works.** `harness/cli.py` maintains a `_SKILLS` dict mapping slash-command prefixes (e.g., `/lit-review`, `/browser`, `/email`) to handler callables. When the CLI detects a `/`-prefixed task, it looks up the handler and calls it instead of the standard `agent.run()`. Each skill can implement pre-hooks (injecting domain-specific context before synthesis), replace the agent loop entirely (the `/browser` skill replaces research with Playwright navigation), or add post-processing (the `/email` skill adds a draft-review step before sending). Five built-in skills are documented in the implementation notes: `/lit-review` (7-stage academic literature review), `/browser` (LLM-guided ARIA navigation), `/email` (Gmail OAuth), `/github` (git + gh CLI automation), and `/deck` (PowerPoint generation from markdown).

**Consequences.** Adding a skill requires one entry in `_SKILLS` and one handler function; no modification of the agent loop is necessary. The distinction between high-leverage skills (`/lit-review`, `/browser`) and low-leverage ones (`/email`) — discussed in Chapter 21 — should inform which tasks are worth implementing as skills versus handled directly through the agent. Skills that replace the agent loop bypass *The Wiggum Loop* (C1) by default; skills that want evaluation must call `wiggum.loop()` explicitly. See also: *The DAG Orchestrator* (D1).

---

### Section E: Security Patterns *(~62 pp.)*

*Patterns that constrain what the agent can do to the host system and what external content can do to the agent's memory.*

---

#### Pattern E1: The AST Guard *(~12 pp.)*

**Intent.** Analyze agent-generated Python code at the syntax-tree level before any execution, blocking code that contains dangerous constructs without requiring sandboxing infrastructure.

**Also known as.** Static Code Guard; Syntax-Tree Code Scanner; Pre-Execution AST Check.

**How it works.** `harness/security.check_python_code(code)` parses the code string using Python's `ast` module (stdlib, no external dependencies) and walks the resulting tree looking for: `os.system` / `subprocess` calls, `__import__` of restricted modules, `open()` with write mode targeting paths outside the sandbox allowlist, `exec()` / `eval()` of dynamic strings, and network socket creation. If any construct is detected, the function returns `(safe=False, reason=str)` and logs a `"block"` severity event to `data/security_events.jsonl` without raising. Execution is prevented by the caller, not the guard itself, preserving the separation between detection and enforcement.

**Consequences.** Catches the most common classes of dangerous agent-generated code without running any code in a subprocess or container. Cannot catch obfuscated code (e.g., base64-encoded `exec` payloads) — that case is addressed by *The Injection Scanner* (E3). Does not sandbox already-running code; it only blocks code that has not yet been executed. False positives (blocking legitimate file writes) are configurable via the allowlist. See also: *The Path Sandbox* (E2), *The Injection Scanner* (E3).

---

#### Pattern E2: The Path Sandbox *(~12 pp.)*

**Intent.** Prevent file read operations from accessing sensitive paths or escaping the project directory by validating all file paths against an allowlist and a blocklist before any `open()` call.

**Also known as.** File Access Control; Canonical Path Guard; Directory Jail.

**How it works.** `harness/security.check_file_path(path)` resolves symlinks and `..` traversals to a canonical absolute path using `pathlib.Path.resolve()`, then checks the result against an allowlist (`data/`, `outputs/`, `skills/screenshots/`) and a blocklist (`.env`, `.ssh/`, `.aws/`, any filename matching `*credential*` or `*secret*`). Path traversal attacks like `../../etc/passwd` are caught at the resolve step, before any string comparison. The canonical path is what matters — not the input string. All file-reading operations in the harness call `check_file_path()` before opening.

**Consequences.** Requires that all legitimate file access paths be known in advance and registered in the allowlist. The blocklist uses suffix matching and is therefore susceptible to novel credential filenames not on the list; the allowlist is the stronger defense. The pattern does not protect against agent code that circumvents the harness's own file-reading functions (e.g., by importing `builtins.open` directly); that case requires *The AST Guard* (E1). See also: *The AST Guard* (E1), *The CDP Guard* (E4).

---

#### Pattern E3: The Injection Scanner *(~12 pp.)*

**Intent.** Detect prompt injection attempts in external content before it reaches the memory store, allowing the content to be logged and the memory write to be blocked, while permitting the content to be used in synthesis with a warning.

**Also known as.** Prompt Safety Gate; External Content Scanner; Injection Detection.

**How it works.** `harness/security.scan_for_injection(text, source)` applies a set of regex patterns against the content: phrases like "ignore previous instructions", role-switch commands ("you are now"), base64-encoded blobs above a length threshold (potential obfuscated payloads), excessive system-level formatting (large blocks of `---SYSTEM---` or `[INST]`), and Unicode confusable characters that disguise instructions. The function returns `(clean=bool, pattern=str)` and a severity. Content scanned from web sources receives `"warn"` severity — it is logged but synthesis is not blocked, because false-positive rates for web content are high. Memory writes from flagged content receive `"block"` severity — the observation is not written to the store — because injected content in the memory store has persistent effects across all future runs.

**Consequences.** The asymmetric severity (warn for synthesis, block for memory) reflects the asymmetric risk: a single injected synthesis prompt is contained to one run; an injected memory observation poisons all future runs that retrieve it. False positive rates for the injection patterns must be monitored via the `data/security_events.jsonl` analytics. The scanner does not provide cryptographic guarantees; it is a heuristic defense that raises the cost of injection attacks. See also: *The Dual-Backend Memory Store* (B3), *The AST Guard* (E1).

---

#### Pattern E4: The CDP Guard *(~10 pp.)*

**Intent.** Prevent the browser skill from navigating to localhost, private IP ranges, or local filesystem paths, blocking Server-Side Request Forgery attacks where a malicious page instructs the agent to fetch internal services.

**Also known as.** Browser Navigation Allowlist; SSRF Prevention; CDP URL Guard.

**How it works.** `harness/security.check_cdp_navigate(url)` parses the URL, resolves the hostname using `urllib.parse` and `ipaddress` (both stdlib), and blocks: `localhost` and `127.0.0.1` (direct SSRF), `file://` scheme (local filesystem access via browser), and RFC 1918 private IP ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`). The block is enforced before the Playwright navigation call; the page never loads. A `"block"` severity event is logged to `data/security_events.jsonl` with the blocked URL and the matched rule.

**Consequences.** Covers the primary SSRF attack surface for browser-capable agents. Does not block DNS rebinding attacks (where a legitimate hostname resolves to a private IP after the check passes); mitigating DNS rebinding requires resolving the hostname at check time and re-validating after navigation. The guard does not restrict which external URLs the browser can visit — only internal and local ones — so it is not a general-purpose content filter. See also: *The Path Sandbox* (E2), *The Skill Registry* (D4).

---

#### Pattern E5: The AIBOM *(~12 pp.)*

**Intent.** Maintain an AI Bill of Materials alongside the software SBOM, enumerating the model artifacts, custom Modelfiles, and cloud endpoints that constitute the other half of an agentic system's supply chain — the components that `pip freeze` cannot enumerate.

**Also known as.** AI Bill of Materials; Model Supply Chain Manifest; Model Provenance Log.

**How it works.** An agentic harness has three supply chain layers that a traditional SBOM covers only partially. The *software layer* (Python packages, system libraries) is fully covered by `pyproject.toml` lock files and PyPI hashes. The *runtime binary layer* (Ollama, llama.cpp, whisper.cpp, ffmpeg) is covered only for packages managed as git submodules; system-installed binaries appear nowhere in pip-managed dependency graphs. The *model layer* — local GGUF files, custom Modelfiles with baked-in system prompts, and cloud API endpoints — is entirely invisible to the software SBOM.

`AIBOM.md` is committed to the repository root alongside `SBOM.json` (generated by `pip-audit` or `cyclonedx-bom`). It is structured into four sections. *Local models* list each `ollama list` entry with its manifest SHA256 prefix, parameter count, and quantization level. *Custom Modelfiles* document each `FROM` base model, the quantization, and a truncated hash of the embedded system prompt — the system prompt hash is critical because a Modelfile is not just a model; it is a model plus instructions, and the instructions can change independently of the model weights. *Cloud endpoints* list each service name, provider, and last-verified date, with an explicit warning badge: cloud models cannot be content-hashed, have no pin mechanism, and may change behavior without notice. The harness's production AIBOM lists two cloud entries: `kimi-k2.5:cloud` (Moonshot AI), which is actively wired to the autoresearch Kimi unblock path and fires after 6 consecutive discards; and `glm-5.1:cloud` (Zhipu AI), which is registered in Ollama but not currently wired to a code path. The distinction between "active" and "registered-only" dependencies is itself part of the AIBOM's value: a registered-but-inactive endpoint is still a supply chain risk if it is activated without review. *Runtime binaries* list the git submodule SHA or system package version for each non-Python executable the harness calls.

The `scripts/generate_aibom.py` utility automates the local-model and Modelfile sections by calling `ollama list --json` and `ollama show --modelfile <name>` for each entry. Cloud endpoints and runtime binaries require manual maintenance. The script is run as a pre-commit hook so that new model entries trigger a review.

**Consequences.** The AIBOM makes model-layer supply chain changes visible in version control: adding a new GGUF, modifying a Modelfile's system prompt, or switching a cloud endpoint all produce a diff in `AIBOM.md` that can be reviewed and approved like any other code change. The cloud endpoint section is the weakest link — it provides naming and documentation but no cryptographic guarantee; the only real mitigation for cloud endpoint risk is the multi-backend design of *The Inference Shim* (A1), which allows a cloud endpoint to be removed from `HARNESS_ENDPOINTS` in a one-line config change. The AIBOM does not replace a traditional SBOM; both are necessary for a complete supply chain picture. See also: *The Inference Shim* (A1), *The AST Guard* (E1), *The JSONL Audit Log* (F2).

---

### Section F: Observability Patterns *(~35 pp.)*

*Patterns that make the pipeline's operation auditable, diagnosable, and analytically tractable.*

---

#### Pattern F1: The RunTrace *(~18 pp.)*

**Intent.** Accumulate all telemetry for a run — stage transitions, LLM messages with token counts, tool calls with novelty scores, Wiggum scores per round, and elapsed time per stage — in a Python context manager that finalizes into a structured record on exit.

**Also known as.** Run Context Manager; Structured Telemetry Accumulator; Pipeline Audit Record.

**How it works.** `harness/logger.RunTrace` is used with a `with` statement wrapping each run. `enter_stage(name)` marks stage transitions and records elapsed time for the preceding stage. `add_message(role, stage, content, tokens)` appends every LLM exchange. `record_tool_call(name, query, urls, result_preview, novelty_score)` captures each search or fetch. `finalize(status, content, duration, wiggum_scores)` writes the complete record to `data/runs.jsonl` as a single JSON line and emits a Chrome Trace Event format file to `data/task_logs/<run_id>.trace.json`. Run IDs are UTC timestamp plus UUID4 hex prefix, making them sortable and collision-resistant.

**Consequences.** The `runs.jsonl` record is the primary artifact for diagnosing pipeline failures (Chapter 3, Chapter 4) and for extracting training data (Section G patterns). The Chrome Trace file, loadable in `ui.perfetto.dev`, reveals which stage accounts for the most latency — a pattern in the flame graph where the planner block is wider than the synthesis block indicates that the planner model is not staying warm in VRAM; see *The Keep-Alive Budget* (A4). The context manager pattern guarantees that `finalize()` runs even if the run raises an exception, so partial run records are always written. See also: *The JSONL Audit Log* (F2).

---

#### Pattern F2: The JSONL Audit Log *(~12 pp.)*

**Intent.** Maintain an append-only log of every completed run as a newline-delimited JSON file, making the full pipeline history queryable with standard tooling (jq, pandas, DuckDB) and auditable without a database server.

**Also known as.** runs.jsonl; Append-Only Run Log; Structured Run History.

**How it works.** Every call to `RunTrace.finalize()` appends exactly one JSON object to `data/runs.jsonl`. The file is never rewritten; revision rounds overwrite the output file but produce a new `runs.jsonl` entry for the revised run. The record schema captures: `run_id`, `task`, `producer_model`, `evaluator_model`, `run_duration_s`, `input_tokens`, `output_tokens`, `wiggum_rounds`, `wiggum_scores`, `wiggum_dimensions`, `final` (PASS/FAIL), `tool_calls` (array of search records with novelty scores), and `output_path`. The `/api/runs` and `/api/data` endpoints serve paginated and aggregated views of this file to the dashboard.

**Consequences.** The plain JSONL format is the pattern's primary advantage: no database schema migrations, no server dependency, full portability. A 1,000-run log is typically under 20 MB and loads into pandas in under a second. The full-scan read pattern degrades at very high run counts; the dashboard caches aggregated views for 30 seconds. The `data/legacy_runs.jsonl` file merges historical records from previous schema versions at read time. See also: *The RunTrace* (F1), *The Data Flywheel* (G1).

---

#### Pattern F3: The Chrome Trace Exporter *(~5 pp.)*

**Intent.** Emit run timing data in Chrome Trace Event format so that per-stage latency can be visualized as a flame graph in Perfetto UI without additional tooling.

**Also known as.** Perfetto Export; Trace Event Emitter; Stage Timeline.

**How it works.** `RunTrace.finalize()` writes a `{"traceEvents": [...]}` JSON file alongside the `runs.jsonl` entry. Each event has `name` (stage or LLM call label), `ph` (phase: `B` for begin, `E` for end), `ts` (microsecond timestamp), and `pid`/`tid` (process and thread IDs for multi-threaded orchestration runs). The file is loadable in `ui.perfetto.dev` by drag-and-drop; no Perfetto installation required.

**Consequences.** Flame graph visualization is the fastest way to identify pipeline bottlenecks — it makes cold-start overhead, slow planner calls, and revision round latency immediately visible as visual anomalies. The export adds negligible overhead. The format is stable (Chrome Trace Events is a long-lived standard); the only maintenance risk is Perfetto UI API changes, which do not affect the file format. See also: *The RunTrace* (F1).

---

### Section G: Self-Improvement Patterns *(~55 pp.)*

*Patterns that close the loop between pipeline outputs and model training, turning every run into a contribution to the next model generation.*

---

#### Pattern G1: The Data Flywheel *(~20 pp.)*

**Intent.** Extract Supervised Fine-Tuning (SFT) and Direct Preference Optimization (DPO) datasets from `runs.jsonl`, using the harness's own Wiggum scores as quality labels and PASS/FAIL pairs as preference signals.

**Also known as.** Run-to-Dataset Extraction; Self-Labeling Training Pipeline; Preference Pair Extraction.

**How it works.** SFT extraction filters `runs.jsonl` for PASS runs above a score threshold, formats each as an instruction-completion pair (task string as instruction, final synthesis as completion), and writes a `.parquet` file compatible with Hugging Face `trl`'s `SFTTrainer`. DPO extraction finds tasks with both a PASS and a FAIL run (or a round-1 draft and a passing final revision for the same run), pairs them as `(chosen, rejected)` preference pairs, and appends the evaluator's dimensional feedback as a reward annotation. The chapter provides the complete extraction script and explains how the Wiggum Loop's revision rounds produce preference pairs automatically — the round-1 draft that scored 6.4 and the round-2 revision that scored 8.3 for the same task are a natural preference pair with no additional labeling required.

**Consequences.** The flywheel works because the harness produces quality labels — Wiggum scores — as a byproduct of normal operation. No human annotation pipeline is required for SFT data; human annotation is optional for DPO data (it improves calibration but is not required to start the flywheel). The quality of the SFT dataset depends directly on the calibration of the Wiggum evaluator; evaluator drift (see *The Evaluator Pool*, A3) degrades label quality. See also: *The RL Rollout* (G2), *The JSONL Audit Log* (F2).

---

#### Pattern G2: The RL Rollout *(~15 pp.)*

**Intent.** Use the Wiggum Loop as an online reward signal during training — generating candidate outputs, evaluating them with the rubric, and using the dimensional scores as a reward function — to produce RLHF-style training data without a separate reward model.

**Also known as.** Wiggum-as-Reward; Online Reward Rollout; Self-Evaluation RL.

**How it works.** During a rollout session, the harness generates N candidate outputs for a task using the producer model with varied temperature and sampling parameters. Each candidate is scored by the evaluator using *The Dimensional Rubric* (C2). The dimensional scores are aggregated into a scalar reward (weighted sum, with weights tuned by task type). Candidate pairs — high-reward vs. low-reward — are written as DPO preference pairs; the full reward-annotated dataset is written for PPO or GRPO training. The rollout session is configured via `HARNESS_ROLLOUT_N` (number of candidates per task) and `HARNESS_ROLLOUT_TASKS` (task list to roll out over). Fine-tuning on the resulting dataset using QLoRA is covered in Chapter 20.

**Consequences.** Eliminates the need for a separate reward model (the evaluator serves that role), at the cost of N × evaluator inference calls per rollout task. The evaluator's calibration becomes the training signal; evaluator errors become training errors. The fine-tuned model, when returned to production, generates higher-scoring outputs on average — which produces better-quality SFT data for the next fine-tuning round, completing the flywheel. The risk is reward hacking: if the fine-tuned model learns to satisfy the rubric rather than to actually improve output quality, scores inflate while true quality stagnates. Periodic holdout evaluation against human raters is the correct diagnostic. See also: *The Data Flywheel* (G1), *The Wiggum Loop* (C1), *The Dimensional Rubric* (C2).

---

#### Pattern G3: The Literature Review Pipeline *(~10 pp.)*

**Intent.** Chain arXiv fetch, Semantic Scholar enrichment, multi-persona curation, per-paper annotation, thematic clustering, and cross-cluster synthesis into a seven-stage skill that produces a peer-review-quality literature survey from a topic string and a date range.

**Also known as.** /lit-review Skill; Academic Survey Pipeline; Seven-Stage Literature Review.

**How it works.** Implemented in `harness/skills/lit_review_skill.py` (~880 LOC). Stage 1: fetch papers from arXiv by topic and date range (CSV output). Stage 2: enrich with Semantic Scholar citation counts and hub scores. Stage 3: curate using five reviewer personas (Pragmatist, Rigorist, Synthesizer, Contrarian, Newcomer) — each scores papers 1–5; papers with mean ≥ 3.5 and no score below 2 pass. Stage 4: annotate each passing paper with a structured summary (contributions, methods, limitations, connections). Stage 5: cluster annotations into 3–5 thematic groups using the producer model. Stage 6: summarize each cluster. Stage 7: synthesize a cross-cluster review with open research questions. The final review passes through *The Wiggum Loop* (C1).

**Consequences.** The multi-persona curation stage is the pipeline's most distinctive feature: it rejects papers that any persona scores below 2 (a single weak-link rule that prevents inclusion of clearly off-topic or methodologically unsound papers) while requiring broad consensus for inclusion (mean ≥ 3.5 across five independent perspectives). Runtime for a 50-paper review is typically 45–90 minutes on standard hardware. The pipeline is implemented as a *Skill Registry* (D4) entry and can be invoked as `oh /lit-review "speculative decoding" --after 2024-01 --max 50`. See also: *The Skill Registry* (D4), *The Wiggum Loop* (C1).

---

#### Pattern G4: The Convergence Detector *(~15 pp.)*

**Intent.** Detect when the autoresearch loop has entered an unproductive stall — due to a corrupted diagnostic, an inflated baseline, or a self-defeating ban list — and intervene automatically before continued experiments exhaust the optimization budget without progress.

**Also known as.** Autoresearch Health Monitor; Stall Detector; Loop Escape Circuit.

**How it works.** Analysis of a 90-experiment autoresearch run on task T_B ("Search for best practices for cost envelope management in production AI agents") revealed zero advances after experiment 25, with the baseline set at 8.740 — a 75th-percentile sample against a true expected composite of approximately 8.53, making the advancement threshold systematically unreachable. The run motivates three convergence-detection mechanisms now implemented in `autoresearch.py`.

*Mechanism 1: Plateau re-gather.* After 3 consecutive discards (`PLATEAU_DISCARDS = 3`), the loop re-gathers research context for the target task and resets the discard counter. The plateau is logged and the refreshed context is injected into the next proposal prompt. This addresses cases where the proposer is recycling ideas because its retrieved context has been exhausted, not because good instruction variants don't exist.

*Mechanism 2: Kimi unblock.* After 6 consecutive discards (`KIMI_STUCK_THRESHOLD = 6`), the loop calls the cloud-hosted `kimi-k2.5` model for a structural reframe of the optimization problem. The Kimi response is injected into the next proposal prompt as external guidance. This escalation path handles cases where the proposer is locally stuck — the plateau re-gather has run twice without producing an advance — and a perspective outside the local model family is needed. When the guidance goes stale (another 6 discards with no advance), it is cleared and re-requested.

*Mechanism 3: Cycling detection.* Within each Wiggum evaluation of an autoresearch candidate, if the composite score and all six dimensional scores match the previous round exactly, the inner loop exits early. This prevents the outer autoresearch loop from recording a "no improvement" result that is actually a measurement artifact of the evaluator producing identical output on repeated presentation of the same content.

**Consequences.** These three mechanisms address stalls caused by context exhaustion, local proposal-model saturation, and evaluation noise. They do not address baseline contamination — a corrupted baseline (too high due to a high-percentile single-run sample) requires manual re-estimation or a flag to reset `--eval-n` before restarting the run. The Kimi unblock path introduces a cloud dependency; see *The AIBOM* (E5) for how this dependency is declared and audited. Each mechanism's activation is logged to `data/autoresearch_log.jsonl` with the consecutive-discard count at trigger time. See also: *The RL Rollout* (G2), *The Data Flywheel* (G1), *The JSONL Audit Log* (F2), *The Wiggum Loop* (C1).

---

## Appendices *(~15 pp.)*

### Appendix A: Environment Variable Reference *(~8 pp.)*

Complete reference table for all harness environment variables — name, type, default value, and the pattern(s) that use it. Covers all inference (`OLLAMA_HOST`, `OLLAMA_KEEP_ALIVE`, `HARNESS_ENDPOINTS`), orchestration (`HARNESS_MCP_ENDPOINTS`, `SUBTASK_MAX_WORKERS`, `HARNESS_CHECKPOINT_SUBTASKS`), evaluation (`HARNESS_EVALUATOR_POOL`), research (`RESEARCH_CACHE`, `SEMANTIC_SCHOLAR_API_KEY`), security (`HARNESS_CDP_PORT`), and self-improvement (`HARNESS_ROLLOUT_N`, `HARNESS_ROLLOUT_TASKS`) variables.

### Appendix B: Hardware Sizing Guide *(~7 pp.)*

Recommended configurations for three deployment profiles — Minimal (CPU-only or 8 GB VRAM, sequential model loading), Standard (24 GB VRAM, all three model roles warm simultaneously), and Production (multi-GPU or dedicated inference server, vLLM for high-throughput) — with model recommendations, expected throughput figures, and the pattern combinations that are feasible at each tier.

---

*Estimated total: ~497 pages*

---

### Pattern Index (alphabetical)

| Pattern | Section | Page |
|---------|---------|------|
| The AIBOM | E5 | — |
| The AST Guard | E1 | — |
| The CDP Guard | E4 | — |
| The Chrome Trace Exporter | F3 | — |
| The Convergence Detector | G4 | — |
| The DAG Orchestrator | D1 | — |
| The Data Flywheel | G1 | — |
| The Dimensional Rubric | C2 | — |
| The Dual-Backend Memory Store | B3 | — |
| The Evaluator Pool | A3 | — |
| The Inference Shim | A1 | — |
| The Injection Scanner | E3 | — |
| The Judge Calibration Guard | C5 | — |
| The JSONL Audit Log | F2 | — |
| The Keep-Alive Budget | A4 | — |
| The Literature Review Pipeline | G3 | — |
| The MCP Dispatch Router | D3 | — |
| The Model Role Separation | A2 | — |
| The Novelty Gate | B2 | — |
| The Path Sandbox | E2 | — |
| The Planner-First | B1 | — |
| The ReAct Comparator | C4 | — |
| The RL Rollout | G2 | — |
| The RunTrace | F1 | — |
| The Semantic Chunker | B4 | — |
| The Skill Registry | D4 | — |
| The Surgical Compressor | C3 | — |
| The Vision Bridge | B5 | — |
| The Wiggum Loop | C1 | — |
| The Worktree Context | D2 | — |
