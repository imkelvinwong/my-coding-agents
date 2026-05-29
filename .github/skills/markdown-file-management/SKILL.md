---
name: markdown-file-management
description: Lightweight markdown mechanics for path resolution, directory creation, and orchestrator-driven archiving. Agents that do not have an explicit output file path in their own prompt must consult this skill to resolve the correct path before writing any file.
---

# Markdown File Management Skill

## Domain
Documentation, File System, Path Resolution, Archive Operations

## Purpose

This skill handles file system mechanics only:
- Resolving output file paths for each agent's deliverables
- Ensuring required directories exist before write operations
- Executing archive operations when instructed by the Orchestrator
- Defining I/O boundaries for Fan-Out staging files to prevent disk race conditions between concurrent sub-agents

**What this skill does not do:** It does not define SDLC semantics, ownership rules, or pipeline policy. Those decisions live in each agent's prompt. This skill is the mechanical execution layer — it answers "where does this file go?", "how does archiving work?", and "which agent is permitted to write to which path?", not "should this file be archived?"

**When to consult this skill:** Any agent whose prompt does not specify an explicit output file path must resolve the path here before writing. Session-recovery uses this skill for all archive operations. Agents must not invent paths or use legacy paths not listed in this skill. During Fan-Out execution, both Agent A and Agent B must consult this skill to confirm their designated staging output paths before writing any file.

---

## Required Directory Layout

The directory structure below reflects the contract artifact lifecycle — which agent writes artifacts first through last. This is not the same as invocation order (the Orchestrator may be invoked first but writes its execution report at pipeline completion or halt).

All governed artifact filenames must follow the format `<FEATURE_NAME>_<DOC_TYPE>.md`, where `<FEATURE_NAME>` is the canonical PascalCase feature name established by the Orchestrator at pipeline start and communicated to every agent at invocation.

```
md_docs/
    planner/
        active/
            <FEATURE_NAME>_ARCHITECTURE.md
        archive/
    designer/
        active/
            <FEATURE_NAME>_DESIGN.md          ← full specification or Non-UI Waiver
        archive/
    researcher/
        active/
            <FEATURE_NAME>_TECHNICAL_VERIFICATION_REPORT.md
        archive/
    developer/
        active/
            <FEATURE_NAME>_DEVELOPER_COMPLETION.md
        archive/
    reviewer/
        active/
            <FEATURE_NAME>_REVIEW_CYCLE_N.md  ← N increments per review cycle
        archive/
    builder/
        active/
            <FEATURE_NAME>_BUILDER_COMPLETION.md
        archive/
    tester/
        active/
            <FEATURE_NAME>_TEST_PLAN.md
            <FEATURE_NAME>_TEST_COMPLETION.md
        staging/
            <FEATURE_NAME>_SPEC_INDEPENDENT_START.md   ← heartbeat signal, Agent A
            <FEATURE_NAME>_UI_DEPENDENT_START.md        ← heartbeat signal, Agent B
            <FEATURE_NAME>_SPEC_INDEPENDENT_TESTS.md    ← Fan-Out test output, Agent A
            <FEATURE_NAME>_UI_DEPENDENT_TESTS.md        ← Fan-Out test output, Agent B
        archive/
    orchestrator/
        active/
            <FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md
        archive/
```

**Note on the designer directory:** The `<FEATURE_NAME>_DESIGN.md` file is written either by the Designer agent (full specification) or by the Orchestrator (Non-UI Waiver). Both cases use the same filename and directory. The Reviewer determines which type it is by reading the document header.

**Note on the reviewer directory:** Review cycle files include a numeric suffix: `<FEATURE_NAME>_REVIEW_CYCLE_1.md`, `<FEATURE_NAME>_REVIEW_CYCLE_2.md`, etc. Always use the highest N present as the current decision. Prior cycle files remain in `active/` until the pipeline completes or halts — they are historical context, not stale contracts.

**Note on the tester staging directory:** The `staging/` directory is used exclusively during Fan-Out test generation. It is not present or required during sequential test generation runs. The four staging files are ephemeral — they are produced by Fan-Out agents A and B, consumed by the Tester's consolidation step, and archived by the Orchestrator at pipeline completion or halt. No agent other than the Tester may read from `md_docs/tester/staging/`. Session-recovery never reads staging files as valid contract artifacts — their presence indicates an incomplete Fan-Out, which session-recovery treats as a Tester-phase failure requiring re-run from the beginning of the Tester phase.

**Note on the researcher directory:** The `<FEATURE_NAME>_TECHNICAL_VERIFICATION_REPORT.md` file is always written by the Researcher agent. It is never written by the Orchestrator or any other agent. The Reviewer reads it to verify ML module conformance. Session-recovery uses the "Ready for Developer" field to determine whether the Researcher phase completed.

---

## Agent Output Path Reference

Use this table to resolve the output path for each agent's deliverable. Agents must write to exactly these paths. Session-recovery uses exactly these paths to locate artifacts during reconstruction. During Fan-Out execution, Agent A and Agent B must write to exactly their designated staging paths — no other path is permitted during a Fan-Out phase.

| Agent | Output File | Path |
|---|---|---|
| Planner | Architecture document | `md_docs/planner/active/<FEATURE_NAME>_ARCHITECTURE.md` |
| Designer | Design specification or Non-UI Waiver | `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md` |
| Orchestrator | Non-UI Waiver (when UI_NOT_REQUIRED) | `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md` |
| Researcher | Technical Verification Report | `md_docs/researcher/active/<FEATURE_NAME>_TECHNICAL_VERIFICATION_REPORT.md` |
| Developer | Developer Completion Report | `md_docs/developer/active/<FEATURE_NAME>_DEVELOPER_COMPLETION.md` |
| Reviewer | Review Decision (per cycle) | `md_docs/reviewer/active/<FEATURE_NAME>_REVIEW_CYCLE_N.md` |
| Builder | Builder Completion Report | `md_docs/builder/active/<FEATURE_NAME>_BUILDER_COMPLETION.md` |
| Tester | Test Plan | `md_docs/tester/active/<FEATURE_NAME>_TEST_PLAN.md` |
| Tester | Test Completion Report | `md_docs/tester/active/<FEATURE_NAME>_TEST_COMPLETION.md` |
| Tester (Fan-Out heartbeat — Agent A) | Spec-Independent Start Signal | `md_docs/tester/staging/<FEATURE_NAME>_SPEC_INDEPENDENT_START.md` |
| Tester (Fan-Out heartbeat — Agent B) | UI-Dependent Start Signal | `md_docs/tester/staging/<FEATURE_NAME>_UI_DEPENDENT_START.md` |
| Tester (Fan-Out staging — Agent A) | Spec-Independent Tests | `md_docs/tester/staging/<FEATURE_NAME>_SPEC_INDEPENDENT_TESTS.md` |
| Tester (Fan-Out staging — Agent B) | UI-Dependent Tests | `md_docs/tester/staging/<FEATURE_NAME>_UI_DEPENDENT_TESTS.md` |
| Orchestrator | Pipeline Execution Report | `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` |

---

## Non-UI Waiver Schema

The Non-UI Waiver is authored by the Orchestrator and written to `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md`. It must contain exactly four elements. This is the single authoritative definition of those elements — all agents that author, validate, or consume a Non-UI Waiver must reference this schema rather than re-stating it independently.

| Element | Label | Required Content |
|---|---|---|
| a | Architecture reference | An explicit reference to `md_docs/planner/active/<FEATURE_NAME>_ARCHITECTURE.md` by its exact path. A generic mention of "the architecture document" without the path does not satisfy this element. |
| b | Scope classification | The exact string `UI_NOT_REQUIRED`. A paraphrase such as "no UI work required" does not satisfy this element. |
| c | Evidence statement | A specific, non-generic statement explaining why no UI surface is affected by this feature. A statement such as "this feature has no UI" without explaining why is not specific enough to satisfy this element. |
| d | Explicit waiver statement | The exact sentence: "No UI/UX design specification is required for this feature. All design conformance checklist items are waived." A paraphrase or partial version of this sentence does not satisfy this element. |

All four elements must be present and non-empty. Do not infer or approximate a missing element from surrounding content. Any agent that finds a waiver with a missing element must halt and report the specific missing element by its letter label (a, b, c, or d) and name before taking any further action.

---

## Core Rules

1. **All contract filenames must follow the `<FEATURE_NAME>_<DOC_TYPE>.md` format.** Examples: `UserAuth_ARCHITECTURE.md`, `UserAuth_DESIGN.md`, `UserAuth_PIPELINE_EXECUTION_REPORT.md`. Review cycle files include a numeric suffix: `UserAuth_REVIEW_CYCLE_1.md`. Deviating from this format breaks session-recovery's artifact inventory scan.

2. **Files in `active/` are the current contract for a feature.** Any agent reading from `active/` is reading the most recent valid version of that contract. No agent should hold a cached copy of a contract from a prior read — always read from `active/` at the time of use.

3. **Agents must read from `active/` and must never read from `archive/`.** Archive files are historical records. They may represent superseded, rejected, or partial-run specifications. An agent that reads an archived file may implement against a contract that has since been corrected.

4. **Only the Orchestrator initiates archive operations.** Individual agents must not call archive operations for their own output files or for any other agent's files. An agent that archives its own file mid-pipeline breaks session-recovery's ability to reconstruct state. Archive operations occur only at pipeline completion or halt, and only when invoked by the Orchestrator or by session-recovery under Orchestrator direction.

5. **Legacy paths are not valid.** The paths `.architecture/` and `.design/` are superseded by `md_docs/planner/active/` and `md_docs/designer/active/` respectively. Any file found at a legacy path during recovery should be treated as an orphan — do not read it as a valid contract.

6. **Staging files are ephemeral and are not contract artifacts.** Files in `md_docs/tester/staging/` are produced during Fan-Out test generation and consumed by the Tester's consolidation step. They do not represent a committed pipeline contract at any point in their lifecycle. No agent other than the Tester — and no session-recovery procedure — may read from `md_docs/tester/staging/` for contract or routing purposes. The Orchestrator archives staging files along with all other feature artifacts at pipeline completion or halt, using the same timestamp-append operation defined in the Archive Feature Files section below and applying the same batch timestamp used for all other artifacts in that archive operation. Staging files are never treated as resumable state by session-recovery — their presence in `staging/` indicates that a Fan-Out was initiated but not consolidated, which session-recovery classifies as a Tester-phase failure and resolves by setting the re-entry point to Tester for a full re-run.

---

## Mechanical Operations

### Resolve Paths

Given an agent name and the expected document type, resolve the output file path using the Agent Output Path Reference table above. If the document type is `REVIEW_CYCLE`, append the current cycle number N. If the document type is a Fan-Out staging file, use the exact path from the four staging rows in the table — do not construct a staging path by inference.

Format for contract artifacts: `md_docs/{agent}/active/<FEATURE_NAME>_<DOC_TYPE>.md`

Format for staging artifacts: `md_docs/tester/staging/<FEATURE_NAME>_<DOC_TYPE>.md`

Example resolutions:
- Planner writing architecture for `UserAuth` → `md_docs/planner/active/UserAuth_ARCHITECTURE.md`
- Reviewer writing cycle 2 decision for `UserAuth` → `md_docs/reviewer/active/UserAuth_REVIEW_CYCLE_2.md`
- Tester writing test plan for `DarkModeToggle` → `md_docs/tester/active/DarkModeToggle_TEST_PLAN.md`
- Agent A writing heartbeat signal for `CsvExport` → `md_docs/tester/staging/CsvExport_SPEC_INDEPENDENT_START.md`
- Agent A writing staging output for `CsvExport` → `md_docs/tester/staging/CsvExport_SPEC_INDEPENDENT_TESTS.md`
- Agent B writing heartbeat signal for `CsvExport` → `md_docs/tester/staging/CsvExport_UI_DEPENDENT_START.md`
- Agent B writing staging output for `CsvExport` → `md_docs/tester/staging/CsvExport_UI_DEPENDENT_TESTS.md`

### Ensure Directories

Before any write operation, confirm that the required directories exist for the agent writing the file. Create them if they do not exist. A write to a non-existent directory will fail — creating the directory first prevents this.

For contract artifact writers:
```
md_docs/{agent}/active/     ← must exist before writing
md_docs/{agent}/archive/    ← must exist before archiving
```

For Fan-Out staging writers (Agent A and Agent B):
```
md_docs/tester/staging/     ← must exist before either agent writes any staging file
```

The `staging/` directory is created by the Tester before Fan-Out agents are spawned. Neither Agent A nor Agent B is responsible for creating the `staging/` directory — they must confirm it exists before writing and halt with a self-termination signal if it does not. A missing `staging/` directory at the point of agent activation is treated as a heartbeat failure: the agent writes no start signal, self-terminates, and reports the missing directory as the failure cause.

### Staging Directory I/O Boundaries

This section defines the file-system-level I/O isolation rules that prevent disk race conditions between concurrent Fan-Out sub-agents. These rules are enforced at the path level — no coordination protocol between agents is required or permitted. Each agent's isolation is structural, not cooperative.

**Rule 1 — One agent, one file, one write handle.** Each Fan-Out agent holds an exclusive write handle to exactly one staging output file and exactly one staging heartbeat signal file. Agent A holds write handles only to `<FEATURE_NAME>_SPEC_INDEPENDENT_START.md` and `<FEATURE_NAME>_SPEC_INDEPENDENT_TESTS.md`. Agent B holds write handles only to `<FEATURE_NAME>_UI_DEPENDENT_START.md` and `<FEATURE_NAME>_UI_DEPENDENT_TESTS.md`. No agent opens, reads, or writes any file outside this two-file set during Fan-Out execution. The write handles are opened at agent activation and closed when the agent's staging output is complete or when the agent self-terminates.

**Rule 2 — No cross-agent reads of staging content.** Agent A must not read `<FEATURE_NAME>_UI_DEPENDENT_START.md` or `<FEATURE_NAME>_UI_DEPENDENT_TESTS.md` at any point. Agent B must not read `<FEATURE_NAME>_SPEC_INDEPENDENT_START.md` or `<FEATURE_NAME>_SPEC_INDEPENDENT_TESTS.md` at any point. An agent that reads the other agent's staging file has violated its isolation boundary. There is no legitimate reason for one Fan-Out agent to read the other's output — their scopes are disjoint by design. Cross-agent reads indicate a configuration error in the Fan-Out setup and must be treated as a concurrency control violation triggering an immediate abort.

**Rule 3 — No caching of staging file content across write operations.** Each agent writes its staging output file in a single sequential pass from first test case to last. An agent must not buffer the other agent's staging content, load the other agent's staging file into memory, or hold a read handle on any staging file other than its own. Caching staging content from a concurrently-writing agent produces stale reads — the cached content reflects the staging file's state at cache time, not at consolidation time — and may silently produce an incomplete merged test suite.

**Rule 4 — No writes to shared directories outside staging.** During Fan-Out execution, both agents operate in read-only mode with respect to all directories except `md_docs/tester/staging/`. Neither agent may create, modify, rename, or delete any file in `md_docs/tester/active/`, `md_docs/tester/archive/`, `md_docs/*/active/`, `md_docs/*/archive/`, or any project source directory. The final consolidated test files in `md_docs/tester/active/` are written exclusively by the Tester after Fan-Out completes, both staging files are present, and both staging files pass validation. Any write to a shared directory during Fan-Out is a concurrency control violation and triggers an immediate abort.

**Rule 5 — Staging directory state is append-only during active Fan-Out.** While Fan-Out agents are executing, no process outside the two spawned agents may write to or delete from `md_docs/tester/staging/`. The Orchestrator reads from `staging/` for polling purposes only — it does not write to it, delete from it, or modify staging file content during polling. Deleting or overwriting a staging file while an agent is writing to it produces an undetectable partial write. If the Orchestrator aborts a Fan-Out, it signals the abort to the agents — it does not delete staging files directly. File deletion in `staging/` occurs only during the archive operation at pipeline completion or halt, after all agents have terminated.

**Rule 6 — Discard-on-abort is total, not selective.** When a Fan-Out abort is triggered — by a heartbeat timeout, a concurrency control violation, an Orchestrator signal, or any other cause — all staging file content written thus far must be discarded entirely. The Tester must not attempt to use, merge, or resume from partial staging output. A partially-written `SPEC_INDEPENDENT_TESTS.md` combined with a complete `UI_DEPENDENT_TESTS.md` produces an asymmetric test suite that will pass coverage metrics on only half the intended test targets. After a discard, the `staging/` directory contents are left for the Orchestrator to archive. The Tester proceeds to sequential test generation from Step 2 of its workflow without reading any staging content.

### Archive Feature Files (Orchestrator or Session-Recovery Invocation Only)

When the Orchestrator archives a completed or halted feature run, or when session-recovery archives stale and incomplete files, apply this exact operation:

1. **Collect all files** matching `md_docs/*/active/<FEATURE_NAME>_*.md`. In addition, collect all files matching `md_docs/tester/staging/<FEATURE_NAME>_*.md` — staging files are archived in the same batch as contract artifacts.
2. **Generate one UTC timestamp** in `YYYYMMDDHHMMSS` format for the entire archive batch. Using one timestamp for all files keeps the batch — including staging files — identifiable as a single archive operation.
3. **For each collected file**, move it from its current path to the corresponding `archive/` path with the timestamp appended before `.md`:
   - Contract artifacts: `md_docs/{agent}/active/<FEATURE_NAME>_<DOC_TYPE>.md` → `md_docs/{agent}/archive/<FEATURE_NAME>_<DOC_TYPE>_YYYYMMDDHHMMSS.md`
   - Staging artifacts: `md_docs/tester/staging/<FEATURE_NAME>_<DOC_TYPE>.md` → `md_docs/tester/archive/<FEATURE_NAME>_<DOC_TYPE>_YYYYMMDDHHMMSS.md`
4. **If a collision occurs** (a file already exists at the archive destination — rare but possible if two archive operations run within the same second), generate a new UTC timestamp and retry the entire batch with the new timestamp. Never overwrite an existing archive file.

**Archive scope for partial archiving (session-recovery):** When session-recovery archives individual stale or incomplete files rather than the full feature, apply the same timestamp format to each individual file. Generate one timestamp per archiving action, not per file. Staging files found in `md_docs/tester/staging/` during session-recovery are always treated as incomplete — archive them individually using the per-file timestamp rule and set the Tester re-entry point regardless of staging file content.

Individual agents must not call archive operations for their own files under any circumstances. Fan-Out agents (Agent A and Agent B) must not archive or delete any staging file — that responsibility belongs exclusively to the Orchestrator.
