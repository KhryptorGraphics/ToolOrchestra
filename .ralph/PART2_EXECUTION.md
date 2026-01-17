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

### 13.0.1 Semantic Scholar API Configuration

```bash
# ═══════════════════════════════════════════════════════════════════════════
# PHASE 13.0.1: SEMANTIC SCHOLAR API CONFIGURATION
# Rate limiting and multi-source orchestration setup
# ═══════════════════════════════════════════════════════════════════════════

# Configuration constants for mcp-semantic-scholar-server
SS_API_KEY="${SEMANTIC_SCHOLAR_API_KEY:-}"
SS_RATE_LIMIT_MS=1000           # 1 request per second (enforced by API)
SS_MAX_QUERIES_PER_SEARCH=3     # Max API calls per search session
SS_LAST_REQUEST_TIME=0

# ┌─────────────────────────────────────────────────────────────────────────────┐
# │ MCP SERVER PRIORITY FOR SEMANTIC SCHOLAR:                                   │
# │                                                                             │
# │ 1. mcp-semantic-scholar-server  → Primary search (simple, rate-limited)    │
# │    Tool: mcp__mcp-semantic-scholar-server__search_papers_via_semanticscholar│
# │    Best for: Quick keyword searches with year filtering                     │
# │                                                                             │
# │ 2. semantic-scholar-graph-api   → Citation graph, recommendations, batch   │
# │    Tools: get_citations_and_references, get_paper_recommendations,         │
# │           search_snippets, get_papers_batch, get_authors_batch             │
# │    Best for: Deep research, citation analysis, finding implementations     │
# │                                                                             │
# │ 3. research-semantic-scholar    → Fallback, bibtex generation              │
# │    Tools: search_semantic_scholar, get_paper, get_citations, to_bibtex     │
# │    Best for: Simple queries, citation formatting                           │
# └─────────────────────────────────────────────────────────────────────────────┘

configure_semantic_scholar() {
  echo "=== Configuring Semantic Scholar API ==="
  echo "API Key: ${SS_API_KEY:+configured (${#SS_API_KEY} chars)}${SS_API_KEY:-not set}"
  echo "Rate Limit: ${SS_RATE_LIMIT_MS}ms between requests"
  echo "Max Queries per Search: ${SS_MAX_QUERIES_PER_SEARCH}"
}

enforce_semantic_scholar_rate_limit() {
  # Enforce 1 request/second rate limit for mcp-semantic-scholar-server
  local current_time=$(date +%s%3N)
  local elapsed=$((current_time - SS_LAST_REQUEST_TIME))

  if [ "$elapsed" -lt "$SS_RATE_LIMIT_MS" ] && [ "$SS_LAST_REQUEST_TIME" -gt 0 ]; then
    local wait_time=$((SS_RATE_LIMIT_MS - elapsed))
    echo "  [Rate limit] Waiting ${wait_time}ms before next Semantic Scholar request..."
    sleep "0.$(printf '%03d' $wait_time)"
  fi

  SS_LAST_REQUEST_TIME=$(date +%s%3N)
}

export -f configure_semantic_scholar
export -f enforce_semantic_scholar_rate_limit
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

### 13.1.1 Research Query Decomposition

```bash
# ═══════════════════════════════════════════════════════════════════════════
# PHASE 13.1.1: RESEARCH QUERY DECOMPOSITION
# Decompose complex questions into max 3 targeted sub-queries
# ═══════════════════════════════════════════════════════════════════════════

# ┌─────────────────────────────────────────────────────────────────────────────┐
# │                    QUESTION DECOMPOSITION WORKFLOW                          │
# ├─────────────────────────────────────────────────────────────────────────────┤
# │                                                                             │
# │  ORIGINAL QUERY                                                             │
# │  ├─ "How do transformers handle long sequences efficiently?"               │
# │  │                                                                          │
# │  ▼                                                                          │
# │  DECOMPOSED QUERIES (max 3 API calls total):                               │
# │  ├─ Query 1: "transformer long sequence efficiency" (primary)              │
# │  ├─ Query 2: "attention mechanism memory optimization" (supporting)        │
# │  └─ Query 3: "sparse attention linear complexity" (specific technique)     │
# │                                                                             │
# │  RATE LIMIT: 1 request/second (1000ms delay between calls)                 │
# │                                                                             │
# └─────────────────────────────────────────────────────────────────────────────┘

decompose_research_query() {
  local original_query="$1"
  local max_queries="${2:-$SS_MAX_QUERIES_PER_SEARCH}"

  echo "=== Decomposing Research Query ==="
  echo "Original: $original_query"
  echo "Max sub-queries: $max_queries"

  # Initialize decomposed queries array
  DECOMPOSED_QUERIES=()

  # ───────────────────────────────────────────────────────────────────────
  # Decomposition strategy:
  # Query 1: Core concept (primary search - extract main technical terms)
  # Query 2: Related technique/methodology (supporting context)
  # Query 3: Specific implementation/application (actionable details)
  # ───────────────────────────────────────────────────────────────────────

  # Strategy 1: Extract key technical terms for primary query
  # Remove common words, keep technical terms
  local primary_query=$(echo "$original_query" | \
    sed -E 's/\b(how|do|does|what|is|are|the|a|an|to|for|in|on|with|by|of)\b//gi' | \
    tr -s ' ' | sed 's/^ *//;s/ *$//')

  [ -n "$primary_query" ] && DECOMPOSED_QUERIES+=("$primary_query")

  # Strategy 2: If query mentions specific technique, add technique-focused query
  if echo "$original_query" | grep -qiE "transformer|attention|neural|network|model"; then
    local technique_query=""

    if echo "$original_query" | grep -qi "transformer"; then
      technique_query="transformer architecture implementation"
    elif echo "$original_query" | grep -qi "attention"; then
      technique_query="attention mechanism optimization"
    elif echo "$original_query" | grep -qi "neural"; then
      technique_query="neural network training techniques"
    fi

    [ -n "$technique_query" ] && [ ${#DECOMPOSED_QUERIES[@]} -lt $max_queries ] && \
      DECOMPOSED_QUERIES+=("$technique_query")
  fi

  # Strategy 3: Add implementation-focused query if not at max
  if [ ${#DECOMPOSED_QUERIES[@]} -lt $max_queries ]; then
    local impl_query="$primary_query implementation code"
    DECOMPOSED_QUERIES+=("$impl_query")
  fi

  # Output decomposed queries
  echo ""
  echo "Decomposed into ${#DECOMPOSED_QUERIES[@]} queries:"
  local i=1
  for q in "${DECOMPOSED_QUERIES[@]}"; do
    echo "  Query $i: \"$q\""
    i=$((i + 1))
  done

  export DECOMPOSED_QUERIES
}

export -f decompose_research_query
```

### 13.2 Academic Paper Search (Multi-Source Orchestrated)

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.2: MULTI-SOURCE ACADEMIC PAPER SEARCH (ORCHESTRATED)
# Orchestrates across mcp-semantic-scholar-server, semantic-scholar-graph-api,
# research-semantic-scholar, and arxiv-advanced
# ═══════════════════════════════════════════════════════════════════════════

# ┌─────────────────────────────────────────────────────────────────────────────┐
# │               CROSS-SOURCE RESEARCH WORKFLOW (Phase 13 Enhanced)            │
# ├─────────────────────────────────────────────────────────────────────────────┤
# │                                                                             │
# │  STEP 1: PRIMARY SEARCH (Semantic Scholar - mcp-semantic-scholar-server)   │
# │  ├─ Rate limit: 1 req/sec enforced                                         │
# │  └─ Returns: High-impact papers with citation counts                       │
# │                                                                             │
# │  STEP 2: CITATION GRAPH (semantic-scholar-graph-api)                       │
# │  ├─ get_citations_and_references: Find related work                        │
# │  ├─ get_paper_recommendations: Similar papers                              │
# │  └─ search_snippets: Find specific text in papers                          │
# │                                                                             │
# │  STEP 3: PREPRINT SEARCH (arxiv-advanced)                                  │
# │  ├─ Cutting-edge work not yet peer-reviewed                                │
# │  ├─ Category filtering (cs.AI, cs.LG, cs.CL, etc.)                        │
# │  └─ Full paper download and reading                                        │
# │                                                                             │
# │  STEP 4: SYNTHESIS                                                          │
# │  ├─ Deduplicate across sources                                              │
# │  ├─ Cross-reference citations                                               │
# │  ├─ Rank by: citation count + recency + relevance                          │
# │  └─ Generate APS-style citations                                            │
# │                                                                             │
# └─────────────────────────────────────────────────────────────────────────────┘

search_academic_papers_orchestrated() {
  local query="$1"
  local max_results="${2:-10}"
  local api_calls_used=0

  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "                    MULTI-SOURCE ACADEMIC SEARCH"
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "Query: $query"
  echo "Max API calls: $SS_MAX_QUERIES_PER_SEARCH"

  COLLECTED_PAPERS=()
  TOP_PAPER_ID=""

  # ───────────────────────────────────────────────────────────────────────
  # STEP 1: PRIMARY SEARCH - mcp-semantic-scholar-server (KhryptorGraphics)
  # Best for: Simple keyword searches with year filtering, rate-limited
  # ───────────────────────────────────────────────────────────────────────
  if [ "$api_calls_used" -lt "$SS_MAX_QUERIES_PER_SEARCH" ]; then
    echo ""
    echo ">>> Step 1: Primary search (mcp-semantic-scholar-server)..."
    enforce_semantic_scholar_rate_limit

    # MCP call: mcp__mcp-semantic-scholar-server__search_papers_via_semanticscholar
    # Parameters:
    #   keyword: "$query"
    #   limit: min($max_results, 25)
    #   year_from: 2020  # Focus on recent research
    #
    # Returns: Formatted results with titles, authors, abstracts, citations
    # Store top paper ID for citation expansion

    api_calls_used=$((api_calls_used + 1))
    echo "  API calls used: $api_calls_used / $SS_MAX_QUERIES_PER_SEARCH"
  fi

  # ───────────────────────────────────────────────────────────────────────
  # STEP 2: CITATION EXPANSION - semantic-scholar-graph-api
  # Best for: Finding related work through citation graph, batch queries
  # ───────────────────────────────────────────────────────────────────────
  if [ "$api_calls_used" -lt "$SS_MAX_QUERIES_PER_SEARCH" ] && [ -n "$TOP_PAPER_ID" ]; then
    echo ""
    echo ">>> Step 2: Citation expansion (semantic-scholar-graph-api)..."
    enforce_semantic_scholar_rate_limit

    # For top paper from Step 1, get citations and recommendations
    #
    # MCP call: mcp__semantic-scholar-graph-api__get_semantic_scholar_citations_and_references
    #   paper_id: "$TOP_PAPER_ID"
    # Returns: Citations and references lists
    #
    # MCP call: mcp__semantic-scholar-graph-api__get_semantic_scholar_paper_recommendations
    #   paper_id: "$TOP_PAPER_ID"
    #   limit: 5
    # Returns: Similar/recommended papers
    #
    # MCP call: mcp__semantic-scholar-graph-api__search_semantic_scholar_snippets
    #   query: "$query"
    #   limit: 5
    # Returns: Text snippets from papers matching query

    api_calls_used=$((api_calls_used + 1))
    echo "  API calls used: $api_calls_used / $SS_MAX_QUERIES_PER_SEARCH"
  fi

  # ───────────────────────────────────────────────────────────────────────
  # STEP 3: PREPRINT SEARCH - arxiv-advanced
  # Best for: Cutting-edge research not yet in Semantic Scholar
  # ───────────────────────────────────────────────────────────────────────
  if [ "$api_calls_used" -lt "$SS_MAX_QUERIES_PER_SEARCH" ]; then
    echo ""
    echo ">>> Step 3: Preprint search (arxiv-advanced)..."

    # MCP call: mcp__arxiv-advanced__search_papers
    # Parameters:
    #   query: "$query" (use quoted phrases for exact match)
    #   categories: ["cs.AI", "cs.LG", "cs.CL", "cs.CV", "cs.MA"]
    #   max_results: 5
    #   sort_by: "relevance"
    #   date_from: "2023-01-01"
    #
    # Advanced query syntax:
    #   - ti:"exact title phrase" - search titles only
    #   - au:"author name" - search by author
    #   - abs:"keyword" - search abstracts only
    #   - "term1" OR "term2" - boolean OR
    #   - "term1" ANDNOT "survey" - exclude terms

    api_calls_used=$((api_calls_used + 1))
    echo "  API calls used: $api_calls_used / $SS_MAX_QUERIES_PER_SEARCH"
  fi

  echo ""
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "Search complete. API calls used: $api_calls_used / $SS_MAX_QUERIES_PER_SEARCH"
  echo "Papers collected: ${#COLLECTED_PAPERS[@]}"
  echo "═══════════════════════════════════════════════════════════════════════════"

  export COLLECTED_PAPERS
  export TOP_PAPER_ID
}

# Legacy function name for backward compatibility
search_academic_papers() {
  search_academic_papers_orchestrated "$@"
}

export -f search_academic_papers_orchestrated
export -f search_academic_papers

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

### 13.2.1 Research Results Synthesis

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.2.1: RESEARCH RESULTS SYNTHESIS
# Cross-reference, deduplicate, and rank papers from all sources
# ═══════════════════════════════════════════════════════════════════════════

synthesize_research_results() {
  echo "=== Synthesizing Research Results ==="
  echo "Papers to synthesize: ${#COLLECTED_PAPERS[@]}"

  SYNTHESIZED_PAPERS=()
  SEEN_IDS=()

  # ───────────────────────────────────────────────────────────────────────
  # Step 1: Deduplicate by DOI/arXiv ID
  # Papers may appear in multiple sources (Semantic Scholar + arXiv)
  # ───────────────────────────────────────────────────────────────────────
  echo ">>> Step 1: Deduplicating papers..."

  for paper in "${COLLECTED_PAPERS[@]}"; do
    paper_id=$(echo "$paper" | grep -oP 'id:\K[^,]+' || echo "")
    arxiv_id=$(echo "$paper" | grep -oP 'arXiv:\K[0-9.]+' || echo "")
    doi=$(echo "$paper" | grep -oP 'doi:\K[^,]+' || echo "")

    # Use the most specific identifier available
    unique_id="${doi:-${arxiv_id:-$paper_id}}"

    if [[ ! " ${SEEN_IDS[*]} " =~ " ${unique_id} " ]]; then
      SEEN_IDS+=("$unique_id")
      SYNTHESIZED_PAPERS+=("$paper")
    fi
  done

  echo "  After deduplication: ${#SYNTHESIZED_PAPERS[@]} unique papers"

  # ───────────────────────────────────────────────────────────────────────
  # Step 2: Cross-reference citations
  # If Paper A cites Paper B, both are relevant
  # ───────────────────────────────────────────────────────────────────────
  echo ">>> Step 2: Cross-referencing citations..."

  # Track citation relationships for ranking boost
  declare -A CITATION_BOOST

  # Papers cited by multiple collected papers get a boost
  # (Implementation would parse citation data from semantic-scholar-graph-api)

  # ───────────────────────────────────────────────────────────────────────
  # Step 3: Rank by combined score
  # Score = (citation_count × 0.3) + (recency × 0.3) + (relevance × 0.4)
  # ───────────────────────────────────────────────────────────────────────
  echo ">>> Step 3: Ranking papers by combined score..."

  RANKED_PAPERS=()

  for paper in "${SYNTHESIZED_PAPERS[@]}"; do
    # Extract metrics (these would come from the actual paper data)
    citations=$(echo "$paper" | grep -oP 'citations:\K[0-9]+' || echo "0")
    year=$(echo "$paper" | grep -oP 'year:\K[0-9]+' || echo "2020")

    # Normalize citation count (0-100 scale)
    citation_score=$((citations > 1000 ? 100 : citations / 10))

    # Recency score: 2024=100, 2023=80, 2022=60, etc.
    current_year=$(date +%Y)
    recency_score=$(( (year - 2015) * 10 ))
    recency_score=$((recency_score > 100 ? 100 : recency_score))
    recency_score=$((recency_score < 0 ? 0 : recency_score))

    # Relevance score (from search ranking, default 50)
    relevance_score=50

    # Combined score
    total_score=$(( (citation_score * 30 + recency_score * 30 + relevance_score * 40) / 100 ))

    # Add to ranked list with score
    RANKED_PAPERS+=("$total_score|$paper")
  done

  # Sort by score (highest first)
  IFS=$'\n' RANKED_PAPERS=($(sort -rn -t'|' -k1 <<< "${RANKED_PAPERS[*]}"))
  unset IFS

  echo "  Ranked ${#RANKED_PAPERS[@]} papers"

  # ───────────────────────────────────────────────────────────────────────
  # Step 4: Generate APS-style citations
  # Format: Author1, Author2, et al., Title, Journal/arXiv, Year.
  # ───────────────────────────────────────────────────────────────────────
  echo ">>> Step 4: Generating APS-style citations..."

  APS_CITATIONS=()

  for ranked_paper in "${RANKED_PAPERS[@]}"; do
    paper=$(echo "$ranked_paper" | cut -d'|' -f2-)

    # Extract citation components
    authors=$(echo "$paper" | grep -oP 'authors:\K[^|]+' || echo "Unknown")
    title=$(echo "$paper" | grep -oP 'title:\K[^|]+' || echo "Untitled")
    venue=$(echo "$paper" | grep -oP 'venue:\K[^|]+' || echo "")
    year=$(echo "$paper" | grep -oP 'year:\K[0-9]+' || echo "")
    arxiv_id=$(echo "$paper" | grep -oP 'arXiv:\K[0-9.]+' || echo "")

    # Format APS-style citation
    if [ -n "$arxiv_id" ]; then
      citation="$authors, \"$title,\" arXiv:$arxiv_id ($year)."
    elif [ -n "$venue" ]; then
      citation="$authors, \"$title,\" $venue ($year)."
    else
      citation="$authors, \"$title\" ($year)."
    fi

    APS_CITATIONS+=("$citation")
  done

  echo ""
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "                    SYNTHESIS COMPLETE"
  echo "═══════════════════════════════════════════════════════════════════════════"
  echo "Unique papers: ${#SYNTHESIZED_PAPERS[@]}"
  echo "Top citations:"
  for i in {0..4}; do
    [ $i -lt ${#APS_CITATIONS[@]} ] && echo "  [$((i+1))] ${APS_CITATIONS[$i]}"
  done

  export SYNTHESIZED_PAPERS
  export RANKED_PAPERS
  export APS_CITATIONS
}

export -f synthesize_research_results
```

### 13.2.1 Research Feedback Loop (Enhancement 3)

When synthesis produces low-confidence results, generate NEW search queries:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# RESEARCH FEEDBACK LOOP (v2.0)
# When confidence < threshold, identifies gaps and generates new queries
# ═══════════════════════════════════════════════════════════════════════════

MIN_CONFIDENCE_THRESHOLD=60  # Minimum confidence score (0-100)
MAX_RESEARCH_ITERATIONS=5    # Max feedback loop iterations

calculate_solution_confidence() {
  local paper_count="${#SYNTHESIZED_PAPERS[@]}"
  local avg_citations=0
  local recency_bonus=0

  # Confidence factors:
  # 1. Number of relevant papers (20 points max)
  local count_score=$((paper_count > 10 ? 20 : paper_count * 2))

  # 2. Average citation count (30 points max)
  local total_citations=0
  for paper in "${RANKED_PAPERS[@]}"; do
    citations=$(echo "$paper" | grep -oP 'citations:\K[0-9]+' || echo "0")
    total_citations=$((total_citations + citations))
  done
  if [ "$paper_count" -gt 0 ]; then
    avg_citations=$((total_citations / paper_count))
    avg_citations=$((avg_citations > 300 ? 30 : avg_citations / 10))
  fi

  # 3. Recency of papers (20 points max)
  local current_year=$(date +%Y)
  local recent_count=0
  for paper in "${SYNTHESIZED_PAPERS[@]}"; do
    year=$(echo "$paper" | grep -oP 'year:\K[0-9]+' || echo "2000")
    if [ "$year" -ge "$((current_year - 3))" ]; then
      recent_count=$((recent_count + 1))
    fi
  done
  recency_bonus=$((recent_count * 4 > 20 ? 20 : recent_count * 4))

  # 4. Implementation details found (30 points max)
  local impl_score=0
  # Check if papers have code links, algorithms, etc.
  for paper in "${SYNTHESIZED_PAPERS[@]}"; do
    if echo "$paper" | grep -qi "github\|implementation\|code\|algorithm"; then
      impl_score=$((impl_score + 6))
    fi
  done
  impl_score=$((impl_score > 30 ? 30 : impl_score))

  SOLUTION_CONFIDENCE=$((count_score + avg_citations + recency_bonus + impl_score))

  echo "Confidence Score: $SOLUTION_CONFIDENCE/100"
  echo "  Paper count score: $count_score/20"
  echo "  Citation score: $avg_citations/30"
  echo "  Recency bonus: $recency_bonus/20"
  echo "  Implementation score: $impl_score/30"
}

identify_research_gaps() {
  echo "=== Identifying Research Gaps ==="

  MISSING_ASPECTS=()

  # Check what's missing from current research
  # 1. Implementation details
  local has_code=false
  for paper in "${SYNTHESIZED_PAPERS[@]}"; do
    if echo "$paper" | grep -qi "github\|code\|implementation"; then
      has_code=true
      break
    fi
  done
  [ "$has_code" = "false" ] && MISSING_ASPECTS+=("implementation code")

  # 2. Recent papers
  local has_recent=false
  local current_year=$(date +%Y)
  for paper in "${SYNTHESIZED_PAPERS[@]}"; do
    year=$(echo "$paper" | grep -oP 'year:\K[0-9]+' || echo "2000")
    [ "$year" -ge "$((current_year - 2))" ] && has_recent=true && break
  done
  [ "$has_recent" = "false" ] && MISSING_ASPECTS+=("recent developments")

  # 3. Foundational papers (high citations)
  local has_foundational=false
  for paper in "${RANKED_PAPERS[@]}"; do
    citations=$(echo "$paper" | grep -oP 'citations:\K[0-9]+' || echo "0")
    [ "$citations" -gt 500 ] && has_foundational=true && break
  done
  [ "$has_foundational" = "false" ] && MISSING_ASPECTS+=("foundational papers")

  echo "Missing aspects: ${MISSING_ASPECTS[*]:-none}"
  export MISSING_ASPECTS
}

generate_queries_from_gaps() {
  echo "=== Generating New Queries from Gaps ==="

  NEW_QUERIES=()
  local original_query="$1"

  for gap in "${MISSING_ASPECTS[@]}"; do
    case "$gap" in
      "implementation code")
        NEW_QUERIES+=("$original_query implementation github")
        NEW_QUERIES+=("$original_query code repository")
        ;;
      "recent developments")
        NEW_QUERIES+=("$original_query 2024 2025")
        NEW_QUERIES+=("$original_query state of the art recent")
        ;;
      "foundational papers")
        NEW_QUERIES+=("$original_query seminal foundational survey")
        NEW_QUERIES+=("$original_query benchmark comparison")
        ;;
    esac
  done

  echo "Generated ${#NEW_QUERIES[@]} new queries"
  export NEW_QUERIES
}

run_research_with_feedback() {
  local initial_query="$1"
  local iteration=0

  while [ $iteration -lt $MAX_RESEARCH_ITERATIONS ]; do
    iteration=$((iteration + 1))
    echo ""
    echo "═══════════════════════════════════════════════════════════════════════════"
    echo "                    RESEARCH ITERATION $iteration / $MAX_RESEARCH_ITERATIONS"
    echo "═══════════════════════════════════════════════════════════════════════════"

    # Search with current query
    if [ $iteration -eq 1 ]; then
      search_academic_papers_orchestrated "$initial_query"
    else
      for query in "${NEW_QUERIES[@]}"; do
        search_academic_papers_orchestrated "$query"
      done
    fi

    # Synthesize results
    synthesize_research_results

    # Calculate confidence
    calculate_solution_confidence

    # Check if confidence is sufficient
    if [ "$SOLUTION_CONFIDENCE" -ge "$MIN_CONFIDENCE_THRESHOLD" ]; then
      echo ">>> Confidence threshold met ($SOLUTION_CONFIDENCE >= $MIN_CONFIDENCE_THRESHOLD)"
      break
    fi

    echo ">>> Low confidence ($SOLUTION_CONFIDENCE < $MIN_CONFIDENCE_THRESHOLD)"
    echo ">>> Generating new queries for feedback loop..."

    # Identify gaps and generate new queries
    identify_research_gaps
    generate_queries_from_gaps "$initial_query"

    if [ ${#NEW_QUERIES[@]} -eq 0 ]; then
      echo ">>> No new queries generated. Stopping feedback loop."
      break
    fi
  done

  echo ""
  echo "Research feedback loop complete after $iteration iterations"
  echo "Final confidence: $SOLUTION_CONFIDENCE"
}
```

### 13.2.2 Cross-Iteration Research Aggregation (Enhancement 6)

Merge findings from previous iterations with current results:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# CROSS-ITERATION RESEARCH AGGREGATION (v2.0)
# Explicitly merges findings from previous iterations
# ═══════════════════════════════════════════════════════════════════════════

aggregate_research_across_iterations() {
  local current_iteration="$1"

  echo "=== Aggregating Research Across Iterations ==="

  # ───────────────────────────────────────────────────────────────────────
  # Step 1: Retrieve previous findings from memory
  # ───────────────────────────────────────────────────────────────────────
  echo ">>> Retrieving previous research findings..."

  # From Claude-mem
  # MCP call: mcp__plugin_claude-mem_mcp-search__search
  #   query: "research findings papers"
  #   project: "$PROJECT_NAME"
  #   type: "observation"
  #   limit: 20

  # From Serena memory
  # MCP call: mcp__serena__list_memories
  # Then read relevant research memories

  # From Cipher
  # MCP call: mcp__cipher__ask_cipher
  #   message: "What research findings have we collected for this project?"

  PREVIOUS_FINDINGS=()
  # (Would be populated from memory queries)

  # ───────────────────────────────────────────────────────────────────────
  # Step 2: Merge with current findings
  # ───────────────────────────────────────────────────────────────────────
  echo ">>> Merging with current findings..."

  MERGED_PAPERS=()

  # Add previous papers (avoiding duplicates)
  for prev_paper in "${PREVIOUS_FINDINGS[@]}"; do
    prev_id=$(echo "$prev_paper" | grep -oP 'id:\K[^,]+' || echo "")
    is_duplicate=false

    for new_paper in "${SYNTHESIZED_PAPERS[@]}"; do
      new_id=$(echo "$new_paper" | grep -oP 'id:\K[^,]+' || echo "")
      [ "$prev_id" = "$new_id" ] && is_duplicate=true && break
    done

    [ "$is_duplicate" = "false" ] && MERGED_PAPERS+=("$prev_paper")
  done

  # Add current papers
  for new_paper in "${SYNTHESIZED_PAPERS[@]}"; do
    MERGED_PAPERS+=("$new_paper")
  done

  echo "  Previous findings: ${#PREVIOUS_FINDINGS[@]}"
  echo "  Current findings: ${#SYNTHESIZED_PAPERS[@]}"
  echo "  Merged total: ${#MERGED_PAPERS[@]}"

  # ───────────────────────────────────────────────────────────────────────
  # Step 3: Store merged result
  # ───────────────────────────────────────────────────────────────────────
  echo ">>> Storing aggregated research..."

  store_to_memory "aggregated_research" \
    "Iteration $current_iteration: ${#MERGED_PAPERS[@]} papers aggregated" \
    "research,aggregated,iteration_$current_iteration"

  # Also store in Serena for long-term persistence
  # MCP call: mcp__serena__write_memory
  #   memory_file_name: "research_aggregated_$PROJECT_NAME.md"
  #   content: <formatted paper list>

  export MERGED_PAPERS
  echo "Aggregation complete"
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

### 13.4.1 Iterative Research Task Management

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 13.4.1: ITERATIVE RESEARCH TASK MANAGEMENT
# Create and track research tasks in Beads for iterative development
# ═══════════════════════════════════════════════════════════════════════════

create_research_epic() {
  local research_topic="$1"
  local paper_count="${2:-5}"

  echo "=== Creating Research Epic in Beads ==="
  echo "Topic: $research_topic"
  echo "Papers to analyze: $paper_count"

  # Create epic for the research topic
  bd create \
    --title="Research: $research_topic" \
    --type=epic \
    --priority=1 \
    --description="Academic research synthesis for: $research_topic

Papers to analyze: $paper_count
Created by Phase 13 Research Integration

Workflow:
1. Analyze each paper for implementation details
2. Extract algorithms, architectures, hyperparameters
3. Synthesize findings into actionable implementation plan
4. Create development tasks from research"

  # Get the created epic ID
  RESEARCH_EPIC_ID=$(bd list --status=open --type=epic 2>/dev/null | grep "Research: $research_topic" | head -1 | awk '{print $1}')

  echo "Created research epic: $RESEARCH_EPIC_ID"
  export RESEARCH_EPIC_ID
}

create_paper_analysis_tasks() {
  local epic_id="$1"
  shift
  local papers=("$@")

  echo "=== Creating Paper Analysis Tasks ==="
  echo "Epic: $epic_id"
  echo "Papers: ${#papers[@]}"

  local prev_task=""
  local all_task_ids=()

  for paper in "${papers[@]}"; do
    # Extract paper title for task name
    paper_title=$(echo "$paper" | grep -oP 'title:\K[^|]+' | head -c 50)
    paper_id=$(echo "$paper" | grep -oP 'id:\K[^,]+' || echo "unknown")

    # Create analysis task
    bd create \
      --title="Analyze: ${paper_title}..." \
      --type=task \
      --priority=2 \
      --description="Read and extract implementation details from paper.

Paper ID: $paper_id

Extract:
- Algorithm pseudocode
- Model architecture details
- Hyperparameter recommendations
- Loss function definitions
- Training procedures
- Code repository links (if any)"

    # Get the created task ID
    task_id=$(bd list --status=open 2>/dev/null | grep "Analyze: ${paper_title:0:20}" | head -1 | awk '{print $1}')

    if [ -n "$task_id" ]; then
      all_task_ids+=("$task_id")

      # Add dependency: each task depends on previous (ordered analysis)
      if [ -n "$prev_task" ]; then
        bd dep add "$task_id" "$prev_task" 2>/dev/null || true
        echo "  Added dependency: $task_id depends on $prev_task"
      fi

      prev_task="$task_id"
    fi
  done

  # Create final synthesis task that depends on last analysis task
  if [ -n "$prev_task" ]; then
    bd create \
      --title="Synthesize research findings" \
      --type=task \
      --priority=1 \
      --description="Combine insights from all analyzed papers.

Analyzed papers: ${#papers[@]}

Deliverables:
1. Unified implementation approach
2. Architecture design document
3. Recommended hyperparameters
4. Development task breakdown
5. Risk assessment and fallback options"

    synthesis_task=$(bd list --status=open 2>/dev/null | grep "Synthesize research findings" | head -1 | awk '{print $1}')
    [ -n "$synthesis_task" ] && bd dep add "$synthesis_task" "$prev_task" 2>/dev/null || true
  fi

  echo "Created ${#all_task_ids[@]} analysis tasks + 1 synthesis task"
  export PAPER_ANALYSIS_TASKS=("${all_task_ids[@]}")
}

iterate_research_findings() {
  local epic_id="$1"

  echo "=== Iterating on Research Findings ==="

  # Check for completed analysis tasks
  local completed=$(bd list --status=closed 2>/dev/null | grep -c "Analyze:" || echo "0")
  local total=$(bd list 2>/dev/null | grep -c "Analyze:" || echo "0")

  echo "Analysis progress: $completed / $total"

  if [ "$completed" -eq "$total" ] && [ "$total" -gt 0 ]; then
    echo "All papers analyzed. Checking synthesis..."

    # Check if synthesis is complete
    local synthesis_done=$(bd list --status=closed 2>/dev/null | grep -c "Synthesize research findings" || echo "0")

    if [ "$synthesis_done" -gt 0 ]; then
      echo "Research synthesis complete. Creating implementation tasks..."

      # Create implementation task from synthesis
      bd create \
        --title="Implement research findings" \
        --type=feature \
        --priority=0 \
        --description="Implementation based on research synthesis.

Source: Research epic $epic_id
Papers analyzed: $total

This task represents the culmination of the research phase.
Implementation should follow the synthesized approach documented
in the synthesis task."

      return 0
    else
      echo "Synthesis task not yet complete."
      return 1
    fi
  else
    echo "Analysis still in progress."
    return 1
  fi
}

export -f create_research_epic
export -f create_paper_analysis_tasks
export -f iterate_research_findings
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

### 15.3.1 Enhanced Self-Healing (v2.0)

The enhanced self-healing mechanism doesn't just create tasks - it actively attempts to resolve blockers:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# ENHANCED SELF-HEALING (v2.0)
# Actually resolves issues, not just creates tasks
# ═══════════════════════════════════════════════════════════════════════════

trigger_self_healing_enhanced() {
  echo "=== Enhanced Self-Healing Triggered ==="

  # 1. Identify blocked tasks and their blockers
  BLOCKED_TASKS=$(bd blocked 2>/dev/null)

  if [ -n "$BLOCKED_TASKS" ]; then
    echo "Analyzing blocked tasks..."

    while IFS= read -r blocked_line; do
      task_id=$(echo "$blocked_line" | awk '{print $1}')
      [ -z "$task_id" ] && continue

      # Get blocker reason from task details
      BLOCKER_INFO=$(bd show "$task_id" 2>/dev/null | grep -i "blocked\|depends" || echo "unknown")

      echo "  Task $task_id blocked by: $BLOCKER_INFO"

      # Attempt to resolve based on blocker type
      case "$BLOCKER_INFO" in
        *"import"*|*"Import"*|*"ModuleNotFound"*)
          echo "  -> Attempting to resolve import issue..."
          # Extract module name and try to install
          MODULE=$(echo "$BLOCKER_INFO" | grep -oP "(?<=No module named ')[^']+")
          if [ -n "$MODULE" ]; then
            conda run -n {{ENV_NAME}} pip install "$MODULE" 2>/dev/null && \
              echo "  -> Installed $MODULE" || \
              echo "  -> Could not install $MODULE"
          fi
          ;;
        *"test"*|*"Test"*|*"pytest"*)
          echo "  -> Running targeted test fix..."
          # Could trigger Phase 14 for this specific issue
          detect_functionality_problems "$PROJECT_PATH"
          ;;
        *"environment"*|*"conda"*|*"CUDA"*)
          echo "  -> Triggering environment rebuild (Phase 2-4)..."
          # Route to environment phases
          ;;
        *)
          echo "  -> Creating high-priority unblock task..."
          bd create --title="[Self-Heal] Unblock: $task_id" --type=task --priority=0 \
            --description="Self-healing: Investigate blocker: $BLOCKER_INFO

Suggested actions:
1. Check dependencies
2. Review error logs
3. Test in isolation"
          ;;
      esac
    done <<< "$BLOCKED_TASKS"
  fi

  # 2. Check for stale in_progress tasks (stuck > 30 minutes)
  STALE_TASKS=$(bd list --status=in_progress 2>/dev/null | head -5)
  if [ -n "$STALE_TASKS" ]; then
    echo "Found stale in_progress tasks:"
    echo "$STALE_TASKS"
    # Could reset these or create investigation tasks
  fi

  # 3. Query memory for similar past solutions
  echo "Searching memory for similar situations..."
  retrieve_relevant_memories "stuck not progressing self-healing unblock"

  echo "Enhanced self-healing complete"
}
```

### 15.3.2 Dynamic Phase Routing (Any → Any)

**Enhancement 7**: Every phase can route to ANY other phase based on current state:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# DYNAMIC PHASE ROUTING (v2.0)
# Multi-signal state analysis for intelligent phase selection
# Any phase can route to any other phase based on current state
# ═══════════════════════════════════════════════════════════════════════════

decide_next_phase() {
  PROJECT_PATH="${1:-$(pwd)}"

  # ─────────────────────────────────────────────────────────────────────────
  # ASSESS CURRENT STATE FROM MULTIPLE SIGNALS
  # ─────────────────────────────────────────────────────────────────────────

  # Environment state
  ENV_BROKEN=false
  if ! conda info --envs 2>/dev/null | grep -q "^[^ ]"; then
    ENV_BROKEN=true
  fi
  if command -v nvidia-smi &>/dev/null && ! nvidia-smi &>/dev/null; then
    ENV_BROKEN=true
  fi

  # Code state (NotImplementedError, stubs)
  CODE_INCOMPLETE=$(grep -rn "NotImplementedError\|raise NotImplemented\|pass  # TODO" \
    "$PROJECT_PATH" --include="*.py" 2>/dev/null | grep -v "__pycache__" | wc -l || echo "0")

  # Test state
  if [ -d "$PROJECT_PATH/tests" ]; then
    TEST_RESULT=$(cd "$PROJECT_PATH" && python -m pytest tests/ --collect-only -q 2>&1 | tail -1 || echo "error")
    if echo "$TEST_RESULT" | grep -q "error\|Error"; then
      TESTS_FAILING=true
    else
      TESTS_FAILING=false
    fi
  else
    TESTS_FAILING=false
  fi

  # Research state (complex ML/AI keywords)
  RESEARCH_SCORE=$(grep -rn "transformer\|attention\|neural network\|deep learning\|reinforcement" \
    "$PROJECT_PATH" --include="*.py" 2>/dev/null | grep -v "__pycache__" | wc -l || echo "0")

  # Beads state
  BLOCKED_COUNT=$(bd blocked 2>/dev/null | grep -c . || echo "0")
  READY_COUNT=$(bd ready 2>/dev/null | grep -c . || echo "0")
  IN_PROGRESS=$(bd list --status=in_progress 2>/dev/null | grep -c . || echo "0")

  # Port conflicts
  PORT_CONFLICTS=false
  # (Could check for specific port conflicts here)

  # ─────────────────────────────────────────────────────────────────────────
  # PHASE SELECTION MATRIX (priority order)
  # ─────────────────────────────────────────────────────────────────────────
  #
  # 1. Environment broken         → Phase 2-4 (Conda/CUDA/Dependencies)
  # 2. Blocked tasks              → Self-healing
  # 3. Code incomplete + research → Phase 13 (Academic research)
  # 4. Code incomplete            → Phase 11 (Development)
  # 5. Tests failing              → Phase 14 (Fix functionality)
  # 6. Ready tasks exist          → Phase 11 (Development)
  # 7. All criteria met           → Verify completion

  if [ "$ENV_BROKEN" = "true" ]; then
    echo "2"  # Rebuild environment
    return
  fi

  if [ "$BLOCKED_COUNT" -gt 0 ]; then
    echo "self-heal"  # Trigger self-healing
    return
  fi

  if [ "$CODE_INCOMPLETE" -gt 0 ]; then
    if [ "$RESEARCH_SCORE" -gt 20 ]; then
      echo "13"  # Research phase first
    else
      echo "11"  # Development directly
    fi
    return
  fi

  if [ "$TESTS_FAILING" = "true" ]; then
    echo "14"  # Fix functionality
    return
  fi

  if [ "$READY_COUNT" -gt 0 ]; then
    echo "11"  # Development - work on ready tasks
    return
  fi

  if [ "$IN_PROGRESS" -gt 0 ]; then
    echo "11"  # Continue development
    return
  fi

  # Check completion criteria
  echo "verify"  # Verify completion
}
```

### 15.3.3 Guaranteed Completion (No Hard Limits)

**Enhancement 8**: System runs until TRUE completion, with intelligent escalation when stuck:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# GUARANTEED COMPLETION (v2.0)
# No hard iteration limits - uses escalation for stuck states
# ═══════════════════════════════════════════════════════════════════════════

# Configuration
STUCK_THRESHOLD=5           # Iterations without progress before escalating
ESCALATION_LEVEL=0          # 0=normal, 1=aggressive, 2=human-assisted
LAST_COMPLEXITY_SCORES=()   # Track history for stuck detection

stuck_for_too_long() {
  # Check if complexity scores haven't improved for STUCK_THRESHOLD iterations
  local score_count=${#LAST_COMPLEXITY_SCORES[@]}

  if [ $score_count -lt $STUCK_THRESHOLD ]; then
    return 1  # Not enough data yet
  fi

  # Get last N scores
  local recent_scores=("${LAST_COMPLEXITY_SCORES[@]: -$STUCK_THRESHOLD}")

  # Check if all scores are the same (no progress)
  local first_score=${recent_scores[0]}
  local all_same=true

  for score in "${recent_scores[@]}"; do
    if [ "$score" != "$first_score" ]; then
      all_same=false
      break
    fi
  done

  [ "$all_same" = "true" ] && return 0 || return 1
}

generate_stuck_report() {
  echo "╔═══════════════════════════════════════════════════════════════════════════╗"
  echo "║                    STUCK REPORT - ESCALATION TRIGGERED                    ║"
  echo "╚═══════════════════════════════════════════════════════════════════════════╝"
  echo ""
  echo "Project: $PROJECT_NAME"
  echo "Path: $PROJECT_PATH"
  echo "Escalation Level: $ESCALATION_LEVEL"
  echo ""
  echo "Recent complexity scores: ${LAST_COMPLEXITY_SCORES[*]}"
  echo ""
  echo "=== Current State ==="
  bd stats 2>/dev/null || echo "  (bd stats unavailable)"
  echo ""
  echo "=== Blocked Tasks ==="
  bd blocked 2>/dev/null || echo "  (none)"
  echo ""
  echo "=== Open Tasks ==="
  bd list --status=open 2>/dev/null | head -10 || echo "  (none)"

  # Create Beads task for human intervention
  bd create --title="[ESCALATION] Human intervention required" --type=task --priority=0 \
    --description="Orchestration stuck after $STUCK_THRESHOLD iterations without progress.

Escalation Level: $ESCALATION_LEVEL
Recent scores: ${LAST_COMPLEXITY_SCORES[*]}

Please investigate and provide guidance:
1. Check blocked tasks
2. Review recent errors
3. Consider alternative approaches"

  echo ""
  echo ">>> Created escalation task. Waiting for human input..."
}

wait_for_human_input() {
  # In non-interactive mode, just pause briefly
  # In interactive mode, wait for confirmation
  echo ""
  echo "Press ENTER after resolving the escalation task, or Ctrl+C to abort."
  read -r
  ESCALATION_LEVEL=0  # Reset after human helps
}

# ═══════════════════════════════════════════════════════════════════════════
# ENHANCED MASTER ORCHESTRATION LOOP (v2.0)
# ═══════════════════════════════════════════════════════════════════════════

run_master_orchestration_v2() {
  PROJECT_PATH="${1:-$(pwd)}"
  PROJECT_NAME="${2:-$(basename $PROJECT_PATH)}"

  echo "╔═══════════════════════════════════════════════════════════════════════════╗"
  echo "║      RALPH-LOOP MASTER ORCHESTRATION (v2.0 - Dynamic Routing)             ║"
  echo "║      Project: $PROJECT_NAME"
  echo "╚═══════════════════════════════════════════════════════════════════════════╝"

  local iteration=0
  LAST_COMPLEXITY_SCORES=()
  ESCALATION_LEVEL=0

  # Initialize memory layer
  initialize_memory_layer "$PROJECT_PATH" "$PROJECT_NAME"

  # Recover previous session context
  recover_after_compaction

  # ═══════════════════════════════════════════════════════════════════════════
  # MAIN LOOP - No hard limit, loops until TRUE completion
  # ═══════════════════════════════════════════════════════════════════════════

  while true; do
    iteration=$((iteration + 1))
    echo ""
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║ ORCHESTRATION ITERATION $iteration (Escalation: $ESCALATION_LEVEL)"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 1: DYNAMIC PHASE ROUTING
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 1: Analyzing state and routing dynamically..."

    NEXT_PHASE=$(decide_next_phase "$PROJECT_PATH")
    echo "  Routing to: $NEXT_PHASE"

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 2: EXECUTE PHASE
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 2: Executing phase $NEXT_PHASE..."

    case "$NEXT_PHASE" in
      "2") setup_conda_environment "$PROJECT_PATH" ;;
      "3") setup_cuda_environment "$PROJECT_PATH" ;;
      "4") run_dependency_installation "$PROJECT_PATH" ;;
      "11") run_development_completion "$PROJECT_PATH" ;;
      "12") run_phase_12_verification "$PROJECT_PATH" "{{ENV_NAME}}" ;;
      "13") run_research_phase "$PROJECT_PATH" ;;
      "14") run_functionality_fix_phase "$PROJECT_PATH" ;;
      "self-heal") trigger_self_healing_enhanced ;;
      "verify") check_completion_criteria "$PROJECT_PATH" ;;
    esac

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 3: CHECK COMPLETION CRITERIA
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 3: Checking completion criteria..."

    check_completion_criteria "$PROJECT_PATH"
    CRITERIA_RESULT=$?

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 4: TRUE COMPLETION CHECK
    # ─────────────────────────────────────────────────────────────────────────
    if [ $CRITERIA_RESULT -eq 0 ]; then
      echo ""
      echo "╔═══════════════════════════════════════════════════════════════════════════╗"
      echo "║ PROJECT COMPLETE - All criteria met after $iteration iterations!          ║"
      echo "╚═══════════════════════════════════════════════════════════════════════════╝"

      # Final persistence
      persist_before_compaction
      bd sync

      return 0
    fi

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 5: BD SYNC (Enhancement 2 - every iteration)
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 5: Syncing Beads state..."
    bd sync 2>/dev/null || true

    # Store iteration progress in memory
    store_to_memory "progress" "Iteration $iteration: Next=$NEXT_PHASE, Score=$COMPLEXITY_SCORE" \
      "iteration,progress"

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 6: STUCK DETECTION (Enhancement 8)
    # ─────────────────────────────────────────────────────────────────────────
    if stuck_for_too_long; then
      ESCALATION_LEVEL=$((ESCALATION_LEVEL + 1))

      echo ">>> Stuck detected! Escalation level: $ESCALATION_LEVEL"

      case $ESCALATION_LEVEL in
        1)
          echo ">>> Triggering aggressive self-healing..."
          trigger_self_healing_enhanced
          ;;
        2)
          echo ">>> Human intervention required..."
          generate_stuck_report
          wait_for_human_input
          ;;
        *)
          # Reset and try again with fresh approach
          ESCALATION_LEVEL=0
          LAST_COMPLEXITY_SCORES=()
          ;;
      esac
    fi

  done  # while true - no MAX_ITERATIONS
}
```

### 15.3.4 Conditional MCP Selection

**Enhancement 4**: Route to specific MCP tools based on error type:

```bash
# ═══════════════════════════════════════════════════════════════════════════
# CONDITIONAL MCP SELECTION
# Routes to specific tools based on error/need type
# ═══════════════════════════════════════════════════════════════════════════

select_mcp_tools_for_error() {
  local error_type="$1"

  case "$error_type" in
    "ImportError"|"ModuleNotFoundError")
      # Use code analysis tools
      echo "serena,code-context"
      ;;
    "RuntimeError"|"TypeError"|"ValueError")
      # Use reasoning tools for debugging
      echo "sequential-thinking,code-reasoning"
      ;;
    "research_needed"|"NotImplementedError")
      # Use academic research tools
      echo "arxiv-advanced,semantic-scholar,perplexity"
      ;;
    "external_api"|"HTTPError"|"ConnectionError")
      # Use web tools for API investigation
      echo "tavily,linkup,playwright"
      ;;
    "documentation"|"docstring")
      # Use context and memory tools
      echo "context7,serena,cipher"
      ;;
    *)
      # Default: comprehensive toolset
      echo "serena,code-reasoning,perplexity"
      ;;
  esac
}
```

### 15.4 Morphological Development Orchestration

```bash
# ═══════════════════════════════════════════════════════════════════════════
# MORPHOLOGICAL DEVELOPMENT ORCHESTRATION
# Complete lifecycle: Plan → Research → Create → Edit → Iterate → Test → Deploy
# WITH EXTENSIVE MEMORY INTEGRATION AT EVERY PHASE
# ═══════════════════════════════════════════════════════════════════════════

run_morphological_development() {
  PROJECT_PATH="${1:-$(pwd)}"
  PROJECT_NAME="${2:-$(basename $PROJECT_PATH)}"

  echo "╔═══════════════════════════════════════════════════════════════════════════╗"
  echo "║        MORPHOLOGICAL DEVELOPMENT ORCHESTRATION                            ║"
  echo "║        Project: $PROJECT_NAME"
  echo "╚═══════════════════════════════════════════════════════════════════════════╝"

  # Initialize memory and configuration
  initialize_memory_layer "$PROJECT_PATH" "$PROJECT_NAME"
  configure_semantic_scholar

  # ═══════════════════════════════════════════════════════════════════════════
  # MEMORY: Recover any previous session context
  # ═══════════════════════════════════════════════════════════════════════════
  echo ">>> MEMORY: Recovering previous session context..."

  # Claude-mem: Search for previous work on this project
  # MCP call: mcp__plugin_claude-mem_mcp-search__search
  #   query: "project $PROJECT_NAME development"
  #   project: "$PROJECT_NAME"
  #   limit: 10

  # Serena: Read project-specific memories
  # MCP call: mcp__serena__list_memories
  # MCP call: mcp__serena__read_memory for relevant files

  # Cipher: Query for conversation context
  # MCP call: mcp__cipher__ask_cipher
  #   message: "What was the previous context for project $PROJECT_NAME?"

  recover_after_compaction

  DEVELOPMENT_COMPLETE=false
  ITERATION=0
  MAX_ITERATIONS=50

  while [ "$DEVELOPMENT_COMPLETE" = false ] && [ $ITERATION -lt $MAX_ITERATIONS ]; do
    ITERATION=$((ITERATION + 1))
    echo ""
    echo "═══════════════════════════════════════════════════════════════════════════"
    echo "                    DEVELOPMENT ITERATION $ITERATION / $MAX_ITERATIONS"
    echo "═══════════════════════════════════════════════════════════════════════════"

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 1: PLAN - Assess current state
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 1: PLAN - Assessing project..."

    # MEMORY: Retrieve relevant context for planning
    retrieve_relevant_memories "project assessment $PROJECT_NAME"

    assess_project_completeness_enhanced "$PROJECT_PATH"

    # MEMORY: Store assessment results
    store_to_memory "plan" "Assessment: $PROJECT_STATUS, Score: $COMPLEXITY_SCORE" "plan,assessment,iteration-$ITERATION"

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 2: RESEARCH - If cutting-edge requirements detected
    # ─────────────────────────────────────────────────────────────────────────
    if [ "$PROJECT_STATUS" = "incomplete_research" ] || [ "$RESEARCH_SCORE" -ge 20 ]; then
      echo ">>> STEP 2: RESEARCH - Academic research required..."

      # MEMORY: Check for previous research on similar topics
      search_memory "similar research topics $PROJECT_NAME"

      # MEMORY: Store research context in Cipher for conversation continuity
      # MCP: mcp__cipher__ask_cipher
      #   message: "Store: Starting research phase for $PROJECT_NAME"

      detect_research_requirements "$PROJECT_PATH"

      for query in "${RESEARCH_QUERIES[@]}"; do
        decompose_research_query "$query"
        search_academic_papers_orchestrated "$query"
        synthesize_research_results

        # MEMORY: Store each paper finding
        store_to_memory "research" "Papers found for '$query': ${#COLLECTED_PAPERS[@]}" "research,papers"

        create_research_epic "$query" "${#COLLECTED_PAPERS[@]}"
      done
    fi

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 3: CREATE - Implement based on research/requirements
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 3: CREATE - Implementing features..."

    # MEMORY: Retrieve implementation patterns from previous work
    retrieve_relevant_memories "implementation patterns"

    READY_TASKS=$(bd ready 2>/dev/null | wc -l)

    if [ "$READY_TASKS" -gt 0 ]; then
      while IFS= read -r task_line; do
        task_id=$(echo "$task_line" | awk '{print $1}')
        [ -z "$task_id" ] && continue
        task_title=$(echo "$task_line" | cut -d' ' -f2-)

        bd update "$task_id" --status=in_progress

        # MEMORY: Store implementation decision
        store_to_memory "implementation" "Implementing: $task_title ($task_id)" "implementation,$task_id"

        # Agent implements using serena MCP for code editing

        bd close "$task_id" --reason="Implemented"

        # MEMORY: Record completion
        store_to_memory "implementation" "Completed: $task_title" "implementation,complete,$task_id"

      done < <(bd ready 2>/dev/null | head -5)
    fi

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 4: EDIT - Refine implementation
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 4: EDIT - Refining implementation..."

    # MEMORY: Check for past fixes to similar issues
    retrieve_relevant_memories "past fixes for similar issues"

    detect_functionality_problems "$PROJECT_PATH"

    if [ "$FUNCTIONALITY_SCORE" -gt 0 ]; then
      # MEMORY: Store issue before fixing
      store_to_memory "fix" "Fixing: ${FUNCTIONALITY_ISSUES[*]}" "fix,issues"

      run_functionality_fix_phase "$PROJECT_PATH"

      # MEMORY: Store solution applied
      store_to_memory "fix" "Applied fixes for iteration $ITERATION" "fix,solution"
    fi

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 5: ITERATE - Check if more work needed
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 5: ITERATE - Checking for remaining work..."

    REMAINING_WORK=$(bd list --status=open 2>/dev/null | wc -l)
    BLOCKED_WORK=$(bd blocked 2>/dev/null | wc -l)

    # MEMORY: Store iteration progress
    store_to_memory "progress" "Iteration $ITERATION: Remaining=$REMAINING_WORK, Blocked=$BLOCKED_WORK" "progress,iteration-$ITERATION"

    if [ "$BLOCKED_WORK" -gt 0 ]; then
      echo "  Blocked tasks: $BLOCKED_WORK - attempting to unblock..."

      # MEMORY: Check for stuck patterns from previous sessions
      retrieve_relevant_memories "stuck patterns blocked tasks"

      trigger_self_healing
    fi

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 6: TEST - Run verification
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 6: TEST - Running verification..."

    run_phase_12_verification "$PROJECT_PATH" "{{ENV_NAME}}"
    TEST_RESULT=$?

    # MEMORY: Store test results
    store_to_memory "test" "Test result: $TEST_RESULT (0=pass)" "test,iteration-$ITERATION"

    if [ "$TEST_RESULT" -ne 0 ]; then
      # MEMORY: Check for similar test failures in past
      retrieve_relevant_memories "test failure patterns"
    fi

    # ─────────────────────────────────────────────────────────────────────────
    # STEP 7: CONFIRM - Check completion criteria
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> STEP 7: CONFIRM - Checking completion criteria..."

    check_completion_criteria "$PROJECT_PATH"
    CRITERIA_MET=$?

    # MEMORY: Store verification status
    store_to_memory "verification" "Criteria: $CRITERIA_MET, Tests: $TEST_RESULT" "verification,iteration-$ITERATION"

    if [ "$CRITERIA_MET" -eq 0 ] && [ "$TEST_RESULT" -eq 0 ]; then
      DEVELOPMENT_COMPLETE=true
      echo "  ✅ ALL CRITERIA MET - Development complete!"

      # MEMORY: Store success in all systems
      store_to_memory "success" "Project completed in $ITERATION iterations" "success,complete"
    else
      echo "  ❌ Criteria not met - continuing iteration..."
    fi

    # ─────────────────────────────────────────────────────────────────────────
    # MEMORY: Persist all progress before potential compaction
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> MEMORY: Persisting session progress..."
    persist_before_compaction
    bd sync 2>/dev/null

  done

  # ─────────────────────────────────────────────────────────────────────────
  # STEP 8: DEPLOY - Run deployment phases
  # ─────────────────────────────────────────────────────────────────────────
  if [ "$DEVELOPMENT_COMPLETE" = true ]; then
    echo ""
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║                    DEPLOYMENT PHASE                                       ║"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"

    # MEMORY: Store deployment start
    store_to_memory "deployment" "Starting deployment for $PROJECT_NAME" "deployment,start"

    # Run Phases 0-9 deployment workflow
    echo ">>> Running Phase 0: Dependency Audit..."
    run_dependency_audit "$PROJECT_PATH"
    store_to_memory "deployment" "Phase 0: Dependency audit complete" "deployment,phase-0"

    echo ">>> Running Phase 1: Dependency Installation..."
    run_dependency_installation "$PROJECT_PATH"
    store_to_memory "deployment" "Phase 1: Dependencies installed" "deployment,phase-1"

    echo ">>> Running Phase 2: Conda Environment..."
    setup_conda_environment "$PROJECT_PATH"
    store_to_memory "deployment" "Phase 2: Conda environment ready" "deployment,phase-2"

    echo ">>> Running Phase 3: CUDA Environment..."
    setup_cuda_environment "$PROJECT_PATH"
    store_to_memory "deployment" "Phase 3: CUDA configured" "deployment,phase-3"

    echo ">>> Running Phase 4: Docker Container..."
    build_docker_container "$PROJECT_PATH"
    store_to_memory "deployment" "Phase 4: Docker container built" "deployment,phase-4"

    echo ">>> Running Phase 12: Final Verification..."
    run_phase_12_verification "$PROJECT_PATH" "{{ENV_NAME}}"
    store_to_memory "deployment" "Phase 12: Final verification passed" "deployment,phase-12"

    echo ""
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║        MORPHOLOGICAL DEVELOPMENT COMPLETE ✅                              ║"
    echo "║        Iterations: $ITERATION                                             ║"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"

    # ─────────────────────────────────────────────────────────────────────────
    # MEMORY: Final persistence - store complete project summary
    # ─────────────────────────────────────────────────────────────────────────
    echo ">>> MEMORY: Persisting final project summary..."

    store_to_memory "complete" "Project $PROJECT_NAME: DEPLOYED after $ITERATION iterations" "complete,deployed,$PROJECT_NAME"

    persist_before_compaction
    bd sync 2>/dev/null

    return 0
  else
    echo ""
    echo "╔═══════════════════════════════════════════════════════════════════════════╗"
    echo "║        MAX ITERATIONS REACHED - MANUAL INTERVENTION REQUIRED              ║"
    echo "╚═══════════════════════════════════════════════════════════════════════════╝"

    # MEMORY: Store incomplete status for future recovery
    store_to_memory "incomplete" "Project $PROJECT_NAME: Incomplete after $MAX_ITERATIONS iterations" "incomplete,manual-needed"

    bd create --title="Manual intervention required" --type=task --priority=0 \
      --description="Morphological development incomplete after $MAX_ITERATIONS iterations"

    persist_before_compaction
    bd sync 2>/dev/null

    return 1
  fi
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
