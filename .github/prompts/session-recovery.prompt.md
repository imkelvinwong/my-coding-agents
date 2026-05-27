---
name: session-recovery
description: Extends the Orchestrator agent with a structured recovery workflow. When the Orchestrator detects a partially-complete pipeline for a named feature, it follows this prompt's 10-step procedure to reconstruct state from md_docs artifacts, identify the correct re-entry point, archive stale and orphaned files, and resume — or safely halt — without restarting from the beginning.
---

# Session Recovery Prompt

## When to Use This Prompt

This prompt is used **by the Orchestrator agent** — not as a standalone analysis tool, but as an extension of the Orchestrator's Pre-Execution Planning phase. It activates automatically whenever the Orchestrator detects that a prior pipeline run exists for the requested feature.

**You do not invoke this prompt separately.** Instead, start a conversation with the Orchestrator agent and describe which feature to resume. The Orchestrator checks for existing artifacts as part of its standard Pre-Execution Planning, finds them, and follows this prompt's 10-step recovery workflow before continuing the pipeline.

Example invocations directed at the Orchestrator:
- `"Resume the UserAuth pipeline"`
- `"Continue where we left off on DarkModeToggle"`
- `"The session restarted — pick up the CsvExport feature"`
- Simply starting a new session with `"Build the UserAuth feature"` when artifacts already exist in `md_docs/*/active/` will trigger recovery automatically.

**Why the Orchestrator and not a general agent:** Session recovery is not analysis — it is execution. The recovery workflow produces archive operations, writes to the Pipeline Execution Report, and resumes live agent invocations. These are all Orchestrator responsibilities. A general agent reading the artifacts could produce a Recovery Report, but it could not actually resume the pipeline because it does not have the `agent` invocation tool or the pipeline's execution authority.

**No `## Instructions` section in this prompt:** Unlike `pipeline-analysis.prompt.md`, which establishes a fresh analyst persona via its `## Instructions` section, this prompt extends the Orchestrator's existing persona. The Orchestrator already knows its role, tools, and pipeline context. This prompt provides a structured procedure — a 10-step workflow — that slots into the Orchestrator's Pre-Execution Planning step. Adding a persona-establishing `## Instructions` section would create a competing identity that conflicts with the Orchestrator's system prompt. The Orchestrator reads this prompt as a procedure to execute, not as a new role to adopt.

---

## Purpose

This prompt is invoked when the Orchestrator detects that a pipeline for a known feature is already partially complete — either because the session restarted mid-execution or because the Orchestrator was invoked on a feature that already has artifacts present in `md_docs/*/active/`.

The Orchestrator must never restart a pipeline from the beginning when recoverable state exists. Restarting discards completed work, forces agents to re-run phases whose outputs are still valid, and loses the context established by prior agent decisions. This prompt defines the exact steps to reconstruct state, assess completeness, determine staleness, clean up orphaned staging artifacts, and resume from the correct re-entry point.

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

### Step 2 — Staging Directory Scan and Orphaned File Cleanup

Before reading any contract artifact from `md_docs/*/active/`, scan `md_docs/tester/staging/` for all files matching `<FEATURE_NAME>_*.md`. This scan must be executed before Step 3 — staging file presence directly overrides Tester phase completion status determined in Step 4, and that override must be established before the artifact inventory is assembled.

Staging files in `md_docs/tester/staging/` are ephemeral Fan-Out artifacts defined in `SKILL.md` Core Rule 6. Their presence during session-recovery means a Fan-Out was initiated but never consolidated in a prior run — either because the Fan-Out was aborted by a heartbeat timeout or a concurrency control violation, or because the pipeline was interrupted between Fan-Out completion and the Orchestrator's archive operation. In either case, staging files are never valid contract state and must be treated as orphans.

**If no files matching `<FEATURE_NAME>_*.md` are found in `md_docs/tester/staging/`:**

No orphaned staging artifacts are present for this feature. Record "Orphaned Staging Cleanup: None detected" in the Recovery Report. Proceed to Step 3.

**If one or more files matching `<FEATURE_NAME>_*.md` are found in `md_docs/tester/staging/`:**

Apply the following cleanup procedure exactly. Do not attempt to read or evaluate the content of orphaned staging files — their content is not valid contract state regardless of what they contain.

1. **Inventory all orphaned staging files.** Record the full path of every file found. Expected orphan filenames are `<FEATURE_NAME>_SPEC_INDEPENDENT_START.md`, `<FEATURE_NAME>_UI_DEPENDENT_START.md`, `<FEATURE_NAME>_SPEC_INDEPENDENT_TESTS.md`, and `<FEATURE_NAME>_UI_DEPENDENT_TESTS.md`. Any subset of these four files may be present — treat every file found as an orphan regardless of how many are present.

2. **Generate one UTC timestamp** in `YYYYMMDDHHMMSS` format for the orphaned staging cleanup batch. All staging files cleaned up in this step share this single timestamp so the cleanup batch is identifiable as one operation.

3. **Archive each orphaned staging file** using the staging archive path defined in `SKILL.md`: move each file from `md_docs/tester/staging/<FEATURE_NAME>_<DOC_TYPE>.md` to `md_docs/tester/archive/<FEATURE_NAME>_<DOC_TYPE>_YYYYMMDDHHMMSS.md`. Create `md_docs/tester/archive/` if it does not exist.

4. **Set the Tester phase override flag.** Regardless of what the Tester contract artifacts in `md_docs/tester/active/` show, the Tester phase must be treated as incomplete. If `<FEATURE_NAME>_TEST_COMPLETION.md` exists in `md_docs/tester/active/` and would otherwise satisfy the Step 4 Tester completion condition, that completion condition is overridden by this flag. The re-entry point determined in Step 4 must be Tester or earlier — it cannot be a phase after Tester. This override exists because the presence of orphaned staging files proves the prior pipeline run's Fan-Out was not completed and consolidated, meaning any TEST_COMPLETION.md present was either written before Fan-Out ran or written in an inconsistent state. Do not trust its content for phase completion purposes.

5. **Record the cleanup disposition.** Write all four details in the Recovery Report's "Orphaned Staging Cleanup" section: the list of files found, the UTC timestamp used, the list of files moved and their archive destinations, and the Tester phase override flag status.

After the cleanup procedure is complete, proceed to Step 3.

---

### Step 3 — Artifact Inventory

Read every file found in Step 1. For each file, record its path, the agent that produced it, its file type, and whether it is complete. Do not rely on file size or modification timestamp alone to determine completeness — read the file content and verify the required fields are present.

**Per-agent completeness criteria — a file is complete only if all listed fields are present and non-empty:**

| Agent | File | Required Fields for Completeness |
|---|---|---|
| Planner | `<FEATURE_NAME>_ARCHITECTURE.md` | All 11 section headers present; Section 10 contains a fenced `json` code block that parses successfully into an object with all four required keys: `feature_name` (string), `total_story_points` (integer), `phase_breakdown` (array), and `execution_recommendation` (one of the three permitted string literals). The prose Orchestrator Scheduling Note line is present for human readability but is not evaluated for completeness — the JSON block is the authoritative completeness signal. |
| Designer | `<FEATURE_NAME>_DESIGN.md` | Either: all 10 section headers present (full specification), OR: the document header contains "No UI/UX design specification is required for this feature" and all four waiver elements defined in the Non-UI Waiver Schema in `SKILL.md` are present and non-empty |
| Developer | `<FEATURE_NAME>_DEVELOPER_COMPLETION.md` | "Files Created", "Files Modified", "Self-Review Checklist", and "Ready for Reviewer" fields all present; "Ready for Reviewer" reads "Yes" |
| Reviewer | `<FEATURE_NAME>_REVIEW_CYCLE_N.md` (highest N) | `decision_code` field present and reads exactly `APPROVED` or `REJECTED` (uppercase, exact string match — do not evaluate surrounding prose as a substitute for this field) |
| Builder | `<FEATURE_NAME>_BUILDER_COMPLETION.md` | "Build Status", "Dev Server" (Status and Port fields), "Smoke Test Results", and "Ready for Tester" fields all present; "Ready for Tester" reads "Yes" or "No" with a reason |
| Tester | `<FEATURE_NAME>_TEST_PLAN.md` | "Unit Tests", "Integration Tests", and "Edge Cases and Boundary Conditions" sections all present |
| Tester | `<FEATURE_NAME>_TEST_COMPLETION.md` | "Test Results" (Total, Passed, Failed, Pass Rate), "Coverage" (all four metrics), and Pipeline Status section present with both `status_code` and `status_detail` sub-fields; `status_code` reads exactly `PASSING` or `BLOCKED` (uppercase, exact string match — do not evaluate `status_detail` prose as a substitute for this field). If the Tester phase override flag was set in Step 2, this file is classified as incomplete regardless of its content. |
| Orchestrator | `<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` | "Phases Executed", "Phase Outcomes", "UI Scope Gate" (Classification and Evidence), and "Next Steps" fields all present |

A file that exists but is missing any required field is classified as **incomplete**. A file that contains all required fields with non-empty values is classified as **complete**. A design document that is a valid Non-UI Waiver is classified as **complete (waiver)**. A Tester completion file that would otherwise be classified as complete but is subject to the Step 2 Tester phase override flag is classified as **incomplete (staging override)**.

Produce an internal Artifact Inventory:

```
ARTIFACT INVENTORY
==================

Feature      : [feature name]
Scan Path    : md_docs/*/active/<FEATURE_NAME>_*.md

Files Found
-----------
[file path] — [agent] — [file type] — [status: complete / incomplete / complete (waiver) / incomplete (staging override)]
[missing required fields if incomplete: list each missing field]

Files Not Found (expected but absent)
--------------------------------------
[file path] — [agent] — [file type]
```

---

### Step 4 — Phase Completion Assessment

Using the Artifact Inventory, determine the last fully completed pipeline phase. Apply the following rules in order — the last phase whose completion condition is fully satisfied is the last completed phase:

| Phase | Completion Condition |
|---|---|
| Planner | `md_docs/planner/active/<FEATURE_NAME>_ARCHITECTURE.md` is classified as complete per Step 3 criteria |
| UI Scope Gate | Architecture document is complete AND contains a recoverable `UI_REQUIRED` or `UI_NOT_REQUIRED` classification (in the document body or derivable from its content per Step 5) |
| Designer | `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md` is classified as complete or complete (waiver) per Step 3 criteria |
| Developer | `md_docs/developer/active/<FEATURE_NAME>_DEVELOPER_COMPLETION.md` is complete with "Ready for Reviewer: Yes" |
| Reviewer | The highest-numbered `md_docs/reviewer/active/<FEATURE_NAME>_REVIEW_CYCLE_N.md` is complete and its `decision_code` reads exactly: `APPROVED` (uppercase, exact string match — do not read surrounding prose for this determination) |
| Builder | `md_docs/builder/active/<FEATURE_NAME>_BUILDER_COMPLETION.md` is complete and its "Ready for Tester" field reads "Yes" |
| Tester | `md_docs/tester/active/<FEATURE_NAME>_TEST_COMPLETION.md` is complete per Step 3 criteria (not subject to staging override) and its `status_code` reads exactly: `PASSING` (uppercase, exact string match — do not read `status_detail` prose for this determination) |

The **re-entry point** is the phase immediately following the last completed phase. If no phase is complete, the re-entry point is Planner. If the Step 2 Tester phase override flag is set, the Tester phase completion condition is treated as not satisfied regardless of the `status_code` value in the completion file — the re-entry point must be Tester or earlier.

---

### Step 5 — UI Scope Gate Recovery

The UI Scope Gate decision must be recovered if it is not already recorded in a persisted file. Apply these recovery steps in order, stopping at the first successful recovery:

1. **Check `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md`** for a "UI Scope Gate" section with a "Classification" field. If present and reads `UI_REQUIRED` or `UI_NOT_REQUIRED`, use this value and record "Source: Found in prior execution report."

2. **Check `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md`** for the explicit waiver statement "No UI/UX design specification is required for this feature." If present, classify as `UI_NOT_REQUIRED` and record "Source: Recovered from Non-UI Waiver in design document."

3. **Read `md_docs/planner/active/<FEATURE_NAME>_ARCHITECTURE.md`** and apply the Orchestrator's classification criteria: `UI_REQUIRED` if the architecture contains frontend files, UI components, interaction state changes, visual state specifications, or accessibility-impacting behavior; `UI_NOT_REQUIRED` if the architecture is exclusively backend, infrastructure, data pipeline, or non-UI workflow. Record "Source: Re-derived from architecture content."

4. **If the architecture document is incomplete or missing:** The UI Scope Gate cannot be recovered. Treat this as a Planner-phase failure and set the re-entry point to Planner. Record "Source: Unrecoverable — architecture document absent or incomplete."

---

### Step 6 — Incomplete File Assessment

For any file classified as incomplete in Step 3, apply the following rules. The goal is to ensure the pipeline resumes against clean, valid contracts — not against partially-written files from an interrupted session:

- **If the incomplete file belongs to a phase earlier than the re-entry point determined in Step 4:** Archive the incomplete file using the archive operation defined in `SKILL.md` (append a UTC timestamp before the `.md` extension and move it to `md_docs/{agent}/archive/`). Set the re-entry point to this earlier phase so the producing agent re-runs and writes a fresh complete contract. Record the disposition in the Recovery Report.

- **If the incomplete file belongs to the re-entry phase itself:** The agent for that phase must re-run from the beginning of its workflow. Archive the incomplete file before the agent writes a new one. An agent that finds an existing file at its output path may append to it rather than replace it — archiving first prevents this.

- **If the incomplete file belongs to a phase after the re-entry point:** It is a forward artifact from a previous partial run that was produced against specifications that may have since changed. Archive it before the pipeline resumes. Do not allow downstream agents to read a file that was produced in a prior, potentially superseded run.

---

### Step 7 — Stale Contract Detection

A file can be complete per Step 3 criteria and still be stale — produced against a prior version of its upstream specification. A stale contract is a silent failure waiting to happen: the downstream agent reads it, implements against it, and the Reviewer catches the discrepancy.

Apply these staleness rules in order. Always apply the earliest (most upstream) re-entry point indicated by any rule — do not resume from a later phase if an earlier phase's output is stale:

- **If the architecture document has a more recent modification timestamp than the design document:** The design document was produced before the most recent architecture change and may no longer reflect the current architecture. Set the re-entry point to Designer at minimum. Archive the stale design document before resuming.

- **If the design document has a more recent modification timestamp than the Developer Completion Report:** The Completion Report was produced before the most recent design change and may no longer reflect current specifications. Set the re-entry point to Developer at minimum. Archive the stale Completion Report.

- **If the Developer Completion Report has a more recent modification timestamp than the most recent Reviewer cycle file:** The Reviewer cycle was produced before the most recent Developer submission and may be evaluating an outdated version of the implementation. Set the re-entry point to Reviewer at minimum. Archive the stale Reviewer cycle file.

- **If the Reviewer's most recent cycle has `decision_code: APPROVED` but the Builder Completion Report is absent:** The Builder phase was not reached or did not complete. Set the re-entry point to Builder.

- **If the Builder Completion Report exists with "Ready for Tester: Yes" but no Test Completion file exists:** The Tester phase was not reached or did not complete. Set the re-entry point to Tester.

---

### Step 8 — Effort Gate Recovery

Read Section 10 of `md_docs/planner/active/<FEATURE_NAME>_ARCHITECTURE.md` and locate the fenced `json` code block. Extract and parse the JSON payload to obtain the `total_story_points` integer value. Do not read the prose Orchestrator Scheduling Note line for threshold decisions — it is present for human readability only and is not the machine-authoritative source for the story point total.

**JSON extraction procedure for effort gate recovery:**

1. Find the opening ` ```json ` fence inside Section 10.
2. Read all content between the opening and closing fence as a JSON string.
3. Parse the JSON string. The resulting object must contain the key `total_story_points` with an integer value.
4. If the JSON block is absent, unparseable, or missing the `total_story_points` key: the effort gate cannot be recovered from the architecture document alone. Check `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` for a "Story Points" field under the "Effort" section. If found, use that value. If both sources are absent or unparseable, treat this as a Planner-phase failure — the architecture document is not fully complete — and set the re-entry point to Planner. Record "Effort Gate: Unrecoverable — JSON payload absent or malformed in Section 10."

Once `total_story_points` is obtained as an integer, apply the following thresholds:

- **If `total_story_points` is 5 or fewer:** No confirmation is required. Proceed to Step 9.

- **If `total_story_points` is 6–13 (milestone confirmation required):** Verify whether user confirmation was already received in a prior session.
  - If `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` exists and its "Confirmation Status" field reads "Confirmed by user on [date]", treat confirmation as received for the confirmed phase. Do not request it again.
  - If no Pipeline Execution Report exists or the Confirmation Status field reads "Awaiting confirmation", present the `phase_breakdown` array from the parsed JSON payload to the user and request confirmation before resuming. Do not proceed without it.

- **If `total_story_points` is 14 or more (phased approval required):** Apply the same logic as 6–13, with the additional requirement that each phase transition must have been explicitly approved. If the Pipeline Execution Report does not show phase-by-phase approval, re-request approval for the next phase before resuming.

---

### Step 9 — Recovery Report

Before resuming the pipeline, produce the following Recovery Report. Append it to `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` under a new section header `## Session Recovery — [UTC timestamp]`. If no Pipeline Execution Report exists yet, create it with this as its initial content.

```
SESSION RECOVERY REPORT
=======================

Feature           : [feature name]
Recovery Timestamp: [UTC timestamp in YYYYMMDDHHMMSS format]

Orphaned Staging Cleanup
------------------------
[One of:]
[None detected — md_docs/tester/staging/ contained no files matching <FEATURE_NAME>_*.md]
[Files found and archived:]
  [file path] → [archive destination path] — archived at [UTC timestamp YYYYMMDDHHMMSS]
  [repeat for each file archived]
  Tester Phase Override: [Active — Tester phase treated as incomplete regardless of TEST_COMPLETION.md content]

Artifact Inventory
------------------
[Paste the complete inventory from Step 3 — all files found, their status, and missing fields if incomplete. Include any files classified as incomplete (staging override).]

Phase Completion Assessment
---------------------------
Last Completed Phase : [phase name — or "None" if no phase is complete]
Re-entry Point       : [phase name]
Re-entry Reason      : [specific description: which condition from Step 4 was not met, or which staleness rule from Step 7 was triggered, or "Tester phase override active — orphaned staging files found in Step 2"]

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
Story Points       : [X sp — parsed from JSON payload in Section 10 of architecture document, or recovered from Pipeline Execution Report "Story Points" field]
JSON Source        : [Parsed from Section 10 JSON block / Recovered from Pipeline Execution Report / Unrecoverable — re-entry set to Planner]
Confirmation Status: [Not required (≤5 sp) / Confirmed in prior run on [date] / Requires confirmation before resuming]

Recovery Decision
-----------------
[One of:]
[Resume from [phase name] — all prior artifacts valid, effort gate satisfied]
[Request confirmation — story points require user approval before [phase name] can run]
[Halt — Unrecoverable — [specific reason: architecture missing / JSON payload absent or malformed / contradictory artifact states / etc.]]
```

---

### Step 10 — Resume, Request Confirmation, or Halt

After producing the Recovery Report, take exactly one of the following actions based on the Recovery Decision recorded in Step 9:

**Resume:** If the re-entry point is determined, all required prior artifacts are valid and non-stale, and any required effort gate confirmation is satisfied, resume the pipeline from the re-entry point. Apply all standard phase validation checkpoints and escalation rules as if this were a normal pipeline run — recovery does not relax any checkpoint requirements.

**Request Confirmation:** If the story point total requires user confirmation and no prior confirmation is recorded in the Pipeline Execution Report, present the following to the user and wait for a reply before taking any further action:
- The `phase_breakdown` array from the parsed JSON payload in Section 10 of the architecture document
- The last completed phase and the proposed re-entry point
- A summary of any stale or incomplete artifacts that were found
- A summary of any orphaned staging files that were archived in Step 2

Wait for the user to reply with one of:
- **"Proceed"** or equivalent → Resume from the re-entry point.
- **"Restart from beginning"** → Archive all current active files for this feature using the archive operation defined in `SKILL.md` and proceed with standard Pre-Execution Planning for a fresh run.
- **"Cancel"** or no response → Halt and produce a halt record in the Pipeline Execution Report.

**Halt — Unrecoverable:** If the architecture document is missing or incomplete and the feature scope cannot be reconstructed, or if the JSON scheduling payload in Section 10 is absent or unparseable and no fallback value exists in the Pipeline Execution Report, or if contradictory artifact states exist that cannot be resolved by any rule in Steps 6 or 7, produce the Recovery Report with a "Halt — Unrecoverable" decision. State the specific reason clearly (e.g., "Architecture document absent — cannot determine scope, UI Scope Gate, or effort gate values" or "Section 10 JSON payload malformed — total_story_points could not be extracted and no Pipeline Execution Report fallback exists"). Present this to the user and request manual guidance. Do not attempt a partial recovery or guess at the re-entry point.

---

## Constraints

- **Do not restart the pipeline from the beginning if any valid completed-phase artifact exists.** Restarting discards valid work. Recovery must resume from the earliest incomplete or stale phase, not from Planner.
- **Do not skip the staging directory scan in Step 2.** Orphaned staging files must be identified and archived before the Artifact Inventory is assembled in Step 3. A staging file found after Step 3 is complete invalidates the Tester completeness determination already made.
- **Do not read or evaluate the content of orphaned staging files.** Staging file content is never valid contract state. Archive orphaned staging files without reading them.
- **Do not treat a Tester completion file as complete when the Step 2 Tester phase override flag is set.** The flag is set when orphaned staging files are found, and it cannot be unset by the content of any contract artifact. The re-entry point must be Tester or earlier regardless of what `status_code` shows in the TEST_COMPLETION.md file.
- **Do not read `status_detail` prose to determine Tester phase completion.** Read `status_code` only. The exact uppercase string `PASSING` is the sole valid complete signal for the Tester phase. Any other value — including a paraphrase, a lowercase variant, or an absent field — means the Tester phase is not complete.
- **Do not read surrounding prose to determine Reviewer phase completion.** Read `decision_code` only. The exact uppercase string `APPROVED` is the sole valid complete signal for the Reviewer phase. Any other value — including the prose "Approved — implementation cleared for Builder", a lowercase variant, or an absent field — means the Reviewer phase is not complete.
- **Do not use the prose Orchestrator Scheduling Note line for effort gate threshold decisions.** Extract `total_story_points` from the fenced JSON block in Section 10. The prose line is present for human readability only and may not match the JSON value if the document was manually edited after generation.
- **Do not skip the staleness check in Step 7.** A file's existence and completeness do not mean it is current. A completed Developer Completion Report produced before an architecture change is stale and must trigger a re-run.
- **Do not archive files without recording the disposition in the Recovery Report.** Every archive action must be traceable. An archive without a record makes future recovery attempts unable to understand why the file was moved. This applies equally to orphaned staging files archived in Step 2 and to contract artifacts archived in Steps 6 and 7.
- **Do not resume past a phase that requires user confirmation without receiving that confirmation in the current session.** Prior session confirmation is not carried over unless it is recorded in the Pipeline Execution Report with a timestamp.
- **Do not modify any specification document during recovery.** Recovery is a read, assess, and route operation. Modifying a specification during recovery invalidates the staleness analysis performed in Step 7.
- **Do not invoke this prompt recursively.** If recovery itself encounters an unresolvable state, produce a Halt report and surface the specific failure to the user. Do not attempt to recover the recovery.
- **Use the archive operation defined in `SKILL.md`** for all archiving actions — append a single UTC timestamp to the filename before `.md` and move to `md_docs/{agent}/archive/` for contract artifacts or `md_docs/tester/archive/` for staging artifacts. Do not invent an alternative archive format.
