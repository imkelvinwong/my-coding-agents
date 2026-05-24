# SDLC Agent Pipeline

A structured, multi-agent software development lifecycle (SDLC) pipeline that takes a feature request from architecture through testing, with formal handoffs, quality gates, and session recovery at every step.

---

## What This Is

This pipeline orchestrates seven specialized AI agents through the complete software delivery process. Each agent has a defined role, produces a specific artifact, and hands off to the next via the Orchestrator. No agent skips ahead, no artifact is optional, and no phase proceeds without the previous one passing a validation checkpoint.

The pipeline is designed for **continuous, compounding improvement**: every artifact produced becomes the contract that the next agent must satisfy, and every failure is classified by root cause and routed to the correct agent for resolution rather than patched by whoever happens to encounter it.

---

## Agents

| Agent | Role | Primary Output |
|---|---|---|
| **Orchestrator** | Pipeline controller and quality gate enforcer | Pipeline Execution Report |
| **Planner** | Software architect | Architecture document (11 sections) |
| **Designer** | Product designer | Design specification or Non-UI Waiver |
| **Developer** | Implementation engineer | Source code + Developer Completion Report |
| **Reviewer** | Code reviewer and quality gate | Review Decision (Approved / Rejected with defect report) |
| **Builder** | Build engineer and DevOps lead | Build + smoke test + Builder Completion Report |
| **Tester** | QA engineer and test lead | Test plan + test suite + Tester Completion Report |

---

## Supporting Files

| File | Purpose |
|---|---|
| `SKILL.md` | Path resolution and archive mechanics — consult this for all file path questions |
| `pipeline-analysis.prompt.md` | Runs a structured pros/cons analysis of every pipeline file |
| `session-recovery.prompt.md` | Reconstructs pipeline state after a session interruption and resumes from the correct point |

---

## Pipeline Flow

```
Feature Request
      │
      ▼
  Orchestrator  ──── Pre-Execution Planning
      │               • Identify entry point
      │               • Map agent sequence
      │               • Record effort gate thresholds
      ▼
   Planner  ──────── Produces ARCHITECTURE.md (11 sections)
      │               • Story point estimate
      │               • Orchestrator Scheduling Note
      ▼
 UI Scope Gate ────── Orchestrator decision
      │               • UI_REQUIRED → invoke Designer
      │               • UI_NOT_REQUIRED → Orchestrator writes Non-UI Waiver
      ▼
  Designer  ─────── Produces DESIGN.md (10 sections)
  (or Orchestrator   • Component specs, async states, accessibility
   writes waiver)    • Structured handoff notification
      ▼
  Developer  ─────── Produces source code + DEVELOPER_COMPLETION.md
      │               • Implements all architecture + design contracts
      │               • Self-review checklist
      ▼
  Reviewer  ──────── Produces REVIEW_CYCLE_N.md
      │               • Approved → proceed to Builder
      │               • Rejected → defect report → route by root cause
      ▼
   Builder  ──────── Produces BUILDER_COMPLETION.md
      │               • Compiles, starts dev server, runs smoke test
      │               • "Ready for Tester: Yes" required to proceed
      ▼
   Tester  ───────── Produces TEST_PLAN.md + TEST_COMPLETION.md
                      • Unit, integration, component, accessibility tests
                      • 100% pass rate + coverage thresholds required
```

---

## Artifact Locations

All artifacts are written to `md_docs/*/active/` and archived to `md_docs/*/archive/` at pipeline completion or halt. The `<FEATURE_NAME>` prefix (PascalCase) is shared across all artifacts for a feature, enabling session recovery to locate them as a set.

```
md_docs/
    planner/active/        <FEATURE_NAME>_ARCHITECTURE.md
    designer/active/       <FEATURE_NAME>_DESIGN.md
    developer/active/      <FEATURE_NAME>_DEVELOPER_COMPLETION.md
    reviewer/active/       <FEATURE_NAME>_REVIEW_CYCLE_N.md
    builder/active/        <FEATURE_NAME>_BUILDER_COMPLETION.md
    tester/active/         <FEATURE_NAME>_TEST_PLAN.md
                           <FEATURE_NAME>_TEST_COMPLETION.md
    orchestrator/active/   <FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md
```

---

## Key Design Decisions

### UI Scope Gate
After Planner completes, the Orchestrator classifies the feature as `UI_REQUIRED` or `UI_NOT_REQUIRED` based on architecture content. This gate determines whether the Designer runs or the Orchestrator writes a Non-UI Waiver. The Reviewer and Tester both check this document type before applying their checklists — it is a first-class pipeline concept, not an edge case.

### Defect Classification
Every defect found by the Reviewer is classified before it is routed:
- **Class A — Developer Error**: The specification was clear; the implementation does not follow it. Routes to Developer.
- **Class B — Specification Ambiguity**: The specification was unclear; the Developer's interpretation was reasonable. Routes to Planner or Designer.
- **Class C — Systemic Pattern**: The same defect class recurs across files or across review cycles. Routes to Orchestrator halt + user escalation.

Routing by root cause rather than by proximity prevents incorrect agents from attempting fixes outside their domain.

### Escalation Caps
Every agent that can self-remediate failures has a retry cap of 3 attempts per error. After 3 attempts, the error is escalated regardless of classification. This prevents infinite loops while giving agents enough attempts to resolve genuinely fixable problems.

### Effort-Aware Execution
The Planner's architecture document includes a machine-readable story point estimate. The Orchestrator reads this and applies:
- **≤ 5 points**: Single-run execution, no confirmation needed
- **6–13 points**: Pause after Planner + Designer for user milestone confirmation
- **≥ 14 points**: Full halt after Planner + Designer — explicit phased approval required before Developer runs

### Session Recovery
If a session is interrupted mid-pipeline, `session-recovery.prompt.md` reconstructs state by reading all `active/` artifacts, assessing completeness per agent-specific criteria, detecting stale contracts via timestamp comparison, and resuming from the earliest valid re-entry point. Recovery never restarts from the beginning if valid completed-phase artifacts exist.

---

## Feedback Loops

The pipeline includes structured feedback loops for every failure type:

| Failure | Detected By | Fixed By | Resume From |
|---|---|---|---|
| Compilation error (≤3 attempts) | Builder | Builder (self-remediate) | Builder |
| Persistent build error | Builder | Developer → Builder | Builder |
| Test assertion error | Tester | Tester (self-remediate) | Tester |
| Non-trivial implementation bug | Tester | Developer → Builder → Tester | Developer |
| Review defect — Class A | Reviewer | Developer → Reviewer | Developer |
| Review defect — Class B | Reviewer | Planner or Designer → Developer → Reviewer | Planner or Designer |
| Review defect — Class C | Reviewer | Halt — user intervention | N/A |
| Missing design spec (UI_REQUIRED) | Developer or Reviewer | Designer → Developer | Designer |
| Invalid Non-UI Waiver | Developer or Reviewer | Orchestrator → Developer | Developer |

---

## Running a Pipeline Analysis

To evaluate any or all pipeline files for weaknesses, improvements, and cross-pipeline gaps, invoke `pipeline-analysis.prompt.md` with the files you want analyzed. The prompt evaluates each file across four dimensions (Clarity, Completeness, Boundary Enforcement, Integration Fitness), produces per-file pros/cons reports, and outputs a Consolidated Action Summary with a Decision Matrix for batching improvements.

```
Files analyzed (in order):
1. orchestrator.agent.md
2. planner.agent.md
3. designer.agent.md
4. developer.agent.md
5. reviewer.agent.md
6. builder.agent.md
7. tester.agent.md
8. SKILL.md
9. pipeline-analysis.prompt.md
10. session-recovery.prompt.md
```

---

## Conventions

- **`<FEATURE_NAME>`**: PascalCase, no spaces, no special characters. Set by the Orchestrator at pipeline start. Never modified by any other agent. Example: `UserAuth`, `DarkModeToggle`, `CsvExport`.
- **Active vs. Archive**: `active/` = current contract. `archive/` = historical record. Agents read only from `active/`. Only the Orchestrator writes to `archive/`.
- **Section references**: Architecture document sections are numbered 1–11. Design document sections are numbered 1–10. Reviewer defect reports cite section numbers (e.g., "ARCHITECTURE.md Section 4") so defects are traceable without ambiguity.
- **Non-UI Waiver**: When no UI work exists, the Orchestrator writes a waiver to `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md` containing four required elements. The Reviewer and Tester check for the waiver header and apply the waiver-specific checklist path instead of the full design conformance checklist.

---

## File Reference

| File | Description |
|---|---|
| `orchestrator.agent.md` | Pipeline controller — planning, validation, routing, and archiving |
| `planner.agent.md` | Architecture specification producer — 11-section output |
| `designer.agent.md` | Design specification producer — 10-section output |
| `developer.agent.md` | Implementation producer — source code and Completion Report |
| `reviewer.agent.md` | Quality gate — inspection, defect classification, and Review Decision |
| `builder.agent.md` | Build and smoke test executor — Completion Report |
| `tester.agent.md` | Test suite producer and executor — test plan and Completion Report |
| `SKILL.md` | File path resolution and archive operation mechanics |
| `pipeline-analysis.prompt.md` | Structured pros/cons analysis of all pipeline files |
| `session-recovery.prompt.md` | State reconstruction and re-entry point determination after session interruption |
