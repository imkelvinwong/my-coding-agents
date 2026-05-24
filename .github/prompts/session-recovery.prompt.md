---
name: session-recovery
description: Extends the Orchestrator agent with a structured recovery workflow. When the Orchestrator detects a partially-complete pipeline for a named feature, it follows this prompt's 9-step procedure to reconstruct state from md_docs artifacts, identify the correct re-entry point, archive stale files, and resume — or safely halt — without restarting from the beginning.
argument-hint: Recover pipeline state for <FEATURE_NAME> — e.g. "resume UserAuth" or "continue the DarkModeToggle pipeline"
agent: agent
---

# Session Recovery Prompt

## When to Use This Prompt

This prompt is used **by the Orchestrator agent** — not as a standalone analysis tool, but as an extension of the Orchestrator's Pre-Execution Planning phase. It activates automatically whenever the Orchestrator detects that a prior pipeline run exists for the requested feature.

**You do not invoke this prompt separately.** Instead, start a conversation with the Orchestrator agent and describe which feature to resume. The Orchestrator checks for existing artifacts as part of its standard Pre-Execution Planning, finds them, and follows this prompt's 9-step recovery workflow before continuing the pipeline.

Example invocations directed at the Orchestrator:
- `"Resume the UserAuth pipeline"`
- `"Continue where we left off on DarkModeToggle"`
- `"The session restarted — pick up the CsvExport feature"`
- Simply starting a new session with `"Build the UserAuth feature"` when artifacts already exist in `md_docs/*/active/` will trigger recovery automatically.

**Why the Orchestrator and not a general agent:** Session recovery is not analysis — it is execution. The recovery workflow produces archive operations, writes to the Pipeline Execution Report, and resumes live agent invocations. These are all Orchestrator responsibilities. A general agent reading the artifacts could produce a Recovery Report, but it could not actually resume the pipeline because it does not have the `agent` invocation tool or the pipeline's execution authority.

**No `## Instructions` section in this prompt:** Unlike `pipeline-analysis.prompt.md`, which establishes a fresh analyst persona via its `## Instructions` section, this prompt extends the Orchestrator's existing persona. The Orchestrator already knows its role, tools, and pipeline context. This prompt provides a structured procedure — a 9-step workflow — that slots into the Orchestrator's Pre-Execution Planning step. Adding a persona-establishing `## Instructions` section would create a competing identity that conflicts with the Orchestrator's system prompt. The Orchestrator reads this prompt as a procedure to execute, not as a new role to adopt.

---

## Purpose

This prompt is invoked when the Orchestrator detects that a pipeline for a known feature is already partially complete — either because the session restarted mid-execution or because the Orchestrator was invoked on a feature that already has artifacts present in `md_docs/*/active/`.

The Orchestrator must never restart a pipeline from the beginning when recoverable state exists. Restarting discards completed work, forces agents to re-run phases whose outputs are still valid, and loses the context established by prior agent decisions. This prompt defines the exact steps to reconstruct state, assess completeness, determine staleness, and resume from the correct re-entry point.

---

## Trigger Conditions

The Orchestrator enters this recovery workflow automatically — before constructing any new Pipeline Execution Plan — if any of the following conditions are true:

- **Any file matching `md_docs/*/active/<FEATURE_NAME>_*.md` exists** for the feature being requested, where `<FEATURE_NAME>` is the canonical feature name communicated by the Orchestrator at invocation. The existence of any active artifact means a prior run started and may still have valid completed phases.
- **The user's request references a feature that was previously started but not completed.** Language like "continue", "resume", "pick up where we left off", or "it stopped mid-way" is an explicit signal to invoke recovery rather than start fresh.
- **`md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` exists** but the Orchestrator has no in-context Pipeline Execution Plan. This means the current session lost its execution context and must reconstruct it from the persisted report.
- **The session was explicitly restarted** and the user has indicated that work was in progress on a named feature.

If none of these conditions are true, this is a new pipeline run. Exit this prompt and proceed with standard Pre-Execution Planning.

---

## Recovery Workflow

### Step 1 — Feature Name Resolution

Derive the canonical `<FEATURE_NAME>` value from the user's request using the same normalization rule applied at pipeline start: convert the feature description to PascalCase with no spaces and no special characters. Examples: "user authentication" → `UserAuth`, "dark mode toggle" → `DarkModeToggle`, "CSV export" → `CsvExport`.

Search `md_docs/*/active/` for all files matching the pattern `<FEATURE_NAME>_*.md`.

**If no matching files are found:** This is a new pipeline run. Exit this prompt and proceed with standard Pre-Execution Planning.

**If matching files are found:** Continue to Step 2 with the collected file list.

---

### Step 2 — Artifact Inventory

Read every file found in Step 1. For each file, record its path, the agent that produced it, its file type, and whether it is complete. Do not rely on file size or modification timestamp alone to determine completeness — read the file content and verify the required fields are present.

**Per-agent completeness criteria — a file is complete only if all listed fields are present and non-empty:**

| Agent | File | Required Fields for Completeness |
|---|---|---|
| Planner | `<FEATURE_NAME>_ARCHITECTURE.md` | All 11 section headers present; Section 10 contains the Orchestrator Scheduling Note in the format `Total: X story points — [recommendation]` |
| Designer | `<FEATURE_NAME>_DESIGN.md` | Either: all 10 section headers present (full specification), OR: the document header contains "No UI/UX design specification is required for this feature" and all four waiver elements are present (architecture reference, `UI_NOT_REQUIRED` classification, evidence statement, explicit waiver statement) |
| Developer | `<FEATURE_NAME>_DEVELOPER_COMPLETION.md` | "Files Created", "Files Modified", "Self-Review Checklist", and "Ready for Reviewer" fields all present; "Ready for Reviewer" reads "Yes" |
| Reviewer | `<FEATURE_NAME>_REVIEW_CYCLE_N.md` (highest N) | "Decision" field present and reads either "Approved — implementation cleared for Builder" or "Rejected" with a Defect Report section |
| Builder | `<FEATURE_NAME>_BUILDER_COMPLETION.md` | "Build Status", "Dev Server" (Status and Port fields), "Smoke Test Results", and "Ready for Tester" fields all present; "Ready for Tester" reads "Yes" or "No" with a reason |
| Tester | `<FEATURE_NAME>_TEST_PLAN.md` | "Unit Tests", "Integration Tests", and "Edge Cases and Boundary Conditions" sections all present |
| Tester | `<FEATURE_NAME>_TEST_COMPLETION.md` | "Test Results" (Total, Passed, Failed, Pass Rate), "Coverage" (all four metrics), and "Pipeline Status" field present; "Pipeline Status" reads "All tests passing — ready for completion" or "Blocked — [reason]" |
| Orchestrator | `<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` | "Phases Executed", "Phase Outcomes", "UI Scope Gate" (Classification and Evidence), and "Next Steps" fields all present |

A file that exists but is missing any required field is classified as **incomplete**. A file that contains all required fields with non-empty values is classified as **complete**. A design document that is a valid Non-UI Waiver is classified as **complete (waiver)**.

Produce an internal Artifact Inventory:

```
ARTIFACT INVENTORY
==================

Feature      : [feature name]
Scan Path    : md_docs/*/active/<FEATURE_NAME>_*.md

Files Found
-----------
[file path] — [agent] — [file type] — [status: complete / incomplete / complete (waiver)]
[missing required fields if incomplete: list each missing field]

Files Not Found (expected but absent)
--------------------------------------
[file path] — [agent] — [file type]
```

---

### Step 3 — Phase Completion Assessment

Using the Artifact Inventory, determine the last fully completed pipeline phase. Apply the following rules in order — the last phase whose completion condition is fully satisfied is the last completed phase:

| Phase | Completion Condition |
|---|---|
| Planner | `md_docs/planner/active/<FEATURE_NAME>_ARCHITECTURE.md` is classified as complete per Step 2 criteria |
| UI Scope Gate | Architecture document is complete AND contains a recoverable `UI_REQUIRED` or `UI_NOT_REQUIRED` classification (in the document body or derivable from its content per Step 4) |
| Designer | `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md` is classified as complete or complete (waiver) per Step 2 criteria |
| Developer | `md_docs/developer/active/<FEATURE_NAME>_DEVELOPER_COMPLETION.md` is complete with "Ready for Reviewer: Yes" |
| Reviewer | The highest-numbered `md_docs/reviewer/active/<FEATURE_NAME>_REVIEW_CYCLE_N.md` is complete and its Decision reads "Approved — implementation cleared for Builder" |
| Builder | `md_docs/builder/active/<FEATURE_NAME>_BUILDER_COMPLETION.md` is complete and its "Ready for Tester" field reads "Yes" |
| Tester | `md_docs/tester/active/<FEATURE_NAME>_TEST_COMPLETION.md` is complete and its "Pipeline Status" reads exactly: "All tests passing — ready for completion" |

The **re-entry point** is the phase immediately following the last completed phase. If no phase is complete, the re-entry point is Planner.

---

### Step 4 — UI Scope Gate Recovery

The UI Scope Gate decision must be recovered if it is not already recorded in a persisted file. Apply these recovery steps in order, stopping at the first successful recovery:

1. **Check `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md`** for a "UI Scope Gate" section with a "Classification" field. If present and reads `UI_REQUIRED` or `UI_NOT_REQUIRED`, use this value and record "Source: Found in prior execution report."

2. **Check `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md`** for the explicit waiver statement "No UI/UX design specification is required for this feature." If present, classify as `UI_NOT_REQUIRED` and record "Source: Recovered from Non-UI Waiver in design document."

3. **Read `md_docs/planner/active/<FEATURE_NAME>_ARCHITECTURE.md`** and apply the Orchestrator's classification criteria: `UI_REQUIRED` if the architecture contains frontend files, UI components, interaction state changes, visual state specifications, or accessibility-impacting behavior; `UI_NOT_REQUIRED` if the architecture is exclusively backend, infrastructure, data pipeline, or non-UI workflow. Record "Source: Re-derived from architecture content."

4. **If the architecture document is incomplete or missing:** The UI Scope Gate cannot be recovered. Treat this as a Planner-phase failure and set the re-entry point to Planner. Record "Source: Unrecoverable — architecture document absent or incomplete."

---

### Step 5 — Incomplete File Assessment

For any file classified as incomplete in Step 2, apply the following rules. The goal is to ensure the pipeline resumes against clean, valid contracts — not against partially-written files from an interrupted session:

- **If the incomplete file belongs to a phase earlier than the re-entry point determined in Step 3:** Archive the incomplete file using the archive operation defined in `SKILL.md` (append a UTC timestamp before the `.md` extension and move it to `md_docs/{agent}/archive/`). Set the re-entry point to this earlier phase so the producing agent re-runs and writes a fresh complete contract. Record the disposition in the Recovery Report.

- **If the incomplete file belongs to the re-entry phase itself:** The agent for that phase must re-run from the beginning of its workflow. Archive the incomplete file before the agent writes a new one. An agent that finds an existing file at its output path may append to it rather than replace it — archiving first prevents this.

- **If the incomplete file belongs to a phase after the re-entry point:** It is a forward artifact from a previous partial run that was produced against specifications that may have since changed. Archive it before the pipeline resumes. Do not allow downstream agents to read a file that was produced in a prior, potentially superseded run.

---

### Step 6 — Stale Contract Detection

A file can be complete per Step 2 criteria and still be stale — produced against a prior version of its upstream specification. A stale contract is a silent failure waiting to happen: the downstream agent reads it, implements against it, and the Reviewer catches the discrepancy.

Apply these staleness rules in order. Always apply the earliest (most upstream) re-entry point indicated by any rule — do not resume from a later phase if an earlier phase's output is stale:

- **If the architecture document has a more recent modification timestamp than the design document:** The design document was produced before the most recent architecture change and may no longer reflect the current architecture. Set the re-entry point to Designer at minimum. Archive the stale design document before resuming.

- **If the design document has a more recent modification timestamp than the Developer Completion Report:** The Completion Report was produced before the most recent design change and may no longer reflect current specifications. Set the re-entry point to Developer at minimum. Archive the stale Completion Report.

- **If the Developer Completion Report has a more recent modification timestamp than the most recent Reviewer cycle file:** The Reviewer cycle was produced before the most recent Developer submission and may be evaluating an outdated version of the implementation. Set the re-entry point to Reviewer at minimum. Archive the stale Reviewer cycle file.

- **If the Reviewer's most recent cycle is Approved but the Builder Completion Report is absent:** The Builder phase was not reached or did not complete. Set the re-entry point to Builder.

- **If the Builder Completion Report exists with "Ready for Tester: Yes" but no Test Completion file exists:** The Tester phase was not reached or did not complete. Set the re-entry point to Tester.

---

### Step 7 — Effort Gate Recovery

Read the Orchestrator Scheduling Note from Section 10 of the architecture document. The story point total determines whether user confirmation was required before the pipeline could proceed past Planner and Designer.

- **If the story point total is 5 or fewer:** No confirmation is required. Proceed to Step 8.

- **If the story point total is 6–13 (milestone confirmation required):** Verify whether user confirmation was already received in a prior session.
  - If `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` exists and its "Confirmation Status" field reads "Confirmed by user on [date]", treat confirmation as received for the confirmed phase. Do not request it again.
  - If no Pipeline Execution Report exists or the Confirmation Status field reads "Awaiting confirmation", present the phase breakdown to the user and request confirmation before resuming. Do not proceed without it.

- **If the story point total is 14 or more (phased approval required):** Apply the same logic as 6–13, with the additional requirement that each phase transition must have been explicitly approved. If the Pipeline Execution Report does not show phase-by-phase approval, re-request approval for the next phase before resuming.

---

### Step 8 — Recovery Report

Before resuming the pipeline, produce the following Recovery Report. Append it to `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` under a new section header `## Session Recovery — [UTC timestamp]`. If no Pipeline Execution Report exists yet, create it with this as its initial content.

```
SESSION RECOVERY REPORT
=======================

Feature           : [feature name]
Recovery Timestamp: [UTC timestamp in YYYYMMDDHHMMSS format]

Artifact Inventory
------------------
[Paste the complete inventory from Step 2 — all files found, their status, and missing fields if incomplete]

Phase Completion Assessment
---------------------------
Last Completed Phase : [phase name — or "None" if no phase is complete]
Re-entry Point       : [phase name]
Re-entry Reason      : [specific description: which condition from Step 3 was not met, or which staleness rule from Step 6 was triggered]

UI Scope Gate
-------------
Classification : [UI_REQUIRED / UI_NOT_REQUIRED]
Source         : [Found in prior execution report / Recovered from Non-UI Waiver in design document / Re-derived from architecture content / Unrecoverable]
Evidence       : [brief rationale — which file and which content supports the classification]

Stale Contract Detections
-------------------------
[For each stale file: file path — reason for staleness (timestamp comparison result) — disposition (archived / re-run required)]
[or: None detected]

Incomplete File Dispositions
----------------------------
[For each incomplete file: file path — missing fields — action taken (archived before re-run / flagged for re-run)]
[or: None detected]

Effort Gate Status
------------------
Story Points       : [X sp — from architecture Section 10 Orchestrator Scheduling Note]
Confirmation Status: [Not required (≤5 sp) / Confirmed in prior run on [date] / Requires confirmation before resuming]

Recovery Decision
-----------------
[One of:]
[Resume from [phase name] — all prior artifacts valid, effort gate satisfied]
[Request confirmation — story points require user approval before [phase name] can run]
[Halt — Unrecoverable — [specific reason: architecture missing / contradictory artifact states / etc.]]
```

---

### Step 9 — Resume, Request Confirmation, or Halt

After producing the Recovery Report, take exactly one of the following actions based on the Recovery Decision recorded in Step 8:

**Resume:** If the re-entry point is determined, all required prior artifacts are valid and non-stale, and any required effort gate confirmation is satisfied, resume the pipeline from the re-entry point. Apply all standard phase validation checkpoints and escalation rules as if this were a normal pipeline run — recovery does not relax any checkpoint requirements.

**Request Confirmation:** If the story point total requires user confirmation and no prior confirmation is recorded in the Pipeline Execution Report, present the following to the user and wait for a reply before taking any further action:
- The phase breakdown from the architecture document
- The last completed phase and the proposed re-entry point
- A summary of any stale or incomplete artifacts that were found

Wait for the user to reply with one of:
- **"Proceed"** or equivalent → Resume from the re-entry point.
- **"Restart from beginning"** → Archive all current active files for this feature using the archive operation defined in `SKILL.md` and proceed with standard Pre-Execution Planning for a fresh run.
- **"Cancel"** or no response → Halt and produce a halt record in the Pipeline Execution Report.

**Halt — Unrecoverable:** If the architecture document is missing or incomplete and the feature scope cannot be reconstructed, or if contradictory artifact states exist that cannot be resolved by any rule in Steps 5 or 6, produce the Recovery Report with a "Halt — Unrecoverable" decision. State the specific reason clearly (e.g., "Architecture document absent — cannot determine scope, UI Scope Gate, or effort gate values"). Present this to the user and request manual guidance. Do not attempt a partial recovery or guess at the re-entry point.

---

## Constraints

- **Do not restart the pipeline from the beginning if any valid completed-phase artifact exists.** Restarting discards valid work. Recovery must resume from the earliest incomplete or stale phase, not from Planner.
- **Do not skip the staleness check in Step 6.** A file's existence and completeness do not mean it is current. A completed Developer Completion Report produced before an architecture change is stale and must trigger a re-run.
- **Do not archive files without recording the disposition in the Recovery Report.** Every archive action must be traceable. An archive without a record makes future recovery attempts unable to understand why the file was moved.
- **Do not resume past a phase that requires user confirmation without receiving that confirmation in the current session.** Prior session confirmation is not carried over unless it is recorded in the Pipeline Execution Report with a timestamp.
- **Do not modify any specification document during recovery.** Recovery is a read, assess, and route operation. Modifying a specification during recovery invalidates the staleness analysis performed in Step 6.
- **Do not invoke this prompt recursively.** If recovery itself encounters an unresolvable state, produce a Halt report and surface the specific failure to the user. Do not attempt to recover the recovery.
- **Use the archive operation defined in `SKILL.md`** for all archiving actions — append a single UTC timestamp to the filename before `.md` and move to `md_docs/{agent}/archive/`. Do not invent an alternative archive format.
