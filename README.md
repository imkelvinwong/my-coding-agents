# SDLC Agent Pipeline

A structured, multi-agent software development lifecycle (SDLC) pipeline that takes a feature request from architecture through testing, with formal handoffs, quality gates, machine-readable outcome contracts, and session recovery at every step.

---

## What This Is

This pipeline orchestrates eight specialized AI agents through the complete software delivery process. Each agent has a defined role, produces a specific artifact, and hands off to the next via the Orchestrator. No agent skips ahead, no artifact is optional, and no phase proceeds without the previous one passing a validation checkpoint.

The pipeline is designed for **continuous, compounding improvement**: every artifact produced becomes the contract that the next agent must satisfy, and every failure is classified by root cause and routed to the correct agent for resolution rather than patched by whoever happens to encounter it.

---

## Agents

| Agent | Role | Primary Output |
| --- | --- | --- |
| **Orchestrator** | Pipeline controller and quality gate enforcer | Pipeline Execution Report |
| **Planner** | Software architect | Architecture document (11 sections) |
| **Designer** | Product designer | Design specification (10 sections + Section 8.1) or Non-UI Waiver |
| **Researcher** | ML verification firewall | Technical Verification Report + verified PyTorch/TensorFlow modules + Pydantic v2 I/O schemas |
| **Developer** | Implementation engineer | Source code + Developer Completion Report |
| **Reviewer** | Code reviewer and quality gate | Review Decision (`decision_code: APPROVED` / `REJECTED`) |
| **Builder** | Build engineer and DevOps lead | Build + smoke test + Builder Completion Report |
| **Tester** | QA engineer and test lead | Test plan + test suite + Tester Completion Report (`status_code: PASSING` / `BLOCKED`) + LLM Eval Results (`eval_status: PASS` / `FAIL`) |

---

## Supporting Files

| File | Purpose |
| --- | --- |
| `markdown-file-management/SKILL.md` | Path resolution, directory creation, archive mechanics, Fan-Out I/O isolation rules, and the canonical Non-UI Waiver Schema |
| `pipeline-analysis.prompt.md` | Runs a structured four-dimension pros/cons analysis of every pipeline file and produces a Decision Matrix |
| `session-recovery.prompt.md` | Reconstructs pipeline state after a session interruption and resumes from the correct re-entry point |

---

## Pipeline Flow

```
Feature Request
      │
      ▼
  Orchestrator  ──── Pre-Execution Planning
      │               • Scan md_docs/*/active/ for prior artifacts (3-branch logic)
      │               • Invoke session-recovery if prior artifacts exist
      │               • Map agent sequence and artifact dependencies
      │               • Record effort gate thresholds
      ▼
   Planner  ──────── Produces ARCHITECTURE.md (11 sections)
      │               • Fenced JSON scheduling payload in Section 10
      │               • Orchestrator parses payload programmatically
      ▼
 UI Scope Gate ────── Orchestrator decision (evidence-based, not name-based)
      │               • UI_REQUIRED  → invoke Designer
      │               • UI_NOT_REQUIRED → Orchestrator writes Non-UI Waiver
      ▼
  Designer  ─────── Produces DESIGN.md (10 sections + Section 8.1)
  (or Orchestrator   • Component specs, async states, streaming states, accessibility
   writes waiver)    • Structured handoff notification to Orchestrator
      ▼
  Developer  ─────── Produces source code + DEVELOPER_COMPLETION.md
      │               • Implements all architecture + design contracts
      │               • Logs specification gaps via 5-key protocol
      │               • Self-review checklist
      ▼
  Reviewer  ──────── Produces REVIEW_CYCLE_N.md
      │               • decision_code: APPROVED → proceed to Builder
      │               • decision_code: REJECTED → defect report → route by root cause
      ▼
   Builder  ──────── Produces BUILDER_COMPLETION.md
      │               • Compiles, starts dev server, runs smoke test
      │               • "Ready for Tester: Yes" required to proceed
      ▼
   Tester  ───────── Produces TEST_PLAN.md + TEST_COMPLETION.md + LLM_EVAL_RESULTS.md
      │               • Derives test plan first; counts total targets
      │               • < 12 targets  → sequential execution
      │               • ≥ 12 targets  → Parallel Fan-Out (Agent A + Agent B + Agent C)
      │               • Agent A: Spec-Independent tests (always)
      │               • Agent B: UI-Dependent tests (skipped if Non-UI Waiver)
      │               • Agent C: LLM Eval tests (skipped if no LLM inference components)
      │               • Heartbeat Monitor: 10-second window (Agents A & B); dynamic window (Agent C)
      │               • Orchestrator polls staging/ at Phase 1 and Phase 2 windows
      │               • 100% pass rate + coverage thresholds + eval_status: PASS required
      ▼
   status_code: PASSING + eval_status: PASS → Orchestrator archives all artifacts

```

---

## Artifact Locations

All artifacts are written to `md_docs/*/active/` and archived to `md_docs/*/archive/` at pipeline completion or halt. The `<FEATURE_NAME>` prefix (PascalCase) is shared across all artifacts for a feature, enabling session recovery to locate them as a set. Legacy root-level paths (`.architecture/` and `.design/`) are fully deprecated and treated as orphans.

```
md_docs/
    planner/active/        <FEATURE_NAME>_ARCHITECTURE.md
    designer/active/       <FEATURE_NAME>_DESIGN.md (Full spec or Non-UI Waiver)
    developer/active/      <FEATURE_NAME>_DEVELOPER_COMPLETION.md
    reviewer/active/       <FEATURE_NAME>_REVIEW_CYCLE_N.md (N increments per cycle)
    builder/active/        <FEATURE_NAME>_BUILDER_COMPLETION.md
    tester/active/         <FEATURE_NAME>_TEST_PLAN.md
                           <FEATURE_NAME>_TEST_COMPLETION.md
                           <FEATURE_NAME>_LLM_EVAL_RESULTS.md (if LLM components present)
    tester/staging/        (Ephemeral Fan-Out files: START/TESTS for Agents A, B & C)
    orchestrator/active/   <FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md

```

---

## Key Design Decisions

### Fenced-JSON Scheduling Payload & Effort-Aware Execution

The Planner emits a strictly structured JSON block inside Section 10 of the architecture document. The Orchestrator extracts and parses this block programmatically to enforce effort-aware operational modes based on `total_story_points`:

* **≤ 5 story points**: Single-run execution; runs the full pipeline without pausing.
* **6–13 story points**: Milestone confirmation required; pauses after Planner and Designer to present the phase breakdown.
* **≥ 14 story points**: Phased approval required; hard halts after Planner and Designer for explicit written user approval.

### Machine-Readable Outcome Contracts

Phase-completion routing decisions are driven strictly by dedicated machine-readable fields, completely ignoring surrounding human prose.

* **Reviewer Phase**: Evaluates code quality and outputs `decision_code` (`APPROVED` or `REJECTED`) within `REVIEW_CYCLE_N.md`.
* **Tester Phase**: Executes the test suites and outputs `status_code` (`PASSING` or `BLOCKED`) within `TEST_COMPLETION.md`.
* **LLM Eval Gate**: For features with LLM inference components, the Tester additionally outputs `eval_status` (`PASS` or `FAIL`) within `LLM_EVAL_RESULTS.md`. A `FAIL` result blocks pipeline completion regardless of `status_code` and routes back to Developer with the specific `eval_score` and `eval_threshold` values.

### UI Scope Gate

The Orchestrator classifies features as `UI_REQUIRED` or `UI_NOT_REQUIRED` based on architectural evidence. When `UI_NOT_REQUIRED`, the Orchestrator writes a formal 4-element Non-UI Waiver to the design track. This waiver triggers the Reviewer and Tester to automatically bypass UI-conformance paths and skips spawning Fan-Out Agent B.

### Streaming Data State Designs (Section 8.1)

The Designer specification now includes Section 8.1 — Streaming Data State Designs for any component that renders streaming data (token-by-token LLM output, server-sent events, or WebSocket-pushed content). Section 8.1 defines four streaming states: **Connecting**, **Streaming**, **Interrupted**, and **Complete**. Section 9 (Accessibility) is correspondingly extended with `aria-busy` state transitions, announcement batching strategies for high-frequency updates, and keyboard-accessible interruption affordances. Section 10 (Design System Consistency Checklist) includes three new checklist items enforcing completeness of Section 8.1 entries before handoff.

### Parallel Fan-Out (Tester Phase)

When a derived test plan contains 12 or more distinct test targets, the Tester activates Fan-Out mode instead of sequential execution. Up to three parallel agents run with strict filesystem-level I/O isolation:

* **Agent A** generates Spec-Independent tests (unit, integration, edge cases) — always spawned.
* **Agent B** generates UI-Dependent tests (component, interaction, accessibility) — skipped if the design document is a Non-UI Waiver.
* **Agent C** generates LLM Eval tests (prompt-response scoring, threshold-gated assertions, regression tests for known failure modes) — skipped if the feature contains no LLM inference components.

A Heartbeat Monitor safety protocol validates initialization of every spawned agent before output polling begins; any missing start signal triggers an immediate abort and fallback to sequential execution. Agent A and Agent B are evaluated against a fixed 10-second window. Agent C is evaluated against a dynamic window derived from `max_inference_latency_ms` in architecture Section 8 using the formula `max(10, ceil(max_inference_latency_ms / 1000) × 3)` — defaulting to 90 seconds if the field is absent. This prevents false aborts on inference-heavy evaluation suites where a single eval case may take 30–120 seconds.

### Defect Classification & Specification Gaps

Reviewer defects are classified by root cause to determine routing: **Class A** (Developer Error; routes back to Developer), **Class B** (Specification Ambiguity; routes back to Planner or Designer), or **Class C** (Systemic Pattern; triggers an immediate pipeline halt). To qualify for a Class B classification, Developers must preemptively log ambiguities in their completion report using a strict 5-key protocol (`gap_document`, `specification_gap_description`, `chosen_interpretation`, `rejected_alternatives`, and `risk_assessment`).

### Escalation Caps

Self-remediating agents (Builder and Tester) are strictly capped at 3 execution attempts per error classification. If remediation fails on the third attempt, the agent writes a detailed escalation report, notifies the Orchestrator, and the pipeline halts globally to prevent infinite loops.

### Session Recovery

If a session is interrupted mid-pipeline, `session-recovery.prompt.md` executes a 10-step state reconstruction workflow. It cleans up orphaned staging files, inventories complete contract artifacts via field-presence criteria (including `eval_status` in `LLM_EVAL_RESULTS.md` for LLM-bearing features), identifies stale upstream contracts via timestamp analysis, re-verifies effort gates, and resumes from the earliest valid re-entry point.

---

## Feedback Loops

The pipeline includes structured feedback loops that route failures strictly by root cause:

| Failure | Detected By | Fixed By | Resume From |
| --- | --- | --- | --- |
| Compilation error (≤ 3 attempts) | Builder | Builder (self-remediate) | Builder |
| Persistent build error (> 3 attempts) | Builder | Developer → Builder | Builder |
| Test assertion error (test-side) | Tester | Tester (self-remediate) | Tester |
| Trivial implementation bug (≤ 3 attempts) | Tester | Tester (targeted single-line fix) | Tester |
| Non-trivial implementation bug | Tester | Developer → Builder → Tester | Developer |
| LLM eval gate failure (`eval_status: FAIL`) | Orchestrator (after Tester) | Developer → Builder → Tester | Developer |
| Review defect — Class A | Reviewer | Developer → Reviewer (new cycle) | Developer |
| Review defect — Class B | Reviewer | Planner or Designer → Developer → Reviewer | Planner or Designer |
| Review defect — Class C | Reviewer | Halt — user intervention required | N/A |
| Missing design spec (`UI_REQUIRED`) | Developer/Reviewer | Designer → Developer | Designer |
| Design document exists but matches neither full-spec nor waiver format | Developer/Reviewer | Designer produces complete document → Developer | Designer |
| Invalid or malformed Non-UI Waiver | Developer/Reviewer | Orchestrator regenerates waiver → Developer | Developer |
| JSON scheduling payload absent/malformed | Orchestrator | Planner re-runs with specific parse error | Planner |
| Fan-Out heartbeat or staging timeout | Orchestrator | Abort Fan-Out; discard staging; fall back | Tester (sequential) |
| Orphaned staging files found at recovery | Session recovery | Archive staging files; set Tester override flag | Tester |

---

## Running a Pipeline Analysis

To evaluate pipeline files for weaknesses, improvements, and cross-pipeline gaps, invoke `pipeline-analysis.prompt.md` using a general AI agent. It reviews files across four quality dimensions and supports two operational modes:

* **Analysis (Default)**: Produces a comprehensive pros/cons report and a Consolidated Action Summary with a Decision Matrix.
* **Refinement**: Automatically applies all `Immediate`-priority fixes from the Decision Matrix directly to the files.

Files analyzed in pipeline execution order:

1. `orchestrator.agent.md`
2. `planner.agent.md`
3. `designer.agent.md`
4. `researcher.agent.md`
5. `developer.agent.md`
6. `reviewer.agent.md`
7. `builder.agent.md`
8. `tester.agent.md`
9. `markdown-file-management/SKILL.md`
10. `pipeline-analysis.prompt.md`
11. `session-recovery.prompt.md`

---

## Conventions

* **`<FEATURE_NAME>`**: PascalCase, no spaces, no special characters, broadcast globally by the Orchestrator and never altered.
* **Active vs. Archive**: All active contracts live in `active/`. Only the Orchestrator moves files to `archive/` upon completion or halt, appending a synchronized UTC timestamp (`<FEATURE_NAME>_<DOC_TYPE>_YYYYMMDDHHMMSS.md`).
* **Section references**: Architecture document sections are numbered 1–11; Design document sections are 1–10 plus Section 8.1 (Streaming Data State Designs). Reviewer defect reports cite these specific sections for exact traceability.
* **Outcome Fields**: `decision_code` (`APPROVED` / `REJECTED`) and `status_code` (`PASSING` / `BLOCKED`) must be uppercase and accompanied by their respective detail blocks to be validated as complete. For LLM-bearing features, `eval_status` (`PASS` / `FAIL`) within `LLM_EVAL_RESULTS.md` is an additional outcome gate evaluated before pipeline completion.
* **Non-UI Waiver**: Must satisfy all four elements of the Non-UI Waiver Schema defined in `SKILL.md`. The schema is the single authoritative source for required content — agents author, validate, and consume the waiver against that definition.
* **Specification Gap Protocol**: Gaps must be documented using exactly five snake_case keys (`gap_document`, `specification_gap_description`, `chosen_interpretation`, `rejected_alternatives`, and `risk_assessment`) or face rejection as a separate minor defect.

---

## File Reference

| File | Description |
| --- | --- |
| `orchestrator.agent.md` | Pipeline controller — planning, validation, routing, archiving, Fan-Out polling (Agents A, B, C), LLM eval gate |
| `planner.agent.md` | Architecture specification producer — 11-section output with fenced-JSON scheduling payload; Section 8 requires `max_inference_latency_ms` for any feature with ML framework dependencies |
| `designer.agent.md` | Design specification producer — 10-section output + Section 8.1 streaming states, or Non-UI Waiver |
| `researcher.agent.md` | ML verification firewall — tensor validation, mathematical proofs, PyTorch/TensorFlow core modules, Pydantic v2 I/O schemas, `Ready for Developer` gate |
| `developer.agent.md` | Implementation producer — source code, Completion Report, Specification Gap Protocol |
| `reviewer.agent.md` | Quality gate — inspection, defect classification (A/B/C), `decision_code` output |
| `builder.agent.md` | Build and smoke test executor — 3-attempt retry cap, escalation report, Completion Report |
| `tester.agent.md` | Test suite producer and executor — Fan-Out mode (Agents A, B, C), Heartbeat Monitor, `status_code` output, LLM eval gate (`eval_status`) |
| `markdown-file-management/SKILL.md` | Path resolution, archive mechanics, Fan-Out I/O isolation rules (six staging files), staging directory contract, and canonical Non-UI Waiver Schema |
| `pipeline-analysis.prompt.md` | Structured four-dimension pros/cons analysis with Decision Matrix and optional refinement mode |
| `session-recovery.prompt.md` | 10-step state reconstruction, staging cleanup, stale contract detection, `eval_status` completeness criterion, and re-entry determination |