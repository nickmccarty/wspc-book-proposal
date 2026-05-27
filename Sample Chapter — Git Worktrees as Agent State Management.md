
# Chapter 14: Git Worktrees as Agent State Management

---

The first production implementation of parallel agent orchestration in this harness ran 43 logged tasks before a problem was detected — and only then because a spot-check revealed a discrepancy at run 61. Eighteen runs had passed evaluation with a hidden defect.

The failure is this: when two research agents working on related subtasks both derive their output filenames from a hash of the task string, the filenames collide. The second writer overwrites the first. The run log shows two completions. The filesystem shows one file.

The assembly model reads the available outputs and produces a document that looks, on the surface, like a complete multi-perspective synthesis. It is not. The evaluator scores it, and it passes — because the evaluator cannot detect the absence of content it never saw. The run goes into `runs.jsonl` as a PASS.

There is no crash. There is no warning. There is no indication that anything went wrong.

This is one of the few failure modes in the architecture that does not surface through the quality evaluation system, because the evaluator can only assess what is present, not what is missing. That makes it the most dangerous kind: silent, plausible-looking, and repeatable across every subsequent parallel run until the root cause is diagnosed.

The fix is forty lines of code. The problem is that you have to know to look for it. This chapter describes the pattern that makes the failure structurally impossible.

---

## The Pattern

**Intent.** Use Git worktrees to provide each concurrent subtask with an isolated, checkpointable, git-native filesystem environment during multi-agent orchestration, eliminating shared-state race conditions without introducing container overhead or abandoning the repository's history.

**Also known as.** Worktree-per-Agent; Branched Execution Contexts; Git-Native State Isolation.

**Motivation.** When the orchestrator decomposes a task into N parallel subtasks and dispatches them to a thread pool, each subtask runs a full agent loop: it writes intermediate files, produces an output document, appends observations to the memory store, and — in long-running variants — emits checkpoint records. Without isolation, any two subtasks working on related topics will:

1. Race on output filenames derived from similar task strings
2. Interleave observation writes to `data/memory.db` in ways that corrupt the memory store's novelty scores
3. Pollute each other's research caches when `RESEARCH_CACHE=1` is enabled

The naive solution is to assign each subtask a unique temp directory. This works for output file isolation but discards the property that makes the harness auditable: every artifact is under git's version control. A subtask that crashes partway through leaves no trace in the repository. A subtask that writes a checkpoint mid-run has no way to commit it. A debugging session that wants to inspect exactly what one subtask produced — separate from what the others produced — has no mechanism to do so.

Git worktrees solve all three problems at once. A worktree is an additional working tree attached to an existing repository: it has its own index and working directory, checks out its own branch, and shares the repository's object store. Creating one takes less than a second and requires no additional dependencies. Everything a subtask writes in its worktree is on its own branch, isolated from every other subtask, inspectable after the fact, and checkpointable via ordinary `git commit`.

**Applicability.** Use this pattern when:

- Two or more subtasks run concurrently and write to paths that could collide
- Long-running subtasks need to checkpoint progress so that a process restart does not lose all intermediate work
- Post-run debugging requires the ability to reconstruct exactly what any individual subtask wrote, independently of the assembly step
- Your deployment environment has git available (which it will, since the harness is a git repository)

**Do not use this pattern when:**

- All subtasks write to disjoint, statically known paths — in that case, simple path-prefixing is sufficient and cheaper
- The repository contains large binary files; each worktree duplicates the working tree, and large binaries make that expensive
- The orchestration spans machines (worktrees are per-host, per-repository) — for cross-machine dispatch, see Chapter 21 on MCP interoperability

---

## What Is a Git Worktree

Before examining the implementation, it is worth being precise about what a worktree is and is not, because it is commonly confused with a clone.

A `git clone` creates a full copy of the repository — the `.git` directory, all objects, all history. Two clones are independent repositories that happen to share a common ancestor. Changes in one are not visible in the other until you push and pull.

A `git worktree add` creates a linked working tree. The `.git` directory — the object store, the refs, the configuration — is shared. The working tree (the checked-out files) and the index (the staging area) are separate. Changes made in one worktree are not visible in another until committed, at which point they appear in the shared object store and can be seen by any worktree that fetches or checks out that branch.

```
project-root/
├── .git/                  ← shared by all worktrees
│   ├── objects/           ← one object store
│   ├── refs/              ← all branches visible to all worktrees
│   └── worktrees/         ← git's internal record of linked worktrees
│       ├── subtask-a7f2/
│       └── subtask-b3e1/
├── harness/               ← main worktree (HEAD branch)
├── data/
└── .worktrees/            ← where we put linked worktrees
    ├── subtask-a7f2/      ← its own working tree, on branch subtask/a7f2
    │   ├── harness/
    │   └── data/
    └── subtask-b3e1/      ← isolated; writes here do not affect a7f2
        ├── harness/
        └── data/
```

Creating a worktree:

```bash
git worktree add -b subtask/a7f2 .worktrees/subtask-a7f2 HEAD
```

This creates a new branch `subtask/a7f2` starting from `HEAD`, checks it out in `.worktrees/subtask-a7f2`, and registers the linked worktree with git. The branch is exclusively checked out in that worktree — git prevents the same branch from being checked out in two worktrees simultaneously, which enforces the isolation guarantee.

Removing it after use:

```bash
git worktree remove --force .worktrees/subtask-a7f2
git branch -d subtask/a7f2
git worktree prune
```

The `--force` flag is necessary when the worktree's working tree contains uncommitted changes. For cleanup after subtask completion, this is the right behavior: we want removal to succeed regardless of what the subtask left behind.

---

## Structure

```
Orchestrator
│
├── _check_dag_cycles(subtasks)          -- Kahn's algorithm; abort if cycles
│
├── _create_worktree(path, branch)       -- one per subtask, before dispatch
│
├── ThreadPoolExecutor(max_workers=4)
│   ├── _execute_subtask(s1, wt1, …)    ─┐
│   ├── _execute_subtask(s2, wt2, …)     ├── concurrent; isolated
│   ├── _execute_subtask(s3, wt3, …)     │   by worktree
│   └── _execute_subtask(s4, wt4, …)    ─┘
│
├── collect outputs from each worktree
│
├── _remove_worktree(path)               -- cleanup, in finally block
│
└── _assemble(task, outputs, model)      -- cross-reference synthesis
```

Each subtask's agent loop runs entirely inside its worktree. The worktree's `data/eval/` directory receives the subtask's output. The worktree's `data/memory.db` is the subtask's private memory store for the duration of the run. No file paths cross worktree boundaries during execution.

---

## Implementation

### Orchestrator Entry Point

```python
# harness/orchestrator.py

import subprocess
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor, as_completed

SUBTASK_MAX_WORKERS = int(os.getenv("SUBTASK_MAX_WORKERS", "4"))
WORKTREE_BASE       = Path(".worktrees")


def orchestrate(task: str, output_path: str, **kwargs) -> str:
    """
    Decompose task into subtasks, execute them concurrently in isolated
    worktrees, and assemble a unified synthesis from all outputs.
    """
    memory_context = memory.get_context(task)
    plan = planner.make_plan(task, memory_context)
    subtasks = plan.subtasks

    if not subtasks:
        # Single-agent path: no orchestration needed
        return agent.run(task, output_path, **kwargs)

    _check_dag_cycles(subtasks)

    # Phase 1: provision worktrees before any subtask starts
    worktrees: dict[str, Path] = {}
    for st in subtasks:
        branch  = f"subtask/{st.id}"
        wt_path = WORKTREE_BASE / st.id
        _create_worktree(wt_path, branch)
        worktrees[st.id] = wt_path

    # Phase 2: concurrent execution
    outputs: dict[str, str] = {}
    errors:  dict[str, str] = {}

    try:
        with ThreadPoolExecutor(max_workers=SUBTASK_MAX_WORKERS) as pool:
            futures = {
                pool.submit(
                    _execute_subtask,
                    subtask=st,
                    worktree=worktrees[st.id],
                    kwargs=kwargs,
                ): st
                for st in subtasks
            }
            for future in as_completed(futures):
                st = futures[future]
                try:
                    outputs[st.id] = future.result()
                except Exception as exc:
                    errors[st.id] = str(exc)
                    logger.warning(f"subtask {st.id} failed: {exc}")
    finally:
        # Phase 3: cleanup — always runs, even on KeyboardInterrupt
        for st_id, wt_path in worktrees.items():
            _remove_worktree(wt_path)
            _delete_branch(f"subtask/{st_id}")
        _prune_worktrees()

    if not outputs:
        raise RuntimeError(
            f"All {len(subtasks)} subtasks failed. "
            f"Errors: {errors}"
        )

    # Phase 4: assembly
    return _assemble(task, outputs, model=kwargs.get("model"), errors=errors)
```

The `finally` block is the most important implementation detail in the entire function. Worktree cleanup must happen regardless of what else goes wrong. An uncleaned worktree leaves an orphaned branch and a locked working tree that git will refuse to check out again. Over enough failed runs, orphaned worktrees accumulate and consume disk space. The `finally` block ensures this cannot happen — but it means the cleanup code cannot raise, which is why `_remove_worktree` uses `capture_output=True` rather than `check=True`.

### Worktree Lifecycle

```python
def _create_worktree(path: Path, branch: str) -> None:
    WORKTREE_BASE.mkdir(parents=True, exist_ok=True)
    subprocess.run(
        ["git", "worktree", "add", "-b", branch, str(path), "HEAD"],
        check=True,
        capture_output=True,
    )


def _remove_worktree(path: Path) -> None:
    """Best-effort removal. Logs failures but does not raise."""
    result = subprocess.run(
        ["git", "worktree", "remove", "--force", str(path)],
        capture_output=True,
    )
    if result.returncode != 0:
        logger.warning(
            f"worktree removal failed for {path}: "
            f"{result.stderr.decode().strip()}"
        )


def _delete_branch(branch: str) -> None:
    subprocess.run(
        ["git", "branch", "-D", branch],
        capture_output=True,  # branch may not exist if worktree creation failed
    )


def _prune_worktrees() -> None:
    subprocess.run(
        ["git", "worktree", "prune"],
        capture_output=True,
    )
```

### Subtask Execution

Each subtask runs a full agent loop rooted in its worktree:

```python
def _execute_subtask(
    subtask: Subtask,
    worktree: Path,
    kwargs: dict,
) -> str:
    """
    Run agent.run() inside the subtask's isolated worktree.
    Returns the output content string.
    """
    # Check whether this subtask should be routed to a remote MCP instance
    endpoint = mcp_dispatch.resolve_endpoint(subtask.task)
    if endpoint:
        result = mcp_dispatch.route_subtask(subtask.task, endpoint)
        return result["content"] if result["ok"] else ""

    # Local execution: run inside the worktree
    output_path = worktree / "data" / "eval" / f"{subtask.id}.md"
    output_path.parent.mkdir(parents=True, exist_ok=True)

    # Agent runs with its working directory set to the worktree root
    # This ensures all file operations land in the isolated environment
    content = agent.run(
        task=subtask.task,
        output_path=str(output_path),
        cwd=str(worktree),         # <-- the key parameter
        **kwargs,
    )

    # Optional: commit the output so it survives worktree removal
    # (useful for debugging; disabled by default)
    if os.getenv("HARNESS_CHECKPOINT_SUBTASKS"):
        _checkpoint(worktree, subtask)

    return content
```

The `cwd=str(worktree)` argument is what makes the isolation real. Every file operation in `agent.run()` — reading the memory database, writing the output, appending to the research cache — resolves relative to the worktree root, not the main working directory. Two subtasks running in different worktrees literally cannot touch the same files.

### Checkpointing

For subtasks that run more than a few minutes, checkpointing is valuable. A checkpoint is a git commit inside the worktree:

```python
def _checkpoint(worktree: Path, subtask: Subtask) -> None:
    """Commit the subtask's current state to its branch."""
    try:
        subprocess.run(
            ["git", "add", "-A"],
            cwd=str(worktree),
            check=True,
            capture_output=True,
        )
        subprocess.run(
            ["git", "commit", "-m",
             f"checkpoint: subtask/{subtask.id} intermediate output"],
            cwd=str(worktree),
            check=True,
            capture_output=True,
        )
    except subprocess.CalledProcessError:
        pass  # nothing staged; no checkpoint needed
```

A checkpoint written to the worktree branch persists even after the worktree is removed, because the commit object is in the shared `.git/objects/` store. If the orchestration process is interrupted before the finally block runs — a SIGKILL, a power failure, an OOM kill — the orphaned worktree branches are still accessible:

```bash
# Recover output from an interrupted subtask
git checkout subtask/a7f2
cat data/eval/a7f2.md
```

This makes debugging interrupted long-running orchestration runs substantially less painful than it would be with temp directories.

### Assembly

After all subtasks complete, the orchestrator assembles their outputs into a unified cross-reference synthesis:

```python
def _assemble(
    task: str,
    outputs: dict[str, str],
    model: str,
    errors: dict[str, str] | None = None,
) -> str:
    """
    Ask the assembly model to synthesize all subtask outputs into a
    unified cross-reference document.
    """
    if not outputs:
        raise RuntimeError("No subtask outputs to assemble.")

    outputs_text = "\n\n---\n\n".join(
        f"## Subtask {st_id}\n\n{content}"
        for st_id, content in outputs.items()
        if content.strip()
    )

    error_note = ""
    if errors:
        error_note = (
            f"\n\nNOTE: The following subtasks failed and are not "
            f"represented in the outputs above: "
            f"{list(errors.keys())}. Account for these gaps in your synthesis."
        )

    prompt = ASSEMBLY_PROMPT_TEMPLATE.format(
        task=task,
        outputs=outputs_text,
        error_note=error_note,
    )

    response = inference.chat(
        model=model or DEFAULT_ASSEMBLY_MODEL,
        messages=[{"role": "user", "content": prompt}],
        options={"temperature": 0.2},
    )

    return response.message.content
```

The assembly model reads all subtask outputs and produces a document that cross-references findings across subtasks, identifies where they agree and disagree, and integrates them into a coherent whole. This is a different cognitive task from the individual subtask's synthesis — it requires reading multiple documents and finding structure across them — which is why the assembly model prompt is distinct from the producer prompt used in the agent loop.

---

## Cycle Detection

Before any worktree is provisioned, the orchestrator verifies that the subtask dependency graph has no cycles. A cycle — subtask A depends on B, B depends on C, C depends on A — would cause a deadlock in the thread pool. The check uses Kahn's algorithm:

```python
def _check_dag_cycles(subtasks: list[Subtask]) -> None:
    """
    Raise OrchestratorError if the subtask dependency graph contains a cycle.
    Uses Kahn's topological sort algorithm.
    """
    # Build adjacency and in-degree maps
    in_degree = {st.id: 0 for st in subtasks}
    adj: dict[str, list[str]] = {st.id: [] for st in subtasks}

    for st in subtasks:
        for dep in st.depends_on:
            if dep not in adj:
                raise OrchestratorError(
                    f"Subtask {st.id} depends on unknown subtask {dep}"
                )
            adj[dep].append(st.id)
            in_degree[st.id] += 1

    queue = [sid for sid, deg in in_degree.items() if deg == 0]
    visited = 0

    while queue:
        node = queue.pop(0)
        visited += 1
        for neighbor in adj[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    if visited != len(subtasks):
        raise OrchestratorError(
            "Subtask dependency graph contains a cycle. "
            "Check the planner's subtask decomposition."
        )
```

This check is fast — linear in the number of subtasks — and runs before any worktree is created, so a bad decomposition fails immediately rather than partway through a multi-minute execution.

---

## What Worktrees Cannot Do

The pattern solves filesystem isolation completely. It does not solve coordination.

Consider a twelve-subtask orchestration where four subtasks run concurrently. Subtask A3 is surveying inference backends; subtask A7 is analyzing benchmark results for those backends. A3 finishes first and its output concludes, based on the evidence it retrieved, that llama.cpp is the optimal backend for the deployment profile under investigation. A7 is now running in its own worktree, unaware of A3's conclusion, and it retrieves benchmark data that leads it to the same conclusion — duplicated effort — or, worse, to a contradictory conclusion based on different sources.

Neither finding is wrong. They were derived from different search rounds, different sources, different search queries. The assembly model will synthesize them, note the agreement or flag the discrepancy, and produce a reasonable final document. But A7 spent eighteen minutes on a question that A3's output could have partially answered — or worse, it spent eighteen minutes building a case for a position that the full dataset contradicts.

The worktree pattern provides isolation. It provides no mechanism for running subtasks to communicate intermediate findings to each other. For that, you need something different: a message-passing layer that allows a completed subtask to publish its conclusions to a shared channel that running subtasks can query. In the harness architecture, this is the role of the MCP server — which exposes completed run results as a resource that any client, including a running subtask, can read.

But using the MCP server for mid-execution coordination introduces a new problem: the subtasks are no longer isolated. They are reading from a shared store of intermediate results, which means the order of completion affects the content of later outputs. If A3 finishes and A7 reads A3's conclusions and incorporates them, the orchestration is no longer a set of independent parallel computations — it is a sequential computation with partial parallelism, and its results are now sensitive to timing and execution order.

This is not a failure of the worktree pattern. It is the next problem up the stack, and it requires a different solution.

---

## Consequences

**What you gain:**

*Deterministic isolation.* Two subtasks writing to the same logical path write to different physical files. There are no race conditions on output files, memory databases, or research caches. The outputs you collect after execution are exactly the outputs each subtask wrote.

*Checkpointability.* Long-running subtasks can commit progress to their branch at any granularity. Interrupted runs leave inspectable branches in the repository. Resumption from a checkpoint is a `git checkout subtask/<id>` away.

*Auditability.* Even after cleanup, the `runs.jsonl` record for an orchestrated run includes each subtask's branch name and the commit hash of its final output (if checkpointing was enabled). You can reconstruct the exact state of any subtask at any point during execution.

*Zero additional dependencies.* Every harness deployment is already a git repository. The worktree pattern requires no additional software, no container runtime, no virtual filesystem, no message broker.

**What you give up:**

*Disk space.* Each worktree duplicates the working tree. A repository with 500 MB of working tree and eight concurrent subtasks will temporarily consume 4 GB of additional disk. For most deployments this is acceptable; for disk-constrained environments, limit `SUBTASK_MAX_WORKERS` accordingly.

*Branch hygiene.* Subtask branches accumulate in the repository. The `_delete_branch()` call in the finally block handles this for normal completions; interrupted runs leave orphaned branches. A `git branch --list "subtask/*"` and a periodic `git worktree prune && git branch -D $(git branch --list 'subtask/*')` are necessary maintenance operations in production deployments.

*Coordination transparency.* Because subtasks are isolated, the final document's coherence depends entirely on the assembly model. If the assembly model fails to integrate contradictory findings from different subtasks, the contradiction reaches the user. The quality evaluation system cannot catch this because the evaluator scores the assembled document against the original task — it cannot know that the contradiction came from independent subtasks that did not share information.

---

## Known Uses

The pattern is implemented in `harness/orchestrator.py` and activated whenever the planner's output includes a non-empty `subtasks` list. It has been used on orchestration runs of up to sixteen concurrent subtasks on a single host. The practical limit imposed by the harness is `SUBTASK_MAX_WORKERS=4`, not because of the worktree mechanism — which scales to the filesystem's inode limit — but because of VRAM: four concurrent agent loops, each holding a model warm in memory, saturate a 24 GB GPU.

---

## Related Patterns

**The DAG Orchestrator** (this chapter) — the pattern within which worktrees are provisioned and managed.

**MCP Interoperability** (Chapter 21) — the coordination layer that becomes necessary when isolated subtasks must share intermediate findings.

**RunTrace** (Chapter 17) — the observability context manager that records each subtask's branch name and output path, making the checkpoint system auditable.

**The Inference Shim** (Chapter 16) — the backend abstraction that allows each subtask's agent loop to use the same inference API regardless of which backend is serving the model, even when subtasks use different models.

---

## Summary

Git worktrees provide isolated, checkpointable, git-native working environments for concurrent subtasks without container overhead or process isolation. The pattern's implementation is forty lines of subprocess calls in `orchestrator.py` and its `finally` block is the most important production engineering decision in the orchestrator. The pattern solves the shared-state problem completely for filesystem isolation. It does not solve coordination — the problem of subtasks that would benefit from each other's intermediate findings — and that limitation is not a defect to be engineered away. It is the boundary between two distinct architectural problems, each with its own solution.

The next chapter examines what happens when the harness escapes the boundary of a single machine entirely: when subtasks are not parallel threads in the same process, but distributed agents running on different hosts, each with different models, different memory stores, and different security boundaries — and the orchestrator must coordinate them over HTTP using the same Model Context Protocol that allows Claude Code to treat the harness as a tool.

At that scale, filesystem isolation becomes the least of your problems.

---

**Code.** All code in this chapter is from `harness/orchestrator.py` and `harness/planner.py`. Environment variables are documented in Appendix B. The checkpoint mechanism is opt-in via `HARNESS_CHECKPOINT_SUBTASKS=1`.
