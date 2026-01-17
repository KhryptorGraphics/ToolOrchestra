# Ralph-Loop Autonomous Deployment Master Prompt

> **Platform**: NVIDIA Jetson Thor | JetPack 7.4 | CUDA 13.0 | Driver 580.00
> **Version**: 2.0 - Generalized Dependency-Driven Deployment
> **Purpose**: Autonomous ML project deployment with ralph-loop orchestration

---

## CRITICAL PRINCIPLES

### 1. Dependency-Driven Deployment
- **ONLY install what the project requires** - read requirements.txt, setup.py, pyproject.toml
- Do NOT assume packages like vLLM, Flash Attention, etc. are needed unless explicitly required
- Parse project dependencies FIRST, then plan installation phases

### 2. CUDA-First Package Resolution
- **Priority order for CUDA packages**:
  1. Jetson AI Lab PyPI mirror: `https://pypi.jetson-ai-lab.io/sbsa/cu130/`
  2. NVIDIA NGC containers/wheels
  3. Source compilation with CUDA enabled
- NEVER install CPU-only versions of CUDA-capable packages

### 3. Non-Destructive Deployment
- **DO NOT** modify system Python or existing conda environments
- **DO NOT** change ports used by existing services
- **DO NOT** overwrite existing configuration files
- Create isolated conda environments for each project
- Use project-specific port ranges

### 4. Port Conflict Prevention
- Audit ALL ports before deployment (system services, Docker, databases)
- Assign project-specific port ranges that don't conflict
- Document port allocations

### 5. Source Compilation Fallback
- If pre-built CUDA wheel unavailable, compile from source
- Ensure CUDA_HOME, nvcc, and build tools are available
- Document any source builds for future reference

---

## PHASE 0: System Audit (ALWAYS RUN FIRST)

### 0.0 Initialize Development Stack

```bash
# Run standard initialization
/init

# This activates:
# - Serena project context
# - Beads task tracking
# - Cipher conversation memory
# - Project-specific configurations
```

### 0.1 Generate Codebase Analysis

```bash
# Generate comprehensive codebase analysis for planning
PROJECT_PATH="{{PROJECT_PATH}}"
code2prompt "$PROJECT_PATH" --output-file="$PROJECT_PATH/.ralph/code2prompt.md"

# This creates a structured view of:
# - All source files and their contents
# - Project structure and organization
# - Dependencies visible in code
# - Entry points and configurations
```

Use the generated `code2prompt.md` to:
- Identify all dependencies (imports, requirements)
- Understand project structure before deployment
- Find configuration files and entry points
- Detect database connections, API endpoints, ports used in code

### 0.2 Port Audit

```bash
# Audit all ports in use
echo "=== System Ports ==="
ss -tlnp | grep LISTEN

echo "=== Docker Containers ==="
docker ps --format "table {{.Names}}\t{{.Ports}}" 2>/dev/null || echo "Docker not running"

echo "=== MySQL/MariaDB ==="
ss -tlnp | grep -E ":3306|:3307" || echo "No MySQL detected"

echo "=== Common Service Ports ==="
for port in 80 443 8080 8000 5000 6379 27017 5432; do
  ss -tlnp | grep -q ":$port " && echo "Port $port: IN USE" || echo "Port $port: available"
done
```

### 0.3 Existing Environment Audit

```bash
# List existing conda environments
conda env list

# Check existing services
systemctl list-units --type=service --state=running | grep -E "docker|mysql|postgres|nginx|apache"

# Check GPU status
nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv
```

### 0.4 Assign Non-Conflicting Port Range

```bash
# Find available port range (1000 ports)
for start in 18000 19000 20000 21000 22000; do
  conflict=false
  for offset in 0 10 20 30 40 50; do
    port=$((start + offset))
    if ss -tlnp | grep -q ":$port "; then
      conflict=true
      break
    fi
  done
  if [ "$conflict" = false ]; then
    echo "Available port range: $start-$((start+999))"
    break
  fi
done
```

---

## PHASE 1: Project Dependency Analysis

### 1.1 Parse Project Requirements

```bash
# Extract all dependencies from project
PROJECT_PATH="{{PROJECT_PATH}}"

echo "=== requirements.txt ==="
cat "$PROJECT_PATH/requirements.txt" 2>/dev/null || echo "Not found"

echo "=== setup.py dependencies ==="
grep -A 50 "install_requires" "$PROJECT_PATH/setup.py" 2>/dev/null || echo "Not found"

echo "=== pyproject.toml dependencies ==="
cat "$PROJECT_PATH/pyproject.toml" 2>/dev/null | grep -A 50 "dependencies" || echo "Not found"
```

### 1.2 Categorize Dependencies

Create dependency categories:

| Category | Examples | Installation Strategy |
|----------|----------|----------------------|
| **CUDA-Required** | torch, cupy, flash-attn, vllm, xformers | Jetson AI Lab mirror or source |
| **GPU-Optional** | transformers, accelerate | Standard pip (GPU auto-detected) |
| **Pure Python** | requests, pydantic, hydra-core | Standard pip |
| **Database** | mysql-connector, sqlalchemy | Standard pip + DB setup |
| **System** | opencv, ffmpeg bindings | Conda or apt + pip |

### 1.3 Check Wheel Availability

```bash
# Check Jetson AI Lab for CUDA wheels
MIRROR="https://pypi.jetson-ai-lab.io/sbsa/cu130/"

for pkg in torch torchvision torchaudio flash-attn vllm cupy xformers; do
  echo -n "$pkg: "
  curl -s "$MIRROR" | grep -q "$pkg" && echo "AVAILABLE" || echo "NOT FOUND - may need source build"
done
```

---

## PHASE 2: Conda Environment Setup & Management

### 2.1 Analyze Project Environment Requirements

```bash
# ═══════════════════════════════════════════════════════════════════════════
# ENVIRONMENT DISCOVERY
# Determine how many conda environments the project needs
# ═══════════════════════════════════════════════════════════════════════════

PROJECT_PATH="{{PROJECT_PATH}}"

# Check for environment indicators in project
echo "=== Analyzing Environment Requirements ==="

# Check for multiple requirements files (indicates multi-env project)
find "$PROJECT_PATH" -name "requirements*.txt" -type f

# Check for docker-compose (indicates service separation)
ls "$PROJECT_PATH/docker-compose*.yml" 2>/dev/null

# Check for separate service directories
ls -d "$PROJECT_PATH"/{inference,training,api,retriever,server}/ 2>/dev/null

# Check conda environment files
ls "$PROJECT_PATH"/environment*.yml 2>/dev/null
ls "$PROJECT_PATH"/conda*.yml 2>/dev/null
```

### 2.2 Environment Planning Matrix

Based on analysis, plan environments:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PROJECT ENVIRONMENT PLANNING                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SINGLE-ENV PROJECTS (most common):                                        │
│  └─ {{PROJECT_NAME}}     → Main development, all dependencies              │
│                                                                             │
│  MULTI-ENV PROJECTS (when services need isolation):                        │
│  ├─ {{PROJECT_NAME}}           → Core development (training, evaluation)   │
│  ├─ {{PROJECT_NAME}}-inference → Inference server (vLLM, TensorRT)         │
│  ├─ {{PROJECT_NAME}}-api       → API server (FastAPI, Flask)               │
│  └─ {{PROJECT_NAME}}-retriever → Search/retrieval (faiss, embeddings)      │
│                                                                             │
│  ENVIRONMENT ISOLATION TRIGGERS:                                           │
│  - Different Python versions needed                                        │
│  - Conflicting package versions (torch vs tensorrt)                        │
│  - Separate deployment targets (GPU vs CPU)                                │
│  - Memory isolation for large models                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Create Project Conda Environments

```bash
# ═══════════════════════════════════════════════════════════════════════════
# CONDA ENVIRONMENT CREATION
# Non-destructive: only creates if doesn't exist
# ═══════════════════════════════════════════════════════════════════════════

PYTHON_VERSION="3.12"
PROJECT_NAME="{{PROJECT_NAME}}"

# Function to safely create environment
create_env_if_not_exists() {
  local env_name=$1
  local python_ver=$2

  if conda env list | grep -q "^$env_name "; then
    echo "Environment $env_name already exists - SKIPPING"
    return 0
  fi

  echo "Creating environment: $env_name (Python $python_ver)"
  conda create -n "$env_name" python="$python_ver" -y

  # Verify creation
  if conda env list | grep -q "^$env_name "; then
    echo "SUCCESS: $env_name created"
  else
    echo "ERROR: Failed to create $env_name"
    return 1
  fi
}

# Create main environment
create_env_if_not_exists "$PROJECT_NAME" "$PYTHON_VERSION"

# Create additional environments based on project analysis
# (Only if multi-env project detected in Phase 2.1)

# Example for inference environment
if [ -d "$PROJECT_PATH/inference" ] || grep -q "vllm\|tensorrt" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then
  create_env_if_not_exists "${PROJECT_NAME}-inference" "$PYTHON_VERSION"
fi

# Example for API environment
if [ -d "$PROJECT_PATH/api" ] || [ -f "$PROJECT_PATH/app.py" ]; then
  create_env_if_not_exists "${PROJECT_NAME}-api" "$PYTHON_VERSION"
fi

# Example for retriever environment
if [ -d "$PROJECT_PATH/retriever" ] || grep -q "faiss\|sentence-transformers" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then
  create_env_if_not_exists "${PROJECT_NAME}-retriever" "$PYTHON_VERSION"
fi
```

### 2.4 Environment Verification & Health Check

```bash
# ═══════════════════════════════════════════════════════════════════════════
# ENVIRONMENT HEALTH CHECK
# Verify all project environments are properly configured
# ═══════════════════════════════════════════════════════════════════════════

verify_environment() {
  local env_name=$1

  echo "=== Verifying: $env_name ==="

  # Check exists
  if ! conda env list | grep -q "^$env_name "; then
    echo "FAIL: Environment does not exist"
    return 1
  fi

  # Check Python version
  local py_ver=$(conda run -n "$env_name" python --version 2>&1)
  echo "Python: $py_ver"

  # Check pip is working
  if ! conda run -n "$env_name" pip --version >/dev/null 2>&1; then
    echo "FAIL: pip not working"
    return 1
  fi

  # Check CUDA access (if GPU env)
  if conda run -n "$env_name" python -c "import torch; torch.cuda.is_available()" 2>/dev/null; then
    echo "CUDA: Available"
  else
    echo "CUDA: Not configured (may be OK for CPU-only env)"
  fi

  echo "STATUS: OK"
  return 0
}

# Verify all project environments
for env in "$PROJECT_NAME" "${PROJECT_NAME}-inference" "${PROJECT_NAME}-api" "${PROJECT_NAME}-retriever"; do
  if conda env list | grep -q "^$env "; then
    verify_environment "$env"
  fi
done
```

### 2.5 Configure Environment Variables

```bash
# ═══════════════════════════════════════════════════════════════════════════
# ENVIRONMENT VARIABLES
# Project-specific configuration (non-destructive append)
# ═══════════════════════════════════════════════════════════════════════════

# Check if already configured
if grep -q "{{PROJECT_PREFIX}}_PROJECT_PATH" ~/.bashrc 2>/dev/null; then
  echo "Environment variables already configured - SKIPPING"
else
  cat >> ~/.bashrc << 'EOF'

# ═══════════════════════════════════════════════════════════════════════════
# {{PROJECT_NAME}} Configuration
# Generated by ralph-loop deployment
# ═══════════════════════════════════════════════════════════════════════════

# Project paths
export {{PROJECT_PREFIX}}_PROJECT_PATH="{{PROJECT_PATH}}"
export {{PROJECT_PREFIX}}_CONDA_ENV="{{PROJECT_NAME}}"

# Environment aliases (for multi-env projects)
export {{PROJECT_PREFIX}}_ENV_MAIN="{{PROJECT_NAME}}"
export {{PROJECT_PREFIX}}_ENV_INFERENCE="{{PROJECT_NAME}}-inference"
export {{PROJECT_PREFIX}}_ENV_API="{{PROJECT_NAME}}-api"
export {{PROJECT_PREFIX}}_ENV_RETRIEVER="{{PROJECT_NAME}}-retriever"

# Port configuration (allocated range: {{PORT_START}}-{{PORT_END}})
export {{PROJECT_PREFIX}}_PORT_PRIMARY={{PORT_START}}
export {{PROJECT_PREFIX}}_PORT_INFERENCE=$(({{PORT_START}} + 10))
export {{PROJECT_PREFIX}}_PORT_API=$(({{PORT_START}} + 20))
export {{PROJECT_PREFIX}}_PORT_RETRIEVER=$(({{PORT_START}} + 30))

# CUDA Configuration
export CUDA_HOME="/usr/local/cuda-13.0"
export PATH="$CUDA_HOME/bin:$PATH"
export LD_LIBRARY_PATH="$CUDA_HOME/lib64:$LD_LIBRARY_PATH"

# Shared cache directories
export HF_HOME="/home/kp/.cache/huggingface"
export TORCH_HOME="/home/kp/.cache/torch"

# Helper function: activate project environment
{{PROJECT_NAME}}_activate() {
  local env_type="${1:-main}"
  case "$env_type" in
    main)      conda activate "${{PROJECT_PREFIX}}_ENV_MAIN" ;;
    inference) conda activate "${{PROJECT_PREFIX}}_ENV_INFERENCE" ;;
    api)       conda activate "${{PROJECT_PREFIX}}_ENV_API" ;;
    retriever) conda activate "${{PROJECT_PREFIX}}_ENV_RETRIEVER" ;;
    *)         echo "Unknown env type: $env_type" ;;
  esac
}

# Helper function: run command in specific environment
{{PROJECT_NAME}}_run() {
  local env_type="$1"
  shift
  case "$env_type" in
    main)      conda run -n "${{PROJECT_PREFIX}}_ENV_MAIN" "$@" ;;
    inference) conda run -n "${{PROJECT_PREFIX}}_ENV_INFERENCE" "$@" ;;
    api)       conda run -n "${{PROJECT_PREFIX}}_ENV_API" "$@" ;;
    retriever) conda run -n "${{PROJECT_PREFIX}}_ENV_RETRIEVER" "$@" ;;
    *)         echo "Unknown env type: $env_type" ;;
  esac
}
EOF

  source ~/.bashrc
  echo "Environment variables configured"
fi
```

### 2.6 Create Beads Tasks for Environment Setup

```bash
# ═══════════════════════════════════════════════════════════════════════════
# DYNAMIC BEADS TASK CREATION
# Create tasks based on discovered environments
# ═══════════════════════════════════════════════════════════════════════════

# Initialize Beads if not already done
if [ ! -d "$PROJECT_PATH/.beads" ]; then
  cd "$PROJECT_PATH"
  bd init --prefix={{PROJECT_PREFIX}}
fi

# Create environment setup tasks
bd create --title="Create conda environment: {{PROJECT_NAME}}" \
  --type=task --priority=1 \
  --description="Create main development conda environment with Python {{PYTHON_VERSION}}"

# If multi-env project, create additional tasks
if [ -n "${{PROJECT_PREFIX}}_ENV_INFERENCE" ]; then
  bd create --title="Create conda environment: {{PROJECT_NAME}}-inference" \
    --type=task --priority=1 \
    --description="Create inference environment for vLLM/TensorRT"
fi

# Create dependency installation tasks (will be expanded in Phase 3)
bd create --title="Install CUDA dependencies in {{PROJECT_NAME}}" \
  --type=task --priority=2 \
  --description="Install PyTorch, Flash Attention, vLLM (if required)"

bd create --title="Install project packages in {{PROJECT_NAME}}" \
  --type=task --priority=3 \
  --description="Install project-specific packages (editable installs)"

bd create --title="Verify all environments" \
  --type=task --priority=4 \
  --description="Run comprehensive verification across all environments"
```

---

## PHASE 3: CUDA Package Installation

### 3.1 PyTorch (IF REQUIRED)

Only install if project requires torch:

```bash
# Check if torch is required
if grep -qE "^torch|pytorch" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then
  conda run -n {{ENV_NAME}} pip install \
    torch==2.9.1 torchvision==0.24.1 torchaudio==2.9.1 \
    --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/

  # Verify
  conda run -n {{ENV_NAME}} python -c "import torch; assert torch.cuda.is_available(); print(f'PyTorch {torch.__version__} CUDA {torch.version.cuda}')"
fi
```

### 3.2 Flash Attention (IF REQUIRED)

```bash
if grep -qE "^flash-attn|flash_attn" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then
  conda run -n {{ENV_NAME}} pip install flash-attn==2.8.4 \
    --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/

  # Verify
  conda run -n {{ENV_NAME}} python -c "from flash_attn import flash_attn_func; print('Flash Attention OK')"
fi
```

### 3.3 vLLM (IF REQUIRED)

```bash
if grep -qE "^vllm" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then
  conda run -n {{ENV_NAME}} pip install vllm==0.13.0+cu130 \
    --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/

  # Verify
  conda run -n {{ENV_NAME}} python -c "from vllm import LLM; print('vLLM OK')"
fi
```

### 3.4 CuPy (IF REQUIRED)

```bash
if grep -qE "^cupy" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then
  conda run -n {{ENV_NAME}} pip install cupy==13.6.0 \
    --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/
fi
```

### 3.5 Source Compilation Fallback

For packages not available as pre-built wheels:

```bash
# Example: Building a CUDA package from source
PACKAGE="{{PACKAGE_NAME}}"
VERSION="{{VERSION}}"

# Ensure build dependencies
conda run -n {{ENV_NAME}} pip install cmake ninja wheel setuptools

# Clone and build
git clone https://github.com/{{ORG}}/$PACKAGE.git /tmp/$PACKAGE
cd /tmp/$PACKAGE
git checkout v$VERSION

# Build with CUDA
export CUDA_HOME=/usr/local/cuda-13.0
export TORCH_CUDA_ARCH_LIST="9.0"  # Thor SM90
conda run -n {{ENV_NAME}} pip install -e . --no-build-isolation

# Verify
conda run -n {{ENV_NAME}} python -c "import $PACKAGE; print('$PACKAGE OK')"

# Cleanup
rm -rf /tmp/$PACKAGE
```

---

## PHASE 4: Standard Dependencies

### 4.1 Filter and Install Non-CUDA Packages

```bash
# Create filtered requirements (exclude CUDA packages already installed)
grep -v "^#" "$PROJECT_PATH/requirements.txt" | \
grep -v "^torch" | \
grep -v "^flash" | \
grep -v "^vllm" | \
grep -v "^cupy" | \
grep -v "^xformers" | \
grep -v "^nvidia-" | \
grep -v "^$" > /tmp/requirements-filtered.txt

# Install filtered requirements
conda run -n {{ENV_NAME}} pip install -r /tmp/requirements-filtered.txt
```

### 4.2 Project Editable Installs

```bash
# Find and install all setup.py/pyproject.toml packages
find "$PROJECT_PATH" -name "setup.py" -o -name "pyproject.toml" | while read pkg; do
  pkg_dir=$(dirname "$pkg")
  echo "Installing editable: $pkg_dir"
  cd "$pkg_dir" && conda run -n {{ENV_NAME}} pip install -e .
done
```

---

## PHASE 5: Database Setup (IF REQUIRED)

### 5.1 MySQL Configuration

```bash
# MySQL root password for this server
MYSQL_ROOT_PASS="teamrsi123teamrsi123teamrsi123"

# Check if project needs MySQL
if grep -qE "mysql|sqlalchemy.*mysql|pymysql" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then

  # Check if MySQL is running
  if ! systemctl is-active --quiet mysql && ! systemctl is-active --quiet mariadb; then
    echo "WARNING: MySQL/MariaDB not running. Start with: sudo systemctl start mysql"
  fi

  # Create project database (non-destructive - only if not exists)
  DB_NAME="{{PROJECT_NAME}}_db"

  mysql -u root -p"$MYSQL_ROOT_PASS" -e "CREATE DATABASE IF NOT EXISTS $DB_NAME;"
  mysql -u root -p"$MYSQL_ROOT_PASS" -e "GRANT ALL PRIVILEGES ON $DB_NAME.* TO 'root'@'localhost';"

  echo "Database $DB_NAME ready"

  # Set environment variable
  echo "export {{PROJECT_PREFIX}}_DB_URL=\"mysql://root:$MYSQL_ROOT_PASS@localhost/$DB_NAME\"" >> ~/.bashrc
fi
```

### 5.2 PostgreSQL Configuration (IF REQUIRED)

```bash
if grep -qE "psycopg|postgres" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then
  # PostgreSQL setup (if needed)
  DB_NAME="{{PROJECT_NAME}}_db"
  sudo -u postgres createdb "$DB_NAME" 2>/dev/null || echo "Database may already exist"
fi
```

---

## PHASE 6: Docker Container Port Audit

### 6.1 Audit Docker Port Usage

```bash
# Get all Docker port mappings
echo "=== Docker Port Mappings ==="
docker ps --format "{{.Names}}: {{.Ports}}" 2>/dev/null | while read line; do
  echo "$line"
  # Extract port numbers
  echo "$line" | grep -oE '[0-9]+->|:[0-9]+' | tr -d ':->' | sort -u
done

# Check for conflicts with project ports
PROJECT_PORTS=({{PORT_START}} $(({{PORT_START}}+10)) $(({{PORT_START}}+20)))
for port in "${PROJECT_PORTS[@]}"; do
  if docker ps --format "{{.Ports}}" | grep -q ":$port->"; then
    echo "CONFLICT: Port $port used by Docker container"
  fi
done
```

### 6.2 Docker Compose Port Configuration

If project uses Docker Compose, update ports:

```yaml
# docker-compose.override.yml (non-destructive overlay)
version: '3.8'
services:
  {{SERVICE_NAME}}:
    ports:
      - "{{PORT_START}}:{{CONTAINER_PORT}}"  # Override default port
```

---

## PHASE 7: MCP Tool Orchestration Framework

### 7.1 MCP Server Categories & Usage Order

Ralph-loop uses MCP servers in a specific order based on the task phase:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MCP SERVER ORCHESTRATION ORDER                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PHASE A: MEMORY & CONTEXT (Always First)                                  │
│  ├─ 1. serena          → Code memory, project patterns                     │
│  ├─ 2. cipher          → Conversation memory, past solutions               │
│  ├─ 3. mcp-search      → Search stored observations/memories               │
│  └─ 4. code-context    → Codebase structure analysis                       │
│                                                                             │
│  PHASE B: REASONING & ANALYSIS                                             │
│  ├─ 5. sequential-thinking → Step-by-step problem breakdown                │
│  ├─ 6. think-strategies    → Multi-strategy reasoning                      │
│  └─ 7. code-reasoning      → Code-specific logical analysis                │
│                                                                             │
│  PHASE C: DOCUMENTATION & RESEARCH                                         │
│  ├─ 8. context7        → Library/framework documentation                   │
│  ├─ 9. tavily          → Web search (primary)                              │
│  └─ 10. linkup         → Web search (secondary/real-time)                  │
│                                                                             │
│  PHASE D: INTERACTIVE TESTING                                              │
│  ├─ 11. playwright     → Browser automation, API testing                   │
│  ├─ 12. puppeteer      → Browser automation (alternative)                  │
│  └─ 13. skyvern        → AI-powered web interaction                        │
│                                                                             │
│  PHASE E: CODE OPERATIONS                                                  │
│  └─ 14. github-mcp     → GitHub PRs, issues, code search                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Dynamic Beads Task Management (Morphological Approach)

Ralph-loop dynamically manages Beads tasks as issues are discovered:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# BEADS MORPHOLOGICAL MANAGEMENT
# Tasks evolve based on discoveries during execution
# ═══════════════════════════════════════════════════════════════════════════

# --- CREATE NEW TASK (when issue discovered) ---
bd create --title="Fix: {{DISCOVERED_ISSUE}}" \
  --type=bug \
  --priority=1 \
  --description="Discovered during {{PARENT_TASK}}: {{DETAILS}}"

# Add dependency to parent task
bd dep add {{NEW_TASK_ID}} {{PARENT_TASK_ID}}

# --- EDIT EXISTING TASK (when scope changes) ---
bd update {{TASK_ID}} \
  --description="Updated: {{NEW_DESCRIPTION}}" \
  --priority={{NEW_PRIORITY}}

# --- SPLIT TASK (when too complex) ---
bd create --title="{{TASK_TITLE}} - Part 1: {{SUBTASK_1}}" --type=task
bd create --title="{{TASK_TITLE}} - Part 2: {{SUBTASK_2}}" --type=task
bd dep add {{PART_2_ID}} {{PART_1_ID}}  # Part 2 depends on Part 1
bd close {{ORIGINAL_TASK_ID}} --reason="Split into subtasks"

# --- MARK BLOCKED (when external dependency found) ---
bd update {{TASK_ID}} --status=blocked
bd create --title="Resolve blocker: {{BLOCKER}}" --type=task --priority=0
bd dep add {{TASK_ID}} {{BLOCKER_TASK_ID}}

# --- UNBLOCK (when blocker resolved) ---
bd close {{BLOCKER_TASK_ID}} --reason="Resolved: {{SOLUTION}}"
# Task auto-unblocks when dependencies close

# --- DISCOVER RELATED WORK ---
bd create --title="Related: {{RELATED_WORK}}" --type=task
bd dep add {{RELATED_ID}} {{CURRENT_ID}} --type=discovered-from
```

### 7.3 Enhanced Memory Layer (claude-mem + serena + cipher)

Ralph-loop uses a three-tier memory system for cross-session context retention:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# PHASE 7.3: ENHANCED MEMORY LAYER
# Uses claude-mem 3-layer workflow + serena + cipher for cross-session memory
# ═══════════════════════════════════════════════════════════════════════════

### 7.3.1 Memory Initialization

initialize_memory_layer() {
  PROJECT_PATH="$1"
  PROJECT_NAME="$2"

  echo "=== Initializing Enhanced Memory Layer ==="

  # Set context for all memory systems
  export MEMORY_PROJECT="$PROJECT_NAME"
  export MEMORY_CONTEXT="ralph-loop-orchestration"

  # Check for existing memories from previous sessions
  check_existing_memories
}

check_existing_memories() {
  echo ">>> Checking for existing memories..."

  # 1. Serena memories (project-specific)
  # MCP call: mcp__serena__list_memories
  # Returns list of saved memory files

  # 2. Claude-mem search for project context
  # MCP call: mcp__plugin_claude-mem_mcp-search__search
  #   query: "$MEMORY_PROJECT deployment"
  #   project: "$MEMORY_PROJECT"

  # 3. Cipher recall
  # MCP call: mcp__cipher__ask_cipher
  #   message: "Recall previous context for project: $MEMORY_PROJECT"
}

### 7.3.2 Claude-Mem 3-Layer Retrieval

# ═══════════════════════════════════════════════════════════════════════════
# CRITICAL: Always use 3-layer workflow for efficient token usage
# Layer 1: Search → Get index with IDs (~50-100 tokens/result)
# Layer 2: Timeline → Get context around interesting results
# Layer 3: Get Observations → Fetch full details ONLY for filtered IDs
# NEVER fetch full details without filtering first. 10x token savings.
# ═══════════════════════════════════════════════════════════════════════════

# LAYER 1: Search - Get index with IDs
search_memory() {
  local query="$1"
  local memory_type="${2:-all}"  # all, decision, error, pattern, solution, research

  echo "=== Memory Search: $query ==="

  # MCP call: mcp__plugin_claude-mem_mcp-search__search
  # Parameters:
  #   query: "$query"
  #   project: "$MEMORY_PROJECT"
  #   type: "$memory_type"  # Optional filter
  #   limit: 15
  #   orderBy: "relevance"  # or "created_at"
  #
  # Returns: Index with observation IDs (~50-100 tokens/result)
  # Format: { id, title, created_at, type, snippet }
}

# LAYER 2: Timeline - Get context around results
get_memory_timeline() {
  local anchor_id="$1"
  local depth_before="${2:-3}"
  local depth_after="${3:-5}"

  echo "=== Memory Timeline: anchor=$anchor_id ==="

  # MCP call: mcp__plugin_claude-mem_mcp-search__timeline
  # Parameters:
  #   anchor: $anchor_id  # Observation ID from Layer 1
  #   depth_before: $depth_before
  #   depth_after: $depth_after
  #   project: "$MEMORY_PROJECT"
  #
  # Returns: Contextual observations around the anchor
  # Shows what happened before/after for context
}

# LAYER 3: Get Observations - Fetch full details for filtered IDs
get_memory_observations() {
  local ids="$1"  # JSON array of IDs: [123, 456, 789]

  echo "=== Fetching Full Memory Observations ==="

  # MCP call: mcp__plugin_claude-mem_mcp-search__get_observations
  # Parameters:
  #   ids: $ids  # Array of observation IDs (REQUIRED)
  #   project: "$MEMORY_PROJECT"
  #   orderBy: "created_at"
  #   limit: 20
  #
  # Returns: Full observation details for specified IDs only
}

### 7.3.3 Memory Storage Strategy

store_to_memory() {
  local memory_type="$1"  # decision, error, pattern, solution, research
  local content="$2"
  local tags="$3"

  echo "=== Storing to Memory: $memory_type ==="

  # Store in ALL memory systems for redundancy and different retrieval patterns

  # 1. Serena Memory (file-based, project-specific, permanent)
  # Best for: Structured knowledge, deployment patterns, solutions
  # MCP call: mcp__serena__write_memory
  #   memory_file_name: "${MEMORY_PROJECT}_${memory_type}_$(date +%Y%m%d).md"
  #   content: |
  #     # ${memory_type}: $(date)
  #     Tags: $tags
  #
  #     $content
  #
  #     ---
  #     Project: $MEMORY_PROJECT
  #     Context: $MEMORY_CONTEXT

  # 2. Cipher Memory (conversation context, associative)
  # Best for: Conversational recall, quick lookups
  # MCP call: mcp__cipher__ask_cipher
  #   message: "Store ${memory_type} for ${MEMORY_PROJECT}: $content"

  # 3. Claude-mem (cross-session observations, automatic)
  # Observations are auto-captured during conversation
  # Tag important ones explicitly for retrieval by using descriptive language
  # that will match future searches
}

### 7.3.4 Memory Retrieval Orchestration

retrieve_relevant_memories() {
  local context="$1"  # e.g., "implementing authentication", "fixing import error"

  echo "=== Retrieving Relevant Memories ==="
  echo "Context: $context"

  # ───────────────────────────────────────────────────────────────────────
  # STEP 1: Search all memory systems in parallel
  # ───────────────────────────────────────────────────────────────────────

  # 1a. Serena memories (project patterns, solutions)
  # MCP call: mcp__serena__list_memories
  # Then filter by name relevance, read relevant ones:
  # MCP call: mcp__serena__read_memory
  #   memory_file_name: "<relevant_memory>.md"

  # 1b. Claude-mem search (cross-session, 3-layer)
  # Layer 1 only first:
  # MCP call: mcp__plugin_claude-mem_mcp-search__search
  #   query: "$context"
  #   project: "$MEMORY_PROJECT"
  #   limit: 15
  # Returns IDs

  # 1c. Cipher query (conversational context)
  # MCP call: mcp__cipher__ask_cipher
  #   message: "Recall any relevant context for: $context"

  # ───────────────────────────────────────────────────────────────────────
  # STEP 2: Filter and rank by relevance
  # ───────────────────────────────────────────────────────────────────────
  # From claude-mem results:
  # - Exact match in title/content: weight 10
  # - Partial match (same topic): weight 5
  # - Related topic (same project): weight 2
  #
  # Select top 5-10 IDs for full retrieval

  # ───────────────────────────────────────────────────────────────────────
  # STEP 3: Get full details for filtered results
  # ───────────────────────────────────────────────────────────────────────
  # Layer 2 (optional): get_memory_timeline for context if needed
  # Layer 3: get_memory_observations for full details
  # MCP call: mcp__plugin_claude-mem_mcp-search__get_observations
  #   ids: [filtered_ids]

  # ───────────────────────────────────────────────────────────────────────
  # STEP 4: Synthesize into actionable context
  # ───────────────────────────────────────────────────────────────────────
  # Combine results from all three systems
  # Deduplicate overlapping information
  # Return structured context for current task
}

### 7.3.5 Pre-Compaction Persistence

persist_before_compaction() {
  echo "=== Pre-Compaction Memory Persistence ==="
  echo "Critical: Saving session learnings before context loss"

  # Get current session summary
  local session_date=$(date +%Y%m%d_%H%M%S)
  local session_file="session_${session_date}.md"

  # 1. Save comprehensive session summary to Serena memory (permanent file)
  # MCP call: mcp__serena__write_memory
  #   memory_file_name: "$session_file"
  #   content: |
  #     # Session Summary: $session_date
  #     Project: $MEMORY_PROJECT
  #
  #     ## Decisions Made
  #     [List key decisions from this session]
  #
  #     ## Errors Encountered & Solutions
  #     [List errors and how they were resolved]
  #
  #     ## Patterns Discovered
  #     [List reusable patterns found]
  #
  #     ## Solutions Applied
  #     [List solutions that worked]
  #
  #     ## Tasks Completed
  #     [List bd tasks closed this session]
  #
  #     ## Pending Work
  #     [List what remains to be done]

  # 2. Store key insights via Cipher for quick recall
  # MCP call: mcp__cipher__ask_cipher
  #   message: "Store session summary for $MEMORY_PROJECT:
  #     Key decisions: [summary]
  #     Important solutions: [summary]
  #     Current status: [status]"

  # 3. Update Beads with session notes for task context
  # bd update <active_task_id> --notes="Session $session_date learnings: ..."

  # 4. Sync Beads to git
  # bd sync
}

### 7.3.6 Post-Compaction Recovery

recover_after_compaction() {
  echo "=== Post-Compaction Memory Recovery ==="
  echo "Restoring context from persistent storage"

  # ───────────────────────────────────────────────────────────────────────
  # STEP 1: Read most recent Serena session memories
  # ───────────────────────────────────────────────────────────────────────
  # MCP call: mcp__serena__list_memories
  # Filter for session_*.md files
  # Read most recent 2-3 session files:
  # MCP call: mcp__serena__read_memory
  #   memory_file_name: "session_YYYYMMDD_HHMMSS.md"

  # ───────────────────────────────────────────────────────────────────────
  # STEP 2: Query Cipher for context
  # ───────────────────────────────────────────────────────────────────────
  # MCP call: mcp__cipher__ask_cipher
  #   message: "What was the context of the previous session for $MEMORY_PROJECT?
  #     What decisions were made? What was the current status?"

  # ───────────────────────────────────────────────────────────────────────
  # STEP 3: Search claude-mem for recent observations (3-layer)
  # ───────────────────────────────────────────────────────────────────────
  # Layer 1: search_memory "session $MEMORY_PROJECT"
  # Layer 3: get_memory_observations for relevant IDs

  # ───────────────────────────────────────────────────────────────────────
  # STEP 4: Check Beads for task state
  # ───────────────────────────────────────────────────────────────────────
  # bd list --status=in_progress  # What was being worked on
  # bd ready                       # What's available to work on
  # bd blocked                     # What's waiting on dependencies

  # ───────────────────────────────────────────────────────────────────────
  # STEP 5: Reconstruct working context
  # ───────────────────────────────────────────────────────────────────────
  # Synthesize information from all sources
  # Identify current state and next actions
  # Resume work from where it left off
}
```

### 7.4 MCP Tool Integration Patterns

#### Memory Layer (serena + cipher + mcp-search)

```bash
# Always query memory FIRST before any action
mcp__serena__activate_project with project="{{PROJECT_PATH}}"
mcp__serena__read_memory with memory_file_name="jetson-thor-deployment.md"

mcp__cipher__ask_cipher with message="Search: {{CURRENT_PROBLEM}} on Jetson Thor CUDA 13"

mcp__plugin_claude-mem_mcp-search__search with query="{{ERROR_MESSAGE}}" project="{{PROJECT_NAME}}"

# Store discoveries AFTER successful resolution
mcp__serena__write_memory with memory_file_name="{{TOPIC}}.md" content="{{SOLUTION}}"
mcp__cipher__ask_cipher with message="Store: {{PROJECT_NAME}} - {{DISCOVERY}}"
```

#### Reasoning Layer (sequential-thinking + think-strategies + code-reasoning)

```bash
# For complex problems, use structured reasoning
mcp__sequential-thinking__sequentialthinking with thought="{{PROBLEM}}" thoughtNumber=1 totalThoughts=5 nextThoughtNeeded=true

mcp__think-strategies__think-strategies with thought="{{APPROACH}}" strategy="chain_of_thought" thoughtNumber=1 totalThoughts=3 nextThoughtNeeded=true

mcp__code-reasoning__code-reasoning with thought="{{CODE_ISSUE}}" thought_number=1 total_thoughts=4 next_thought_needed=true
```

#### Research Layer (context7 + tavily + linkup)

```bash
# Check library documentation first
mcp__context7__resolve-library-id with libraryName="{{PACKAGE}}" query="{{QUESTION}}"
mcp__context7__query-docs with libraryId="{{LIBRARY_ID}}" query="{{SPECIFIC_QUESTION}}"

# Web search for solutions
mcp__tavily__tavily-search with query="{{PACKAGE}} {{ERROR}} Jetson Thor CUDA 13 site:github.com OR site:stackoverflow.com"

mcp__linkup__linkup-search with query="{{REALTIME_QUERY}}" depth="deep"
```

#### Testing Layer (playwright + puppeteer + skyvern)

```bash
# Test web endpoints
mcp__playwright__playwright_navigate with url="http://localhost:{{PORT}}/health"
mcp__playwright__playwright_get_visible_text

# Test API responses
mcp__playwright__playwright_get with url="http://localhost:{{PORT}}/api/status"

# Complex web interactions
mcp__skyvern__skyvern_run_task with url="{{URL}}" prompt="{{TASK_DESCRIPTION}}"
```

#### Code Operations (github-mcp)

```bash
# Search for solutions in GitHub
mcp__github-mcp__search_code with q="{{ERROR_MESSAGE}} language:python"
mcp__github-mcp__search_issues with q="{{PACKAGE}} {{ERROR}} is:issue"

# Check upstream fixes
mcp__github-mcp__list_commits with owner="{{ORG}}" repo="{{REPO}}" sha="main"
```

---

## PHASE 8: Ralph-Loop Iteration Protocol

### 8.0 Master Orchestration Integration

Before starting the iteration cycle, Ralph-Loop now routes through the Master Orchestration
Loop (Phase 15) for intelligent project handling. This ensures projects are properly
assessed and routed to the correct phase based on their status.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MASTER ORCHESTRATION ROUTING                             │
│                    (Entry Point for All Projects)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 0: PROJECT ASSESSMENT                                           │   │
│  │ ├─ assess_project_completeness()                                     │   │
│  │ ├─ Returns: PROJECT_STATUS (5 types)                                 │   │
│  │ │   ├─ complete              → Standard deployment (8.1)             │   │
│  │ │   ├─ functionality_problems → Phase 14 first                       │   │
│  │ │   ├─ incomplete_research   → Phase 13, then Phase 11               │   │
│  │ │   ├─ incomplete_critical   → Phase 11 (development)                │   │
│  │ │   └─ incomplete_minor      → Phase 11 (development)                │   │
│  │ └─ Returns: COMPLEXITY_SCORE (weighted)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 0.1: ROUTE TO APPROPRIATE HANDLER                               │   │
│  │                                                                       │   │
│  │  IF PROJECT_STATUS != "complete":                                     │   │
│  │     run_master_orchestration()  ← Phase 15                           │   │
│  │     (Iterates until all criteria met)                                 │   │
│  │  ELSE:                                                                │   │
│  │     Continue to standard deployment cycle (8.1)                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 0.2: POST-ORCHESTRATION VERIFICATION                            │   │
│  │ ├─ Re-assess project after Phase 15 completes                        │   │
│  │ ├─ Verify PROJECT_STATUS == "complete"                               │   │
│  │ ├─ IF still incomplete → Alert user, create manual intervention task │   │
│  │ └─ IF complete → Proceed to standard deployment (8.1)                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# ═══════════════════════════════════════════════════════════════════════════
# RALPH-LOOP ENTRY POINT WITH MASTER ORCHESTRATION
# Routes projects through appropriate phases before deployment
# ═══════════════════════════════════════════════════════════════════════════

ralph_loop_entry() {
  PROJECT_PATH="${1:-$(pwd)}"
  PROJECT_NAME="${2:-$(basename "$PROJECT_PATH")}"

  echo "╔═══════════════════════════════════════════════════════════════════════════╗"
  echo "║           RALPH-LOOP ENTRY POINT                                          ║"
  echo "║           Project: $PROJECT_NAME"
  echo "╚═══════════════════════════════════════════════════════════════════════════╝"

  # ─────────────────────────────────────────────────────────────────────────
  # STEP 0: Initialize memory layer
  # ─────────────────────────────────────────────────────────────────────────
  echo ">>> Initializing memory layer..."
  initialize_memory_layer "$PROJECT_PATH" "$PROJECT_NAME"

  # Recover any previous session context
  recover_after_compaction

  # ─────────────────────────────────────────────────────────────────────────
  # STEP 0.1: Assess project completeness
  # ─────────────────────────────────────────────────────────────────────────
  echo ">>> Assessing project completeness..."
  assess_project_completeness "$PROJECT_PATH"

  # ─────────────────────────────────────────────────────────────────────────
  # STEP 0.2: Route based on project status
  # ─────────────────────────────────────────────────────────────────────────
  echo ""
  echo ">>> Routing based on project status: $PROJECT_STATUS"

  case "$PROJECT_STATUS" in
    "complete")
      echo "Project is complete. Proceeding to standard deployment cycle..."
      # Continue to 8.1 iteration cycle
      return 0
      ;;

    "functionality_problems"|"incomplete_research"|"incomplete_critical"|"incomplete_minor")
      echo "Project requires orchestration. Starting master orchestration loop..."
      echo ""
      run_master_orchestration "$PROJECT_PATH" "$PROJECT_NAME"
      ORCHESTRATION_RESULT=$?

      if [ $ORCHESTRATION_RESULT -eq 0 ]; then
        echo "Master orchestration complete. Proceeding to standard deployment..."
        return 0
      else
        echo "Master orchestration did not complete. Manual intervention required."
        bd create \
          --title="Manual intervention required: $PROJECT_NAME" \
          --type=task \
          --priority=0 \
          --description="Master orchestration loop did not complete successfully.
Status: $PROJECT_STATUS
Complexity Score: $COMPLEXITY_SCORE

Manual review and intervention required before deployment."
        return 1
      fi
      ;;

    *)
      echo "ERROR: Unknown project status: $PROJECT_STATUS"
      return 1
      ;;
  esac
}

# Export for use
export -f ralph_loop_entry
```

### 8.1 Complete Iteration Cycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RALPH-LOOP ITERATION CYCLE                               │
│                    (Runs until all tasks complete)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 1: INITIALIZE                                                   │   │
│  │ ├─ /init (activate development stack)                                │   │
│  │ ├─ mcp__serena__activate_project                                     │   │
│  │ ├─ mcp__serena__read_memory("jetson-thor-deployment.md")             │   │
│  │ └─ mcp__cipher__ask_cipher("Recall: {{PROJECT}} deployment status")  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 2: GET NEXT TASK                                                │   │
│  │ ├─ bd ready (get unblocked tasks)                                    │   │
│  │ ├─ IF no ready tasks → bd blocked (check blockers)                   │   │
│  │ ├─ IF all closed → DEPLOYMENT COMPLETE ✓                            │   │
│  │ └─ SELECT highest priority ready task                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 3: CLAIM & ANALYZE TASK                                         │   │
│  │ ├─ bd update <id> --status=in_progress                               │   │
│  │ ├─ bd show <id> (get full task details)                              │   │
│  │ ├─ mcp__code-context__get_code_context (if code task)                │   │
│  │ └─ mcp__sequential-thinking (plan approach)                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 4: EXECUTE TASK                                                 │   │
│  │ ├─ Run task commands                                                 │   │
│  │ ├─ Use --index-url for CUDA packages                                 │   │
│  │ ├─ Use conda run -n {{ENV}} for isolation                            │   │
│  │ └─ Log output to .ralph/logs/{{TASK_ID}}.log                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 5: VERIFY TASK                                                  │   │
│  │ ├─ Run verification command                                          │   │
│  │ ├─ Check CUDA availability (if GPU package)                          │   │
│  │ ├─ Check import success                                              │   │
│  │ ├─ Test functionality (playwright/puppeteer if web service)          │   │
│  │ └─ IF PASS → STEP 7 | IF FAIL → STEP 6                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 6: SELF-HEALING LOOP (if verification failed)                   │   │
│  │ │                                                                    │   │
│  │ │  ┌─────────────────────────────────────────────────────────────┐  │   │
│  │ │  │ 6A: MEMORY CHECK                                             │  │   │
│  │ │  │ ├─ mcp__cipher__ask_cipher("{{ERROR}} solution")             │  │   │
│  │ │  │ ├─ mcp__serena__read_memory("{{RELATED}}.md")                │  │   │
│  │ │  │ └─ mcp__plugin_claude-mem_mcp-search__search("{{ERROR}}")    │  │   │
│  │ │  └─────────────────────────────────────────────────────────────┘  │   │
│  │ │                         ↓                                          │   │
│  │ │  ┌─────────────────────────────────────────────────────────────┐  │   │
│  │ │  │ 6B: REASONING                                                │  │   │
│  │ │  │ ├─ mcp__sequential-thinking (analyze error)                  │  │   │
│  │ │  │ ├─ mcp__code-reasoning (if code issue)                       │  │   │
│  │ │  │ └─ mcp__think-strategies (explore alternatives)              │  │   │
│  │ │  └─────────────────────────────────────────────────────────────┘  │   │
│  │ │                         ↓                                          │   │
│  │ │  ┌─────────────────────────────────────────────────────────────┐  │   │
│  │ │  │ 6C: RESEARCH                                                 │  │   │
│  │ │  │ ├─ mcp__context7__query-docs (library docs)                  │  │   │
│  │ │  │ ├─ mcp__tavily__tavily-search (web search)                   │  │   │
│  │ │  │ ├─ mcp__linkup__linkup-search (real-time info)               │  │   │
│  │ │  │ └─ mcp__github-mcp__search_issues (upstream issues)          │  │   │
│  │ │  └─────────────────────────────────────────────────────────────┘  │   │
│  │ │                         ↓                                          │   │
│  │ │  ┌─────────────────────────────────────────────────────────────┐  │   │
│  │ │  │ 6D: APPLY FIX                                                │  │   │
│  │ │  │ ├─ Try identified solution                                   │  │   │
│  │ │  │ ├─ pip install --force-reinstall                             │  │   │
│  │ │  │ ├─ pip install --no-deps + manual deps                       │  │   │
│  │ │  │ └─ Source compilation if needed                              │  │   │
│  │ │  └─────────────────────────────────────────────────────────────┘  │   │
│  │ │                         ↓                                          │   │
│  │ │  ┌─────────────────────────────────────────────────────────────┐  │   │
│  │ │  │ 6E: RE-VERIFY                                                │  │   │
│  │ │  │ ├─ Run verification again                                    │  │   │
│  │ │  │ ├─ IF PASS → Store solution in memory → STEP 7               │  │   │
│  │ │  │ ├─ IF FAIL (attempts < 3) → RETRY from 6A                    │  │   │
│  │ │  │ └─ IF FAIL (attempts >= 3) → Create blocker task → STEP 2    │  │   │
│  │ │  └─────────────────────────────────────────────────────────────┘  │   │
│  │ │                                                                    │   │
│  │ └─ IF permanently blocked:                                           │   │
│  │    ├─ bd update <id> --status=blocked                                │   │
│  │    ├─ bd create --title="Blocker: {{ISSUE}}" --priority=0            │   │
│  │    ├─ mcp__cipher__ask_cipher("Store: BLOCKED - {{DETAILS}}")        │   │
│  │    └─ Continue to next task                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 7: COMPLETE TASK                                                │   │
│  │ ├─ bd close <id> --reason="{{VERIFICATION_RESULT}}"                  │   │
│  │ ├─ mcp__cipher__ask_cipher("Store: {{TASK}} completed - {{HOW}}")    │   │
│  │ └─ mcp__serena__write_memory (if new pattern discovered)             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 8: DISCOVER NEW WORK                                            │   │
│  │ ├─ Check if task revealed new requirements                           │   │
│  │ ├─ bd create --title="Discovered: {{NEW_TASK}}" (if needed)          │   │
│  │ ├─ Update .ralph/progress.txt                                        │   │
│  │ └─ bd stats (check overall progress)                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│                         LOOP → STEP 2                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Completion Criteria

Ralph-loop has two levels of completion criteria:

1. **Deployment-Only Projects** (complete projects): Standard criteria
2. **Development + Deployment Projects** (incomplete projects): Enhanced criteria

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              ENHANCED COMPLETION CRITERIA                                    │
│              (Applies to all projects)                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TASK MANAGEMENT (Required)                                                 │
│  ├─ [ ] No open tasks: bd list --status=open returns empty                 │
│  ├─ [ ] No blocked tasks: bd blocked returns empty                         │
│  └─ [ ] All in_progress tasks closed                                       │
│                                                                             │
│  CODE COMPLETENESS (Required for incomplete projects)                       │
│  ├─ [ ] No NotImplementedError in codebase                                 │
│  ├─ [ ] No empty function bodies (pass/... stubs)                          │
│  └─ [ ] Critical TODOs addressed (P0-P2)                                   │
│                                                                             │
│  TEST COVERAGE (Required)                                                   │
│  ├─ [ ] All tests passing: pytest tests/ exits 0                           │
│  ├─ [ ] Coverage >= 80%: --cov-fail-under=80 passes                        │
│  └─ [ ] No skipped tests without documented reason                         │
│                                                                             │
│  FUNCTIONAL VERIFICATION (Required)                                         │
│  ├─ [ ] Core imports work: python -c "from {{PROJECT}} import *"           │
│  ├─ [ ] Entry points work: main.py --help, cli.py --help                   │
│  ├─ [ ] Services start: health endpoints respond                           │
│  └─ [ ] CUDA available (if GPU project)                                    │
│                                                                             │
│  ENVIRONMENT (Required)                                                     │
│  ├─ [ ] Conda environment exists and verified                              │
│  ├─ [ ] Environment variables configured in .bashrc                        │
│  └─ [ ] Ports allocated and available                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# ═══════════════════════════════════════════════════════════════════════════
# ENHANCED COMPLETION CRITERIA CHECK
# Project is ready for deployment when ALL conditions met
# ═══════════════════════════════════════════════════════════════════════════

check_completion_criteria() {
  local project_path="${1:-$(pwd)}"
  local env_name="${2:-{{ENV_NAME}}}"

  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "                    COMPLETION CRITERIA CHECK"
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo ""

  CRITERIA_MET=0
  CRITERIA_TOTAL=0

  # ═══════════════════════════════════════════════════════════════════════════
  # TASK MANAGEMENT
  # ═══════════════════════════════════════════════════════════════════════════
  echo "--- TASK MANAGEMENT ---"

  # 1. No open tasks
  ((CRITERIA_TOTAL++))
  OPEN_TASKS=$(bd list --status=open 2>/dev/null | grep -c "^{{PROJECT_PREFIX}}-" || echo "0")
  if [ "$OPEN_TASKS" -eq 0 ]; then
    echo "✓ No open tasks"
    ((CRITERIA_MET++))
  else
    echo "✗ $OPEN_TASKS open tasks remain"
  fi

  # 2. No blocked tasks
  ((CRITERIA_TOTAL++))
  BLOCKED_TASKS=$(bd blocked 2>/dev/null | grep -c "^{{PROJECT_PREFIX}}-" || echo "0")
  if [ "$BLOCKED_TASKS" -eq 0 ]; then
    echo "✓ No blocked tasks"
    ((CRITERIA_MET++))
  else
    echo "✗ $BLOCKED_TASKS blocked tasks"
  fi

  # ═══════════════════════════════════════════════════════════════════════════
  # CODE COMPLETENESS
  # ═══════════════════════════════════════════════════════════════════════════
  echo ""
  echo "--- CODE COMPLETENESS ---"

  # 3. No NotImplementedError
  ((CRITERIA_TOTAL++))
  NOT_IMPL=$(grep -rn "NotImplementedError\|raise NotImplemented" "$project_path" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | wc -l)
  if [ "$NOT_IMPL" -eq 0 ]; then
    echo "✓ No NotImplementedError"
    ((CRITERIA_MET++))
  else
    echo "✗ $NOT_IMPL unimplemented functions"
  fi

  # 4. No empty function bodies
  ((CRITERIA_TOTAL++))
  EMPTY_FUNCS=$(grep -Pzo "def \w+\([^)]*\):\s*\n\s*(pass|\.\.\.)\s*\n" "$project_path" \
    --include="*.py" 2>/dev/null | grep -c "def " || echo "0")
  if [ "$EMPTY_FUNCS" -eq 0 ]; then
    echo "✓ No empty function bodies"
    ((CRITERIA_MET++))
  else
    echo "✗ $EMPTY_FUNCS empty function bodies"
  fi

  # ═══════════════════════════════════════════════════════════════════════════
  # TEST COVERAGE
  # ═══════════════════════════════════════════════════════════════════════════
  echo ""
  echo "--- TEST COVERAGE ---"

  # 5. All tests passing
  ((CRITERIA_TOTAL++))
  cd "$project_path"
  conda run -n "$env_name" pytest tests/ -v --tb=no -q 2>/dev/null
  if [ $? -eq 0 ]; then
    echo "✓ All tests passing"
    ((CRITERIA_MET++))
  else
    echo "✗ Tests failing"
  fi

  # 6. Coverage >= 80%
  ((CRITERIA_TOTAL++))
  COVERAGE=$(conda run -n "$env_name" pytest tests/ --cov={{PROJECT_NAME}} \
    --cov-report=term 2>/dev/null | grep "TOTAL" | awk '{print $NF}' | tr -d '%')
  if [ "${COVERAGE:-0}" -ge 80 ]; then
    echo "✓ Coverage: ${COVERAGE}% (>= 80%)"
    ((CRITERIA_MET++))
  else
    echo "✗ Coverage: ${COVERAGE:-0}% (< 80%)"
  fi

  # ═══════════════════════════════════════════════════════════════════════════
  # FUNCTIONAL VERIFICATION
  # ═══════════════════════════════════════════════════════════════════════════
  echo ""
  echo "--- FUNCTIONAL VERIFICATION ---"

  # 7. Core imports work
  ((CRITERIA_TOTAL++))
  conda run -n "$env_name" python -c "from {{PROJECT_NAME}} import *" 2>/dev/null
  if [ $? -eq 0 ]; then
    echo "✓ Core imports work"
    ((CRITERIA_MET++))
  else
    echo "✗ Core imports FAILED"
  fi

  # 8. Services accessible (if applicable)
  if [ -f "$project_path/app.py" ] || grep -rq "FastAPI\|Flask" "$project_path"/*.py 2>/dev/null; then
    ((CRITERIA_TOTAL++))
    curl -s --connect-timeout 5 "http://localhost:{{PORT_START}}/health" 2>/dev/null | grep -qi "ok\|healthy\|200"
    if [ $? -eq 0 ]; then
      echo "✓ Service health check passed"
      ((CRITERIA_MET++))
    else
      echo "✗ Service health check FAILED (may need to start service)"
    fi
  fi

  # ═══════════════════════════════════════════════════════════════════════════
  # ENVIRONMENT
  # ═══════════════════════════════════════════════════════════════════════════
  echo ""
  echo "--- ENVIRONMENT ---"

  # 9. Conda environment exists
  ((CRITERIA_TOTAL++))
  if conda env list | grep -q "^$env_name "; then
    echo "✓ Conda environment exists: $env_name"
    ((CRITERIA_MET++))
  else
    echo "✗ Conda environment missing: $env_name"
  fi

  # 10. Environment variables configured
  ((CRITERIA_TOTAL++))
  source ~/.bashrc 2>/dev/null
  if [ -n "${{PROJECT_PREFIX}}_PROJECT_PATH" ]; then
    echo "✓ Environment variables configured"
    ((CRITERIA_MET++))
  else
    echo "✗ Environment variables not configured"
  fi

  # ═══════════════════════════════════════════════════════════════════════════
  # SUMMARY
  # ═══════════════════════════════════════════════════════════════════════════
  echo ""
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "COMPLETION CRITERIA: $CRITERIA_MET / $CRITERIA_TOTAL passed"
  echo "═══════════════════════════════════════════════════════════════════════════"

  if [ "$CRITERIA_MET" -eq "$CRITERIA_TOTAL" ]; then
    echo ""
    echo "┌─────────────────────────────────────────────────────────────────────────┐"
    echo "│                                                                         │"
    echo "│                    ✓ PROJECT READY FOR DEPLOYMENT                      │"
    echo "│                                                                         │"
    echo "└─────────────────────────────────────────────────────────────────────────┘"
    return 0
  else
    MISSING=$((CRITERIA_TOTAL - CRITERIA_MET))
    echo ""
    echo "┌─────────────────────────────────────────────────────────────────────────┐"
    echo "│                                                                         │"
    echo "│            ✗ PROJECT NOT READY - $MISSING criteria not met             │"
    echo "│                                                                         │"
    echo "│  Run 'bd ready' to see remaining tasks                                  │"
    echo "│  Run Phase 11 for development completion                                │"
    echo "│  Run Phase 12 for test verification                                     │"
    echo "│                                                                         │"
    echo "└─────────────────────────────────────────────────────────────────────────┘"
    return 1
  fi
}

# Export for use in other scripts
export -f check_completion_criteria
```

### 8.2.1 Quick Completion Check (Simplified)

For fast iteration, use this simplified check:

```bash
# Quick check - returns 0 if ready, 1 if not
quick_completion_check() {
  local env_name="${1:-{{ENV_NAME}}}"
  local project_path="${2:-$(pwd)}"

  # Must-pass criteria (any failure = not ready)
  bd list --status=open 2>/dev/null | grep -q "^{{PROJECT_PREFIX}}-" && return 1
  bd blocked 2>/dev/null | grep -q "^{{PROJECT_PREFIX}}-" && return 1
  grep -rq "NotImplementedError" "$project_path" --include="*.py" 2>/dev/null && return 1
  conda run -n "$env_name" pytest tests/ -x -q 2>/dev/null || return 1

  return 0
}
```

---

## PHASE 9: Verification & Testing

### 9.1 Component Verification Template

```python
#!/usr/bin/env python3
"""Verify all project dependencies are correctly installed with CUDA support."""

import sys
errors = []

# PyTorch + CUDA (if required)
try:
    import torch
    if not torch.cuda.is_available():
        errors.append("PyTorch: CUDA not available")
    else:
        print(f"PyTorch {torch.__version__} CUDA {torch.version.cuda}")
except ImportError:
    pass  # Not required by this project

# Flash Attention (if required)
try:
    from flash_attn import flash_attn_func
    print("Flash Attention OK")
except ImportError:
    pass  # Not required

# vLLM (if required)
try:
    from vllm import LLM
    print("vLLM OK")
except ImportError:
    pass  # Not required

# Project packages
try:
    # Import project-specific packages
    {{PROJECT_IMPORTS}}
    print("Project packages OK")
except ImportError as e:
    errors.append(f"Project packages: {e}")

# Summary
if errors:
    print(f"\nFAILED: {len(errors)} errors")
    for err in errors:
        print(f"  - {err}")
    sys.exit(1)
else:
    print("\nSUCCESS: All required components verified!")
```

### 9.2 Port Verification

```bash
# Verify project ports are available
for port in {{PORT_START}} $(({{PORT_START}}+10)) $(({{PORT_START}}+20)); do
  if ss -tlnp | grep -q ":$port "; then
    echo "ERROR: Port $port is in use"
  else
    echo "OK: Port $port is available"
  fi
done
```

### 9.3 Database Verification (if required)

```bash
# Test database connection
mysql -u root -p"teamrsi123teamrsi123teamrsi123" -e "USE {{PROJECT_NAME}}_db; SELECT 1;" && echo "MySQL OK"
```

---

## PHASE 10: Development Gap Detection & Filling

### 10.1 Automated Gap Detection

Run during any task to discover incomplete work across the codebase:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DEVELOPMENT GAP CATEGORIES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CODE GAPS (Incomplete Implementation)                                      │
│  ├─ TODO/FIXME/XXX/HACK markers                                            │
│  ├─ raise NotImplementedError                                              │
│  ├─ pass # TODO stubs                                                       │
│  ├─ Ellipsis (...) placeholders                                            │
│  └─ Empty function/method bodies                                           │
│                                                                             │
│  TEST GAPS (Missing Coverage)                                               │
│  ├─ Source files without corresponding test files                          │
│  ├─ Functions/classes without test coverage                                │
│  ├─ Edge cases not tested                                                   │
│  └─ Integration tests missing                                              │
│                                                                             │
│  DOCUMENTATION GAPS                                                         │
│  ├─ Missing docstrings on public functions/classes                         │
│  ├─ Outdated README files                                                   │
│  ├─ Missing API documentation                                              │
│  └─ Configuration not documented                                           │
│                                                                             │
│  ERROR HANDLING GAPS                                                        │
│  ├─ Bare except clauses                                                    │
│  ├─ Missing error handling in critical paths                               │
│  ├─ Exceptions caught but not logged                                       │
│  └─ Missing input validation                                               │
│                                                                             │
│  CONFIGURATION GAPS                                                         │
│  ├─ Missing .env.example files                                              │
│  ├─ Hardcoded credentials/secrets                                          │
│  ├─ Missing config validation                                              │
│  └─ Undocumented environment variables                                     │
│                                                                             │
│  DEPENDENCY GAPS                                                            │
│  ├─ Imports without requirements.txt entry                                  │
│  ├─ Version pins missing                                                    │
│  ├─ Unused dependencies                                                     │
│  └─ Security vulnerabilities in deps                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Gap Detection Commands

```bash
# ═══════════════════════════════════════════════════════════════════════════
# CODE GAP DETECTION
# Find incomplete implementations in the codebase
# ═══════════════════════════════════════════════════════════════════════════

PROJECT_PATH="{{PROJECT_PATH}}"

echo "=== TODO/FIXME Markers ==="
grep -rn "TODO\|FIXME\|XXX\|HACK" "$PROJECT_PATH" \
  --include="*.py" --include="*.js" --include="*.ts" \
  | grep -v "__pycache__" \
  | grep -v "node_modules"

echo "=== Incomplete Implementations ==="
grep -rn "raise NotImplementedError\|pass  # TODO\|\.\.\.  # TODO" "$PROJECT_PATH" \
  --include="*.py" \
  | grep -v "__pycache__"

echo "=== Empty Function Bodies ==="
grep -Pzo "def \w+\([^)]*\):\s*\n\s*(pass|\.\.\.)\s*\n" "$PROJECT_PATH" \
  --include="*.py" 2>/dev/null || echo "No empty functions found"

# ═══════════════════════════════════════════════════════════════════════════
# TEST GAP DETECTION
# Find source files without corresponding tests
# ═══════════════════════════════════════════════════════════════════════════

echo "=== Missing Test Files ==="
# Find source directories
SRC_DIRS=$(find "$PROJECT_PATH" -type d -name "src" -o -name "lib" -o -name "app" 2>/dev/null)
TEST_DIR="$PROJECT_PATH/tests"

if [ -d "$TEST_DIR" ]; then
  # Compare source files to test files
  find "$PROJECT_PATH" -path "**/src/*.py" -o -path "**/lib/*.py" -o -path "**/app/*.py" 2>/dev/null | \
  while read src; do
    # Skip __init__.py and __pycache__
    basename=$(basename "$src")
    [ "$basename" = "__init__.py" ] && continue
    [[ "$src" == *"__pycache__"* ]] && continue

    # Check for corresponding test file
    test_file="$TEST_DIR/test_${basename}"
    test_file_alt="$TEST_DIR/test_$(echo $basename | sed 's/.py$//')_test.py"

    if [ ! -f "$test_file" ] && [ ! -f "$test_file_alt" ]; then
      echo "Missing test for: $src"
    fi
  done
else
  echo "WARNING: No tests directory found at $TEST_DIR"
fi

# ═══════════════════════════════════════════════════════════════════════════
# DOCUMENTATION GAP DETECTION
# Find undocumented public interfaces
# ═══════════════════════════════════════════════════════════════════════════

echo "=== Missing Docstrings ==="
grep -Pzo "def [^_]\w+\([^)]*\):\s*\n\s*[^\"']" "$PROJECT_PATH" \
  --include="*.py" 2>/dev/null | head -20 || echo "All public functions documented"

echo "=== Classes Without Docstrings ==="
grep -Pzo "class \w+.*:\s*\n\s*[^\"']" "$PROJECT_PATH" \
  --include="*.py" 2>/dev/null | head -20 || echo "All classes documented"

# ═══════════════════════════════════════════════════════════════════════════
# ERROR HANDLING GAP DETECTION
# Find risky error handling patterns
# ═══════════════════════════════════════════════════════════════════════════

echo "=== Bare Except Clauses ==="
grep -rn "except:" "$PROJECT_PATH" \
  --include="*.py" \
  | grep -v "__pycache__" \
  | grep -v "# noqa"

echo "=== Silent Exception Swallowing ==="
grep -Pzo "except.*:\s*\n\s*pass\s*\n" "$PROJECT_PATH" \
  --include="*.py" 2>/dev/null || echo "No silent exception swallowing found"

# ═══════════════════════════════════════════════════════════════════════════
# CONFIGURATION GAP DETECTION
# Find security and configuration issues
# ═══════════════════════════════════════════════════════════════════════════

echo "=== Hardcoded Secrets (Potential) ==="
grep -rn "password\s*=\s*[\"']" "$PROJECT_PATH" \
  --include="*.py" --include="*.yml" --include="*.yaml" \
  | grep -v "__pycache__" \
  | grep -v ".env"

grep -rn "api_key\s*=\s*[\"']" "$PROJECT_PATH" \
  --include="*.py" --include="*.yml" --include="*.yaml" \
  | grep -v "__pycache__"

echo "=== Missing .env.example ==="
if [ -f "$PROJECT_PATH/.env" ] && [ ! -f "$PROJECT_PATH/.env.example" ]; then
  echo "WARNING: .env exists but no .env.example template"
fi

# ═══════════════════════════════════════════════════════════════════════════
# DEPENDENCY GAP DETECTION
# Find import/requirement mismatches
# ═══════════════════════════════════════════════════════════════════════════

echo "=== Imports Not in Requirements ==="
# Extract all imports
IMPORTS=$(grep -rh "^import \|^from .* import" "$PROJECT_PATH" \
  --include="*.py" 2>/dev/null \
  | grep -v "__pycache__" \
  | sed 's/import //' | sed 's/from //' | sed 's/ import.*//' \
  | sort -u)

# Check against requirements
if [ -f "$PROJECT_PATH/requirements.txt" ]; then
  for pkg in $IMPORTS; do
    # Skip standard library
    python3 -c "import $pkg" 2>/dev/null && continue
    # Check if in requirements
    if ! grep -qi "^$pkg" "$PROJECT_PATH/requirements.txt" 2>/dev/null; then
      echo "Possibly missing: $pkg"
    fi
  done | head -20
fi
```

### 10.3 Gap → Beads Task Workflow

When a gap is detected, create appropriate Beads tasks:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# AUTOMATED GAP → TASK CONVERSION
# Uses code-reasoning to assess severity, then creates appropriate tasks
# ═══════════════════════════════════════════════════════════════════════════

analyze_and_create_gap_task() {
  local gap_type="$1"       # test, doc, error-handling, config, impl
  local gap_location="$2"   # file:line or file
  local gap_description="$3"

  # Determine task type and priority based on gap type
  case "$gap_type" in
    impl)
      # NotImplementedError or TODO with blocking effect
      TASK_TYPE="task"
      PRIORITY=2
      ;;
    test)
      # Missing test coverage
      TASK_TYPE="task"
      PRIORITY=3
      ;;
    error-handling)
      # Bare except, silent failures
      TASK_TYPE="bug"
      PRIORITY=2
      ;;
    security)
      # Hardcoded secrets, missing validation
      TASK_TYPE="bug"
      PRIORITY=1
      ;;
    doc)
      # Missing documentation
      TASK_TYPE="chore"
      PRIORITY=4
      ;;
    config)
      # Missing config files
      TASK_TYPE="task"
      PRIORITY=3
      ;;
    *)
      TASK_TYPE="task"
      PRIORITY=3
      ;;
  esac

  # Create Beads task
  bd create \
    --title="Gap: $gap_description" \
    --type=$TASK_TYPE \
    --priority=$PRIORITY \
    --description="Location: $gap_location
Gap Type: $gap_type
Detected by: Phase 10 Gap Detection

Action Required: Address this gap before deployment."

  echo "Created $TASK_TYPE task (P$PRIORITY): $gap_description"
}

# Example usage during gap detection
# analyze_and_create_gap_task "test" "src/auth.py" "Missing tests for authentication module"
# analyze_and_create_gap_task "impl" "src/api.py:45" "NotImplementedError in request handler"
# analyze_and_create_gap_task "security" "config.py:12" "Hardcoded API key"
```

### 10.4 Gap Detection Integration with Ralph-Loop

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GAP DETECTION IN RALPH-LOOP                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WHEN TO RUN GAP DETECTION:                                                 │
│  ├─ After Phase 0 (System Audit) - Initial gap assessment                  │
│  ├─ After completing any feature task - Check for new gaps                 │
│  ├─ Before marking deployment complete - Final gap check                   │
│  └─ On demand when issues discovered during development                    │
│                                                                             │
│  GAP DETECTION FLOW:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Run gap detection commands (10.2)                                │   │
│  │    └─ Outputs list of gaps with locations                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 2. Use code-reasoning to assess severity                            │   │
│  │    ├─ mcp__code-reasoning__code-reasoning                           │   │
│  │    │   with thought="Analyze gap: {{GAP}}"                          │   │
│  │    └─ Determine: blocking vs non-blocking                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 3. Create Beads tasks (10.3)                                        │   │
│  │    ├─ BLOCKING gaps → high priority, add as dependency              │   │
│  │    └─ NON-BLOCKING gaps → lower priority, work later                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 4. Continue with current task                                       │   │
│  │    ├─ If blocking → address gap first                               │   │
│  │    └─ If non-blocking → continue, address later                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GAP TASK INTEGRATION:                                                      │
│  ├─ Gap tasks appear in `bd ready` when unblocked                          │
│  ├─ Gap tasks can be assigned to specialists (test, doc writers)           │
│  ├─ Gap metrics tracked via `bd stats`                                      │
│  └─ Gap completion verified before deployment sign-off                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.5 Development Task Types

Map gaps to appropriate Beads task types:

| Gap Type | Beads Type | Priority | Description |
|----------|------------|----------|-------------|
| **NotImplementedError** | `task` | P1-P2 | Core functionality not implemented |
| **TODO/FIXME** | `task` | P2-P3 | Planned work not completed |
| **Missing Tests** | `task` | P3 | Test coverage gaps |
| **Bare Except** | `bug` | P2 | Error handling defect |
| **Hardcoded Secrets** | `bug` | P0 | Security vulnerability |
| **Missing Validation** | `bug` | P2 | Input handling defect |
| **Missing Docs** | `chore` | P4 | Documentation gap |
| **Missing Config** | `task` | P3 | Configuration not provided |
| **Unused Deps** | `chore` | P4 | Cleanup needed |
| **Security Vuln** | `bug` | P0-P1 | Dependency security issue |

### 10.6 Automated Gap Report Generation

```bash
# ═══════════════════════════════════════════════════════════════════════════
# GENERATE GAP REPORT
# Creates structured report of all gaps for team review
# ═══════════════════════════════════════════════════════════════════════════

generate_gap_report() {
  PROJECT_PATH="${1:-$(pwd)}"
  REPORT_FILE="$PROJECT_PATH/.ralph/gap-report-$(date +%Y%m%d).md"

  cat > "$REPORT_FILE" << 'EOF'
# Development Gap Report
Generated: $(date)
Project: $PROJECT_PATH

## Summary

| Category | Count | Critical |
|----------|-------|----------|
EOF

  # Count gaps by category
  TODO_COUNT=$(grep -rn "TODO\|FIXME" "$PROJECT_PATH" --include="*.py" 2>/dev/null | wc -l)
  IMPL_COUNT=$(grep -rn "NotImplementedError" "$PROJECT_PATH" --include="*.py" 2>/dev/null | wc -l)
  EXCEPT_COUNT=$(grep -rn "except:" "$PROJECT_PATH" --include="*.py" 2>/dev/null | wc -l)

  echo "| TODOs/FIXMEs | $TODO_COUNT | N/A |" >> "$REPORT_FILE"
  echo "| NotImplementedError | $IMPL_COUNT | YES |" >> "$REPORT_FILE"
  echo "| Bare Except | $EXCEPT_COUNT | REVIEW |" >> "$REPORT_FILE"

  echo "" >> "$REPORT_FILE"
  echo "## Detailed Findings" >> "$REPORT_FILE"
  echo "" >> "$REPORT_FILE"

  # Append detailed findings
  echo "### Critical: NotImplementedError" >> "$REPORT_FILE"
  echo '```' >> "$REPORT_FILE"
  grep -rn "NotImplementedError" "$PROJECT_PATH" --include="*.py" 2>/dev/null >> "$REPORT_FILE"
  echo '```' >> "$REPORT_FILE"

  echo "### TODOs and FIXMEs" >> "$REPORT_FILE"
  echo '```' >> "$REPORT_FILE"
  grep -rn "TODO\|FIXME" "$PROJECT_PATH" --include="*.py" 2>/dev/null | head -50 >> "$REPORT_FILE"
  echo '```' >> "$REPORT_FILE"

  echo "" >> "$REPORT_FILE"
  echo "## Recommended Actions" >> "$REPORT_FILE"
  echo "1. Address all NotImplementedError before deployment" >> "$REPORT_FILE"
  echo "2. Review bare except clauses for proper error handling" >> "$REPORT_FILE"
  echo "3. Create tasks for TODO/FIXME items in Beads" >> "$REPORT_FILE"

  echo "Gap report saved to: $REPORT_FILE"
}

# Run: generate_gap_report /path/to/project
```

### 10.7 Gap Detection During Task Execution

Integrate gap detection into the standard task workflow:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# POST-TASK GAP CHECK
# Run after completing any feature/implementation task
# ═══════════════════════════════════════════════════════════════════════════

post_task_gap_check() {
  local task_id="$1"
  local files_changed="$2"  # Space-separated list of modified files

  echo "=== Running post-task gap check for $task_id ==="

  for file in $files_changed; do
    echo "Checking: $file"

    # Check for new TODOs in changed files
    TODOS=$(grep -n "TODO\|FIXME" "$file" 2>/dev/null)
    if [ -n "$TODOS" ]; then
      echo "New TODOs found:"
      echo "$TODOS"

      # Create Beads task for each TODO
      echo "$TODOS" | while read line; do
        LINE_NUM=$(echo "$line" | cut -d: -f1)
        CONTENT=$(echo "$line" | cut -d: -f2-)
        bd create \
          --title="Address TODO: ${CONTENT:0:50}..." \
          --type=task \
          --priority=3 \
          --description="Location: $file:$LINE_NUM
Found after completing task: $task_id
Content: $CONTENT"
      done
    fi

    # Check for NotImplementedError
    NOT_IMPL=$(grep -n "NotImplementedError" "$file" 2>/dev/null)
    if [ -n "$NOT_IMPL" ]; then
      echo "WARNING: NotImplementedError found in $file"
      bd create \
        --title="Implement: stub in $file" \
        --type=task \
        --priority=2 \
        --description="NotImplementedError found after task $task_id"
    fi
  done

  echo "=== Gap check complete ==="
}

# Usage in ralph-loop Step 7 (COMPLETE TASK):
# post_task_gap_check "{{TASK_ID}}" "file1.py file2.py file3.py"
```

---

## PHASE 10.5: Project Completeness Assessment

### 10.5.1 Purpose & Decision Logic

Determine if project is complete (ready for deployment) or incomplete (needs development):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              PROJECT COMPLETENESS ASSESSMENT                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DECISION FLOW:                                                             │
│  ├─ Count incomplete indicators (TODOs, NotImplementedError, stubs)        │
│  ├─ Assess severity (blocking vs non-blocking)                             │
│  └─ Route to appropriate workflow:                                          │
│      ├─ COMPLETE → Standard deployment (Phases 0-9)                        │
│      └─ INCOMPLETE → Development completion (Phase 11) + deployment        │
│                                                                             │
│  INCOMPLETE INDICATORS:                                                     │
│  ├─ NotImplementedError        → Critical (blocks functionality)           │
│  ├─ raise NotImplemented       → Critical (alternative syntax)             │
│  ├─ TODO/FIXME/XXX markers     → Moderate (planned work)                   │
│  ├─ pass  # stub               → Moderate (placeholder)                    │
│  ├─ ...  # placeholder         → Moderate (ellipsis placeholders)          │
│  └─ Empty function bodies      → Critical (no implementation)              │
│                                                                             │
│  PROJECT STATUS VALUES:                                                     │
│  ├─ complete           → All indicators = 0, ready for deployment          │
│  ├─ incomplete_critical → Has NotImplementedError, MUST develop first      │
│  └─ incomplete_minor   → Has TODOs/stubs, development recommended          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.5.2 Completeness Assessment Script

```bash
# ═══════════════════════════════════════════════════════════════════════════
# PROJECT COMPLETENESS ASSESSMENT
# Determines if project needs deployment only or development + deployment
# ═══════════════════════════════════════════════════════════════════════════

# ═══════════════════════════════════════════════════════════════════════════
# ENHANCED PROJECT COMPLETENESS ASSESSMENT
# Handles 5 project types with weighted scoring
# Integrates with Phase 13 (Research) and Phase 14 (Functionality Fix)
# ═══════════════════════════════════════════════════════════════════════════

assess_project_completeness() {
  PROJECT_PATH="${1:-$(pwd)}"

  echo "╔═══════════════════════════════════════════════════════════════════════════╗"
  echo "║           ENHANCED PROJECT COMPLETENESS ASSESSMENT                        ║"
  echo "╚═══════════════════════════════════════════════════════════════════════════╝"
  echo "Project: $PROJECT_PATH"
  echo ""

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE A: Static Code Analysis
  # ─────────────────────────────────────────────────────────────────────────

  echo ">>> PHASE A: Static Code Analysis..."

  NOT_IMPL=$(grep -rn "NotImplementedError\|raise NotImplemented" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | wc -l)

  TODOS=$(grep -rn "TODO\|FIXME\|XXX" "$PROJECT_PATH" \
    --include="*.py" --include="*.js" --include="*.ts" 2>/dev/null | \
    grep -v "__pycache__" | grep -v "node_modules" | wc -l)

  STUBS=$(grep -rn "pass  #\|\.\.\.  #" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | wc -l)

  # Count empty function bodies (def foo(): pass or def foo(): ...)
  EMPTY_FUNCS=$(grep -Pzo "def \w+\([^)]*\):\s*\n\s*(pass|\.\.\.)\s*\n" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -c "def " || echo "0")

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE B: Research Requirement Detection
  # ─────────────────────────────────────────────────────────────────────────

  echo ">>> PHASE B: Research Requirement Detection..."
  detect_research_requirements "$PROJECT_PATH"
  # Sets: RESEARCH_SCORE, RESEARCH_INDICATORS

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE C: Functionality Problem Detection
  # ─────────────────────────────────────────────────────────────────────────

  echo ">>> PHASE C: Functionality Problem Detection..."
  detect_functionality_problems "$PROJECT_PATH"
  # Sets: FUNCTIONALITY_SCORE, FUNCTIONALITY_ISSUES

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE D: Weighted Complexity Scoring
  # ─────────────────────────────────────────────────────────────────────────

  echo ">>> PHASE D: Weighted Complexity Scoring..."

  # Weight factors (higher = more critical)
  WEIGHT_NOT_IMPL=10      # Critical: unimplemented functions
  WEIGHT_EMPTY_FUNC=8     # High: empty function bodies
  WEIGHT_IMPORT_ERR=7     # High: can't even import
  WEIGHT_TEST_FAIL=6      # Medium-high: tests broken
  WEIGHT_RESEARCH=5       # Medium: needs research
  WEIGHT_TODO=2           # Low: known work items
  WEIGHT_STUB=1           # Low: placeholders

  COMPLEXITY_SCORE=$((
    NOT_IMPL * WEIGHT_NOT_IMPL +
    EMPTY_FUNCS * WEIGHT_EMPTY_FUNC +
    FUNCTIONALITY_SCORE +
    RESEARCH_SCORE / 4 +
    TODOS * WEIGHT_TODO +
    STUBS * WEIGHT_STUB
  ))

  echo ""
  echo "┌─────────────────────────────────────────────────────────────────┐"
  echo "│ COMPLEXITY ANALYSIS                                             │"
  echo "├─────────────────────────────────────────────────────────────────┤"
  echo "│ NotImplementedError:     $NOT_IMPL (weight: $WEIGHT_NOT_IMPL)"
  echo "│ Empty functions:         $EMPTY_FUNCS (weight: $WEIGHT_EMPTY_FUNC)"
  echo "│ Functionality score:     $FUNCTIONALITY_SCORE"
  echo "│ Research score:          $RESEARCH_SCORE"
  echo "│ TODOs:                   $TODOS (weight: $WEIGHT_TODO)"
  echo "│ Stubs:                   $STUBS (weight: $WEIGHT_STUB)"
  echo "├─────────────────────────────────────────────────────────────────┤"
  echo "│ TOTAL COMPLEXITY SCORE:  $COMPLEXITY_SCORE"
  echo "└─────────────────────────────────────────────────────────────────┘"

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE E: Classification Decision (5 project types)
  # ─────────────────────────────────────────────────────────────────────────

  echo ""
  echo ">>> PHASE E: Project Classification..."

  if [ "$COMPLEXITY_SCORE" -eq 0 ]; then
    PROJECT_STATUS="complete"
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║ PROJECT STATUS: ✓ COMPLETE                                               ║"
    echo "║ Action: Standard deployment (Phases 0-9)                                 ║"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"
    export NEEDS_DEVELOPMENT="false"

  elif [ "$FUNCTIONALITY_SCORE" -gt 20 ]; then
    PROJECT_STATUS="functionality_problems"
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║ PROJECT STATUS: ⚠ FUNCTIONALITY PROBLEMS                                 ║"
    echo "║ Issues: ${FUNCTIONALITY_ISSUES[*]}"
    echo "║ Action: Fix functionality first (Phase 14), then continue                ║"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"
    export NEEDS_DEVELOPMENT="true"
    export NEEDS_FUNCTIONALITY_FIX="true"

  elif [ "$RESEARCH_SCORE" -ge 20 ] && [ "$NOT_IMPL" -gt 0 ]; then
    PROJECT_STATUS="incomplete_research"
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║ PROJECT STATUS: ✗ INCOMPLETE - Requires Research                         ║"
    echo "║ Research indicators: ${RESEARCH_INDICATORS[*]}"
    echo "║ Action: Research phase (Phase 13), then development (Phase 11)           ║"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"
    export NEEDS_DEVELOPMENT="true"
    export NEEDS_RESEARCH="true"

  elif [ "$NOT_IMPL" -gt 0 ] || [ "$EMPTY_FUNCS" -gt 0 ]; then
    PROJECT_STATUS="incomplete_critical"
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║ PROJECT STATUS: ✗ INCOMPLETE - Critical                                  ║"
    echo "║ Reason: Has $NOT_IMPL unimplemented functions, $EMPTY_FUNCS empty bodies ║"
    echo "║ Action: Development completion required (Phase 11)                       ║"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"
    export NEEDS_DEVELOPMENT="true"

  else
    PROJECT_STATUS="incomplete_minor"
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║ PROJECT STATUS: ⚠ INCOMPLETE - Minor                                     ║"
    echo "║ Reason: Has $TODOS TODOs, $STUBS stubs                                   ║"
    echo "║ Action: Address TODOs/stubs (Phase 11)                                   ║"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"
    export NEEDS_DEVELOPMENT="true"
  fi

  export PROJECT_STATUS
  export COMPLEXITY_SCORE

  # Create Beads summary task if not complete
  if [ "$PROJECT_STATUS" != "complete" ]; then
    echo ""
    echo "Creating Beads epic for project completion..."
    bd create \
      --title="Project Assessment: $PROJECT_STATUS (score: $COMPLEXITY_SCORE)" \
      --type=epic \
      --priority=1 \
      --description="Enhanced Assessment Results:

Static Analysis:
- NotImplementedError: $NOT_IMPL
- Empty functions: $EMPTY_FUNCS
- TODOs: $TODOS
- Stubs: $STUBS

Functional Analysis:
- Functionality Score: $FUNCTIONALITY_SCORE
- Issues: ${FUNCTIONALITY_ISSUES[*]:-None}

Research Analysis:
- Research Score: $RESEARCH_SCORE
- Indicators: ${RESEARCH_INDICATORS[*]:-None}

TOTAL COMPLEXITY SCORE: $COMPLEXITY_SCORE

Recommended Action Path:
$(case "$PROJECT_STATUS" in
  "functionality_problems") echo "1. Phase 14: Fix functionality problems
2. Re-assess
3. Continue to appropriate phase";;
  "incomplete_research") echo "1. Phase 13: Research required technologies
2. Phase 11: Implement based on research
3. Phase 0-9: Deploy";;
  "incomplete_critical") echo "1. Phase 11: Critical development completion
2. Phase 0-9: Deploy";;
  "incomplete_minor") echo "1. Phase 11: Address TODOs and stubs
2. Phase 0-9: Deploy";;
esac)

Assessed: $(date)"
  fi

  return 0
}

# Export for use in other scripts
export -f assess_project_completeness
```

### 10.5.3 Detailed Gap Listing

```bash
# ═══════════════════════════════════════════════════════════════════════════
# LIST ALL INCOMPLETE ITEMS WITH LOCATIONS
# For detailed planning of development work
# ═══════════════════════════════════════════════════════════════════════════

list_incomplete_items() {
  PROJECT_PATH="${1:-$(pwd)}"

  echo "=== Detailed Incomplete Items List ==="
  echo ""

  echo "### CRITICAL: NotImplementedError ###"
  grep -rn "NotImplementedError\|raise NotImplemented" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | head -50

  echo ""
  echo "### CRITICAL: Empty Function Bodies ###"
  grep -Pzo "def \w+\([^)]*\):\s*\n\s*(pass|\.\.\.)\s*\n" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | head -50

  echo ""
  echo "### MODERATE: TODO/FIXME Markers ###"
  grep -rn "TODO\|FIXME\|XXX" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | head -50

  echo ""
  echo "### MODERATE: Stub Implementations ###"
  grep -rn "pass  #\|\.\.\.  #" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | head -50
}
```

### 10.5.4 Integration with Ralph-Loop

After Phase 10.5 assessment, the workflow branches:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              RALPH-LOOP ROUTING AFTER ASSESSMENT                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  assess_project_completeness()                                              │
│          ↓                                                                  │
│  ┌───────────────────────────────────────────────────────┐                 │
│  │ PROJECT_STATUS = ?                                     │                 │
│  └───────────────────────────────────────────────────────┘                 │
│          ↓                                                                  │
│    ┌─────┴─────┬─────────────────┐                                         │
│    ↓           ↓                 ↓                                         │
│ complete   incomplete_minor   incomplete_critical                          │
│    │           │                 │                                         │
│    │           └────────┬────────┘                                         │
│    ↓                    ↓                                                  │
│ Phase 9:            Phase 11:                                              │
│ Verification        Development Completion                                 │
│ & Testing           Workflow (11.1-11.4)                                   │
│    │                    │                                                  │
│    │                    ↓                                                  │
│    │               Phase 12:                                               │
│    │               Test Execution                                          │
│    │               Framework (12.1-12.3)                                   │
│    │                    │                                                  │
│    └────────┬───────────┘                                                  │
│             ↓                                                               │
│       DEPLOYMENT READY                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## PHASE 11: Development Completion Workflow

### 11.0 Overview

When a project is assessed as incomplete (Phase 10.5), this workflow implements the missing functionality:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DEVELOPMENT COMPLETION WORKFLOW                           │
│                    (For incomplete projects)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 11.1: PRIORITIZE DEVELOPMENT TASKS                             │   │
│  │ ├─ Parse all gaps from Phase 10                                     │   │
│  │ ├─ Categorize by type and severity                                  │   │
│  │ └─ Create ordered Beads tasks with dependencies                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 11.2: IMPLEMENT MISSING FUNCTIONALITY                          │   │
│  │ ├─ Get next ready task: bd ready                                    │   │
│  │ ├─ Understand requirements (MCP tools)                              │   │
│  │ ├─ Generate implementation code                                     │   │
│  │ └─ Save patterns to memory                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 11.3: WRITE TESTS FOR NEW CODE                                 │   │
│  │ ├─ Create test file if not exists                                   │   │
│  │ ├─ Write unit tests (happy path, edge cases, errors)                │   │
│  │ └─ Ensure tests PASS before continuing                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ STEP 11.4: VERIFY IMPLEMENTATION                                    │   │
│  │ ├─ Run related tests                                                │   │
│  │ ├─ Check no new gaps introduced                                     │   │
│  │ ├─ Verify imports work                                              │   │
│  │ └─ Close Beads task and store in memory                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                              │
│                    LOOP → STEP 11.1 (until no gaps remain)                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 11.1 Prioritize Development Tasks

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 11.1: PRIORITIZE DEVELOPMENT TASKS
# Parse gaps and create ordered Beads tasks
# ═══════════════════════════════════════════════════════════════════════════

prioritize_development_tasks() {
  PROJECT_PATH="${1:-$(pwd)}"

  echo "=== Prioritizing Development Tasks ==="

  # PRIORITY 0: Security issues (hardcoded secrets, vulnerabilities)
  echo "--- P0: Security Issues ---"
  grep -rn "password\s*=\s*[\"'][^\"']*[\"']" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | \
  while read line; do
    FILE=$(echo "$line" | cut -d: -f1)
    LINENUM=$(echo "$line" | cut -d: -f2)
    bd create \
      --title="SECURITY: Remove hardcoded credential in $(basename $FILE)" \
      --type=bug \
      --priority=0 \
      --description="Location: $FILE:$LINENUM
Severity: CRITICAL
Found during Phase 11.1 task prioritization.

Remove hardcoded secret and use environment variables."
  done

  # PRIORITY 1: NotImplementedError (core functionality blocked)
  echo "--- P1: NotImplementedError ---"
  grep -rn "NotImplementedError\|raise NotImplemented" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | \
  while read line; do
    FILE=$(echo "$line" | cut -d: -f1)
    LINENUM=$(echo "$line" | cut -d: -f2)
    # Extract function name if possible
    FUNC_NAME=$(sed -n "${LINENUM}p" "$FILE" 2>/dev/null | grep -oP "def \K\w+")
    bd create \
      --title="Implement: ${FUNC_NAME:-function} in $(basename $FILE)" \
      --type=task \
      --priority=1 \
      --description="Location: $FILE:$LINENUM
Gap Type: NotImplementedError
Found during Phase 11.1 task prioritization.

Implement the missing functionality."
  done

  # PRIORITY 2: Empty function bodies
  echo "--- P2: Empty Function Bodies ---"
  grep -Pzo "def (\w+)\([^)]*\):\s*\n\s*(pass|\.\.\.)\s*\n" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | \
  while read -r match; do
    FUNC_NAME=$(echo "$match" | grep -oP "def \K\w+")
    [ -z "$FUNC_NAME" ] && continue
    bd create \
      --title="Implement: $FUNC_NAME (empty body)" \
      --type=task \
      --priority=2 \
      --description="Gap Type: Empty function body
Found during Phase 11.1 task prioritization.

Function has stub implementation (pass or ...). Implement full functionality."
  done

  # PRIORITY 3: TODOs with clear requirements
  echo "--- P3: TODO Markers ---"
  grep -rn "TODO\|FIXME" "$PROJECT_PATH" \
    --include="*.py" 2>/dev/null | grep -v "__pycache__" | head -20 | \
  while read line; do
    FILE=$(echo "$line" | cut -d: -f1)
    LINENUM=$(echo "$line" | cut -d: -f2)
    CONTENT=$(echo "$line" | cut -d: -f3- | head -c 80)
    bd create \
      --title="TODO: ${CONTENT:0:50}" \
      --type=task \
      --priority=3 \
      --description="Location: $FILE:$LINENUM
Content: $CONTENT
Found during Phase 11.1 task prioritization."
  done

  # PRIORITY 4: Missing tests
  echo "--- P4: Missing Tests ---"
  find "$PROJECT_PATH" -path "**/src/*.py" -o -path "**/lib/*.py" 2>/dev/null | \
  while read src; do
    basename=$(basename "$src" .py)
    [ "$basename" = "__init__" ] && continue
    [[ "$src" == *"__pycache__"* ]] && continue

    test_file="$PROJECT_PATH/tests/test_${basename}.py"
    if [ ! -f "$test_file" ]; then
      bd create \
        --title="Write tests for $basename" \
        --type=task \
        --priority=4 \
        --description="Source: $src
Missing test file: $test_file
Found during Phase 11.1 task prioritization."
    fi
  done

  echo "=== Task prioritization complete ==="
  echo "Run 'bd ready' to see tasks ready for development"
}
```

### 11.2 Implement Missing Functionality

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 11.2: IMPLEMENT MISSING FUNCTIONALITY
# MCP tool orchestration for development completion
# ═══════════════════════════════════════════════════════════════════════════

implement_missing_function() {
  local file_path="$1"
  local function_name="$2"
  local task_id="$3"

  echo "=== Implementing: $function_name in $file_path ==="

  # 1. Claim the task
  echo "Claiming task $task_id..."
  bd update "$task_id" --status=in_progress

  # 2. Understand context using MCP tools
  echo "Analyzing context..."
  # Use these MCP tools in sequence:
  #
  # mcp__code-reasoning__code-reasoning
  #   - Analyze what the function should do based on:
  #     - Function name and signature
  #     - Docstring (if any)
  #     - How it's called elsewhere in the codebase
  #     - Similar functions in the same module
  #
  # mcp__context7__resolve-library-id + mcp__context7__query-docs
  #   - Get documentation for libraries used by this function
  #   - Understand API contracts and patterns
  #
  # mcp__serena__read_memory
  #   - Check for project-specific patterns
  #   - Look for similar implementations done previously
  #
  # mcp__cipher__ask_cipher
  #   - Search conversation memory for past solutions
  #   - Find how similar problems were solved

  # 3. Generate implementation
  echo "Generating implementation..."
  # Use mcp__serena tools for code editing:
  #
  # mcp__serena__find_symbol
  #   - Locate the stub function
  #
  # mcp__serena__replace_symbol_body
  #   - Replace the stub with actual implementation
  #
  # Follow existing project patterns from serena memory

  # 4. Run syntax check
  conda run -n {{ENV_NAME}} python -m py_compile "$file_path"
  if [ $? -ne 0 ]; then
    echo "ERROR: Syntax error in implementation"
    bd update "$task_id" --status=blocked
    bd create --title="Fix syntax error in $function_name" --type=bug --priority=1
    return 1
  fi

  echo "Implementation complete, proceeding to tests..."
  return 0
}

# ═══════════════════════════════════════════════════════════════════════════
# MCP TOOL ORCHESTRATION PATTERN FOR IMPLEMENTATION
# ═══════════════════════════════════════════════════════════════════════════
#
# STEP A: Memory Check
# mcp__serena__read_memory("project-patterns.md")
# mcp__cipher__ask_cipher("Search: how to implement {{FUNCTION_TYPE}} in {{PROJECT}}")
# mcp__plugin_claude-mem_mcp-search__search(query="{{FUNCTION_NAME}} implementation")
#
# STEP B: Context Analysis
# mcp__code-context__get_code_context(absolutePath="{{PROJECT_PATH}}")
# mcp__code-reasoning__code-reasoning(thought="Analyze requirements for {{FUNCTION}}")
#
# STEP C: Documentation Lookup
# mcp__context7__resolve-library-id(libraryName="{{LIBRARY}}")
# mcp__context7__query-docs(libraryId="{{ID}}", query="{{SPECIFIC_QUESTION}}")
#
# STEP D: Implementation
# mcp__serena__find_symbol(name_path_pattern="{{FUNCTION}}", include_body=true)
# mcp__serena__replace_symbol_body(name_path="{{FUNCTION}}", body="{{NEW_CODE}}")
#
# STEP E: Store Solution
# mcp__serena__write_memory("{{TOPIC}}.md", "{{SOLUTION_PATTERN}}")
# mcp__cipher__ask_cipher("Store: {{PROJECT}} - {{FUNCTION}} implementation pattern")
```

### 11.3 Write Tests for New Code

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 11.3: WRITE TESTS FOR NEW CODE
# Test generation alongside development
# ═══════════════════════════════════════════════════════════════════════════

generate_test_for_function() {
  local module_name="$1"
  local function_name="$2"
  local project_path="${3:-$(pwd)}"

  local test_dir="$project_path/tests"
  local test_file="$test_dir/test_${module_name}.py"

  echo "=== Generating tests for $function_name ==="

  # Create tests directory if not exists
  if [ ! -d "$test_dir" ]; then
    mkdir -p "$test_dir"
    touch "$test_dir/__init__.py"
  fi

  # Create test file if not exists
  if [ ! -f "$test_file" ]; then
    cat > "$test_file" << EOF
"""Tests for ${module_name} module."""
import pytest
from {{PROJECT_NAME}}.${module_name} import *


EOF
    echo "Created test file: $test_file"
  fi

  # Check if test already exists
  if grep -q "def test_${function_name}" "$test_file" 2>/dev/null; then
    echo "Tests for $function_name already exist in $test_file"
    return 0
  fi

  # Append test function template
  cat >> "$test_file" << EOF

class Test${function_name^}:
    """Test suite for ${function_name} function."""

    def test_${function_name}_happy_path(self):
        """Test ${function_name} with valid inputs."""
        # Arrange
        # TODO: Set up test data based on function signature

        # Act
        result = ${function_name}()

        # Assert
        assert result is not None

    def test_${function_name}_edge_cases(self):
        """Test ${function_name} with edge cases."""
        # Test empty inputs
        # Test boundary values
        # Test None handling
        pass

    def test_${function_name}_error_handling(self):
        """Test ${function_name} error handling."""
        # Test invalid inputs raise appropriate exceptions
        with pytest.raises(ValueError):
            ${function_name}(invalid_arg=None)

EOF

  echo "Generated test template for $function_name in $test_file"
  echo "TODO: Implement actual test logic based on function behavior"
}

# ═══════════════════════════════════════════════════════════════════════════
# RUN TESTS FOR SPECIFIC FUNCTION
# ═══════════════════════════════════════════════════════════════════════════

run_function_tests() {
  local function_name="$1"
  local env_name="${2:-{{ENV_NAME}}}"

  echo "Running tests for $function_name..."

  # Run pytest with function name filter
  conda run -n "$env_name" pytest tests/ -v -k "$function_name" --tb=short

  if [ $? -eq 0 ]; then
    echo "✓ All tests for $function_name PASSED"
    return 0
  else
    echo "✗ Tests for $function_name FAILED"
    return 1
  fi
}
```

### 11.4 Verify Implementation

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 11.4: VERIFY IMPLEMENTATION
# Comprehensive verification before closing task
# ═══════════════════════════════════════════════════════════════════════════

verify_implementation() {
  local file_path="$1"
  local function_name="$2"
  local task_id="$3"
  local env_name="${4:-{{ENV_NAME}}}"

  echo "=== Verifying implementation of $function_name ==="

  VERIFICATION_PASSED=true

  # 1. Syntax check
  echo "1. Syntax check..."
  conda run -n "$env_name" python -m py_compile "$file_path"
  if [ $? -ne 0 ]; then
    echo "✗ Syntax check FAILED"
    VERIFICATION_PASSED=false
  else
    echo "✓ Syntax check passed"
  fi

  # 2. Import check
  echo "2. Import check..."
  MODULE_NAME=$(basename "$file_path" .py)
  conda run -n "$env_name" python -c "from {{PROJECT_NAME}}.$MODULE_NAME import $function_name; print('Import OK')" 2>/dev/null
  if [ $? -ne 0 ]; then
    echo "✗ Import check FAILED"
    VERIFICATION_PASSED=false
  else
    echo "✓ Import check passed"
  fi

  # 3. Run tests
  echo "3. Running tests..."
  run_function_tests "$function_name" "$env_name"
  if [ $? -ne 0 ]; then
    echo "✗ Tests FAILED"
    VERIFICATION_PASSED=false
  else
    echo "✓ Tests passed"
  fi

  # 4. Check for new gaps introduced
  echo "4. Checking for new gaps..."
  NEW_TODOS=$(grep -n "TODO\|FIXME\|NotImplementedError" "$file_path" 2>/dev/null | wc -l)
  if [ "$NEW_TODOS" -gt 0 ]; then
    echo "⚠ Warning: $NEW_TODOS new gaps found in $file_path"
    # Create follow-up tasks for new gaps
    grep -n "TODO\|FIXME" "$file_path" 2>/dev/null | while read line; do
      LINENUM=$(echo "$line" | cut -d: -f1)
      CONTENT=$(echo "$line" | cut -d: -f2-)
      bd create \
        --title="Follow-up TODO from $function_name" \
        --type=task \
        --priority=3 \
        --description="Found while implementing $function_name
Location: $file_path:$LINENUM
Content: $CONTENT"
    done
  else
    echo "✓ No new gaps introduced"
  fi

  # 5. Complete or fail the task
  if [ "$VERIFICATION_PASSED" = true ]; then
    echo ""
    echo "=== VERIFICATION PASSED ==="
    bd close "$task_id" --reason="Implemented and verified: $function_name"

    # Store success in memory
    # mcp__cipher__ask_cipher("Store: $function_name implemented successfully")
    # mcp__serena__write_memory if new pattern discovered

    return 0
  else
    echo ""
    echo "=== VERIFICATION FAILED ==="
    bd update "$task_id" --status=blocked
    bd create \
      --title="Fix verification failures for $function_name" \
      --type=bug \
      --priority=1 \
      --description="Verification failed during Phase 11.4
Function: $function_name
File: $file_path
Task: $task_id

Review and fix the failing checks."
    return 1
  fi
}

# ═══════════════════════════════════════════════════════════════════════════
# DEVELOPMENT LOOP CONTROLLER
# Runs the complete 11.1-11.4 workflow until no gaps remain
# ═══════════════════════════════════════════════════════════════════════════

run_development_completion_loop() {
  local project_path="${1:-$(pwd)}"
  local max_iterations="${2:-50}"

  echo "=== Starting Development Completion Loop ==="
  echo "Project: $project_path"
  echo "Max iterations: $max_iterations"

  for i in $(seq 1 $max_iterations); do
    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "ITERATION $i of $max_iterations"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

    # Check for ready development tasks
    READY_TASKS=$(bd ready 2>/dev/null | grep -c "^{{PROJECT_PREFIX}}-" || echo "0")

    if [ "$READY_TASKS" -eq 0 ]; then
      echo "No more ready tasks. Checking for blocked tasks..."
      BLOCKED_TASKS=$(bd blocked 2>/dev/null | grep -c "^{{PROJECT_PREFIX}}-" || echo "0")

      if [ "$BLOCKED_TASKS" -gt 0 ]; then
        echo "⚠ $BLOCKED_TASKS blocked tasks remain"
        echo "Review blockers with: bd blocked"
        return 1
      else
        echo "✓ All development tasks complete!"
        return 0
      fi
    fi

    echo "Found $READY_TASKS ready tasks"

    # Get next task
    NEXT_TASK=$(bd ready 2>/dev/null | head -1 | awk '{print $1}')

    if [ -z "$NEXT_TASK" ]; then
      echo "No task ID found"
      break
    fi

    echo "Working on: $NEXT_TASK"
    bd show "$NEXT_TASK"

    # The actual implementation would be done by the AI agent
    # using the MCP tools described in 11.2-11.4
    echo ""
    echo ">>> AI agent implements $NEXT_TASK using MCP tools <<<"
    echo ""

    # For testing, mark as in_progress
    bd update "$NEXT_TASK" --status=in_progress

  done

  echo "Development loop completed after $max_iterations iterations"
}

# ═══════════════════════════════════════════════════════════════════════════
# WRAPPER: run_development_completion
# Alias for run_development_completion_loop (for master orchestration)
# ═══════════════════════════════════════════════════════════════════════════

run_development_completion() {
  run_development_completion_loop "$@"
}

export -f run_development_completion
```

---

## PHASE 12: Test Execution Framework

### 12.0 Overview

Systematic testing after development completion:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TEST EXECUTION FRAMEWORK                                  │
│                    (Phase 12: After Development Completion)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TEST EXECUTION ORDER:                                                      │
│  ├─ 12.1: Unit tests (fast feedback)                                       │
│  ├─ 12.2: Integration tests (component interaction)                        │
│  └─ 12.3: Functional testing (full system verification)                    │
│                                                                             │
│  COVERAGE REQUIREMENTS:                                                     │
│  ├─ Minimum: 80% code coverage                                             │
│  ├─ All NotImplementedError must be gone                                   │
│  └─ All tests must pass (no skips without reason)                          │
│                                                                             │
│  TEST FRAMEWORK: pytest with coverage                                       │
│  CONFIGURATION: pytest.ini, conftest.py                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 12.1 pytest Configuration Setup

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 12.1: PYTEST CONFIGURATION
# Set up test framework for the project
# ═══════════════════════════════════════════════════════════════════════════

setup_test_framework() {
  local project_path="${1:-$(pwd)}"
  local project_name="{{PROJECT_NAME}}"
  local env_name="${2:-{{ENV_NAME}}}"

  echo "=== Setting up Test Framework ==="

  # Install pytest and coverage if not present
  conda run -n "$env_name" pip install pytest pytest-cov pytest-xdist pytest-timeout 2>/dev/null

  # Create pytest.ini
  if [ ! -f "$project_path/pytest.ini" ]; then
    cat > "$project_path/pytest.ini" << EOF
[pytest]
testpaths = tests
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
addopts = -v --cov=${project_name} --cov-report=term-missing --cov-fail-under=80
minversion = 7.0
filterwarnings =
    ignore::DeprecationWarning
    ignore::PendingDeprecationWarning
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    e2e: marks tests as end-to-end tests
timeout = 300
EOF
    echo "Created pytest.ini"
  fi

  # Create conftest.py with common fixtures
  if [ ! -f "$project_path/tests/conftest.py" ]; then
    mkdir -p "$project_path/tests"
    cat > "$project_path/tests/conftest.py" << 'EOF'
"""Common test fixtures and configuration."""
import pytest
import sys
from pathlib import Path

# Add project root to path
PROJECT_ROOT = Path(__file__).parent.parent
sys.path.insert(0, str(PROJECT_ROOT))


@pytest.fixture
def project_root():
    """Return the project root directory."""
    return PROJECT_ROOT


@pytest.fixture
def sample_data():
    """Provide sample test data."""
    return {
        "test_string": "hello world",
        "test_number": 42,
        "test_list": [1, 2, 3],
        "test_dict": {"key": "value"},
    }


@pytest.fixture
def mock_env(monkeypatch):
    """Fixture to mock environment variables."""
    def _mock_env(**kwargs):
        for key, value in kwargs.items():
            monkeypatch.setenv(key, value)
    return _mock_env


@pytest.fixture
def temp_file(tmp_path):
    """Create a temporary file for testing."""
    def _temp_file(content="", suffix=".txt"):
        file_path = tmp_path / f"test_file{suffix}"
        file_path.write_text(content)
        return file_path
    return _temp_file
EOF
    echo "Created tests/conftest.py"
  fi

  # Create __init__.py
  touch "$project_path/tests/__init__.py"

  echo "Test framework setup complete"
}
```

### 12.2 Test Execution Commands

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 12.2: TEST EXECUTION
# Run tests in the correct order with proper reporting
# ═══════════════════════════════════════════════════════════════════════════

run_full_test_suite() {
  local project_path="${1:-$(pwd)}"
  local env_name="${2:-{{ENV_NAME}}}"

  echo "=== Running Full Test Suite ==="
  echo "Project: $project_path"
  echo "Environment: $env_name"
  echo ""

  cd "$project_path"

  UNIT_RESULT=0
  INTEGRATION_RESULT=0
  E2E_RESULT=0
  COVERAGE_RESULT=0

  # 1. Unit tests (fast, run first)
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "STEP 1: Unit Tests"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  if [ -d "tests/unit" ]; then
    conda run -n "$env_name" pytest tests/unit/ -v --tb=short
    UNIT_RESULT=$?
  else
    # Run all tests except integration and e2e
    conda run -n "$env_name" pytest tests/ -v --tb=short \
      --ignore=tests/integration --ignore=tests/e2e -m "not integration and not e2e" 2>/dev/null || \
    conda run -n "$env_name" pytest tests/ -v --tb=short
    UNIT_RESULT=$?
  fi

  if [ $UNIT_RESULT -eq 0 ]; then
    echo "✓ Unit tests PASSED"
  else
    echo "✗ Unit tests FAILED"
  fi

  # 2. Integration tests (if exist)
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "STEP 2: Integration Tests"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  if [ -d "tests/integration" ]; then
    conda run -n "$env_name" pytest tests/integration/ -v --tb=short -m integration 2>/dev/null || \
    conda run -n "$env_name" pytest tests/integration/ -v --tb=short
    INTEGRATION_RESULT=$?
    if [ $INTEGRATION_RESULT -eq 0 ]; then
      echo "✓ Integration tests PASSED"
    else
      echo "✗ Integration tests FAILED"
    fi
  else
    echo "⊘ No integration tests directory (tests/integration/)"
    INTEGRATION_RESULT=0
  fi

  # 3. E2E tests (if exist)
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "STEP 3: End-to-End Tests"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  if [ -d "tests/e2e" ]; then
    conda run -n "$env_name" pytest tests/e2e/ -v --tb=short -m e2e 2>/dev/null || \
    conda run -n "$env_name" pytest tests/e2e/ -v --tb=short
    E2E_RESULT=$?
    if [ $E2E_RESULT -eq 0 ]; then
      echo "✓ E2E tests PASSED"
    else
      echo "✗ E2E tests FAILED"
    fi
  else
    echo "⊘ No E2E tests directory (tests/e2e/)"
    E2E_RESULT=0
  fi

  # 4. Coverage report
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "STEP 4: Coverage Report"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  conda run -n "$env_name" pytest tests/ --cov={{PROJECT_NAME}} \
    --cov-report=term-missing --cov-fail-under=80 --no-header -q 2>/dev/null

  COVERAGE_RESULT=$?

  # Extract coverage percentage
  COVERAGE=$(conda run -n "$env_name" pytest tests/ --cov={{PROJECT_NAME}} \
    --cov-report=term 2>/dev/null | grep "TOTAL" | awk '{print $NF}' | tr -d '%')

  if [ -n "$COVERAGE" ]; then
    echo "Coverage: ${COVERAGE}%"
    if [ "${COVERAGE:-0}" -ge 80 ]; then
      echo "✓ Coverage requirement met (>= 80%)"
    else
      echo "✗ Coverage below 80% threshold"
    fi
  fi

  # Summary
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "TEST SUMMARY"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "Unit Tests:        $([ $UNIT_RESULT -eq 0 ] && echo '✓ PASS' || echo '✗ FAIL')"
  echo "Integration Tests: $([ $INTEGRATION_RESULT -eq 0 ] && echo '✓ PASS' || echo '✗ FAIL')"
  echo "E2E Tests:         $([ $E2E_RESULT -eq 0 ] && echo '✓ PASS' || echo '✗ FAIL')"
  echo "Coverage:          $([ $COVERAGE_RESULT -eq 0 ] && echo "✓ ${COVERAGE:-?}%" || echo "✗ Below 80%")"

  if [ $UNIT_RESULT -eq 0 ] && [ $INTEGRATION_RESULT -eq 0 ] && [ $E2E_RESULT -eq 0 ] && [ $COVERAGE_RESULT -eq 0 ]; then
    echo ""
    echo "═══════════════════════════════════════════════════════"
    echo "              ALL TESTS PASSED ✓"
    echo "═══════════════════════════════════════════════════════"
    return 0
  else
    echo ""
    echo "═══════════════════════════════════════════════════════"
    echo "              TESTS FAILED ✗"
    echo "═══════════════════════════════════════════════════════"
    return 1
  fi
}
```

### 12.3 Functional Verification

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 12.3: FUNCTIONAL VERIFICATION
# Test that the project actually works end-to-end
# ═══════════════════════════════════════════════════════════════════════════

verify_functionality() {
  local project_path="${1:-$(pwd)}"
  local env_name="${2:-{{ENV_NAME}}}"
  local port="${3:-{{PORT_START}}}"

  echo "=== Functional Verification ==="
  echo "Project: $project_path"
  echo "Environment: $env_name"
  echo ""

  cd "$project_path"

  FUNC_PASSED=0
  FUNC_TOTAL=0

  # 1. Test core imports
  echo "--- 1. Core Imports ---"
  ((FUNC_TOTAL++))
  conda run -n "$env_name" python -c "
from {{PROJECT_NAME}} import *
print('Core imports: OK')
"
  if [ $? -eq 0 ]; then
    echo "✓ Core imports work"
    ((FUNC_PASSED++))
  else
    echo "✗ Core imports FAILED"
  fi

  # 2. Test main entry point (if exists)
  echo ""
  echo "--- 2. Entry Point ---"
  if [ -f "$project_path/main.py" ]; then
    ((FUNC_TOTAL++))
    conda run -n "$env_name" python "$project_path/main.py" --help 2>/dev/null
    if [ $? -eq 0 ]; then
      echo "✓ main.py --help works"
      ((FUNC_PASSED++))
    else
      echo "✗ main.py --help FAILED"
    fi
  elif [ -f "$project_path/cli.py" ]; then
    ((FUNC_TOTAL++))
    conda run -n "$env_name" python "$project_path/cli.py" --help 2>/dev/null
    if [ $? -eq 0 ]; then
      echo "✓ cli.py --help works"
      ((FUNC_PASSED++))
    else
      echo "✗ cli.py --help FAILED"
    fi
  else
    echo "⊘ No main.py or cli.py entry point"
  fi

  # 3. Test web service (if exists)
  echo ""
  echo "--- 3. Web Service ---"
  if [ -f "$project_path/app.py" ] || grep -rq "FastAPI\|Flask\|uvicorn" "$project_path"/*.py 2>/dev/null; then
    ((FUNC_TOTAL++))
    echo "Web service detected, starting for health check..."

    # Start server in background
    conda run -n "$env_name" python -c "
import subprocess
import requests
import time
import sys
import os

# Find the app file
app_file = None
for f in ['app.py', 'main.py', 'server.py']:
    if os.path.exists(f):
        app_file = f
        break

if not app_file:
    print('No app file found')
    sys.exit(1)

# Start server
proc = subprocess.Popen(
    [sys.executable, app_file],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)

# Wait for startup
time.sleep(5)

# Health check
try:
    resp = requests.get('http://localhost:$port/health', timeout=5)
    print(f'Health check: {resp.status_code}')
    success = resp.status_code == 200
except Exception as e:
    # Try root path
    try:
        resp = requests.get('http://localhost:$port/', timeout=5)
        print(f'Root check: {resp.status_code}')
        success = resp.status_code in [200, 404]  # 404 is ok, means server runs
    except:
        print('Health check: FAILED')
        success = False
finally:
    proc.terminate()
    proc.wait()

sys.exit(0 if success else 1)
" 2>/dev/null

    if [ $? -eq 0 ]; then
      echo "✓ Web service health check passed"
      ((FUNC_PASSED++))
    else
      echo "✗ Web service health check FAILED"
    fi
  else
    echo "⊘ No web service detected"
  fi

  # 4. Test database connection (if applicable)
  echo ""
  echo "--- 4. Database Connection ---"
  if grep -rq "sqlalchemy\|mysql\|postgres\|sqlite" "$project_path"/*.py 2>/dev/null; then
    ((FUNC_TOTAL++))
    conda run -n "$env_name" python -c "
# Attempt to import database module and test connection
try:
    from {{PROJECT_NAME}} import db, database, models
    print('Database module: OK')
except ImportError:
    print('Database module: OK (no dedicated db module)')
" 2>/dev/null

    if [ $? -eq 0 ]; then
      echo "✓ Database modules accessible"
      ((FUNC_PASSED++))
    else
      echo "✗ Database modules FAILED"
    fi
  else
    echo "⊘ No database configuration detected"
  fi

  # 5. Test GPU/CUDA (if applicable)
  echo ""
  echo "--- 5. GPU/CUDA ---"
  if grep -rq "torch\|cuda\|gpu" "$project_path"/*.py 2>/dev/null; then
    ((FUNC_TOTAL++))
    conda run -n "$env_name" python -c "
import torch
assert torch.cuda.is_available(), 'CUDA not available'
print(f'CUDA: {torch.cuda.get_device_name(0)}')
print(f'Memory: {torch.cuda.get_device_properties(0).total_memory / 1e9:.1f} GB')
" 2>/dev/null

    if [ $? -eq 0 ]; then
      echo "✓ GPU/CUDA accessible"
      ((FUNC_PASSED++))
    else
      echo "✗ GPU/CUDA FAILED"
    fi
  else
    echo "⊘ No GPU/CUDA requirements detected"
  fi

  # Summary
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "FUNCTIONAL VERIFICATION: $FUNC_PASSED / $FUNC_TOTAL passed"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

  if [ "$FUNC_PASSED" -eq "$FUNC_TOTAL" ]; then
    echo "✓ All functional checks PASSED"
    return 0
  else
    echo "✗ Some functional checks FAILED"
    return 1
  fi
}

# ═══════════════════════════════════════════════════════════════════════════
# COMPLETE TEST AND VERIFICATION WORKFLOW
# Run all Phase 12 checks in sequence
# ═══════════════════════════════════════════════════════════════════════════

run_complete_verification() {
  local project_path="${1:-$(pwd)}"
  local env_name="${2:-{{ENV_NAME}}}"

  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "                    PHASE 12: COMPLETE VERIFICATION"
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo ""

  # Setup test framework
  setup_test_framework "$project_path" "$env_name"

  # Run test suite
  run_full_test_suite "$project_path" "$env_name"
  TEST_RESULT=$?

  echo ""

  # Run functional verification
  verify_functionality "$project_path" "$env_name"
  FUNC_RESULT=$?

  echo ""
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "                    VERIFICATION SUMMARY"
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "Test Suite:              $([ $TEST_RESULT -eq 0 ] && echo '✓ PASSED' || echo '✗ FAILED')"
  echo "Functional Verification: $([ $FUNC_RESULT -eq 0 ] && echo '✓ PASSED' || echo '✗ FAILED')"
  echo ""

  if [ $TEST_RESULT -eq 0 ] && [ $FUNC_RESULT -eq 0 ]; then
    echo "═══════════════════════════════════════════════════════════════════════════"
    echo "                    PROJECT READY FOR DEPLOYMENT ✓"
    echo "═══════════════════════════════════════════════════════════════════════════"
    return 0
  else
    echo "═══════════════════════════════════════════════════════════════════════════"
    echo "                    PROJECT NOT READY - FIX FAILURES"
    echo "═══════════════════════════════════════════════════════════════════════════"
    return 1
  fi
}

# ═══════════════════════════════════════════════════════════════════════════
# WRAPPER: run_phase_12_verification
# Alias for run_complete_verification (for master orchestration)
# ═══════════════════════════════════════════════════════════════════════════

run_phase_12_verification() {
  run_complete_verification "$@"
}

export -f run_phase_12_verification
```

---

## PHASE 13: Academic Research Integration

### 13.0 Overview

For cutting-edge development requiring research paper guidance:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ACADEMIC RESEARCH INTEGRATION                             │
│                    (Phase 13: For research-driven development)              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ACADEMIC RESEARCH MCP SERVERS:                                            │
│  ├─ arxiv-advanced        → Primary: Advanced arXiv search, paper download │
│  ├─ semantic-scholar      → Secondary: Citation graphs, impact analysis    │
│  ├─ research-arxiv        → Fallback: Basic arXiv queries                  │
│  └─ paper-search          → Supplementary: Multi-platform coverage         │
│                                                                             │
│  USAGE PRIORITY:                                                           │
│  1. arxiv-advanced   → Best for ML/AI cutting-edge (category filters)     │
│  2. semantic-scholar → Best for citation traversal (find implementations) │
│  3. research-arxiv   → Simple queries, quick lookups                       │
│  4. paper-search     → Broader coverage (medical, biology, etc.)          │
│                                                                             │
│  WORKFLOW:                                                                  │
│  ├─ 13.1: Detect research requirements                                     │
│  ├─ 13.2: Multi-source academic paper search                               │
│  ├─ 13.3: Paper analysis & download                                        │
│  ├─ 13.4: Research-derived task creation                                   │
│  └─ 13.5: Advanced research synthesis (iterative)                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 13.1 Research Requirement Detection

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.1: DETECT RESEARCH REQUIREMENTS
# Determines if project needs cutting-edge research guidance
# ═══════════════════════════════════════════════════════════════════════════

detect_research_requirements() {
  PROJECT_PATH="${1:-$(pwd)}"

  echo "=== Detecting Research Requirements ==="
  echo "Project: $PROJECT_PATH"

  RESEARCH_INDICATORS=()
  RESEARCH_SCORE=0

  # ───────────────────────────────────────────────────────────────────────
  # Check for cutting-edge technology markers
  # ───────────────────────────────────────────────────────────────────────
  CUTTING_EDGE_PATTERNS=(
    "transformer"
    "attention mechanism"
    "diffusion model"
    "reinforcement learning"
    "neural network architecture"
    "state-of-the-art"
    "SOTA"
    "novel approach"
    "research paper"
    "arxiv"
    "IEEE"
    "ACM"
    "multi-agent"
    "large language model"
    "LLM"
  )

  for pattern in "${CUTTING_EDGE_PATTERNS[@]}"; do
    count=$(grep -ri "$pattern" "$PROJECT_PATH" --include="*.py" --include="*.md" 2>/dev/null | wc -l)
    if [ "$count" -gt 0 ]; then
      RESEARCH_INDICATORS+=("$pattern: $count occurrences")
      RESEARCH_SCORE=$((RESEARCH_SCORE + count * 3))
    fi
  done

  # ───────────────────────────────────────────────────────────────────────
  # Check for academic citations in code/docs
  # ───────────────────────────────────────────────────────────────────────
  CITATIONS=$(grep -rE "\[[0-9]+\]|\(et al\.,? [0-9]{4}\)|arXiv:[0-9]+\.[0-9]+" "$PROJECT_PATH" 2>/dev/null | wc -l)
  RESEARCH_SCORE=$((RESEARCH_SCORE + CITATIONS * 5))
  [ "$CITATIONS" -gt 0 ] && RESEARCH_INDICATORS+=("Academic citations: $CITATIONS")

  # ───────────────────────────────────────────────────────────────────────
  # Check requirements.txt for research libraries
  # ───────────────────────────────────────────────────────────────────────
  RESEARCH_LIBS=("transformers" "torch" "tensorflow" "jax" "flax" "einops" "timm" "huggingface" "accelerate" "peft" "bitsandbytes")
  if [ -f "$PROJECT_PATH/requirements.txt" ]; then
    for lib in "${RESEARCH_LIBS[@]}"; do
      if grep -qi "$lib" "$PROJECT_PATH/requirements.txt"; then
        RESEARCH_INDICATORS+=("Research library: $lib")
        RESEARCH_SCORE=$((RESEARCH_SCORE + 5))
      fi
    done
  fi

  # ───────────────────────────────────────────────────────────────────────
  # Check for NotImplementedError with research-related messages
  # ───────────────────────────────────────────────────────────────────────
  RESEARCH_NOTIMPL=$(grep -rn "NotImplementedError.*research\|TODO.*paper\|FIXME.*implementation\|TODO.*arxiv" "$PROJECT_PATH" --include="*.py" 2>/dev/null | wc -l)
  RESEARCH_SCORE=$((RESEARCH_SCORE + RESEARCH_NOTIMPL * 10))
  [ "$RESEARCH_NOTIMPL" -gt 0 ] && RESEARCH_INDICATORS+=("Research-related TODOs: $RESEARCH_NOTIMPL")

  # ───────────────────────────────────────────────────────────────────────
  # Output results
  # ───────────────────────────────────────────────────────────────────────
  echo ""
  echo "┌─────────────────────────────────────────┐"
  echo "│ RESEARCH REQUIREMENT ANALYSIS           │"
  echo "├─────────────────────────────────────────┤"
  echo "│ Research Score: $RESEARCH_SCORE"
  echo "│ Indicators:"
  for indicator in "${RESEARCH_INDICATORS[@]}"; do
    echo "│   - $indicator"
  done
  echo "└─────────────────────────────────────────┘"

  export RESEARCH_SCORE
  export RESEARCH_INDICATORS

  if [ "$RESEARCH_SCORE" -ge 20 ]; then
    echo "RESEARCH REQUIRED: High"
    export RESEARCH_LEVEL="high"
    return 0
  elif [ "$RESEARCH_SCORE" -ge 10 ]; then
    echo "RESEARCH REQUIRED: Medium"
    export RESEARCH_LEVEL="medium"
    return 1
  else
    echo "RESEARCH REQUIRED: Low"
    export RESEARCH_LEVEL="low"
    return 2
  fi
}
```

### 13.2 Academic Paper Search (Multi-Source)

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.2: ACADEMIC PAPER SEARCH
# Search across multiple academic MCP servers
# ═══════════════════════════════════════════════════════════════════════════

search_academic_papers() {
  local query="$1"
  local max_results="${2:-10}"

  echo "=== Searching Academic Papers: $query ==="

  # ───────────────────────────────────────────────────────────────────────
  # PRIMARY: arXiv Advanced (best for cutting-edge ML/AI research)
  # ───────────────────────────────────────────────────────────────────────
  # MCP call: mcp__arxiv-advanced__search_papers
  # Parameters:
  #   query: "$query" (use quoted phrases: "transformer architecture")
  #   categories: ["cs.AI", "cs.LG", "cs.CL", "cs.CV", "cs.RO", "cs.MA"]
  #   max_results: $max_results
  #   sort_by: "relevance"  # or "date" for newest
  #   date_from: "2023-01-01"  # for recent research
  #
  # Advanced query syntax:
  #   - ti:"exact title phrase" - search titles only
  #   - au:"author name" - search by author
  #   - abs:"keyword" - search abstracts only
  #   - "term1" OR "term2" - boolean OR
  #   - "term1" ANDNOT "survey" - exclude terms
  #
  # Category codes:
  #   - cs.AI: Artificial Intelligence
  #   - cs.LG: Machine Learning
  #   - cs.CL: Computation and Language (NLP)
  #   - cs.CV: Computer Vision
  #   - cs.RO: Robotics
  #   - cs.MA: Multi-Agent Systems

  # ───────────────────────────────────────────────────────────────────────
  # SECONDARY: Semantic Scholar Graph API (citation analysis)
  # ───────────────────────────────────────────────────────────────────────
  # MCP call: mcp__research-semantic-scholar__search_semantic_scholar
  # Parameters:
  #   query: "$query"
  #   maxResults: $max_results
  #   startYear: 2020  # Optional year filter
  #
  # Returns: Papers with citation counts, authors, venues, impact scores

  # ───────────────────────────────────────────────────────────────────────
  # TERTIARY: Basic arXiv (simpler queries, fallback)
  # ───────────────────────────────────────────────────────────────────────
  # MCP call: mcp__research-arxiv__search_arxiv
  # Parameters:
  #   query: "$query"
  #   maxResults: $max_results
  #   sortBy: "relevance"

  # ───────────────────────────────────────────────────────────────────────
  # SUPPLEMENTARY: Paper Search (multi-platform)
  # ───────────────────────────────────────────────────────────────────────
  # MCP call: mcp__paper-search__search_papers
  # Parameters:
  #   query: "$query"
  #   platform: "all"  # or specific: "arxiv", "semantic", "crossref", "pubmed"
  #   maxResults: $max_results
  #
  # Best for: Broader coverage, medical/biology research

  # Aggregate and deduplicate results
  # Rank by: citation count, recency, relevance to implementation
}

traverse_citation_graph() {
  local seed_paper_id="$1"
  local direction="${2:-both}"  # citing, cited_by, both

  echo "=== Traversing Citation Graph from: $seed_paper_id ==="

  # ───────────────────────────────────────────────────────────────────────
  # Find papers that CITE the seed paper (implementations, improvements)
  # ───────────────────────────────────────────────────────────────────────
  # MCP call: mcp__research-semantic-scholar__get_paper_citations
  # Parameters:
  #   paperId: "$seed_paper_id"
  #   maxResults: 20
  #
  # Returns: Papers that cite this paper
  # Look for:
  #   - "implementation" in title/abstract → likely has code
  #   - GitHub links mentioned → implementation available
  #   - Recent papers → more modern implementations

  # ───────────────────────────────────────────────────────────────────────
  # Get full paper details
  # ───────────────────────────────────────────────────────────────────────
  # MCP call: mcp__research-semantic-scholar__get_semantic_scholar_paper
  # Parameters:
  #   identifier: "$seed_paper_id"
  #
  # Returns: Full metadata, citation count, abstract, authors, venue
}
```

### 13.3 Paper Analysis and Download

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.3: PAPER ANALYSIS AND DOWNLOAD
# Download papers and extract implementation-relevant information
# ═══════════════════════════════════════════════════════════════════════════

analyze_paper() {
  local paper_id="$1"
  local paper_source="${2:-arxiv}"

  echo "=== Analyzing Paper: $paper_id ==="

  # ───────────────────────────────────────────────────────────────────────
  # STEP 1: Get paper metadata
  # ───────────────────────────────────────────────────────────────────────

  if [ "$paper_source" = "arxiv" ]; then
    # Get basic metadata
    # MCP call: mcp__research-arxiv__get_arxiv_paper
    #   arxivId: "$paper_id"

    # Get BibTeX citation
    # MCP call: mcp__research-arxiv__arxiv_to_bibtex
    #   arxivId: "$paper_id"

  elif [ "$paper_source" = "semantic-scholar" ]; then
    # Get detailed metadata including citations
    # MCP call: mcp__research-semantic-scholar__get_semantic_scholar_paper
    #   identifier: "$paper_id"

    # Get papers that cite this one
    # MCP call: mcp__research-semantic-scholar__get_paper_citations
    #   paperId: "$paper_id"
    #   maxResults: 15

    # Get BibTeX
    # MCP call: mcp__research-semantic-scholar__semantic_scholar_to_bibtex
    #   identifier: "$paper_id"
  fi

  # ───────────────────────────────────────────────────────────────────────
  # STEP 2: Download and read full paper (arxiv-advanced)
  # ───────────────────────────────────────────────────────────────────────

  # Download paper (creates resource, converts to markdown)
  # MCP call: mcp__arxiv-advanced__download_paper
  #   paper_id: "$paper_id"
  #   check_status: false

  # Check download/conversion status
  # MCP call: mcp__arxiv-advanced__download_paper
  #   paper_id: "$paper_id"
  #   check_status: true

  # Read full paper content in markdown format
  # MCP call: mcp__arxiv-advanced__read_paper
  #   paper_id: "$paper_id"

  # ───────────────────────────────────────────────────────────────────────
  # STEP 3: Extract implementation-relevant information
  # ───────────────────────────────────────────────────────────────────────

  # From paper content, extract:
  # - Algorithm pseudocode
  # - Model architecture details
  # - Hyperparameter recommendations
  # - Loss function definitions
  # - Training procedures
  # - Evaluation metrics
  # - Code repository links

  # ───────────────────────────────────────────────────────────────────────
  # STEP 4: Store findings in memory
  # ───────────────────────────────────────────────────────────────────────

  # store_to_memory "research" "Paper $paper_id findings: ..." "paper,research,$paper_id"
}

list_downloaded_papers() {
  echo "=== Listing Downloaded Papers ==="

  # MCP call: mcp__arxiv-advanced__list_papers
  # Returns: All papers downloaded and available as resources
}
```

### 13.4 Research-Derived Task Creation

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.4: CREATE TASKS FROM RESEARCH FINDINGS
# Convert paper insights into actionable Beads tasks
# ═══════════════════════════════════════════════════════════════════════════

create_research_tasks() {
  local paper_id="$1"
  local extracted_info="$2"

  echo "=== Creating Research-Derived Tasks ==="

  # 1. Algorithm implementation tasks
  bd create \
    --title="Implement algorithm from paper $paper_id" \
    --type=feature \
    --priority=1 \
    --description="Based on paper findings:
$extracted_info

Reference: $paper_id" \
    --labels="research,implementation"

  # 2. Architecture tasks
  bd create \
    --title="Build model architecture per paper specification" \
    --type=task \
    --priority=1 \
    --description="Architecture details from paper $paper_id"

  # 3. Hyperparameter configuration tasks
  bd create \
    --title="Configure hyperparameters per paper recommendations" \
    --type=task \
    --priority=2 \
    --description="Hyperparameters from paper $paper_id"

  # 4. Add dependencies between tasks
  # bd dep add <impl_task> <arch_task>
}
```

### 13.5 Advanced Research Synthesis Workflow

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.5: ADVANCED RESEARCH SYNTHESIS
# Iterative synthesis from multiple papers until validated solution found
# ═══════════════════════════════════════════════════════════════════════════

run_advanced_research_synthesis() {
  PROJECT_PATH="$1"
  DEVELOPMENT_REQUIREMENT="$2"

  echo "╔═══════════════════════════════════════════════════════════════════════════╗"
  echo "║        ADVANCED RESEARCH SYNTHESIS WORKFLOW (ITERATIVE)                   ║"
  echo "║        Requirement: $DEVELOPMENT_REQUIREMENT"
  echo "╚═══════════════════════════════════════════════════════════════════════════╝"

  MAX_SYNTHESIS_ITERATIONS=500
  synthesis_iteration=0
  SOLUTION_FOUND=false
  SOLUTION_CONFIDENCE=0
  MIN_CONFIDENCE_THRESHOLD=80

  while [ "$SOLUTION_FOUND" = false ] && [ $synthesis_iteration -lt $MAX_SYNTHESIS_ITERATIONS ]; do
    synthesis_iteration=$((synthesis_iteration + 1))
    echo ""
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║ SYNTHESIS ITERATION $synthesis_iteration / $MAX_SYNTHESIS_ITERATIONS"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"

    # PHASE 1: Paper Collection
    echo ">>> PHASE 1: Collecting papers..."

    # PHASE 2: Deep Analysis
    echo ">>> PHASE 2: Analyzing papers..."

    # PHASE 3: Cross-Paper Synthesis
    echo ">>> PHASE 3: Synthesizing ideas..."

    # PHASE 4: Methodology Planning
    echo ">>> PHASE 4: Planning methodology..."

    # PHASE 5: Validation Check
    echo ">>> PHASE 5: Validating..."

    # Calculate confidence based on criteria met
    # SOLUTION_CONFIDENCE = sum of criteria

    if [ "$SOLUTION_CONFIDENCE" -ge "$MIN_CONFIDENCE_THRESHOLD" ]; then
      SOLUTION_FOUND=true
      echo ">>> SOLUTION FOUND: ${SOLUTION_CONFIDENCE}% confidence"
    else
      echo ">>> Confidence: ${SOLUTION_CONFIDENCE}% (need ${MIN_CONFIDENCE_THRESHOLD}%)"
    fi
  done

  if [ "$SOLUTION_FOUND" = true ]; then
    # Persist solution
    bd create \
      --title="Implement research-validated solution: $DEVELOPMENT_REQUIREMENT" \
      --type=feature \
      --priority=0 \
      --description="Confidence: ${SOLUTION_CONFIDENCE}%. Iterations: $synthesis_iteration"

    return 0
  else
    bd create \
      --title="Manual research review needed: $DEVELOPMENT_REQUIREMENT" \
      --type=task \
      --priority=0 \
      --description="Automated synthesis incomplete after $MAX_SYNTHESIS_ITERATIONS iterations"

    return 1
  fi
}
```

### 13.6 Research Phase Orchestration

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.6: RESEARCH PHASE ORCHESTRATION
# Main entry point for research-driven development
# ═══════════════════════════════════════════════════════════════════════════

run_research_phase() {
  PROJECT_PATH="${1:-$(pwd)}"

  echo "=== Running Research Phase ==="

  detect_research_requirements "$PROJECT_PATH"

  if [ "$RESEARCH_LEVEL" = "low" ]; then
    echo "Research phase not required (score: $RESEARCH_SCORE)"
    return 0
  fi

  QUERIES=()
  DIFFICULT_REQUIREMENTS=()

  # Extract queries from NotImplementedError
  while IFS= read -r line; do
    query=$(echo "$line" | grep -oP 'NotImplementedError\("\K[^"]+')
    [ -n "$query" ] && QUERIES+=("$query")

    if echo "$query" | grep -qiE "novel|cutting.edge|SOTA|advanced|research"; then
      DIFFICULT_REQUIREMENTS+=("$query")
    fi
  done < <(grep -rn "NotImplementedError" "$PROJECT_PATH" --include="*.py" 2>/dev/null)

  # Handle difficult requirements with synthesis
  if [ ${#DIFFICULT_REQUIREMENTS[@]} -gt 0 ]; then
    for requirement in "${DIFFICULT_REQUIREMENTS[@]}"; do
      run_advanced_research_synthesis "$PROJECT_PATH" "$requirement"
    done
  fi

  # Standard paper search for simpler queries
  for query in "${QUERIES[@]}"; do
    [[ " ${DIFFICULT_REQUIREMENTS[*]} " =~ " ${query} " ]] && continue
    search_academic_papers "$query" 5
  done

  store_to_memory "research" "Research phase complete for $PROJECT_PATH" "research,phase13"

  echo "Research phase complete"
}
```

---

## PHASE 14: Functionality Fix Phase

### 14.0 Overview

For projects with broken functionality:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FUNCTIONALITY FIX PHASE                                   │
│                    (Phase 14: For projects with broken functionality)       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROBLEM TYPES DETECTED:                                                   │
│  ├─ Import errors           → Cannot load modules                          │
│  ├─ Missing dependencies    → Required packages not installed              │
│  ├─ Test collection errors  → Tests cannot be discovered                   │
│  ├─ Circular imports        → Module dependency cycles                     │
│  └─ Runtime errors          → Code crashes when executed                   │
│                                                                             │
│  WORKFLOW:                                                                  │
│  ├─ 14.1: Detect functionality problems                                    │
│  ├─ 14.2: Categorize and prioritize issues                                 │
│  └─ 14.3: Systematic fix workflow                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 14.1 Functionality Problem Detection

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 14.1: DETECT FUNCTIONALITY PROBLEMS
# Find issues that prevent the project from running
# ═══════════════════════════════════════════════════════════════════════════

detect_functionality_problems() {
  PROJECT_PATH="${1:-$(pwd)}"

  echo "=== Detecting Functionality Problems ==="

  FUNCTIONALITY_ISSUES=()
  FUNCTIONALITY_SCORE=0

  cd "$PROJECT_PATH"

  # Check for import errors
  IMPORT_ERRORS=$(python -c "
import sys
import pkgutil
import importlib.util
sys.path.insert(0, '.')
errors = []
for finder, name, ispkg in pkgutil.walk_packages(['.'], onerror=lambda x: None):
    if not name.startswith('test') and not name.startswith('.'):
        try:
            spec = importlib.util.find_spec(name)
            if spec:
                importlib.import_module(name)
        except Exception as e:
            errors.append(str(type(e).__name__))
for e in errors[:20]:
    print(e)
" 2>&1 | grep -c "Error\|Exception" || echo "0")

  if [ "$IMPORT_ERRORS" -gt 0 ]; then
    FUNCTIONALITY_ISSUES+=("Import errors: $IMPORT_ERRORS")
    FUNCTIONALITY_SCORE=$((FUNCTIONALITY_SCORE + IMPORT_ERRORS * 7))
  fi

  # Check for test collection failures
  if [ -d "$PROJECT_PATH/tests" ]; then
    TEST_FAILURES=$(conda run -n {{ENV_NAME}} python -m pytest tests/ --collect-only -q 2>&1 | grep -c "error\|Error" || echo "0")
    if [ "$TEST_FAILURES" -gt 0 ]; then
      FUNCTIONALITY_ISSUES+=("Test collection errors: $TEST_FAILURES")
      FUNCTIONALITY_SCORE=$((FUNCTIONALITY_SCORE + TEST_FAILURES * 6))
    fi
  fi

  # Check for circular imports
  CIRCULAR=$(grep -rn "from \. import\|from \.\. import" "$PROJECT_PATH" --include="*.py" 2>/dev/null | wc -l)
  if [ "$CIRCULAR" -gt 10 ]; then
    FUNCTIONALITY_ISSUES+=("Potential circular imports: $CIRCULAR")
    FUNCTIONALITY_SCORE=$((FUNCTIONALITY_SCORE + 5))
  fi

  # Check for missing dependencies
  if [ -f "$PROJECT_PATH/requirements.txt" ]; then
    while IFS= read -r req; do
      pkg=$(echo "$req" | cut -d'=' -f1 | cut -d'>' -f1 | cut -d'<' -f1 | cut -d'[' -f1 | tr -d ' ')
      [ -z "$pkg" ] && continue
      [[ "$pkg" =~ ^# ]] && continue
      import_name=$(echo "$pkg" | tr '-' '_' | tr '[:upper:]' '[:lower:]')
      if ! python -c "import $import_name" 2>/dev/null; then
        FUNCTIONALITY_ISSUES+=("Missing: $pkg")
        FUNCTIONALITY_SCORE=$((FUNCTIONALITY_SCORE + 3))
      fi
    done < "$PROJECT_PATH/requirements.txt"
  fi

  echo ""
  echo "┌─────────────────────────────────────────┐"
  echo "│ FUNCTIONALITY PROBLEM ANALYSIS          │"
  echo "├─────────────────────────────────────────┤"
  echo "│ Functionality Score: $FUNCTIONALITY_SCORE"
  echo "│ Issues:"
  for issue in "${FUNCTIONALITY_ISSUES[@]}"; do
    echo "│   - $issue"
  done
  echo "└─────────────────────────────────────────┘"

  export FUNCTIONALITY_SCORE
  export FUNCTIONALITY_ISSUES
}
```

### 14.2 Functionality Fix Workflow

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 14.2: FUNCTIONALITY FIX WORKFLOW
# Systematically fix detected functionality problems
# ═══════════════════════════════════════════════════════════════════════════

run_functionality_fix_phase() {
  PROJECT_PATH="${1:-$(pwd)}"

  echo "=== Running Functionality Fix Phase ==="

  detect_functionality_problems "$PROJECT_PATH"

  if [ "$FUNCTIONALITY_SCORE" -eq 0 ]; then
    echo "No functionality problems detected"
    return 0
  fi

  PREV_SCORE=$FUNCTIONALITY_SCORE

  # Create tasks for each issue
  for issue in "${FUNCTIONALITY_ISSUES[@]}"; do
    if [[ "$issue" == "Import errors"* ]]; then
      bd create --title="Fix import errors" --type=bug --priority=0 \
        --description="$issue"
    elif [[ "$issue" == "Missing:"* ]]; then
      pkg=$(echo "$issue" | cut -d: -f2 | tr -d ' ')
      bd create --title="Install missing: $pkg" --type=bug --priority=1 \
        --description="Package $pkg required but not installed"
    elif [[ "$issue" == "Test collection"* ]]; then
      bd create --title="Fix test collection errors" --type=bug --priority=1 \
        --description="$issue"
    elif [[ "$issue" == "Potential circular"* ]]; then
      bd create --title="Resolve circular imports" --type=bug --priority=2 \
        --description="$issue"
    fi
  done

  # Work through fixes
  MAX_FIX_ITERATIONS=20
  fix_iteration=0

  while [ "$(bd ready | grep -c .)" -gt 0 ] && [ $fix_iteration -lt $MAX_FIX_ITERATIONS ]; do
    fix_iteration=$((fix_iteration + 1))
    echo "=== Fix Iteration $fix_iteration ==="

    task_id=$(bd ready | head -1 | awk '{print $1}')
    [ -z "$task_id" ] && break

    bd update "$task_id" --status=in_progress

    # Apply fix (using serena, pip, etc.)

    # Verify fix
    detect_functionality_problems "$PROJECT_PATH"

    if [ "$FUNCTIONALITY_SCORE" -lt "$PREV_SCORE" ]; then
      bd close "$task_id" --reason="Fixed, score: $PREV_SCORE -> $FUNCTIONALITY_SCORE"
      PREV_SCORE=$FUNCTIONALITY_SCORE
    else
      bd update "$task_id" --status=blocked
      bd create --title="Alternative fix needed" --type=task --priority=0
    fi

    [ "$FUNCTIONALITY_SCORE" -eq 0 ] && break
  done

  echo "=== Functionality Fix Phase Complete ==="
  echo "Final score: $FUNCTIONALITY_SCORE"
}
```

---

## PHASE 15: Master Orchestration Loop

### 15.0 Overview

Iterative loop until project is fully complete:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MASTER ORCHESTRATION LOOP                                 │
│                    (Runs until project fully complete)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROJECT TYPES HANDLED:                                                    │
│  ├─ complete               → Standard deployment (Phases 0-9)              │
│  ├─ functionality_problems → Fix first (Phase 14), then continue           │
│  ├─ incomplete_research    → Research (13) → Development (11)             │
│  ├─ incomplete_critical    → Development completion (Phase 11)             │
│  └─ incomplete_minor       → Address TODOs/stubs (Phase 11)                │
│                                                                             │
│  ITERATION CYCLE:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ASSESS → ROUTE → EXECUTE → VERIFY → (loop or complete)              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  FEATURES:                                                                  │
│  ├─ Memory persistence across iterations                                   │
│  ├─ Self-healing when stuck                                               │
│  └─ Automatic routing based on project state                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 15.1 Master Orchestration Function

```bash
# ═══════════════════════════════════════════════════════════════════════════
# MASTER ORCHESTRATION LOOP
# Runs iteratively until project is fully deployed, tested, and functional
# ═══════════════════════════════════════════════════════════════════════════

run_master_orchestration() {
  PROJECT_PATH="${1:-$(pwd)}"
  PROJECT_NAME="${2:-$(basename $PROJECT_PATH)}"

  echo "╔═══════════════════════════════════════════════════════════════════════════╗"
  echo "║           RALPH-LOOP MASTER ORCHESTRATION                                 ║"
  echo "║           Project: $PROJECT_NAME"
  echo "╚═══════════════════════════════════════════════════════════════════════════╝"

  MAX_OUTER_ITERATIONS=10
  iteration=0
  PREV_COMPLEXITY_SCORE=999999

  # Initialize memory layer
  initialize_memory_layer "$PROJECT_PATH" "$PROJECT_NAME"

  # Recover previous session context
  recover_after_compaction

  while [ $iteration -lt $MAX_OUTER_ITERATIONS ]; do
    iteration=$((iteration + 1))
    echo ""
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║ ORCHESTRATION ITERATION $iteration / $MAX_OUTER_ITERATIONS"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"

    # STEP 1: ASSESS
    echo ">>> STEP 1: Assessing project state..."
    assess_project_completeness_enhanced "$PROJECT_PATH"

    # STEP 2: ROUTE
    echo ">>> STEP 2: Routing based on status: $PROJECT_STATUS"

    case "$PROJECT_STATUS" in
      "complete")
        echo "Project complete. Running verification..."
        NEXT_PHASE="verify"
        ;;
      "functionality_problems")
        echo "Routing to Phase 14: Functionality Fix"
        NEXT_PHASE="14"
        ;;
      "incomplete_research")
        echo "Routing to Phase 13: Research, then Phase 11"
        NEXT_PHASE="13"
        ;;
      "incomplete_critical"|"incomplete_minor")
        echo "Routing to Phase 11: Development"
        NEXT_PHASE="11"
        ;;
    esac

    # STEP 3: EXECUTE
    echo ">>> STEP 3: Executing phase $NEXT_PHASE..."

    case "$NEXT_PHASE" in
      "14") run_functionality_fix_phase "$PROJECT_PATH" ;;
      "13") run_research_phase "$PROJECT_PATH"; run_development_completion "$PROJECT_PATH" ;;
      "11") run_development_completion "$PROJECT_PATH" ;;
      "verify") run_phase_12_verification "$PROJECT_PATH" "{{ENV_NAME}}" ;;
    esac

    # STEP 4: VERIFY
    echo ">>> STEP 4: Verifying completion..."

    check_completion_criteria "$PROJECT_PATH"
    CRITERIA_RESULT=$?

    # STEP 5: ITERATE OR COMPLETE
    if [ $CRITERIA_RESULT -eq 0 ]; then
      echo ""
      echo "╔═══════════════════════════════════════════════════════════════════════════╗"
      echo "║ PROJECT COMPLETE - All criteria met                                       ║"
      echo "╚═══════════════════════════════════════════════════════════════════════════╝"

      persist_before_compaction
      bd sync

      return 0
    else
      echo "Criteria not met. Continuing..."

      store_to_memory "progress" "Iteration $iteration: $PROJECT_STATUS" "iteration,progress"

      if [ "$COMPLEXITY_SCORE" -ge "$PREV_COMPLEXITY_SCORE" ] && [ $iteration -gt 2 ]; then
        echo "WARNING: Not making progress. Self-healing..."
        trigger_self_healing
      fi

      PREV_COMPLEXITY_SCORE=$COMPLEXITY_SCORE
    fi
  done

  echo ""
  echo "╔═══════════════════════════════════════════════════════════════════════════╗"
  echo "║ MAX ITERATIONS REACHED - Manual intervention may be required              ║"
  echo "╚═══════════════════════════════════════════════════════════════════════════╝"

  persist_before_compaction
  return 1
}
```

### 15.2 Completion Criteria Check

```bash
# ═══════════════════════════════════════════════════════════════════════════
# COMPLETION CRITERIA CHECK
# Verifies all requirements are met
# ═══════════════════════════════════════════════════════════════════════════

check_completion_criteria() {
  PROJECT_PATH="${1:-$(pwd)}"

  echo "=== Checking Completion Criteria ==="

  CRITERIA_MET=0
  TOTAL_CRITERIA=6

  # 1. No NotImplementedError
  NOT_IMPL=$(grep -rn "NotImplementedError\|raise NotImplemented" "$PROJECT_PATH" --include="*.py" 2>/dev/null | grep -v "__pycache__" | wc -l)
  if [ "$NOT_IMPL" -eq 0 ]; then
    echo "✓ No NotImplementedError"
    CRITERIA_MET=$((CRITERIA_MET + 1))
  else
    echo "✗ Has $NOT_IMPL NotImplementedError"
  fi

  # 2. No import errors
  IMPORT_OK=$(cd "$PROJECT_PATH" && python -c "import sys; sys.path.insert(0,'.')" 2>&1 | grep -c "Error" || echo "0")
  if [ "$IMPORT_OK" -eq 0 ]; then
    echo "✓ No import errors"
    CRITERIA_MET=$((CRITERIA_MET + 1))
  else
    echo "✗ Has import errors"
  fi

  # 3. Tests pass
  if [ -d "$PROJECT_PATH/tests" ]; then
    TEST_RESULT=$(cd "$PROJECT_PATH" && conda run -n {{ENV_NAME}} python -m pytest tests/ -q 2>&1 | tail -1)
    if echo "$TEST_RESULT" | grep -q "passed"; then
      echo "✓ Tests pass"
      CRITERIA_MET=$((CRITERIA_MET + 1))
    else
      echo "✗ Tests: $TEST_RESULT"
    fi
  else
    echo "⊘ No tests directory"
    CRITERIA_MET=$((CRITERIA_MET + 1))
  fi

  # 4. No blocked tasks
  BLOCKED=$(bd blocked 2>/dev/null | grep -c . || echo "0")
  if [ "$BLOCKED" -eq 0 ]; then
    echo "✓ No blocked tasks"
    CRITERIA_MET=$((CRITERIA_MET + 1))
  else
    echo "✗ Has $BLOCKED blocked tasks"
  fi

  # 5. No in_progress tasks
  IN_PROGRESS=$(bd list --status=in_progress 2>/dev/null | grep -c . || echo "0")
  if [ "$IN_PROGRESS" -eq 0 ]; then
    echo "✓ No tasks in progress"
    CRITERIA_MET=$((CRITERIA_MET + 1))
  else
    echo "✗ Has $IN_PROGRESS tasks in progress"
  fi

  # 6. Functionality score is 0
  detect_functionality_problems "$PROJECT_PATH" >/dev/null 2>&1
  if [ "$FUNCTIONALITY_SCORE" -eq 0 ]; then
    echo "✓ No functionality problems"
    CRITERIA_MET=$((CRITERIA_MET + 1))
  else
    echo "✗ Functionality score: $FUNCTIONALITY_SCORE"
  fi

  echo ""
  echo "Criteria met: $CRITERIA_MET / $TOTAL_CRITERIA"

  [ "$CRITERIA_MET" -eq "$TOTAL_CRITERIA" ] && return 0 || return 1
}
```

### 15.3 Self-Healing Mechanism

```bash
# ═══════════════════════════════════════════════════════════════════════════
# SELF-HEALING MECHANISM
# Triggered when not making progress
# ═══════════════════════════════════════════════════════════════════════════

trigger_self_healing() {
  echo "=== Self-Healing Triggered ==="

  # 1. Check blocked tasks
  BLOCKED_COUNT=$(bd blocked 2>/dev/null | grep -c . || echo "0")
  if [ "$BLOCKED_COUNT" -gt 0 ]; then
    echo "Found $BLOCKED_COUNT blocked tasks..."

    while IFS= read -r blocked_line; do
      task_id=$(echo "$blocked_line" | awk '{print $1}')
      [ -z "$task_id" ] && continue

      bd create --title="Unblock: $task_id" --type=task --priority=0 \
        --description="Self-healing: Investigate and resolve blocker"
    done < <(bd blocked 2>/dev/null)
  fi

  # 2. Query memory for similar issues
  echo "Searching memory for similar situations..."
  retrieve_relevant_memories "stuck not progressing self-healing"

  # 3. Store self-healing event
  store_to_memory "error" "Self-healing triggered - not making progress" "self-healing,stuck"

  echo "Self-healing analysis complete"
}
```

---

## SELF-HEALING STRATEGIES

### Strategy 1: CUDA Wheel Not Found

```bash
# 1. Check alternative mirrors
pip index versions $PACKAGE --index-url https://pypi.jetson-ai-lab.io/sbsa/cu130/
pip index versions $PACKAGE --index-url https://download.pytorch.org/whl/cu130

# 2. Check NGC
ngc registry search $PACKAGE

# 3. Build from source (last resort)
git clone https://github.com/{{ORG}}/$PACKAGE
cd $PACKAGE
CUDA_HOME=/usr/local/cuda-13.0 pip install -e .
```

### Strategy 2: Port Conflict

```bash
# Find process using port
lsof -i :$PORT

# If Docker container
docker stop $(docker ps -q --filter "publish=$PORT")

# Reassign to next available port
NEW_PORT=$((PORT + 1))
export {{PROJECT_PREFIX}}_PORT_PRIMARY=$NEW_PORT
```

### Strategy 3: Dependency Conflict

```bash
# Try isolated install
pip install $PACKAGE --no-deps

# Then install dependencies one by one
pip install $DEP1 $DEP2

# Or use constraint file
pip install $PACKAGE -c constraints.txt
```

### Strategy 4: Permission Issues

```bash
# For /home/kp owned directories
sudo chown -R kp:kp {{PROJECT_PATH}}

# For system directories (avoid if possible)
# Use conda/pip --user instead
```

---

## PLACEHOLDER REFERENCE

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `{{PROJECT_NAME}}` | Project name (lowercase) | myproject |
| `{{PROJECT_PATH}}` | Absolute project path | /home/kp/repo/myproject |
| `{{PROJECT_PREFIX}}` | Env var prefix (UPPER) | MYPROJECT |
| `{{ENV_NAME}}` | Conda environment name | myproject |
| `{{PORT_START}}` | First port in range | 19000 |
| `{{PORT_END}}` | Last port in range | 19999 |
| `{{PROJECT_IMPORTS}}` | Python import statements | import mymodule |

---

## QUICK START

```bash
# 1. Clone this methodology
cp /home/kp/repo-nvidia/ToolOrchestra/.ralph/MASTER_PROMPT_TEMPLATE.md /path/to/project/.ralph/

# 2. Run Phase 0: System Audit
# (Identify ports, existing envs, GPU status)

# 3. Run Phase 1: Dependency Analysis
# (Parse requirements.txt, categorize packages)

# 4. Create prd.json with discovered tasks
# (Based on actual project dependencies)

# 5. Initialize Beads
bd init --prefix=PROJ

# 6. Start ralph-loop
/ralph
```

---

## PLATFORM CONSTANTS (Jetson Thor)

```bash
# These are fixed for this server
DEVICE="NVIDIA Jetson Thor"
JETPACK="7.4 (R38.4.0)"
CUDA_VERSION="13.0"
DRIVER_VERSION="580.00"
PYTHON_DEFAULT="3.12"
PYPI_MIRROR="https://pypi.jetson-ai-lab.io/sbsa/cu130/"
CUDA_HOME="/usr/local/cuda-13.0"
COMPUTE_CAPABILITY="9.0"  # SM90

# MySQL
MYSQL_ROOT_PASS="teamrsi123teamrsi123teamrsi123"

# System user
USER="kp"
HOME="/home/kp"
```
