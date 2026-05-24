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

**What this skill does not do:** It does not define SDLC semantics, ownership rules, or pipeline policy. Those decisions live in each agent's prompt. This skill is the mechanical execution layer — it answers "where does this file go?" and "how does archiving work?", not "should this file be archived?"

**When to consult this skill:** Any agent whose prompt does not specify an explicit output file path must resolve the path here before writing. Session-recovery uses this skill for all archive operations. Agents must not invent paths or use legacy paths not listed in this skill.

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
        archive/
    orchestrator/
        active/
            <FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md
        archive/
```

**Note on the designer directory:** The `<FEATURE_NAME>_DESIGN.md` file is written either by the Designer agent (full specification) or by the Orchestrator (Non-UI Waiver). Both cases use the same filename and directory. The Reviewer determines which type it is by reading the document header.

**Note on the reviewer directory:** Review cycle files include a numeric suffix: `<FEATURE_NAME>_REVIEW_CYCLE_1.md`, `<FEATURE_NAME>_REVIEW_CYCLE_2.md`, etc. Always use the highest N present as the current decision. Prior cycle files remain in `active/` until the pipeline completes or halts — they are historical context, not stale contracts.

---

## Agent Output Path Reference

Use this table to resolve the output path for each agent's deliverable. Agents must write to exactly these paths. Session-recovery uses exactly these paths to locate artifacts during reconstruction.

| Agent | Output File | Path |
|---|---|---|
| Planner | Architecture document | `md_docs/planner/active/<FEATURE_NAME>_ARCHITECTURE.md` |
| Designer | Design specification or Non-UI Waiver | `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md` |
| Orchestrator | Non-UI Waiver (when UI_NOT_REQUIRED) | `md_docs/designer/active/<FEATURE_NAME>_DESIGN.md` |
| Developer | Developer Completion Report | `md_docs/developer/active/<FEATURE_NAME>_DEVELOPER_COMPLETION.md` |
| Reviewer | Review Decision (per cycle) | `md_docs/reviewer/active/<FEATURE_NAME>_REVIEW_CYCLE_N.md` |
| Builder | Builder Completion Report | `md_docs/builder/active/<FEATURE_NAME>_BUILDER_COMPLETION.md` |
| Tester | Test Plan | `md_docs/tester/active/<FEATURE_NAME>_TEST_PLAN.md` |
| Tester | Test Completion Report | `md_docs/tester/active/<FEATURE_NAME>_TEST_COMPLETION.md` |
| Orchestrator | Pipeline Execution Report | `md_docs/orchestrator/active/<FEATURE_NAME>_PIPELINE_EXECUTION_REPORT.md` |

---

## Core Rules

1. **All contract filenames must follow the `<FEATURE_NAME>_<DOC_TYPE>.md` format.** Examples: `UserAuth_ARCHITECTURE.md`, `UserAuth_DESIGN.md`, `UserAuth_PIPELINE_EXECUTION_REPORT.md`. Review cycle files include a numeric suffix: `UserAuth_REVIEW_CYCLE_1.md`. Deviating from this format breaks session-recovery's artifact inventory scan.

2. **Files in `active/` are the current contract for a feature.** Any agent reading from `active/` is reading the most recent valid version of that contract. No agent should hold a cached copy of a contract from a prior read — always read from `active/` at the time of use.

3. **Agents must read from `active/` and must never read from `archive/`.** Archive files are historical records. They may represent superseded, rejected, or partial-run specifications. An agent that reads an archived file may implement against a contract that has since been corrected.

4. **Only the Orchestrator initiates archive operations.** Individual agents must not call archive operations for their own output files or for any other agent's files. An agent that archives its own file mid-pipeline breaks session-recovery's ability to reconstruct state. Archive operations occur only at pipeline completion or halt, and only when invoked by the Orchestrator or by session-recovery under Orchestrator direction.

5. **Legacy paths are not valid.** The paths `.architecture/` and `.design/` are superseded by `md_docs/planner/active/` and `md_docs/designer/active/` respectively. Any file found at a legacy path during recovery should be treated as an orphan — do not read it as a valid contract.

---

## Mechanical Operations

### Resolve Paths

Given an agent name and the expected document type, resolve the output file path using the Agent Output Path Reference table above. If the document type is `REVIEW_CYCLE`, append the current cycle number N.

Format: `md_docs/{agent}/active/<FEATURE_NAME>_<DOC_TYPE>.md`

Example resolutions:
- Planner writing architecture for `UserAuth` → `md_docs/planner/active/UserAuth_ARCHITECTURE.md`
- Reviewer writing cycle 2 decision for `UserAuth` → `md_docs/reviewer/active/UserAuth_REVIEW_CYCLE_2.md`
- Tester writing test plan for `DarkModeToggle` → `md_docs/tester/active/DarkModeToggle_TEST_PLAN.md`

### Ensure Directories

Before any write operation, confirm that both the `active/` and `archive/` directories exist for the agent writing the file. Create them if they do not exist. A write to a non-existent directory will fail — creating the directory first prevents this:

```
md_docs/{agent}/active/     ← must exist before writing
md_docs/{agent}/archive/    ← must exist before archiving
```

### Archive Feature Files (Orchestrator or Session-Recovery Invocation Only)

When the Orchestrator archives a completed or halted feature run, or when session-recovery archives stale and incomplete files, apply this exact operation:

1. **Collect all files** matching `md_docs/*/active/<FEATURE_NAME>_*.md`.
2. **Generate one UTC timestamp** in `YYYYMMDDHHMMSS` format for the entire archive batch. Using one timestamp for all files keeps the batch identifiable as a single archive operation.
3. **For each collected file**, move it from its `active/` path to the corresponding `archive/` path with the timestamp appended before `.md`:
   - `md_docs/{agent}/active/<FEATURE_NAME>_<DOC_TYPE>.md`
   - → `md_docs/{agent}/archive/<FEATURE_NAME>_<DOC_TYPE>_YYYYMMDDHHMMSS.md`
4. **If a collision occurs** (a file already exists at the archive destination — rare but possible if two archive operations run within the same second), generate a new UTC timestamp and retry the entire batch with the new timestamp. Never overwrite an existing archive file.

**Archive scope for partial archiving (session-recovery):** When session-recovery archives individual stale or incomplete files rather than the full feature, apply the same timestamp format to each individual file. Generate one timestamp per archiving action, not per file.

Individual agents must not call archive operations for their own files under any circumstances.
