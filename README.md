# GPU Task Runner Skill

一个轻量 GPU 批量任务调度 skill，配合其他 skill（如 `research[E]-coding`）使用。对话中已有任务背景时，自动探测 GPU 状态，计算并发方案，在已有脚本上追加调度逻辑。

## 使用方式

```
/GPU-task-runner
/GPU-task-runner [具体指令]
```

具体指令示例：
- `/GPU-task-runner 只用 GPU 0 和 1`
- `/GPU-task-runner 每个任务限制 8GB 显存`
- `/GPU-task-runner 生成脚本但不运行`

**优先级**：用户当前指令 > 对话历史信息 > skill 默认行为

## 功能

- **探测 GPU 状态**：通过 `nvidia-smi` 获取每张卡的利用率和空闲显存，跳过负载过高的卡
- **自动提取任务信息**：从对话历史复用已有的命令、脚本、环境，不重复询问
- **计算并发方案**：根据每任务显存需求和空闲显存，计算每张卡的并发槽数
- **追加调度块**：在已有 `.sh` 脚本末尾追加调度逻辑；无已有脚本时生成独立 `run_batch.sh`
- **日志重定向**：每个子任务的输出写入 `logs/YYYY-MM-DD_HH-MM-SS/<任务标识>.log`
- **多卡任务支持**：检测 `torchrun`/`deepspeed`/`accelerate` 命令，将连续空闲卡分组分配
- **直接运行**：默认追加完成后立即执行脚本；对话中明确要求"仅生成"时只输出脚本

## 输出示例

```
━━━━━━━━━━ GPU 调度方案 ━━━━━━━━━━
任务总数：24   并发槽：5   预计轮次：5

GPU  空闲显存   并发槽
 0   18.2 GB    2
 1   22.1 GB    2
 2   10.4 GB    1
 3   [跳过]      ← 利用率 91%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

已追加调度逻辑到 ./train.sh
运行：bash train.sh
日志目录：logs/2026-07-07_14-32-01/
```

## 安装

### Claude Code

```bash
mkdir -p ~/.claude/skills/GPU-task-runner
cp SKILL.md ~/.claude/skills/GPU-task-runner/SKILL.md
```

### Codex (OpenAI)

```bash
mkdir -p ~/.codex/skills/GPU-task-runner
cp SKILL.md ~/.codex/skills/GPU-task-runner/SKILL.md
```
