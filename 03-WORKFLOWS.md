<div align="center">

[🏠 Home](README.md) • [📖 Overview](00-OVERVIEW.md) • **03 Workflows**

━━━━━━━━━●━━━━━━━━━━━━━━━━━━━━━ `3/8`

[← 02 Architecture](02-LAYER-ARCHITECTURE.md) • [04 Agents →](04-AGENTS.md)

</div>

---

# Workflows

> **Definition (Anthropic):** Systems where LLMs and tools are orchestrated through **predefined code paths**.
>
> — [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), December 2024

### Anthropic's Progression

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart LR
    classDef block fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#ffffff
    classDef workflow fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef agent fill:#f59e0b,stroke:#d97706,stroke-width:2px,color:#ffffff

    A["🧱 Augmented LLM<br/><i>Building block</i>"]:::block
    W["⚙️ Workflows<br/><i>Composed blocks</i>"]:::workflow
    AG["🤖 Agents<br/><i>Loops + feedback</i>"]:::agent

    A -->|"compose"| W
    W -->|"add autonomy"| AG
```

| Baseline | Workflows | Agents |
|----------|-----------|--------|
| [0. 🏎️ Direct Execution](#0-️-baseline-direct-execution) | [1. ⛓️ Prompt Chaining](#1-️-prompt-chaining) | [6. 🐉 Autonomous](04-AGENTS.md#1--autonomous-agents) |
| *(single augmented LLM call)* | [2. 🚦 Routing](#2--routing) | *(self-directed loops)* |
| | [3. 🛤️ Parallelization](#3-️-parallelization) | |
| | [4. 🦑 Orchestrator-Workers](#4--orchestrator-workers) | |
| | [5. 🩻 Evaluator-Optimizer](#5--evaluator-optimizer) | |

> **Key characteristic:** The **CODE** controls the flow, not the LLM

---

## Building Block: The Augmented LLM

> The basic building block of agentic systems is an LLM enhanced with **retrieval**, **tools**, and **memory**.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TB
    classDef retrieval fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#ffffff
    classDef tools fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef memory fill:#06b6d4,stroke:#0891b2,stroke-width:2px,color:#ffffff
    classDef llm fill:#f59e0b,stroke:#d97706,stroke-width:2px,color:#ffffff

    R[("Retrieval<br/>RAG, search")]:::retrieval
    T{{"Tools<br/>MCP, Bash"}}:::tools
    M[/"Memory<br/>Context"/]:::memory

    LLM(["LLM<br/>Generates • Selects • Decides"]):::llm

    R --> LLM
    T --> LLM
    M --> LLM
```

> **Key insight:** Focus on tailoring capabilities to your specific use case and ensuring they provide an easy, well-documented interface for the LLM.

All workflows below assume each LLM call has access to these augmented capabilities.

---

## 📑 Table of Contents

| # | Pattern | Description | Complexity |
|---|---------|-------------|:----------:|
| 0 | [🏎️ Baseline](#0--baseline-direct-execution) | Single augmented LLM call | None |
| 1 | [⛓️ Prompt Chaining](#1-️-prompt-chaining) | Sequential steps, output→input | Low |
| 2 | [🚦 Routing](#2--routing) | Classify then dispatch | Low |
| 3 | [🛤️ Parallelization](#3-️-parallelization) | Concurrent independent tasks | Medium |
| 4 | [🦑 Orchestrator-Workers](#4--orchestrator-workers) | Manager + specialized workers | High |
| 5 | [🩻 Evaluator-Optimizer](#5--evaluator-optimizer) | Generate → Evaluate → Improve | Medium |
| | [Variants](#workflow-variants) | Wizard, Parallel Tools, Clone | — |
| | [Mechanisms](#implementation-mechanisms) | Progressive Skills, Programmatic | — |

> **Note:** Anthropic lists 5 workflows. We include "Baseline" (Direct Execution) as pattern #0 to show the progression from simple to complex. It represents the foundational single LLM call before orchestration is added.

---

## Terminology

| Symbol | Term | Description |
|:------:|------|-------------|
| 🐔 | **Main Agent** | Claude Code orchestrator (the hen that coordinates) |
| 🐦 | **Subagent** | Delegated worker spawned via Task (the bird) |
| 🪺 | **Spawn (Task)** | Action to create 🐦 subagents (via Task built-in tool) |
| 📚 | **Skill** | Loaded knowledge that enhances 🐔 capabilities |
| 🚧 | **Gate** | Checkpoint that validates output before proceeding to next step |

### Hierarchy

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart LR
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef main fill:#f59e0b,stroke:#d97706,stroke-width:2px,color:#ffffff
    classDef sub fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#ffffff
    classDef blocked fill:#ef4444,stroke:#dc2626,stroke-width:2px,color:#ffffff

    U1["🙋‍♀️📥 User"]:::user
    MA["🐔 Main Agent"]:::main
    SA["🐦 Subagent"]:::sub
    U2["💁‍♀️📤 User"]:::user

    U1 -->|request| MA
    MA -->|"🪺 spawn"| SA
    SA -->|result| MA
    MA -->|response| U2

    SA x-.-x|"❌ cannot spawn"| SA2["🐦 Subagent"]:::blocked
```

> **Rule:** 🐦 Subagents CANNOT spawn other 🐦 subagents (flat hierarchy)

---

## Decision Tree

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TD
    classDef question fill:#f59e0b,stroke:#d97706,stroke-width:2px,color:#ffffff
    classDef workflow fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef simple fill:#64748b,stroke:#475569,stroke-width:2px,color:#ffffff

    START["🙋‍♀️📥 Task Received"] --> Q1{"Single step?"}:::question

    Q1 -->|Yes| P1["🏎️ Direct Execution"]:::simple
    Q1 -->|No| Q2{"Steps dependent?"}:::question

    Q2 -->|Yes, sequential| P2["⛓️ Prompt Chaining"]:::workflow
    Q2 -->|No, parallel| Q3{"Same or different tasks?"}:::question

    Q3 -->|Same task, different data| P4["🛤️ Parallelization"]:::workflow
    Q3 -->|Different tasks| P5["🦑 Orchestrator-Workers"]:::workflow

    Q2 -->|Need classification first| P3["🚦 Routing"]:::workflow

    START --> Q4{"Quality critical?"}:::question
    Q4 -->|Yes, needs iteration| P6["🩻 Evaluator-Optimizer"]:::workflow
```

---

## 0. 🏎️ Baseline (Direct Execution)

> **Definition:** Single augmented LLM call without orchestration — the foundation before adding workflow complexity. Not counted as a workflow by Anthropic, but included here to show the full progression.

### Diagram

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart LR
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff

    USER["🙋‍♀️📥 User Request"]:::user --> MA["🐔💭 Main Agent"]:::main
    MA -->|"🐔📤"| OUT["💁‍♀️📤 User Receives"]:::user
```

### When to use this workflow

- Simple, single-step tasks
- No need for specialization
- Quick operations (file read, simple edit, search)

### Examples where direct execution is useful

- "What's in the config.json file?"
- "Add a console.log statement to this function"
- "Search for usages of `useState`"

### When NOT to use

- Complex multi-step workflows
- Tasks requiring multiple specializations
- Large-scale operations

---

## 1. ⛓️ Prompt Chaining

> **Definition:** Decompose a task into a sequence of steps, where each LLM call processes the output of the previous one.

### Diagram

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart LR
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef gate fill:#f59e0b,stroke:#d97706,stroke-width:2px,color:#ffffff
    classDef exit fill:#ef4444,stroke:#dc2626,stroke-width:2px,color:#ffffff

    USER["🙋‍♀️📥"]:::user --> P1["🐔💭 Step 1"]:::main
    P1 -->|"🐔📤"| G1{"🚧 Gate"}:::gate
    G1 -->|Pass| P2["🐔💭 Step 2"]:::main
    G1 -.->|Fail| EXIT["❌ Exit"]:::exit
    P2 -->|"🐔📤"| G2{"🚧 Gate"}:::gate
    G2 -->|Pass| P3["🐔💭 Step 3"]:::main
    G2 -.->|Fail| EXIT
    P3 -->|"🐔📤"| OUT["💁‍♀️📤"]:::user
```

### 🚧 Gate

> **Definition:** A checkpoint between steps that validates the output before proceeding. If validation fails, the chain exits early instead of propagating errors downstream.

**Gates can check for:**
- Output format/structure validity
- Quality thresholds (confidence scores, completeness)
- Safety checks (content moderation, guardrails)
- Business rules (required fields, constraints)

### When to use this workflow

This workflow is ideal for situations where the task can be easily and cleanly decomposed into fixed subtasks. The main goal is to trade off latency for higher accuracy, by making each LLM call an easier task.

### Examples where prompt chaining is useful

| Use Case | Chain |
|----------|-------|
| Marketing | Generate copy → Translate to target language |
| Documents | Write outline → Validate criteria → Write document |
| Code generation | Plan → Implement → Review |
| Data transformation | Parse → Transform → Validate |

### Example Flow

```
Step 1: "Extract all function names from this code"
        → [list of functions]

Step 2: "For each function, identify parameters and return types"
        → [function signatures]

Step 3: "Generate documentation for each function"
        → [documented code]
```

### When NOT to use

- Steps can be done independently (use 🛤️ Parallelization)
- Simple single-step tasks (use 🏎️ Direct Execution)

### Variant: 🧙 Wizard Workflow

Multi-step process with explicit 🙆‍♀️ user confirmation at each phase using ❓ `AskUserQuestion`.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
stateDiagram-v2
    [*] --> Analysis: 🙋‍♀️📥 User Request

    Analysis --> Confirm1: Present findings
    Confirm1 --> Planning: 🙆‍♀️✅ User approves
    Confirm1 --> Analysis: 🙆‍♀️❓ User requests changes

    Planning --> Confirm2: Present plan
    Confirm2 --> Implementation: 🙆‍♀️✅ User approves
    Confirm2 --> Planning: 🙆‍♀️❓ User requests changes

    Implementation --> Confirm3: Show changes
    Confirm3 --> Verification: 🙆‍♀️✅ User approves
    Confirm3 --> Implementation: 🙆‍♀️❓ User requests changes

    Verification --> [*]: ✅ Complete
```

**Use 🧙 Wizard for:**
- Destructive operations (migrations, deletions)
- Complex refactoring
- Multi-stakeholder decisions

---

## 2. 🚦 Routing

> **Definition:** Classify an input and direct it to a specialized followup task. This allows separation of concerns and more specialized prompts.

### Diagram

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TB
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef subagent fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#ffffff
    classDef idle fill:#94a3b8,stroke:#64748b,stroke-width:2px,color:#ffffff

    INPUT["🙋‍♀️📥 User Request"]:::user --> ROUTER{"🐔🚦 Classify & Route"}:::main

    ROUTER -.->|"Type A"| HA["🐦💤 Handler A"]:::idle
    ROUTER -->|"🐔🪺 Type B"| HB["🐦⚡ Handler B"]:::subagent
    ROUTER -.->|"Type C"| HC["🐦💤 Handler C"]:::idle
    ROUTER -.->|"Unknown"| DEFAULT["🐔💤 Default"]:::idle

    HB -->|"🐦📤"| FINAL["💁‍♀️📤 User Receives"]:::user
```

### Key Insight

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🚦 ROUTING: Choose ONE branch                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Logic: if/else, switch/case                                               │
│  Question: "Where should I send this?"                                      │
│  Result: Single output from chosen handler                                  │
│                                                                             │
│  Analogy: Train switch → One train takes ONE track                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### When to use this workflow

Routing works well for complex tasks where there are distinct categories that are better handled separately, and where classification can be handled accurately.

### Examples where routing is useful

| Use Case | Routes |
|----------|--------|
| Customer support | Bug → Tech Team, Billing → Finance, General → FAQ |
| Code tasks | Bug fix → Debugger, New feature → Builder |
| Model routing | Easy → Claude Haiku 4.5, Hard → Claude Sonnet 4.5 |
| Content | Question → Q&A handler, Task → Executor |

### When NOT to use

- All inputs require same processing
- Classification is unreliable
- Categories overlap significantly

---

## 3. 🛤️ Parallelization

> **Definition:** Execute independent tasks simultaneously and aggregate outputs programmatically. Manifests in two key variations: **Sectioning** and **Voting**.

### Core Concept

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart LR
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef parallel fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#ffffff

    IN["🙋‍♀️📥"]:::user --> SPLIT["🐔🔀 Split"]:::main
    SPLIT -->|"🐔🪺"| A["🐦⚡"]:::parallel
    SPLIT -->|"🐔🪺"| B["🐦⚡"]:::parallel
    SPLIT -->|"🐔🪺"| C["🐦⚡"]:::parallel
    A -->|"🐦📤"| MERGE["🐔🌀 Merge"]:::main
    B -->|"🐦📤"| MERGE
    C -->|"🐦📤"| MERGE
    MERGE -->|"🐔📤"| OUT["💁‍♀️📤"]:::user
```

### Key Insight

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ⚠️  IMPORTANT: Parallelization vs Orchestrator-Workers                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  In Parallelization, all spawned subagents are IDENTICAL.                   │
│  Same prompt, same capabilities. They are INTERCHANGEABLE.                  │
│                                                                             │
│  🛤️ Parallelization:        🐦⚡ = 🐦⚡ = 🐦⚡   (clones)                      │
│  🦑 Orchestrator-Workers:  🐦🔒 ≠ 🐦⚡ ≠ 🐦🎨   (specialists)                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2 Types of Parallelization

#### Type 1: 🛤️ Sectioning (Split DATA)

Break a task into independent subtasks run in parallel, then combine ALL results.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart LR
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef parallel fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#ffffff

    S_IN["🙋‍♀️📥 100 files"]:::user --> S_SPLIT["🐔🛤️"]:::main
    S_SPLIT -->|"🐔🪺"| S1["🐦⚡ Files 1-50"]:::parallel
    S_SPLIT -->|"🐔🪺"| S2["🐦⚡ Files 51-100"]:::parallel
    S1 -->|"🐦📤"| S_MERGE["🐔🌀 Combine ALL"]:::main
    S2 -->|"🐦📤"| S_MERGE
    S_MERGE -->|"🐔📤"| S_OUT["💁‍♀️📤"]:::user
```

**Examples:**
- Guardrails: One instance processes queries, another screens for inappropriate content
- Evals: Each LLM call evaluates a different aspect of model performance

#### Type 2: 🗳️ Voting (Same TASK, pick BEST)

Run the same task multiple times to get diverse outputs, then select the best.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart LR
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef parallel fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#ffffff
    classDef wizard fill:#14b8a6,stroke:#0d9488,stroke-width:2px,color:#ffffff
    classDef success fill:#10b981,stroke:#059669,stroke-width:2px,color:#ffffff

    V_IN["🙋‍♀️📥 Write headline"]:::user --> V_COPY["🐔🔀 3 attempts"]:::main
    V_COPY -->|"🐔🪺"| V1["🐦⚡ Version A"]:::parallel
    V_COPY -->|"🐔🪺"| V2["🐦⚡ Version B"]:::parallel
    V_COPY -->|"🐔🪺"| V3["🐦⚡ Version C"]:::parallel
    V1 -->|"🐦📤"| VOTE{"🐔🗳️ Compare"}:::wizard
    V2 -->|"🐦📤"| VOTE
    V3 -->|"🐦📤"| VOTE
    VOTE -->|"🐔✅ B wins"| BEST["🏆 Best"]:::success
```

**Examples:**
- Code vulnerability review with multiple prompts
- Content moderation with different vote thresholds

### Summary

| Type | Workers | Input | Output |
|------|---------|-------|--------|
| **🛤️ Sectioning** | IDENTICAL | Different DATA chunks | Combine ALL |
| **🗳️ Voting** | IDENTICAL | Same DATA | Pick ONE best |

### When to use this workflow

Parallelization is effective when the divided subtasks can be parallelized for speed, or when multiple perspectives are needed for higher confidence results.

### When NOT to use

- Tasks depend on each other's output
- Sequential order matters
- Limited resources

### Variant: 🚂 Parallel Tool Calling

Execute multiple independent 🔧 tool calls in a single message for efficiency.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TB
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef parallel fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#ffffff
    classDef state fill:#10b981,stroke:#059669,stroke-width:2px,color:#ffffff

    MA["🐔 Main Agent"]:::main -->|Single Message| TOOLS

    subgraph TOOLS["🚂 Parallel Tool Calls"]
        T1["🔧 Read file1.ts"]
        T2["🔧 Read file2.ts"]
        T3["🔧 Read file3.ts"]
    end

    T1 --> RESULTS["✅ All Results"]:::state
    T2 --> RESULTS
    T3 --> RESULTS

    RESULTS --> MA

    style TOOLS fill:#dbeafe,stroke:#3b82f6,stroke-width:2px
```

### Variant: 🧬 Master-Clone

Spawn multiple isolated 🐦 instances handling independent domains with no shared state.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TB
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef subagent fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#ffffff
    classDef state fill:#10b981,stroke:#059669,stroke-width:2px,color:#ffffff

    MA["🐔 Main Agent"]:::main

    MA -->|"Context: fr-FR"| C1["🐦 Clone: fr-FR"]:::subagent
    MA -->|"Context: es-ES"| C2["🐦 Clone: es-ES"]:::subagent
    MA -->|"Context: de-DE"| C3["🐦 Clone: de-DE"]:::subagent

    C1 -->|9 files| R1[Result: fr-FR]
    C2 -->|9 files| R2[Result: es-ES]
    C3 -->|9 files| R3[Result: de-DE]

    R1 --> MERGE["✅ Merge Results"]:::state
    R2 --> MERGE
    R3 --> MERGE

    MERGE --> MA
```

---

## 4. 🦑 Orchestrator-Workers

> **Definition:** A central LLM dynamically breaks down tasks, delegates them to worker LLMs, and synthesizes their results.

### Diagram

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TB
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef subagent fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#ffffff

    INPUT["🙋‍♀️📥 Review this PR"]:::user --> ORCH["🐔🦑 Orchestrator"]:::main

    ORCH -->|"🐔🪺 Check vulns"| W1["🐦🔒 Security Expert"]:::subagent
    ORCH -->|"🐔🪺 Check perf"| W2["🐦⚡ Performance Expert"]:::subagent
    ORCH -->|"🐔🪺 Check style"| W3["🐦🎨 Style Expert"]:::subagent

    W1 -->|"🐦📤 2 SQLi found"| SYNTH["🐔🌀 Synthesize"]:::main
    W2 -->|"🐦📤 O(n²) loop"| SYNTH
    W3 -->|"🐦📤 3 violations"| SYNTH

    SYNTH -->|"🐔📤"| OUTPUT["💁‍♀️📤 Final Report"]:::user
```

### Key Insight

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🦑 ORCHESTRATOR-WORKERS: Different specialists                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Each 🐦 subagent has a DIFFERENT expertise and does a DIFFERENT task.     │
│                                                                             │
│  Key difference from 🛤️ Parallelization: subtasks aren't pre-defined,      │
│  but determined by the orchestrator based on the specific input.            │
│                                                                             │
│  Analogy: Hospital team → Different experts collaborate                     │
│           (Chef + Pastry + Sommelier, not 3 cooks making same recipe)      │
│                                                                             │
│  Compare:                                                                   │
│  - 🛤️ Parallelization: Same worker + Different data (assembly line)        │
│  - 🦑 Orchestration: Different workers + Same data (expert team)           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### When to use this workflow

This workflow is well-suited for complex tasks where you can't predict the subtasks needed. The key difference from parallelization is its flexibility—subtasks aren't pre-defined, but determined by the orchestrator based on the specific input.

### Examples where orchestrator-workers is useful

| Use Case | Orchestration |
|----------|---------------|
| Coding products | Make complex changes to multiple files dynamically |
| Search tasks | Gather and analyze from multiple sources |
| PR Review | Security + Performance + Style experts |

### Main Agent Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **Decomposition** | Break complex task into subtasks |
| **Assignment** | Route subtasks to appropriate 🐦 subagents |
| **Monitoring** | Track 🐦 subagent progress |
| **Synthesis** | Combine results into coherent output |

### 🐦 Subagent Definition

```markdown
# .claude/agents/code-reviewer.md

---
name: code-reviewer
description: Reviews code for quality, security, and best practices
tools: Read, Grep, Glob
---

You are a senior code reviewer specializing in security and quality.

## Your Task
Review the provided code and report:
1. Security vulnerabilities
2. Performance issues
3. Code quality concerns
4. Suggested improvements

## Output Format
- ❌ CRITICAL: Issues requiring immediate attention
- ⚠️ WARNING: Should be addressed
- ℹ️ INFO: Suggestions for improvement
```

### Critical Rules

| Rule | Explanation |
|------|-------------|
| **No nested subagents** | 🐦 Subagents cannot spawn other 🐦 subagents |
| **Isolated context** | Each 🐦 subagent starts fresh, no shared memory |
| **Report to orchestrator** | Results flow back to 🐔 Main Agent only |

### When NOT to use

- Simple tasks not worth decomposition overhead
- Workers need heavy inter-communication

---

## 5. 🩻 Evaluator-Optimizer

> **Definition:** One LLM call generates a response while another provides evaluation and feedback in a loop until quality threshold is met.

### Diagram

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TB
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef data fill:#06b6d4,stroke:#0891b2,stroke-width:2px,color:#ffffff
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef wizard fill:#14b8a6,stroke:#0d9488,stroke-width:2px,color:#ffffff
    classDef success fill:#10b981,stroke:#059669,stroke-width:2px,color:#ffffff
    classDef error fill:#ef4444,stroke:#dc2626,stroke-width:2px,color:#ffffff

    INPUT["🙋‍♀️📥 Task"]:::user --> GEN["🐔💭 Generate"]:::main
    GEN --> CAND["🐔📤 Candidate"]:::data
    CAND --> EVAL{"🐔🩻 Evaluate"}:::wizard

    EVAL -->|"🐔✅ Pass"| OUTPUT["💁‍♀️📤 Output"]:::success
    EVAL -->|"🐔❌ Fail"| FEEDBACK["🐔🔄 Feedback"]:::error
    FEEDBACK --> GEN
```

### Detailed Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
sequenceDiagram
    participant U as 🙋‍♀️ User
    participant G as 🐔💭 Generator
    participant E as 🐔🩻 Evaluator

    U->>G: 🙋‍♀️📥 Request
    loop 🔄 Until quality threshold
        G->>G: 🐔💭 Generate candidate
        G->>E: 🐔📤 Submit for evaluation
        E->>E: 🐔👀 Score candidate
        alt ✅ Score >= threshold
            E->>U: 💁‍♀️📤 Accept
        else ❌ Score < threshold
            E->>G: 🐔🔄 Feedback for improvement
        end
    end
```

### When to use this workflow

This workflow is particularly effective when we have clear evaluation criteria, and when iterative refinement provides measurable value. Two signs of good fit:
1. LLM responses can be demonstrably improved when feedback is articulated
2. The LLM can provide such feedback

### Examples where evaluator-optimizer is useful

| Domain | Criteria | Use Case |
|--------|----------|----------|
| **Code** | Tests pass, lint clean, no security issues | Code generation |
| **Text** | Clarity score, factual accuracy, tone match | Literary translation |
| **Search** | Comprehensiveness, relevance | Complex research tasks |

### Example: Code Generation

```
Generator: Write function to parse CSV

Attempt 1: Basic implementation
Evaluator: "Missing error handling for malformed input"

Attempt 2: Added try/catch
Evaluator: "Not handling empty files"

Attempt 3: Complete implementation
Evaluator: "Pass - all criteria met"
```

### Advanced: Self-Correction Chains

You can chain prompts to have Claude **review its own work**. This catches errors and refines outputs, especially for high-stakes tasks.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
sequenceDiagram
    participant U as 🙋‍♀️ User
    participant G as 🐔💭 Generator
    participant R as 🐔🔍 Reviewer

    U->>G: 🙋‍♀️📥 "Summarize this research paper"
    G->>G: 🐔💭 Generate summary
    G->>R: 🐔📤 Submit for self-review
    R->>R: 🐔🔍 Check accuracy, clarity, completeness
    alt ✅ Quality OK
        R->>U: 💁‍♀️📤 Final summary
    else ❌ Issues found
        R->>G: 🐔🔄 "Missing methodology details"
        G->>G: 🐔💭 Regenerate with feedback
        G->>R: 🐔📤 Submit improved version
    end
```

**Use Self-Correction for:**
- Research summaries requiring accuracy
- Code that must meet strict criteria
- Content requiring specific style/tone

### When NOT to use

- First attempt is usually good enough
- No clear quality metrics
- Time constraints prevent iteration

---

## Workflow Variants

| Variant | Parent | Emoji | Description |
|---------|--------|-------|-------------|
| **Wizard Workflow** | ⛓️ Prompt Chaining | 🧙 | Human checkpoints via AskUserQuestion |
| **Parallel Tool Calling** | 🛤️ Parallelization | 🚂 | Multiple tools in single response |
| **Master-Clone** | 🛤️ Parallelization | 🧬 | Same agent, parallel instances |

---

## Implementation Mechanisms

These are **implementation mechanisms** in Claude Code, not workflows themselves.

### 📚 Progressive Skills

Load 📚 skills on-demand to enhance 🐔 agent capabilities for specific task types.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TB
    classDef main fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef skill fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#ffffff
    classDef decision fill:#f59e0b,stroke:#d97706,stroke-width:2px,color:#ffffff

    REQ["🙋‍♀️📥 User Request"] --> MA["🐔 Main Agent"]:::main
    MA --> CHECK{"📚 Match Skills?"}:::decision

    CHECK -->|TDD Task| TDD["📚 test-driven-development"]:::skill
    CHECK -->|Debug Task| DEBUG["📚 systematic-debugging"]:::skill
    CHECK -->|Review Task| REVIEW["📚 code-review"]:::skill
    CHECK -->|None| DIRECT[Direct Execution]

    TDD --> EXEC["✅ Enhanced Execution"]
    DEBUG --> EXEC
    REVIEW --> EXEC
    DIRECT --> EXEC
```

**Purpose:** Route execution through specialized methodologies (implements 🚦 Routing pattern).

### 🎛️ Programmatic Orchestration

External code controls 🐔 agent invocation and workflow logic rather than pure prompt-based control.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#64748b'}}}%%
flowchart TB
    classDef user fill:#6366f1,stroke:#4f46e5,stroke-width:2px,color:#ffffff
    classDef subagent fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#ffffff

    CODE["🎛️ External Code"]:::user --> LOOP{For each item}

    LOOP --> INVOKE["🐔 Invoke Claude"]:::subagent
    INVOKE --> RESULT[Get Result]
    RESULT --> PROCESS["🎛️ Process in Code"]
    PROCESS --> LOOP

    LOOP -->|Done| FINAL["✅ Final Output"]
```

**Purpose:** ⛓️ Prompt Chaining with external control (CI/CD, batch processing, API automation).

**Implementation:**
```python
# 🎛️ External Python script orchestrating Claude
import anthropic

client = anthropic.Anthropic()

locales = ["fr-FR", "es-ES", "de-DE"]
results = []

for locale in locales:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        messages=[{"role": "user", "content": f"Generate for {locale}"}]
    )
    results.append({"locale": locale, "content": response.content})
    save_progress(results)  # 🎛️ Code-controlled checkpointing
```

---

## Workflow Summary

```
┌──────────────────────────┬─────────────┬─────────────┬──────────────┬───────────┐
│ Pattern                  │ Complexity  │ Parallelism │ Human-Loop   │ Iteration │
├──────────────────────────┼─────────────┼─────────────┼──────────────┼───────────┤
│ 0. 🏎️ Baseline           │ None        │ None        │ None         │ None      │
├──────────────────────────┼─────────────┼─────────────┼──────────────┼───────────┤
│ 1. ⛓️ Prompt Chaining     │ Low         │ None        │ Optional     │ Linear    │
│ 2. 🚦 Routing             │ Low         │ None        │ None         │ None      │
│ 3. 🛤️ Parallelization     │ Medium      │ High        │ Optional     │ None      │
│ 4. 🦑 Orchestrator-Workers│ High        │ High        │ Optional     │ As needed │
│ 5. 🩻 Evaluator-Optimizer │ Medium      │ Optional    │ Optional     │ Loop      │
└──────────────────────────┴─────────────┴─────────────┴──────────────┴───────────┘
```

---

## Best Practices

### Permission Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `default` | Asks permission for each tool call | Read-only operations, validation |
| `acceptEdits` | Auto-approves Write/Edit operations | Generation after 🧙 user confirmation |
| `bypassPermissions` | All tools auto-approved | Trusted autonomous workflows |

### Parallelization Limits

| Type | Recommended Max | Risk if Exceeded |
|------|-----------------|------------------|
| 🐦 Concurrent Subagents | **10-15** | Context overflow, memory pressure |
| 🔌 MCP calls per subagent | **5** | Rate limit errors |
| 🪺 Task calls per message | **10** | API limits, timeouts |

---

<div align="center">

**━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━**

[← 02 Architecture](02-LAYER-ARCHITECTURE.md) • [🏠 Home](README.md) • [04 Agents →](04-AGENTS.md)

</div>
