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

### 7.3 MCP Tool Integration Patterns

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

```bash
# Ralph-loop terminates when ALL conditions met:

# 1. No open tasks
bd list --status=open  # Returns empty

# 2. No blocked tasks (all resolved)
bd blocked  # Returns empty

# 3. All verifications pass
conda run -n {{ENV}} python -c "{{FINAL_VERIFICATION}}"

# 4. Services accessible (if applicable)
curl -s http://localhost:{{PORT}}/health | grep -q "ok"

# 5. Environment properly configured
source ~/.bashrc && test -n "${{PROJECT_PREFIX}}_PROJECT_PATH"
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
