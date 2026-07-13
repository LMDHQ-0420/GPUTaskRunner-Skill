---
name: GPU-task-runner
description: >
  GPU batch task scheduler: probes live GPU state, extracts task info from conversation history, estimates MIN_CONCURRENCY, and appends a dynamic dispatch block to an existing shell script (or writes run_batch.sh).
version: 2.0.0
---

# GPU Task Runner

A lightweight scheduling skill meant to be used alongside other skills (e.g. `research[E]-coding`). The conversation already contains task context — this skill adds only the GPU dispatch layer.

## Priority Rules

**User's current instruction > conversation history > skill defaults**

All parameters, paths, and run modes follow the user's latest instruction first, then what was established in the conversation, then skill defaults as fallback.

## Invocation

```
/GPU-task-runner
/GPU-task-runner <inline instruction>
```

Inline instruction examples:
- `/GPU-task-runner use only GPU 0 and 1`
- `/GPU-task-runner cap VRAM per task at 8 GB`
- `/GPU-task-runner generate script only, do not run`

---

## Design Principles

- **Reuse first**: commands, scripts, environments, and data already in the conversation are used directly — never re-collected
- **Append preferred**: if an existing `.sh` script was produced in the conversation, append the dispatch block to it; otherwise write `run_batch.sh`
- **Single round of questions**: all missing info is collected in one message, never across multiple turns
- **Pure bash**: the dispatch layer uses only shell built-ins, `CUDA_VISIBLE_DEVICES`, and `wait`
- **Logs per task**: each subprocess's stdout/stderr is redirected to `logs/YYYY-MM-DD_HH-MM-SS/<task_id>.log`
- **No `conda run`**: when running the script directly, activate the environment first with `source activate <env>` or `source <venv>/bin/activate`

---

## Execution Flow

### Step 1: Parse inline instruction

If the user appended an instruction to the skill call, extract overrides first:
- specific GPU indices → restrict to those GPUs only
- VRAM cap → override inferred `PER_TASK_VRAM`
- `MIN_CONCURRENCY` value → skip estimation, use directly
- "generate only" / "do not run" → override default run behavior

### Step 2: Probe GPU state

```bash
nvidia-smi --query-gpu=index,utilization.gpu,memory.free,memory.total \
           --format=csv,noheader,nounits
```

Parse per GPU: index, util%, free VRAM (MiB), total VRAM (MiB).

If `nvidia-smi` is not available: abort with error, prompt user to check drivers.

### Step 3: Extract task info from conversation history

| Info | How to extract |
|------|---------------|
| Task type | Command contains `torchrun`/`deepspeed`/`accelerate`/`--nproc` → multi-GPU; otherwise single-GPU |
| GPUs per task | From `--nproc_per_node` / `--num_gpus` / `--num_processes` |
| VRAM per task (MiB) | Explicitly mentioned > inferred from model name/batch size/precision > add to missing items |
| Task list | File path (one item per line) / integer N / inline array |
| Command template | Existing command with `{item}` as the task parameter placeholder |
| Existing script path | Any `.sh` file produced earlier in the conversation |
| Python environment | conda env name or venv path |
| Run mode | "generate only" → do not execute; default → run immediately |

### Step 4: Collect missing info

If any required item is still unknown after Step 3, ask once in a single message listing only the missing items:

```
A few things needed to generate the dispatch script:

1. Command template for a single task? (use {item} as the task argument, e.g. python eval.py --input {item})
2. Task list? (file path / count N / inline list)
3. Estimated VRAM per task? (GB — a rough range is fine)
```

### Step 5: Estimate MIN_CONCURRENCY and confirm with user

```
for each gpu:
    usable_vram = free_vram × 0.85
    if usable_vram >= PER_TASK_VRAM and util% < 80:
        estimated_slots += floor(usable_vram / PER_TASK_VRAM)
```

Present the GPU summary table and estimated value, ask user to confirm or override:

```
━━━━━━━━━━ GPU Status ━━━━━━━━━━
GPU  Free VRAM   Util   Est. slots
 0   18.2 GB     12%    2
 1   22.1 GB      8%    2
 2   10.4 GB     45%    1
 3    2.1 GB     91%    0  (OOM risk)
─────────────────────────────────
Estimated MIN_CONCURRENCY: 5
Tasks in queue: 24  →  ~5 rounds

Confirm MIN_CONCURRENCY? [5]
```

### Step 6: Fill template and write script

Select template based on task type:
- Single-GPU tasks → `dispatcher_single.sh`
- Multi-GPU tasks → `dispatcher_multi.sh`

Fill in the config block: `TASKS`, `CMD_TMPL`, `MIN_CONCURRENCY`, `PER_TASK_VRAM`, `UTIL_THRESHOLD` (default 80), `LOG_DIR`.

Append the filled block to the existing script, or write `run_batch.sh` if no existing script was found.

### Step 7: Run or deliver

- **Run (default)**: activate the Python environment first, then execute:
  ```bash
  source activate <env>          # conda
  # or: source <venv>/bin/activate
  bash <script>
  ```
- **Generate only**: print the script path and the run command, do not execute.

---

## Edge Cases

| Situation | Handling |
|-----------|----------|
| No `nvidia-smi` | Abort, prompt to check drivers |
| All GPUs OOM | Allow `running < MIN_CONCURRENCY`; keep waiting for running tasks to finish, then re-scan |
| Multi-GPU: fewer than k free GPUs | Warn, wait for running tasks to free cards, then retry |
| No existing script in conversation | Write `run_batch.sh` |
| `wait -n` unsupported (bash < 4.3) | Fallback to plain `wait` (waits for all children) |
