<p align="center">
  <img src="logo.png" alt="GPU Task Runner" width="180"/>
</p>

<h1 align="center">GPU Task Runner</h1>

<p align="center">
  A Claude / Codex skill that turns your conversation context into a ready-to-run GPU batch dispatch script — no config files, no boilerplate.
</p>

---

## What It Does

When you've been working with an AI coding skill and have a list of experiments or inference jobs to run, `/GPU-task-runner` steps in as the final mile. It reads everything already established in the conversation — the command, the task list, the model size, the environment — probes your live GPU state, and produces a shell script that dispatches all tasks dynamically across your available GPUs.

**You don't write a scheduler. You don't babysit GPU utilization. You just run the script.**

---

## Features

### Dynamic dispatch, not static batching
Tasks are assigned to GPUs only when a GPU slot opens up. Every time a task finishes, the dispatcher re-scans GPU state and fills idle capacity immediately. No fixed round-robin, no wasted idle time between rounds.

### Two-pass scheduling with OOM protection
- **Pass 1** — assigns tasks to GPUs sorted by utilization (lowest first). Skips any GPU where util% exceeds the threshold *or* where remaining VRAM would be insufficient.
- **Pass 2** — if running tasks fall below `MIN_CONCURRENCY` after pass 1, the util constraint is relaxed and tasks are pushed onto the least-loaded GPUs that can still safely accommodate them. VRAM safety is always enforced — a task is never assigned if it would OOM.

### Minimum concurrency guarantee
The skill estimates a sensible `MIN_CONCURRENCY` from your GPU state and asks you to confirm before writing the script. The dispatcher tries to keep at least that many tasks running at all times, except when all remaining GPUs would OOM.

### Single-GPU and multi-GPU task support
- `dispatcher_single.sh` — each task runs on one GPU (`CUDA_VISIBLE_DEVICES=<idx>`)
- `dispatcher_multi.sh` — each task spans *k* consecutive GPUs (`CUDA_VISIBLE_DEVICES=0,1,2,3 torchrun ...`), selected to minimize cross-NUMA traffic

### Per-task logs
Every subprocess's stdout and stderr is redirected to `logs/YYYY-MM-DD_HH-MM-SS/<task_id>.log`. Logs from a single run are grouped under one timestamped directory so reruns never overwrite previous results.

### Context-first, zero re-collection
The skill reads your conversation history before asking a single question. Command template, task list, VRAM estimate, conda/venv environment — anything already discussed is reused directly. Only genuinely missing information is requested, in one message, never across multiple turns.

### Priority rules
**User's current instruction > conversation history > skill defaults.**
Inline overrides like `use only GPU 2 and 3` or `cap VRAM at 10 GB` are applied before anything else.

---

## Usage

```
/GPU-task-runner
/GPU-task-runner <inline instruction>
```

Examples:
```
/GPU-task-runner
/GPU-task-runner use only GPU 0 and 1
/GPU-task-runner cap VRAM per task at 8 GB
/GPU-task-runner generate script only, do not run
/GPU-task-runner MIN_CONCURRENCY 3
```

---

## What the Skill Produces

1. A filled-in `dispatcher_single.sh` or `dispatcher_multi.sh`, appended to your existing run script (or written as `run_batch.sh` if none exists)
2. A GPU status summary with estimated concurrency
3. Optionally, immediate execution with environment activation — no `conda run`, just `source activate` then `bash <script>`

---

## Installation

### Claude Code

```bash
cp -r GPUTaskRunner ~/.claude/skills/
```

### Codex (OpenAI)

```bash
cp -r GPUTaskRunner ~/.codex/skills/
```

---

## Requirements

- `nvidia-smi` (NVIDIA driver installed)
- bash ≥ 4.3 (for `wait -n`; older versions fall back to `wait`)

---

## Repository Layout

```
GPUTaskRunner/
  SKILL.md                ← skill definition (Claude side)
  dispatcher_single.sh    ← single-GPU dispatch template
  dispatcher_multi.sh     ← multi-GPU dispatch template
README.md
logo.png
LICENSE
```
