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
      echo "Project requires development. Starting morphological development orchestration..."
      echo ""
      # Use morphological development for complete lifecycle:
      # Plan → Research → Create → Edit → Iterate → Test → Confirm → Deploy
      run_morphological_development "$PROJECT_PATH" "$PROJECT_NAME"
      ORCHESTRATION_RESULT=$?

      if [ $ORCHESTRATION_RESULT -eq 0 ]; then
        echo "Morphological development complete. Project deployed successfully."
        return 0
      else
        echo "Morphological development did not complete. Manual intervention required."
        bd create \
          --title="Manual intervention required: $PROJECT_NAME" \
          --type=task \
          --priority=0 \
          --description="Morphological development did not complete successfully.
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

