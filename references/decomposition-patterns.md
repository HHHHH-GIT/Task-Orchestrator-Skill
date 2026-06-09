# Decomposition Patterns Reference

This file contains detailed patterns for common task types. Read this when you
need guidance on how to decompose a specific kind of task.

## Table of Contents

1. [Decision Tree](#decision-tree)
2. [Code Patterns](#code-patterns)
3. [Research Patterns](#research-patterns)
4. [Data Patterns](#data-patterns)
5. [Testing Patterns](#testing-patterns)
6. [Anti-Patterns](#anti-patterns)

---

## Decision Tree

Use this to decide your decomposition strategy:

```
Is the task about code changes?
├── Yes
│   ├── Single file? → Don't decompose, do it directly
│   ├── Multiple files, same concern? → Decompose by file
│   ├── Multiple concerns? → Decompose by concern, then by file
│   └── Cross-cutting (refactor, migration)? → Decompose by phase
│       (audit → plan → implement per-module → verify)
│
├── No, it's research/knowledge work
│   ├── Single source? → Don't decompose
│   ├── Multiple sources? → One subagent per source, then synthesize
│   ├── Multiple questions? → One subagent per question
│   └── Unknown scope? → Explore first, then decompose
│
├── No, it's data processing
│   ├── Can records be processed independently? → Fan-out by batch
│   ├── Pipeline of transformations? → Decompose by stage
│   └── Single-pass aggregation? → Don't decompose
│
└── No, it's something else
    ├── Can you list 3+ independent deliverables? → Decompose
    └── Everything is sequential? → Don't decompose
```

## Code Patterns

### Multi-File Refactor

When renaming, restructuring, or changing a pattern across many files:

```
Round 1: [Explore] Inventory all files affected, categorize by complexity
Round 2-N: [Refactor] One subagent per file (or per small group of files)
Round N+1: [Verify] Run tests, check for missed references
Round N+2: [Cleanup] Remove any temporary scaffolding
```

**Key insight**: Each file refactor is independent. The explore phase produces
a manifest that each subagent uses to know its scope.

### New Feature with API + UI + DB

```
Round 1: [Design] Define the API contract and data model
Round 2:
  - [API] Implement endpoints (depends on: Design)
  - [DB] Implement migrations/models (depends on: Design)
  - [UI] Implement components (depends on: Design, can use mock API)
Round 3:
  - [Integration] Wire API ↔ DB (depends on: API, DB)
  - [UI-API] Connect UI to real API (depends on: API, UI)
Round 4: [E2E] End-to-end tests (depends on: Integration, UI-API)
Round 5: [Docs] API docs + user guide (depends on: Design)
```

**Key insight**: Design is the bottleneck. Invest extra care there because
everything else depends on it.

### Bug Fix Across Multiple Components

```
Round 1: [Reproduce] Write a failing test that demonstrates the bug
Round 2: [Investigate] Parallel hypothesis testing:
  - [Hypothesis-A] Check if the bug is in component X
  - [Hypothesis-B] Check if the bug is in component Y
  - [Hypothesis-C] Check if it's a configuration issue
Round 3: [Fix] Implement the fix based on confirmed hypothesis
Round 4: [Verify] Run the failing test, add regression tests
```

**Key insight**: Speculative parallel investigation is faster than sequential
trial-and-error when the root cause is uncertain.

## Research Patterns

### Multi-Topic Research

When the user asks about multiple related topics:

```
Round 1-N: One subagent per topic
  - Each subagent searches, reads, and produces a focused report
Round N+1: [Synthesize] One subagent reads all reports and produces
  a unified analysis with cross-references and recommendations
```

**Prompt for synthesis subagent**: Include all prior reports in the context so
it can find connections, contradictions, and gaps.

### Competitive Analysis

```
Round 1-N: One subagent per competitor/product/technology
  - Each produces a structured comparison template
Round N+1: [Compare] Merge into a comparison matrix
Round N+2: [Recommend] Analyze the matrix and make recommendations
```

## Data Patterns

### Batch Processing

When processing a large dataset where each record is independent:

```
Round 1: [Chunk] Split data into N batches (or identify natural groupings)
Round 2-N: One subagent per batch
  - Each processes its batch and writes results to a separate file
Round N+1: [Merge] Combine all batch results
Round N+2: [Validate] Check for consistency across batches
```

**Subagent sizing**: Aim for batches that complete in roughly equal time.
If some records are much more complex, weight the batches accordingly.

### ETL Pipeline

```
Stage 1: [Extract] One subagent per data source
Stage 2: [Transform] Can often be parallelized per-record or per-field
Stage 3: [Load] Depends on all transforms completing → single subagent
```

## Testing Patterns

### Test Suite Generation

```
Round 1: [Explore] Inventory all public APIs/functions to test
Round 2-N: One subagent per module/component
  - Each generates unit tests for its assigned module
Round N+1: [Integration] Generate integration tests that span modules
Round N+2: [Review] Check test coverage, identify gaps
```

### Test-Driven Development

```
Round 1: [Write Tests] Define all test cases first (can be parallelized
  by test category: unit, integration, e2e)
Round 2: [Implement] Write code to pass the tests
  - Can be parallelized if different components are independent
Round 3: [Verify] Run all tests, fix failures
```

## Anti-Patterns

These are common mistakes in task decomposition. Avoid them:

### Over-Decomposition
**Symptom**: 20+ subtasks, most taking less than a minute
**Problem**: Overhead of spawning subagents exceeds the parallelism benefit
**Fix**: Group related micro-tasks into coherent units

### False Dependencies
**Symptom**: Everything is in Round 1 or everything is sequential
**Problem**: You've declared dependencies that don't actually exist
**Fix**: For each dependency, ask "does subtask B literally need subtask A's
output to start, or could it work with a reasonable assumption?"

### Vague Subtasks
**Symptom**: Subagent comes back asking "what exactly should I do?"
**Problem**: Subtask description is ambiguous
**Fix**: Each subtask should have: a clear title, specific input, specific
output format, and a concrete success criterion

### Ignored Synthesis
**Symptom**: User sees disjointed outputs that don't form a coherent whole
**Problem**: You skipped or rushed the synthesis phase
**Fix**: Always have a synthesis step that reads ALL subtask outputs and
produces a unified result. The synthesis subagent should have context from
every other subtask.

### Premature Parallelism
**Symptom**: Subagents produce conflicting results that need major rework
**Problem**: You parallelized before the design was solid enough
**Fix**: Ensure the "design" or "plan" phase is complete and approved before
dispatching implementation subagents. The design phase is the synchronization
point.

---

## Quick Reference: Subtask Sizing

| Task Complexity | Recommended Subtasks | Parallelism |
|----------------|---------------------|-------------|
| Simple (1-2 files) | 1-2 | Low |
| Medium (3-10 files) | 3-6 | Medium |
| Large (10+ files) | 5-10 | High |
| Massive (migration) | 10+ with recursion | Very High |

When in doubt, start with fewer, larger subtasks. You can always decompose
further in the next iteration if the results are too coarse.
