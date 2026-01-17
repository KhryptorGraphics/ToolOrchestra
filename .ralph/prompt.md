# ToolOrchestra Deployment - Ralph Loop Instructions

## CRITICAL: READ FIRST

You are running in autonomous ralph-loop mode to complete ToolOrchestra deployment on Jetson Thor.
Your goal is to iterate through Beads tasks until all are complete and verified.

## Platform Configuration
- **Device**: NVIDIA Jetson Thor
- **JetPack**: 7.4 (R38.4.0)
- **CUDA**: 13.0
- **Driver**: 580.00
- **Python**: 3.12.12
- **Conda Environment**: toolorchestra

## CRITICAL: Jetson AI Lab PyPI Mirror

ALL CUDA packages MUST be installed using this index:
```bash
--index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/
```

## Available Pre-built Wheels
| Package | Version | Command |
|---------|---------|---------|
| torch | 2.9.1 | `pip install torch==2.9.1 --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/` |
| torchvision | 0.24.1 | `pip install torchvision==0.24.1 --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/` |
| torchaudio | 2.9.1 | `pip install torchaudio==2.9.1 --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/` |
| flash-attn | 2.8.4 | `pip install flash-attn==2.8.4 --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/` |
| vllm | 0.13.0+cu130 | `pip install vllm==0.13.0+cu130 --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/` |
| cupy | 13.6.0 | `pip install cupy==13.6.0 --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/` |

## Development Stack Integration

### Serena (Code Memory)
```bash
# Activate at start of each iteration
mcp__serena__activate_project with project="/home/kp/repo-nvidia/ToolOrchestra"
mcp__serena__switch_modes with modes=["editing", "interactive"]

# Read deployment patterns
mcp__serena__read_memory with memory_file_name="jetson-deployment.md"

# Write new learnings
mcp__serena__write_memory with memory_file_name="<filename>.md" content="..."
```

### Cipher (Conversation Memory)
```bash
# Query for past context
mcp__cipher__ask_cipher with message="What was the issue with..."

# Store new discoveries
mcp__cipher__ask_cipher with message="Store: <new information>"
```

### Beads (Task Tracking)
```bash
# Check available tasks
bd ready

# Claim a task
bd update <id> --status=in_progress

# Complete a task
bd close <id> --reason="Completed: <brief description>"

# Check task details
bd show <id>

# View all open tasks
bd list --status=open
```

## Iteration Protocol

FOR EACH ITERATION:

1. **Initialize**
   ```bash
   # Activate Serena
   mcp__serena__activate_project with project="/home/kp/repo-nvidia/ToolOrchestra"

   # Read deployment memory
   mcp__serena__read_memory with memory_file_name="jetson-deployment.md"
   ```

2. **Get Next Task**
   ```bash
   bd ready
   ```
   - If no ready tasks, check `bd blocked` for dependency issues
   - If all tasks closed, deployment is COMPLETE

3. **Claim Task**
   ```bash
   bd update <id> --status=in_progress
   ```

4. **Execute Task**
   - Run the appropriate commands based on task title
   - Always use `conda run -n toolorchestra pip install ...` for pip commands
   - Always use `--index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/` for CUDA packages

5. **Verify Task**
   - Run verification command for the task
   - If verification fails, try to fix and retry
   - If still failing, log to Cipher and progress.txt

6. **Complete Task**
   ```bash
   bd close <id> --reason="Completed: <verification result>"
   ```

7. **Log Progress**
   - Append any learnings to .ralph/progress.txt
   - Store important discoveries in Cipher

## Task-Specific Commands

### PyTorch Installation
```bash
conda run -n toolorchestra pip install torch==2.9.1 torchvision==0.24.1 torchaudio==2.9.1 \
  --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/
```
Verification:
```bash
conda run -n toolorchestra python -c "import torch; assert torch.cuda.is_available(); print(f'CUDA {torch.version.cuda}')"
```

### Flash Attention Installation
```bash
conda run -n toolorchestra pip uninstall -y flash-attn
conda run -n toolorchestra pip install flash-attn==2.8.4 \
  --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/
```
Verification:
```bash
conda run -n toolorchestra python -c "from flash_attn import flash_attn_func; print('Flash Attention OK')"
```

### vLLM Installation
```bash
conda run -n toolorchestra pip install vllm==0.13.0+cu130 \
  --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/
```
Verification:
```bash
conda run -n toolorchestra python -c "from vllm import LLM; print('vLLM OK')"
```

### Core ML Packages Installation
```bash
# CuPy with CUDA 13.0
conda run -n toolorchestra pip install cupy==13.6.0 \
  --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/

# Filtered requirements (exclude torch, flash-attn, vllm, cupy, langfuse, xformers, nvidia-*)
grep -v "^#" /home/kp/repo-nvidia/ToolOrchestra/requirements.txt | \
grep -v "^flash" | grep -v "^xformers" | grep -v "^langfuse" | \
grep -v "^vllm" | grep -v "^torch" | grep -v "^cupy" | grep -v "^nvidia-" | \
grep -v "^$" > /tmp/requirements-filtered.txt

conda run -n toolorchestra pip install -r /tmp/requirements-filtered.txt
```
Verification:
```bash
conda run -n toolorchestra python -c "import transformers, accelerate, datasets, ray, wandb, hydra; import cupy; print('Core ML OK')"
```

### Project Packages Installation
```bash
cd /home/kp/repo-nvidia/ToolOrchestra/evaluation/tau2-bench && conda run -n toolorchestra pip install -e .
cd /home/kp/repo-nvidia/ToolOrchestra/training/rollout && conda run -n toolorchestra pip install -e .
cd /home/kp/repo-nvidia/ToolOrchestra/training && conda run -n toolorchestra pip install -e .
```
Verification:
```bash
conda run -n toolorchestra python -c "import tau2, verl; print('Project packages OK')"
conda run -n toolorchestra tau2 --help
```

### Environment Variables Configuration
```bash
cat >> ~/.bashrc << 'EOF'

# ToolOrchestra Configuration
export HF_HOME="/home/kp/.cache/huggingface"
export REPO_PATH="/home/kp/repo-nvidia/ToolOrchestra"
export INDEX_DIR="/home/kp/repo-nvidia/ToolOrchestra/index"
export CKPT_DIR="/home/kp/repo-nvidia/ToolOrchestra/Nemotron-Orchestrator-8B"
export CUDA_HOME="/usr/local/cuda-13.0"

# ToolOrchestra Port Configuration (18000-18999 reserved range)
export TOOLORCHESTRA_VLLM_PORT=18000
export TOOLORCHESTRA_VLLM_PORT_SECONDARY=18001
export TOOLORCHESTRA_TAU2_API_PORT=18010
export TOOLORCHESTRA_TAU2_WS_PORT=18011
export TOOLORCHESTRA_RAY_DASHBOARD_PORT=18020
export TOOLORCHESTRA_WANDB_PORT=18030

# Port conflict check function
check_port() {
  if ss -tlnp 2>/dev/null | grep -q ":$1 "; then
    echo "Port $1 is IN USE"
    return 1
  else
    echo "Port $1 is available"
    return 0
  fi
}
EOF
source ~/.bashrc
```
Verification:
```bash
source ~/.bashrc && test -n "$CUDA_HOME" && test -d "$INDEX_DIR" && echo "Environment OK"
```

### Final Verification
```bash
conda run -n toolorchestra python << 'EOF'
import torch
assert torch.cuda.is_available(), "CUDA not available"
print(f"PyTorch {torch.__version__} CUDA {torch.version.cuda}")

from flash_attn import flash_attn_func
print("Flash Attention OK")

from vllm import LLM
print("vLLM OK")

import cupy as cp
print(f"CuPy CUDA {cp.cuda.runtime.runtimeGetVersion()}")

import transformers, accelerate, datasets, ray, wandb, hydra
print("Core ML OK")

import tau2, verl
print("Project packages OK")

print("\n SUCCESS: All components verified!")
EOF
```

## Self-Healing Strategies

If a step fails:

1. **Check Cipher for past solutions**
   ```bash
   mcp__cipher__ask_cipher with message="How was <error> resolved before?"
   ```

2. **Check Serena for platform patterns**
   ```bash
   mcp__serena__read_memory with memory_file_name="jetson-deployment.md"
   ```

3. **Try alternative approach**
   - For pip failures: try `--no-deps` or `--force-reinstall`
   - For import errors: check if dependency is missing

4. **Log and continue**
   - Store the error in Cipher
   - Append to progress.txt
   - Move to next task if blocked

## Completion Criteria

Deployment is COMPLETE when:
- [ ] `bd list --status=open` returns no tasks
- [ ] `bd stats` shows all tasks closed
- [ ] PyTorch CUDA 13.0 available
- [ ] Flash Attention functional
- [ ] vLLM inference working
- [ ] All project packages importable
- [ ] Environment variables set

When complete, update progress.txt with final status and summary.
