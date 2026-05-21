
# Author CV and Brief Bio

---

## Brief Bio *(for proposal cover page)*

**Nicholas McCarty** is Principal Consultant at Upskilled Consulting, where he designs and deploys production agentic systems for research-intensive organizations. He is the architect of the open-source Upskilled harness — a full-stack Python and TypeScript scaffolding framework for autonomous research agents running on local hardware — and the creator of the *Harness Engineering for AI Agents* course series. His work focuses on the infrastructure layer of AI systems: the pipeline scaffolding, evaluation loops, memory architectures, and observability tooling that determine whether a capable model delivers capable results. He is based in Sacramento, California.

---

## Curriculum Vitae

**Nicholas McCarty**  
Principal Consultant, Upskilled Consulting  
nick@upskilled.consulting  
https://upskilled.consulting  
2108 N St., Ste. N, Sacramento, CA 95816  

---

### Current Position

**Principal Consultant** — Upskilled Consulting *(2023 – present)*  
Designs and deploys agentic AI systems for research, knowledge management, and document synthesis workflows. Practice areas include harness architecture, LLM evaluation loop design, retrieval-augmented generation pipelines, and local inference infrastructure. Clients include research institutions, professional services firms, and technology companies implementing autonomous agent workflows.

---

### Open-Source Projects

**Upskilled Harness** *(2024 – present)*  
Author and primary maintainer of a full-stack agentic harness implementing 11 orchestrated subsystems — from CLI entry point through multi-model inference, novelty-gated research, evaluation loops, persistent memory, and structured observability. Written in Python and TypeScript. Runs entirely on local hardware; no external API keys required.

Key components authored:
- `harness/agent.py` — Core research agent: plan → search → synthesize → evaluate → persist
- `harness/wiggum.py` — Producer–evaluator separation loop with six-dimension rubric
- `harness/orchestrator.py` — DAG-based multi-subtask decomposition with parallel execution
- `harness/inference.py` — Unified inference shim routing to Ollama, vLLM, and llama.cpp
- `harness/memory.py` — Dual-backend memory store (ChromaDB + SQLite FTS5) with novelty gating
- `harness/mcp_server.py` — Model Context Protocol server exposing harness capabilities as tools
- `dashboard/` — React 18 / TypeScript live monitoring dashboard with WebSocket run streaming
- `harness/skills/` — Skills ecosystem: `/lit-review`, `/browser`, `/email`, `/github`, `/deck`

**Upskilled Platform** *(2023 – present)*  
Author of the Upskilled educational platform hosting free ML and AI courses for practitioners. Courses published include: *Harness Engineering for AI Agents* (5 modules, 12 hours estimated); mathematics prerequisites in linear algebra, matrix algebra, calculus, and probability; and domain supplements on neural network architectures, activation functions, loss functions, optimizers, regularization, and normalization.

---

### Teaching and Course Development

**Harness Engineering for AI Agents** — Upskilled Platform *(2025)*  
Five-module intermediate course covering pipeline architecture, context engineering and memory, verification and failure modes, production systems, and self-improvement. Includes readings, quizzes, and laboratory exercises. Published at upskilled.consulting/harness-engineering.

*Module 1: The Harness Thesis* — Pipeline architecture, experimental methodology, model role separation  
*Module 2: Context Engineering & Memory* — Planning, dual-search, saturation gating, persistent memory  
*Module 3: Verification & Failure Modes* — Wiggum loop, evaluation rubric, failure taxonomy  
*Module 4: Production Systems* — Skills hook system, orchestration, security layer, observability  
*Module 5: Self-Improvement* — Data flywheel, SFT/DPO dataset generation, QLoRA fine-tuning, literature review pipeline

---

### Selected Technical Areas

- **Agentic pipeline architecture:** Producer/evaluator separation, DAG-based orchestration, skill hook systems, MCP interoperability
- **Evaluation methodology:** LLM-as-a-Judge calibration, multi-dimension rubrics, evaluator pool rotation, failure taxonomy
- **Context engineering:** Novelty-gated search, dual-backend memory retrieval, large-document chunking, semantic search
- **Local inference infrastructure:** Ollama, vLLM, llama.cpp (GGUF), GGUF quantization, VRAM management, keep-alive tuning
- **Self-improving systems:** Data flywheel design, SFT and DPO dataset extraction from run logs, QLoRA fine-tuning
- **Security:** AST-level code scanning, path sandboxing, prompt injection detection, SSRF prevention in browser agents
- **Observability:** RunTrace telemetry, JSONL audit trails, Chrome Trace / Perfetto integration, real-time WebSocket dashboards
- **Languages and frameworks:** Python, TypeScript, React, FastAPI, SQLite FTS5, ChromaDB, Playwright

---

### Education

*(To be completed by author.)*

---

### Professional Affiliations and Recognitions

*(To be completed by author.)*

---

### Suggested Expert Reviewers

The following experts in the areas covered by this book have been identified as potential peer reviewers:

| Name | Affiliation | Email |
|------|-------------|-------|
| Peter Steinberger | OpenAI | peter@steipete.me |
| Philipp Schmid | Google DeepMind | schmidphillipp1995@gmail.com |
| Sebastian Raschka | Lightning AI | mail@sebastianraschka.com |
