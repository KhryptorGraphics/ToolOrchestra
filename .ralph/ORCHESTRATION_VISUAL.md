# Ralph Orchestration Visual Documentation

> **Purpose**: Comprehensive visual guide showing how Ralph orchestration works from start to finish.
> **Version**: 2.0.0 | **Updated**: 2026-01-17

---

## Table of Contents

0. [v2.0 Dynamic Routing (NEW)](#0-v20-dynamic-routing-new)
1. [Master Orchestration Flow](#1-master-orchestration-flow)
2. [Phase Relationship Diagram](#2-phase-relationship-diagram)
3. [Project Status Decision Tree](#3-project-status-decision-tree)
4. [Memory Integration System](#4-memory-integration-system)
5. [MCP Tool Priority Orchestration](#5-mcp-tool-priority-orchestration)
6. [ASCII Art Diagrams](#6-ascii-art-diagrams-terminal-display)

---

## 0. v2.0 Dynamic Routing (NEW)

The v2.0 orchestration uses `decide_next_phase()` for dynamic routing - any phase can route to any other phase based on current state analysis.

```mermaid
flowchart TD
    subgraph ENTRY["GLOBAL CLI"]
        CLI1["ralph-init PROJECT_PATH"] --> INIT["Initialize .ralph/"]
        INIT --> CLI2["ralph-loop --orchestrator"]
    end

    CLI2 --> ASSESS

    subgraph V2LOOP["v2.0 MASTER LOOP (No Hard Limits)"]
        ASSESS["STEP 1: Assess State<br/>- Environment state<br/>- Code state (NotImpl)<br/>- Test state<br/>- Research score<br/>- Beads state"]

        ASSESS --> DECIDE{"decide_next_phase()"}

        DECIDE -->|"ENV_BROKEN"| P2["Phase 2-4<br/>Environment Rebuild"]
        DECIDE -->|"BLOCKED_COUNT > 0"| HEAL["Self-Healing<br/>trigger_self_healing_enhanced()"]
        DECIDE -->|"CODE_INCOMPLETE && RESEARCH > 20"| P13["Phase 13<br/>Academic Research"]
        DECIDE -->|"CODE_INCOMPLETE"| P11["Phase 11<br/>Development"]
        DECIDE -->|"TESTS_FAILING"| P14["Phase 14<br/>Fix Functionality"]
        DECIDE -->|"READY_COUNT > 0"| P11
        DECIDE -->|"All clear"| VERIFY["Verify Completion"]

        P2 --> EXEC["STEP 2: Execute Phase"]
        HEAL --> EXEC
        P13 --> EXEC
        P11 --> EXEC
        P14 --> EXEC
        VERIFY --> EXEC

        EXEC --> CHECK["STEP 3: Check Criteria<br/>check_completion_criteria()"]

        CHECK --> COMPLETE{CRITERIA_RESULT == 0?}
        COMPLETE -->|Yes| DONE([" PROJECT COMPLETE"])

        COMPLETE -->|No| SYNC["STEP 5: bd sync"]
        SYNC --> STUCK{"stuck_for_too_long()?"}

        STUCK -->|No| ASSESS
        STUCK -->|Yes| ESCALATE["Escalation Level++"]

        ESCALATE -->|"Level 1"| HEAL2["Aggressive Self-Healing"]
        ESCALATE -->|"Level 2"| HUMAN["Human Intervention<br/>generate_stuck_report()"]
        ESCALATE -->|"Level 3+"| RESET["Reset & Retry"]

        HEAL2 --> ASSESS
        HUMAN --> ASSESS
        RESET --> ASSESS
    end

    style DONE fill:#2e7d32,color:#fff
    style ASSESS fill:#1976d2,color:#fff
    style DECIDE fill:#7b1fa2,color:#fff
```

### Key v2.0 Features

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    v2.0 ENHANCEMENT SUMMARY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. DYNAMIC ROUTING (decide_next_phase)                                     │
│     ├─ Multi-signal state analysis                                          │
│     ├─ Any phase → Any phase routing                                        │
│     └─ Priority-based selection matrix                                      │
│                                                                             │
│  2. GUARANTEED COMPLETION (No MAX_ITERATIONS)                               │
│     ├─ while true loop (no hard cap)                                        │
│     ├─ Stuck detection with escalation                                      │
│     └─ Human-assisted mode for truly stuck states                           │
│                                                                             │
│  3. ENHANCED SELF-HEALING                                                   │
│     ├─ Actually resolves blockers (not just creates tasks)                  │
│     ├─ Import error → pip install                                           │
│     ├─ Test failure → Phase 14                                              │
│     └─ Memory query for similar past solutions                              │
│                                                                             │
│  4. BD SYNC EVERY ITERATION                                                 │
│     └─ Beads state persisted continuously                                   │
│                                                                             │
│  5. CONDITIONAL MCP SELECTION (select_mcp_tools_for_error)                  │
│     ├─ ImportError → serena, code-context                                   │
│     ├─ RuntimeError → sequential-thinking, code-reasoning                   │
│     ├─ research_needed → arxiv, semantic-scholar, perplexity                │
│     └─ external_api → tavily, linkup, playwright                            │
│                                                                             │
│  6. RESEARCH FEEDBACK LOOP                                                  │
│     ├─ Confidence calculation (0-100)                                       │
│     ├─ Gap identification (missing aspects)                                 │
│     └─ New query generation until threshold met                             │
│                                                                             │
│  7. CROSS-ITERATION AGGREGATION                                             │
│     └─ Merges findings from previous sessions                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Master Orchestration Flow

Complete end-to-end flow from `ralph-init` through project completion.

```mermaid
flowchart TD
    subgraph ENTRY["ENTRY POINTS"]
        START([ralph-init]) --> INIT[Initialize .ralph/]
        INIT --> PRD[Create prd.json]
        PRD --> PROMPT[Generate prompt.md]
        PROMPT --> LOOP([ralph-loop])
    end

    subgraph INFRASTRUCTURE["PART 1: INFRASTRUCTURE (Phases 0-9)"]
        LOOP --> P0[Phase 0: System Audit]
        P0 --> P1[Phase 1: Dependency Analysis]
        P1 --> P2[Phase 2: Conda Environment]
        P2 --> P3[Phase 3: CUDA Installation]
        P3 --> P4[Phase 4: Standard Dependencies]
        P4 --> P5[Phase 5: Database Setup]
        P5 --> P6[Phase 6: Docker Port Audit]
        P6 --> P7[Phase 7: MCP Tool Orchestration]
        P7 --> P8[Phase 8: Ralph-Loop Entry]
        P8 --> P9[Phase 9: Verification & Testing]
    end

    subgraph EXECUTION["PART 2: EXECUTION (Phases 10-15)"]
        P9 --> P10[Phase 10: Gap Detection]
        P10 --> P105[Phase 10.5: Assess Completeness]

        P105 --> STATUS{assess_project_completeness}

        STATUS -->|complete| DEPLOY[Deploy & Verify]
        STATUS -->|functionality_problems| P14[Phase 14: Fix Issues]
        STATUS -->|incomplete_research| P13[Phase 13: Research]
        STATUS -->|incomplete_critical| P11[Phase 11: Dev Completion]
        STATUS -->|incomplete_minor| P11

        P13 --> P11
        P14 --> P105
        P11 --> P12[Phase 12: Test Execution]
        P12 --> P105
    end

    subgraph COMPLETE["COMPLETION"]
        DEPLOY --> VERIFY{All Criteria Met?}
        VERIFY -->|Yes| DONE([PROJECT COMPLETE])
        VERIFY -->|No| P105
    end

    subgraph MASTER["Phase 15: Master Orchestration Controller"]
        direction TB
        M1[ASSESS: Project State]
        M2[ROUTE: Based on Status]
        M3[EXECUTE: Run Phase]
        M4[VERIFY: Check Criteria]
        M5[ITERATE or COMPLETE]
        M1 --> M2 --> M3 --> M4 --> M5
        M5 -->|iterate| M1
    end

    STATUS -.-> MASTER
```

---

## 2. Phase Relationship Diagram

Shows how phases connect and which phases call which.

```mermaid
flowchart LR
    subgraph PART1["PART 1: INFRASTRUCTURE (Phases 0-9)"]
        direction TB
        P0["0: System Audit<br/>run_system_audit()"]
        P1["1: Dependency Analysis<br/>analyze_project_dependencies()"]
        P2["2: Conda Environment<br/>create_env_if_not_exists()"]
        P3["3: CUDA Installation<br/>install_cuda_packages()"]
        P4["4: Standard Dependencies<br/>install_standard_deps()"]
        P5["5: Database Setup<br/>setup_database()"]
        P6["6: Docker Port Audit<br/>audit_docker_ports()"]
        P7["7: MCP Tool Orchestration<br/>configure_mcp_servers()"]
        P8["8: Ralph-Loop Entry<br/>ralph_loop_entry()"]
        P9["9: Verification & Testing<br/>verify_environment()"]

        P0 --> P1 --> P2 --> P3 --> P4 --> P5 --> P6 --> P7 --> P8 --> P9
    end

    subgraph PART2["PART 2: EXECUTION (Phases 10-15)"]
        direction TB
        P10["10: Gap Detection<br/>analyze_and_create_gap_task()"]
        P105["10.5: Project Assessment<br/>assess_project_completeness()"]
        P11["11: Dev Completion<br/>run_development_completion()"]
        P12["12: Test Execution<br/>run_all_tests()"]
        P13["13: Academic Research<br/>run_academic_research()"]
        P14["14: Functionality Fix<br/>fix_functionality_problems()"]
        P15["15: Master Orchestration<br/>run_master_orchestration()"]

        P10 --> P105
        P105 --> P11
        P105 --> P13
        P105 --> P14
        P13 --> P11
        P11 --> P12
        P12 --> P105
        P14 --> P105
        P15 -.->|controls| P105
    end

    PART1 ==> PART2
    P9 --> P10

    style P0 fill:#e1f5fe
    style P15 fill:#fff3e0
    style P105 fill:#f3e5f5
```

---

## 3. Project Status Decision Tree

Routing logic based on `assess_project_completeness()` results.

```mermaid
flowchart TD
    ASSESS[["assess_project_completeness()<br/>Returns PROJECT_STATUS"]]

    ASSESS --> COMPLETE{complete?}
    COMPLETE -->|Yes| DEPLOY["Phases 0-9: Deploy<br/>run_phase_12_verification()"]
    DEPLOY --> VERIFY{All Criteria Met?}
    VERIFY -->|Yes| DONE(["PROJECT COMPLETE"])
    VERIFY -->|No| ASSESS

    COMPLETE -->|No| FUNC{functionality_problems?}
    FUNC -->|Yes| P14["Phase 14: Functionality Fix<br/>run_functionality_fix_phase()"]
    P14 --> ASSESS

    FUNC -->|No| RESEARCH{incomplete_research?}
    RESEARCH -->|Yes| P13["Phase 13: Academic Research<br/>run_research_phase()"]
    P13 --> P11A["Phase 11: Dev Completion<br/>run_development_completion()"]

    RESEARCH -->|No| CRITICAL{incomplete_critical<br/>or incomplete_minor?}
    CRITICAL -->|Yes| P11B["Phase 11: Dev Completion<br/>run_development_completion()"]

    P11A --> P12A["Phase 12: Test Execution<br/>run_all_tests()"]
    P11B --> P12B["Phase 12: Test Execution<br/>run_all_tests()"]

    P12A --> ASSESS
    P12B --> ASSESS

    style ASSESS fill:#e8eaf6
    style DONE fill:#c8e6c9
    style P14 fill:#ffcdd2
    style P13 fill:#fff9c4
    style P11A fill:#bbdefb
    style P11B fill:#bbdefb
```

---

## 4. Memory Integration System

Three-tier memory system for cross-session context retention.

```mermaid
flowchart TD
    subgraph MEMORY["THREE-TIER MEMORY SYSTEM"]
        direction TB

        subgraph TIER1["TIER 1: SERENA MCP"]
            S1[Project Patterns]
            S2[Code Memory]
            S3[Symbol Relationships]
            S4[Architecture Insights]
        end

        subgraph TIER2["TIER 2: CIPHER MCP"]
            C1[Conversation Memory]
            C2[Past Solutions]
            C3[Decision History]
            C4[Error Patterns]
        end

        subgraph TIER3["TIER 3: CLAUDE-MEM (3-Layer)"]
            direction TB
            L1["Layer 1: SEARCH<br/>mcp__plugin_claude-mem__search<br/>Get index with IDs (~50-100 tokens)"]
            L2["Layer 2: TIMELINE<br/>mcp__plugin_claude-mem__timeline<br/>Get context around results"]
            L3["Layer 3: GET_OBSERVATIONS<br/>mcp__plugin_claude-mem__get_observations<br/>Fetch full details for filtered IDs"]
            L1 --> L2 --> L3
        end
    end

    subgraph FUNCTIONS["MEMORY FUNCTIONS"]
        F1[initialize_memory_layer]
        F2[store_to_memory]
        F3[retrieve_relevant_memories]
        F4[persist_before_compaction]
        F5[recover_after_compaction]
    end

    subgraph USAGE["WHEN TO USE"]
        U1["Session Start: recover_after_compaction()"]
        U2["Task Complete: store_to_memory()"]
        U3["Error Found: search_memory()"]
        U4["Session End: persist_before_compaction()"]
    end

    TIER1 --> F1
    TIER2 --> F2
    TIER3 --> F3
    FUNCTIONS --> USAGE

    style TIER1 fill:#e3f2fd
    style TIER2 fill:#f3e5f5
    style TIER3 fill:#e8f5e9
```

---

## 5. MCP Tool Priority Orchestration

Five phases of tool orchestration based on task type.

```mermaid
flowchart TB
    subgraph PHASEA["PHASE A: MEMORY & CONTEXT (Always First)"]
        direction LR
        A1["1. serena<br/>Code memory, patterns"]
        A2["2. cipher<br/>Conversation memory"]
        A3["3. mcp-search<br/>Stored observations"]
        A4["4. code-context<br/>Codebase structure"]
        A1 --> A2 --> A3 --> A4
    end

    subgraph PHASEB["PHASE B: REASONING & ANALYSIS"]
        direction LR
        B1["5. sequential-thinking<br/>Step-by-step breakdown"]
        B2["6. think-strategies<br/>Multi-strategy reasoning"]
        B3["7. code-reasoning<br/>Code-specific analysis"]
        B1 --> B2 --> B3
    end

    subgraph PHASEC["PHASE C: DOCUMENTATION & RESEARCH"]
        direction LR
        C1["8. context7<br/>Library docs"]
        C2["9. tavily<br/>Web search (primary)"]
        C3["10. linkup<br/>Web search (secondary)"]
        C1 --> C2 --> C3
    end

    subgraph PHASED["PHASE D: INTERACTIVE TESTING"]
        direction LR
        D1["11. playwright<br/>Browser automation"]
        D2["12. puppeteer<br/>Alternative browser"]
        D3["13. skyvern<br/>AI web interaction"]
        D1 --> D2 --> D3
    end

    subgraph PHASEE["PHASE E: CODE OPERATIONS"]
        direction LR
        E1["14. github-mcp<br/>PRs, issues, code search"]
    end

    PHASEA ==> PHASEB ==> PHASEC ==> PHASED ==> PHASEE

    style PHASEA fill:#e8eaf6
    style PHASEB fill:#fff3e0
    style PHASEC fill:#e0f2f1
    style PHASED fill:#fce4ec
    style PHASEE fill:#f3e5f5
```

### Research-Specific MCP Priority (Phase 13)

```mermaid
flowchart LR
    subgraph SEMANTIC["SEMANTIC SCHOLAR (Primary)"]
        SS1["mcp-semantic-scholar-server<br/>Simple keyword search<br/>1 req/sec, max 3 queries"]
        SS2["semantic-scholar-graph-api<br/>Citations, recommendations<br/>Batch operations"]
        SS3["research-semantic-scholar<br/>Fallback, BibTeX"]
        SS1 --> SS2 --> SS3
    end

    subgraph ARXIV["arXiv"]
        AX1["arxiv-advanced<br/>ML/AI preprints<br/>Full-text download"]
    end

    subgraph OTHER["OTHER SOURCES"]
        O1["paper-search<br/>Multi-platform"]
        O2["perplexity<br/>Real-time research"]
        O3["linkup<br/>Web extraction"]
    end

    SEMANTIC --> ARXIV --> OTHER

    style SEMANTIC fill:#e3f2fd
    style ARXIV fill:#fff8e1
    style OTHER fill:#f1f8e9
```

---

## 6. ASCII Art Diagrams (Terminal Display)

For display in terminals without Mermaid rendering support.

### 6.1 Master Flow (80-char width)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RALPH MASTER ORCHESTRATION FLOW                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐              │
│  │ ralph-init   │─────►│ Initialize   │─────►│ ralph-loop   │              │
│  │              │      │ .ralph/      │      │              │              │
│  └──────────────┘      └──────────────┘      └──────┬───────┘              │
│                                                      │                      │
│  ════════════════════════════════════════════════════╪════════════════════  │
│  PART 1: INFRASTRUCTURE (Phases 0-9)                 ▼                      │
│  ════════════════════════════════════════════════════════════════════════   │
│                                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│  │ Phase 0 │►│ Phase 1 │►│ Phase 2 │►│ Phase 3 │►│ Phase 4 │              │
│  │ Audit   │ │ Deps    │ │ Conda   │ │ CUDA    │ │ Install │              │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └────┬────┘              │
│                                                       │                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │                     │
│  │ Phase 9 │◄│ Phase 8 │◄│ Phase 7 │◄│ Phase 6 │◄────┘                     │
│  │ Verify  │ │ Entry   │ │ MCP     │ │ Docker  │ ◄─── Phase 5 (DB)         │
│  └────┬────┘ └─────────┘ └─────────┘ └─────────┘                           │
│       │                                                                     │
│  ═════╪═════════════════════════════════════════════════════════════════   │
│  PART│2: EXECUTION (Phases 10-15)                                          │
│  ═════╪═════════════════════════════════════════════════════════════════   │
│       ▼                                                                     │
│  ┌──────────┐     ┌────────────┐                                           │
│  │ Phase 10 │────►│ Phase 10.5 │                                           │
│  │ Gap Det. │     │ Assessment │◄──────────────────────────────────┐       │
│  └──────────┘     └─────┬──────┘                                   │       │
│                         │                                          │       │
│              ┌──────────┼──────────┬──────────┬──────────┐         │       │
│              ▼          ▼          ▼          ▼          ▼         │       │
│         ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐     │       │
│         │complete│ │critical│ │ minor  │ │research│ │func_   │     │       │
│         └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ │problems│     │       │
│             │          │          │          │      └───┬────┘     │       │
│             ▼          │          │          ▼          │          │       │
│        ┌────────┐      │          │     ┌────────┐      │          │       │
│        │ DEPLOY │      └──────────┼────►│Phase 13│      │          │       │
│        │ (0-9)  │                 │     │Research│      ▼          │       │
│        └───┬────┘                 │     └───┬────┘ ┌────────┐      │       │
│            │                      ▼         │      │Phase 14│      │       │
│            ▼                 ┌────────┐     │      │  Fix   │      │       │
│       ┌─────────┐            │Phase 11│◄────┘      └───┬────┘      │       │
│       │ VERIFY  │            │  Dev   │                │          │       │
│       └────┬────┘            └───┬────┘                └──────────┘       │
│            │                     │                                         │
│     ┌──────┴──────┐              ▼                                         │
│     │             │         ┌────────┐                                     │
│     ▼             ▼         │Phase 12│                                     │
│  ┌──────┐    ┌────────┐     │ Test   │                                     │
│  │ DONE │    │ ITERATE│     └───┬────┘                                     │
│  │      │    │        │         │                                          │
│  └──────┘    └────┬───┘         └─────────────────────────────────────────►┤
│                   │                                                         │
│                   └────────────────────────────────────────────────────────►┤
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Phase Groups

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RALPH ORCHESTRATION PHASES                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────┐    ┌─────────────────────────────┐        │
│  │     INFRASTRUCTURE          │    │       EXECUTION              │        │
│  │     (PART1: 0-9)           │    │       (PART2: 10-15)        │        │
│  │                             │    │                             │        │
│  │  0. System Audit ◄──────────┼────┤  10.  Gap Detection         │        │
│  │  1. Dependency Analysis     │    │  10.5 Project Assessment ◄──┼─┐      │
│  │  2. Conda Environment       │    │  11.  Dev Completion     ───┼─┤      │
│  │  3. CUDA Installation       │    │  12.  Test Execution     ───┼─┤      │
│  │  4. Standard Dependencies   │    │  13.  Academic Research  ───┼─┤      │
│  │  5. Database Setup          │    │  14.  Functionality Fix  ───┼─┘      │
│  │  6. Docker Port Audit       │    │  15.  Master Orchestration  │        │
│  │  7. MCP Tool Orchestration  │    │                             │        │
│  │  8. Ralph-Loop Entry ───────┼────►                             │        │
│  │  9. Verification & Testing  │    │                             │        │
│  │                             │    │                             │        │
│  └─────────────────────────────┘    └─────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Decision Routing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     PROJECT STATUS ROUTING                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                    assess_project_completeness()                            │
│                              │                                              │
│              ┌───────────────┼───────────────┐                              │
│              ▼               ▼               ▼                              │
│         ┌────────┐     ┌──────────┐    ┌────────────┐                       │
│         │complete│     │incomplete│    │functionality│                      │
│         └────┬───┘     └────┬─────┘    │  problems  │                       │
│              │              │          └─────┬──────┘                       │
│              ▼              │                │                              │
│      ┌──────────────┐       │                ▼                              │
│      │ Deploy (0-9) │       │         ┌────────────┐                        │
│      │              │       │         │ Phase 14   │                        │
│      │ >>> COMPLETE │       │         │ Fix Issues │                        │
│      └──────────────┘       │         └─────┬──────┘                        │
│                             │               │                               │
│              ┌──────────────┼───────────────┘                               │
│              ▼              ▼                                               │
│         ┌─────────┐    ┌──────────┐                                         │
│         │critical │    │ research │                                         │
│         │ /minor  │    │          │                                         │
│         └────┬────┘    └────┬─────┘                                         │
│              │              │                                               │
│              │              ▼                                               │
│              │      ┌────────────┐                                          │
│              │      │ Phase 13   │                                          │
│              │      │ Research   │                                          │
│              │      └─────┬──────┘                                          │
│              │            │                                                 │
│              └─────┬──────┘                                                 │
│                    ▼                                                        │
│            ┌────────────┐                                                   │
│            │ Phase 11   │                                                   │
│            │ Dev Work   │                                                   │
│            └─────┬──────┘                                                   │
│                  │                                                          │
│                  ▼                                                          │
│            ┌────────────┐                                                   │
│            │ Phase 12   │                                                   │
│            │ Testing    │◄──────────────────────────────────┐               │
│            └─────┬──────┘                                   │               │
│                  │                                          │               │
│                  ▼                                          │               │
│            ┌────────────┐      ┌─────────┐                  │               │
│            │ Re-assess  │─────►│ ITERATE │──────────────────┘               │
│            │            │      │ (Loop)  │                                  │
│            └────────────┘      └─────────┘                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 Memory System

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THREE-TIER MEMORY SYSTEM                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ TIER 1: SERENA MCP                                                    │  │
│  │ ├─ Project patterns        (mcp__serena__read_memory)                 │  │
│  │ ├─ Code memory             (mcp__serena__write_memory)                │  │
│  │ ├─ Symbol relationships    (mcp__serena__find_symbol)                 │  │
│  │ └─ Architecture insights   (mcp__serena__list_memories)               │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                     │                                       │
│                                     ▼                                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ TIER 2: CIPHER MCP                                                    │  │
│  │ ├─ Conversation memory     (mcp__cipher__ask_cipher)                  │  │
│  │ ├─ Past solutions          "Recall previous context for: X"           │  │
│  │ ├─ Decision history        "What decisions were made about: Y"        │  │
│  │ └─ Error patterns          "What errors occurred with: Z"             │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                     │                                       │
│                                     ▼                                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ TIER 3: CLAUDE-MEM (3-Layer Workflow)                                 │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │ LAYER 1: SEARCH                                                 │  │  │
│  │  │ mcp__plugin_claude-mem_mcp-search__search                       │  │  │
│  │  │ Returns: Index with IDs (~50-100 tokens/result)                 │  │  │
│  │  └────────────────────────────┬────────────────────────────────────┘  │  │
│  │                               ▼                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │ LAYER 2: TIMELINE                                               │  │  │
│  │  │ mcp__plugin_claude-mem_mcp-search__timeline                     │  │  │
│  │  │ Returns: Context around interesting results                     │  │  │
│  │  └────────────────────────────┬────────────────────────────────────┘  │  │
│  │                               ▼                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │ LAYER 3: GET_OBSERVATIONS                                       │  │  │
│  │  │ mcp__plugin_claude-mem_mcp-search__get_observations             │  │  │
│  │  │ Returns: Full details for filtered IDs ONLY (10x token savings) │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  KEY FUNCTIONS:                                                             │
│  ├─ initialize_memory_layer()    - Setup at session start                   │
│  ├─ store_to_memory()            - Persist learning after task              │
│  ├─ retrieve_relevant_memories() - Search for context                       │
│  ├─ persist_before_compaction()  - Save before /clear                       │
│  └─ recover_after_compaction()   - Restore after /clear                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.5 MCP Tool Priority

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
├─────────────────────────────────────────────────────────────────────────────┤
│  RESEARCH-SPECIFIC (Phase 13 Academic Research):                           │
│                                                                             │
│  SEMANTIC SCHOLAR (Primary - use in order):                                │
│  ├─ 1. mcp-semantic-scholar-server → Simple keyword (1 req/sec, max 3)     │
│  ├─ 2. semantic-scholar-graph-api  → Citations, recommendations, batch     │
│  └─ 3. research-semantic-scholar   → Fallback, BibTeX generation           │
│                                                                             │
│  arXiv:                                                                    │
│  └─ arxiv-advanced → ML/AI preprints, full-text download                   │
│                                                                             │
│  Other Sources:                                                            │
│  ├─ paper-search → Multi-platform aggregation                              │
│  ├─ perplexity   → Real-time web research                                  │
│  └─ linkup       → Web search and extraction                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RALPH QUICK REFERENCE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ENTRY COMMANDS:                                                            │
│  ├─ ralph-init <PROJECT_PATH>     → Initialize orchestration                │
│  └─ ralph-loop <PROJECT_PATH>     → Start/continue loop                     │
│                                                                             │
│  PROJECT STATUS VALUES:                                                     │
│  ├─ complete               → Deploy and verify                              │
│  ├─ functionality_problems → Route to Phase 14 (Fix)                        │
│  ├─ incomplete_research    → Route to Phase 13 → 11                         │
│  ├─ incomplete_critical    → Route to Phase 11 (Dev)                        │
│  └─ incomplete_minor       → Route to Phase 11 (Dev)                        │
│                                                                             │
│  ITERATION CYCLE:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ASSESS → ROUTE → EXECUTE → VERIFY → (loop or complete)              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMPLETION CRITERIA:                                                       │
│  [ ] No NotImplementedError in code                                         │
│  [ ] All imports work                                                       │
│  [ ] All tests pass                                                         │
│  [ ] No blocked Beads tasks                                                 │
│  [ ] No open Beads tasks                                                    │
│  [ ] No in_progress Beads tasks                                             │
│  [ ] FUNCTIONALITY_SCORE = 0                                                │
│                                                                             │
│  CONTEXT RECOVERY:                                                          │
│  1. mcp__serena__read_memory("orchestration-essentials.md")                 │
│  2. bd ready                                                                │
│  3. cat .ralph/progress.txt | tail -20                                      │
│                                                                             │
│  SIGNAL COMPLETION:                                                         │
│  <promise>COMPLETE</promise>                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Related Documentation

| Document | Purpose |
|----------|---------|
| `MASTER_PROMPT_TEMPLATE.md` | Full orchestration documentation |
| `PART1_INFRASTRUCTURE.md` | Phases 0-9 detailed |
| `PART2_EXECUTION.md` | Phases 10-15 detailed |
| `ORCHESTRATION_ESSENTIALS.md` | Quick reference guide |
| `prd.json` | Current project definition |
| `progress.txt` | Iteration log |
