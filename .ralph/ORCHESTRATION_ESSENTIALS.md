# Ralph-Loop Orchestration Essentials (v2.0)

> **Quick Reference**: Condensed orchestration guide for token-efficient operations.
> For full documentation, use `ralph-context <phase>` to retrieve specific phases.

---

## v2.0 Enhancements Summary

| Enhancement | Function | Description |
|-------------|----------|-------------|
| **Dynamic Routing** | `decide_next_phase()` | Any phase can route to any other based on state |
| **Guaranteed Completion** | `run_master_orchestration_v2()` | No hard limits, escalation-based |
| **Self-Healing** | `trigger_self_healing_enhanced()` | Actually resolves blockers |
| **bd sync Every Iteration** | Built-in | Persists Beads state continuously |
| **Conditional MCP Selection** | `select_mcp_tools_for_error()` | Routes to tools by error type |
| **Research Feedback Loop** | `run_research_with_feedback()` | Iterates until confidence threshold |
| **Cross-Iteration Aggregation** | `aggregate_research_across_iterations()` | Merges findings from previous sessions |

### Global CLI Commands
```bash
ralph-init [PROJECT_PATH]       # Initialize orchestration in any project
ralph-loop --orchestrator       # Run full autonomous orchestration (v2.0)
ralph-loop --phase N            # Run specific phase only
```

---

## Phase Reference

| Phase | File | Purpose | Entry Point |
|-------|------|---------|-------------|
| 0 | PART1 | System Audit (ALWAYS FIRST) | `run_system_audit()` |
| 1 | PART1 | Dependency Analysis | `analyze_project_dependencies()` |
| 2 | PART1 | Conda Environment | `create_env_if_not_exists()` |
| 3 | PART1 | CUDA Installation | `install_cuda_packages()` |
| 4 | PART1 | Standard Dependencies | `install_standard_deps()` |
| 5 | PART1 | Database Setup | `setup_database()` |
| 6 | PART1 | Docker Port Audit | `audit_docker_ports()` |
| 7 | PART1 | MCP Tool Orchestration | `configure_mcp_servers()` |
| 8 | PART1 | Ralph-Loop Protocol | `ralph_loop_entry()` |
| 9 | PART1 | Verification & Testing | `verify_environment()` |
| 10 | PART2 | Gap Detection | `analyze_and_create_gap_task()` |
| 10.5 | PART2 | Project Completeness | `assess_project_completeness()` |
| 11 | PART2 | Development Completion | `run_development_completion()` |
| 12 | PART2 | Test Execution | `run_all_tests()` |
| 13 | PART2 | Academic Research | `run_academic_research()` |
| 14 | PART2 | Functionality Fix | `fix_functionality_problems()` |
| 15 | PART2 | Master Orchestration | `run_master_orchestration()` |

---

## Core Functions

### Entry Points
```bash
ralph_loop_entry(PROJECT_PATH)          # Main loop entry
run_morphological_development(PROJECT_PATH, PROJECT_NAME)  # Full dev cycle
run_master_orchestration()              # Orchestration controller
```

### Assessment Functions
```bash
assess_project_completeness()           # Returns: complete|incomplete_critical|incomplete_minor|incomplete_research|functionality_problems
assess_project_completeness_enhanced()  # Extended version with FUNCTIONALITY_SCORE
check_completion_criteria()             # Final verification
quick_completion_check()                # Fast pre-check
```

### Development Functions
```bash
run_development_completion()            # Execute dev tasks
implement_missing_function()            # Stub implementation
generate_test_for_function()            # Create tests
trigger_self_healing()                  # Fix broken code
```

### Memory Functions (claude-mem MCP)
```bash
initialize_memory_layer()               # Setup memory
store_to_memory()                       # Persist learning
retrieve_relevant_memories()            # Context recovery
persist_before_compaction()             # Pre-clear save
recover_after_compaction()              # Post-clear restore
```

---

## MCP Server Priority (Research)

### Semantic Scholar (Primary)
```
1. mcp-semantic-scholar-server    → Simple keyword search (1 req/sec, max 3 queries)
   Tool: mcp__mcp-semantic-scholar-server__search_papers_via_semanticscholar

2. semantic-scholar-graph-api     → Citations, recommendations, batch operations
   Tools: get_citations_and_references, get_paper_recommendations, search_snippets

3. research-semantic-scholar      → Fallback, BibTeX generation
   Tools: search_semantic_scholar, get_paper, semantic_scholar_to_bibtex
```

### arXiv
```
arxiv-advanced                    → ML/AI preprints, full-text download
Tools: search_papers, download_paper, read_paper
```

### Other Sources
```
paper-search                      → Multi-platform aggregation
perplexity                        → Real-time web research
linkup                            → Web search and extraction
```

---

## Project Status Routing (v2.0 Dynamic)

```
┌─────────────────────────────────────────────────────────────────┐
│          DYNAMIC PHASE ROUTING (v2.0 - decide_next_phase)        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRIORITY ORDER:                                                │
│  1. Environment broken     → Phase 2-4 (Conda/CUDA/Deps)        │
│  2. Blocked tasks          → Self-healing                       │
│  3. Code incomplete        → Phase 13 (if research_score > 20)  │
│                           → Phase 11 (otherwise)                │
│  4. Tests failing          → Phase 14 (Fix functionality)       │
│  5. Ready tasks exist      → Phase 11 (Development)             │
│  6. All criteria met       → Verify completion                  │
│                                                                 │
│  STUCK DETECTION:                                               │
│  - No progress for 5 iterations → Escalation Level 1            │
│    → Aggressive self-healing                                    │
│  - Still stuck → Escalation Level 2                             │
│    → Human intervention (creates task, waits)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Legacy Routing (v1.0 - still supported)
```
complete                → Phase 9 (Deploy/Verify)
functionality_problems  → Phase 14 (Fix) → Re-assess
incomplete_research     → Phase 13 (Research) → Phase 11
incomplete_critical     → Phase 11 (Dev Completion)
incomplete_minor        → Phase 11 (Dev Completion)
```

---

## Completion Criteria Checklist

```
[ ] No NotImplementedError in code
    grep -rn "NotImplementedError" . --include="*.py" | grep -v __pycache__

[ ] All imports work
    python -c "import sys; sys.path.insert(0,'.'); import <main_module>"

[ ] All tests pass
    conda run -n <env> pytest tests/ -x -q

[ ] No blocked Beads tasks
    bd blocked  # Should return empty

[ ] No open Beads tasks
    bd list --status=open  # Should return empty

[ ] No in_progress Beads tasks
    bd list --status=in_progress  # Should return empty

[ ] FUNCTIONALITY_SCORE = 0
    (No broken imports, no runtime errors, no failed tests)
```

---

## Quick Decision Trees

### Gap Detection Priority
```
1. NotImplementedError/raise NotImplemented
2. TODO/FIXME/HACK comments
3. "pass" only functions
4. Uncovered branches (test coverage)
5. Missing docstrings
```

### Test Failure Resolution
```
ImportError      → Fix imports, check __init__.py
AttributeError   → Check class definitions, method names
AssertionError   → Fix test logic or implementation
TypeError        → Check function signatures
RuntimeError     → Check execution flow
```

### Self-Healing Triggers
```
- Test failure after code change
- Import error in new module
- Circular import detected
- Missing dependency
- Configuration error
```

---

## Beads Integration

### Task States
```
open         → Available work
in_progress  → Being worked on
blocked      → Waiting on dependency
closed       → Complete
```

### Key Commands
```bash
bd ready                            # Find available work
bd list --status=in_progress        # Current work
bd show <id>                        # Task details
bd update <id> --status=in_progress # Claim work
bd close <id>                       # Complete work
bd dep add <task> <depends-on>      # Add dependency
bd sync                             # Sync with git
```

---

## Iteration Protocol Summary

1. **Context Recovery** (if compacted)
   ```bash
   mcp__serena__read_memory memory_file_name="orchestration-essentials.md"
   bd ready
   ```

2. **Get Current Task**
   - Read `.ralph/prd.json`
   - Or `bd list --status=in_progress`

3. **Phase-Specific Docs**
   ```bash
   ralph-context <phase-number>
   ```

4. **Execute Task**
   - Follow phase documentation
   - Use appropriate MCP tools

5. **Update State**
   - Mark prd.json task: `passes: true`
   - Update Beads: `bd close <id>`
   - Append to progress.txt

6. **Completion Signal**
   ```
   <promise>COMPLETE</promise>
   ```

---

## File Locations

```
.ralph/
├── PART1_INFRASTRUCTURE.md      # Phases 0-9 (~1656 lines)
├── PART2A_PHASES_10-12.md       # Phases 10-12: Gap Detection, Assessment, Development
├── PART2B_PHASE_13.md           # Phase 13: Academic Research Integration
├── PART2C_PHASES_14-15.md       # Phases 14-15: Functionality Fix, Master Orchestration
├── ORCHESTRATION_ESSENTIALS.md  # This file (~400 lines)
├── ORCHESTRATION_VISUAL.md      # Visual diagrams and flow charts
├── prd.json                     # Current task definition
├── progress.txt                 # Iteration log
└── prompt.md                    # Loop iteration prompt
```

---

## Emergency Recovery

If context is lost or session is unclear:

```bash
# 1. Check what's happening
bd list --status=in_progress
cat .ralph/progress.txt | tail -20

# 2. Recover orchestration context
mcp__serena__read_memory memory_file_name="orchestration-essentials.md"

# 3. Get phase documentation
ralph-context <current-phase>

# 4. Resume from last checkpoint
```
