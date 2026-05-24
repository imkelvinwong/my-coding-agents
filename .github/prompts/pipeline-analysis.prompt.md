---
name: pipeline-analysis
description: Instructs a dedicated general analysis agent to evaluate every agent, skill, and prompt file in the SDLC pipeline and produce a structured pros/cons report per file, a cross-pipeline gap analysis, and a prioritized Consolidated Action Summary with a Decision Matrix. Run with a general agent, not the Orchestrator. Supports two modes — analysis only (default) or analysis + Immediate fix application (refinement mode).
argument-hint: Analyze pipeline files for pros and cons (pass file paths or folders)
agent: agent
---

# Pipeline Analysis Prompt

## When to Use This Prompt

Run this prompt whenever you need a structured evaluation of the pipeline files. Common triggers:

- **After editing any agent or prompt file** — to catch regressions introduced by the change and verify the edit resolves the intended weakness without creating new ones.
- **Before starting a new project run** — to confirm the pipeline is sound before committing to a full delivery cycle.
- **When an agent produces unexpected output** — to determine whether the failure is a gap in the agent's own file or a gap in an upstream file that produces bad input.
- **On a regular improvement cadence** — after every 3–5 pipeline runs, to surface accumulated drift between files.

**Which agent to use:** Run this prompt with a **general Claude agent** — not the Orchestrator. Pipeline analysis is a meta-task: evaluating the pipeline, not running it. The Orchestrator's role is pipeline execution and delegation; using it here conflates its evaluation-subject role with its executor role. A general agent with `read` and `search` tool access is sufficient. No `agent` invocation tool is required because this prompt produces a report, not a pipeline run.

**Analysis mode vs. Refinement mode:**

- **Analysis mode (default):** The agent reads all files, applies the four evaluation dimensions, and produces the full structured report — per-file analyses, Cross-Pipeline Summary, and Consolidated Action Summary with Decision Matrix. No files are modified. The human reviews the Decision Matrix and decides which actions to execute, in what order, and with what scope.
- **Refinement mode:** After producing the full analysis report, the agent applies all `Immediate`-priority fixes from the Decision Matrix directly to the affected files. `Next revision` and `Backlog` items remain as the report's action list for human execution. To activate: set `mode: refinement` in the frontmatter, or append "apply Immediate fixes" to your invocation message.

**Why analysis-first is the default:** The Reviewer agent's philosophy applies here — inspect, classify, and decide are separate from fix. An agent that simultaneously finds weaknesses and rewrites files may resolve the symptom without diagnosing the root cause, introduce new issues not covered by the analysis, or make changes the human would have chosen to handle differently. The Decision Matrix is the handoff point between analysis and action, and the human is the decision-maker at that handoff.

---

## Purpose

This prompt instructs a general analysis agent to evaluate the agent files, skill files, and prompt files that comprise the SDLC pipeline and produce a structured pros and cons report for each one. The output is used to identify specific, actionable improvements before the next revision cycle. Every finding must be traceable to a specific mechanism in a specific file — general observations and vague praise are not acceptable outputs.

---

## Instructions

You are a senior software engineering consultant specializing in agentic system design. Your task is to read each file in the pipeline and produce a structured pros and cons analysis. You evaluate what works, what fails, and — critically — where files interact poorly with each other.

Apply all four evaluation dimensions to every file, regardless of file type. A skill file is evaluated with the same rigor as an agent file. A prompt file is evaluated the same way as either.

If `mode: refinement` is active (set in frontmatter or appended to the invocation), apply all `Immediate`-priority fixes from the Decision Matrix to the affected files after the analysis report is complete. Append a `## Refinement Actions Taken` section to the report listing every file changed and every change made. Do not apply `Next revision` or `Backlog` fixes.

---

## Evaluation Dimensions

### Dimension 1 — Clarity of Instruction

Assess whether the instructions in the file are precise enough for an LLM agent to follow without ambiguity. An instruction is ambiguous if a reasonable reader could follow it in two or more different ways and produce different outputs — both of which would seem correct to the agent.

Flag any instruction that:
- Could be interpreted in more than one reasonable way.
- Uses vague qualifiers such as "appropriate", "as needed", "where applicable", or "where possible" without defining what appropriate, needed, applicable, or possible means in that specific context.
- Depends on agent judgment for a decision that should be deterministic (e.g., "use your discretion to determine whether the file is complete" vs. "a file is complete if it contains all N required sections as listed in [reference]").

### Dimension 2 — Completeness of Coverage

Assess whether the file covers all scenarios the agent will encounter in practice. Identify gaps — situations the agent will face that the file provides no instruction for.

Distinguish between two gap types, because their impact differs:
- **Silent failure gaps**: The agent encounters an unaddressed scenario, makes a plausible guess, proceeds incorrectly, and the error propagates downstream before being detected. These are more dangerous because they are harder to catch.
- **Hard halt gaps**: The agent encounters an unaddressed scenario and cannot proceed at all. These are visible immediately and easier to diagnose, but they still block the pipeline.

### Dimension 3 — Boundary Enforcement

Assess whether the file clearly defines what the agent must NOT do, in addition to what it must do. An agent without clear boundaries will drift into adjacent responsibilities when its own scope is unclear, producing conflicts with other agents.

Identify:
- Any action this agent takes that another agent file also claims or implies responsibility for. This is a responsibility conflict — it means two agents may independently attempt the same action, producing duplicate or conflicting outputs.
- Any action this file permits that a different file in the pipeline prohibits.
- Any constraint in the Constraints section that contradicts an instruction in the Workflow section of the same file.

### Dimension 4 — Integration Fitness

Assess how well the file integrates with the other files in the pipeline, treating the pipeline as an interconnected system rather than a collection of independent agents.

Identify:
- Any input this agent expects that no other agent is guaranteed to produce. The agent will either fail to start (hard halt) or read from an absent or stale source (silent failure).
- Any output this agent produces that no other agent is guaranteed to consume. Uncollected output means work was done that has no downstream effect.
- Any assumption this file makes about another agent's behavior, output format, or file path that is not enforced in that other agent's file. Unenforced assumptions are the most common source of silent integration failures.

---

## Output Format

Produce the analysis in the following format for each file. Do not summarize across files within an individual analysis block. Each file receives its own complete and self-contained analysis.

```
FILE ANALYSIS
=============

File        : [filename including extension]
File Type   : [Agent / Skill / Prompt]
Version     : [version number if tracked, or "initial" if no versioning exists]

PROS
----
[Dimension label]: [Description of what the file does well on this dimension]

Each pro must reference the specific section, rule, mechanism, or instruction that
makes it effective. "Well-structured" is not a pro. "The retry cap in Step 3 prevents
infinite loops by halting after 3 attempts and triggering Orchestrator escalation" is a pro.

List each pro as a separate labeled entry.

CONS
----
[Dimension label]: [Description of the specific weakness]
Impact       : [Silent failure / Hard halt / Downstream error / Responsibility conflict]
Location     : [Section name, step number, or instruction that contains the gap — be specific enough that a reader can find it without searching]
Improvement  : [Specific, actionable change that would address this weakness — describe what the new instruction should say, not just what the problem is]

Each con must include all four fields. A con without a location cannot be fixed. A con
without an improvement action is an observation, not an analysis result.

If a file has zero cons on a given dimension, write: "[Dimension label]: No weaknesses identified on this dimension."
If a file has zero cons on all dimensions, write a single block: "No cons identified. This file's weaknesses may only be visible in cross-pipeline context — see Cross-Pipeline Summary."

PRIORITY IMPROVEMENTS
---------------------
List priority improvements in the following order:
1. ALL High-priority cons — list every High con regardless of count.
2. Up to 3 Medium-priority cons — select the 3 with the highest downstream impact.
3. Up to 3 Low-priority cons — select the 3 most actionable.

Priority is determined by the severity of downstream impact if left unaddressed.
Label each entry with its priority and the dimension it comes from.

Format:
1. [High/Medium/Low] — [Con description in one sentence] — [Dimension]
```

---

## Files to Analyze

Analyze the following files in the order listed. The order reflects pipeline execution sequence for agent files, followed by supporting files. Every file listed must be analyzed — do not skip a file because it appears simple or because earlier files cover similar content.

### Agent Files (in pipeline execution order)

1. **`orchestrator.agent.md`** — The pipeline controller. Evaluate how clearly it defines entry points, validation checkpoints, escalation routing, effort-aware execution gates, and the UI Scope Gate decision. Pay special attention to: whether the session-recovery prompt invocation trigger is unambiguous, whether the Non-UI Waiver authoring instructions are complete, and whether the Pipeline Execution Report template cross-references all downstream artifact paths correctly.

2. **`planner.agent.md`** — The architecture specification producer. Evaluate whether its 11-section output structure is fully defined, whether the Orchestrator Scheduling Note format is machine-readable as a string match, whether Section numbers are explicitly referenced in the output instructions (so Reviewer checklist cross-references work), and whether the handoff notification to the Orchestrator is unambiguous.

3. **`designer.agent.md`** — The design specification producer. Evaluate whether every section of the design document is actionable by the Developer, whether the Non-UI Waiver path (halt if the prompt is triggered from a Non-UI Waiver context) is handled, whether the structured handoff notification format is present and specific, and whether the fallback for a missing design system is adequately defined.

4. **`developer.agent.md`** — The implementation producer. Evaluate whether the Non-UI Waiver design document type is fully handled (which checklist items are skipped, how this is noted in the report), whether the output file path for the Completion Report is specified, whether the self-review checklist covers all specification dimensions, and whether the deviation documentation requirement is specific enough to be actionable.

5. **`reviewer.agent.md`** — The quality gate. Evaluate whether the Non-UI Waiver checklist path is complete and covers all four required waiver elements, whether the defect classification definitions are tight enough to prevent misclassification, whether the routing instructions correctly separate "Reviewer documents" from "Orchestrator routes", and whether the cycle tracking rules prevent infinite review loops.

6. **`builder.agent.md`** — The compilation and smoke test executor. Evaluate whether the output file path for the Completion Report is specified, whether the expected port determination rule covers all cases (config file, .env, architecture document, framework default), whether the error classification table covers all common build error types, and whether the smoke test checklist covers both frontend and backend-only application types.

7. **`tester.agent.md`** — The test suite producer and executor. Evaluate whether the Non-UI Waiver path correctly skips all UI-dependent test categories (component, accessibility, interaction), whether both output file paths are specified, whether the Pipeline Status field wording exactly matches what session-recovery expects as a string, and whether the coverage exclusion documentation requirement prevents gaming the thresholds.

### Skill Files

8. **`SKILL.md`** (`markdown-file-management`) — The path resolution and archiving skill. Evaluate whether the Agent Output Path Reference table covers every agent's deliverable, whether the archive operation instructions are complete enough to be executed without consulting another file, whether the prohibition on individual agents archiving their own files is unambiguous, and whether the legacy path deprecation notice is sufficient to prevent agents from writing to the wrong location.

### Prompt Files

9. **`pipeline-analysis.prompt.md`** (this file) — The pipeline evaluation prompt. Evaluate whether the `## When to Use This Prompt` section covers all realistic invocation triggers, whether the analysis-vs-refinement mode distinction is clear, whether the files-to-analyze descriptions are specific enough to guide evaluation focus, and whether the Consolidated Action Summary instructions guarantee that every con appears exactly once.

10. **`session-recovery.prompt.md`** — The pipeline recovery prompt. Evaluate whether the per-agent completeness criteria in Step 2 are specific enough to distinguish complete from incomplete without agent judgment, whether the staleness detection rules in Step 6 cover all cases where an upstream change invalidates a downstream artifact, whether the three recovery decision branches (Resume / Request Confirmation / Halt) cover all possible artifact states, and whether the absence of an `## Instructions` section (intentional — it runs within the Orchestrator context) is sufficiently explained.

### Processing Rule for Additional Files

If additional files are provided beyond those listed above, append them to the appropriate group (Agent / Skill / Prompt / Other) and analyze them in the order received. Apply all four dimensions regardless of file type. The format defined above applies to every file.

---

## Cross-Pipeline Summary

After all individual file analyses, produce a single cross-pipeline summary. This section is not optional. It captures failure modes that are invisible when reading any single file in isolation.

### Handoff Gaps

A handoff gap exists when what a producing agent guarantees to output does not fully satisfy what the consuming agent requires as input. List every agent-to-agent handoff and evaluate it:

```
Handoff      : [Producer agent] → [Consumer agent]
Produced     : [What the producer file explicitly guarantees to output — cite the specific section or instruction]
Expected     : [What the consumer file explicitly requires as input — cite the specific section or instruction]
Gap          : [Description of the mismatch — or "None" if the handoff is fully covered]
```

Evaluate these handoffs at minimum:
- Planner → Orchestrator (UI Scope Gate read)
- Orchestrator → Designer (Non-UI Waiver authoring)
- Designer → Orchestrator (handoff notification)
- Orchestrator → Developer (design document type communication)
- Developer → Reviewer (Completion Report and source files)
- Reviewer → Orchestrator (Review Decision routing instructions)
- Orchestrator → Builder (Approved decision signal)
- Builder → Tester (smoke test pass confirmation)
- Tester → Orchestrator (Pipeline Status field)
- Any agent → session-recovery.prompt.md (artifact completeness fields)

### Responsibility Conflicts

List any responsibility claimed or implied by more than one agent file. Conflicts are not always explicit — two agents instructed to "verify" the same artifact is a latent conflict, because both agents may perform the verification differently and produce contradictory conclusions.

For each conflict, state:
- Which agents claim the responsibility
- What the specific shared action is
- Which agent should own it and which should defer

### Undefined Paths

List every pipeline scenario that is not handled by any file — states the pipeline could reach where no agent has an instruction for what to do next. These are the most dangerous gaps because they produce behaviors that cannot be predicted or reproduced reliably.

Examples to evaluate:
- What happens if the Orchestrator receives a Non-UI Waiver from Designer that is later determined to be incorrect by the Reviewer?
- What happens if the Developer Completion Report is present but the source files are absent?
- What happens if the Builder produces a "Ready for Tester: No" and escalation to Developer fails three times?

### Missing File Types

Identify any skill file, prompt file, or agent file that would meaningfully improve the pipeline but does not currently exist. For each proposed addition, state:

- File type (Agent / Skill / Prompt)
- Proposed filename
- The specific pipeline gap it would address
- Which existing file's weakness it resolves

---

## Consolidated Action Summary

After the Cross-Pipeline Summary, produce the Consolidated Action Summary. This section is the decision-making reference. It aggregates every pro and con from every file into one flat, prioritized view so that improvements can be actioned without reading individual file blocks.

### All Pros

List every pro from every file analysis in a single table, sorted by dimension so that patterns across files are visible. Do not group by file.

```
ALL PROS
========

| File | Dimension | Pro (one-line summary referencing the specific mechanism) |
|------|-----------|-----------------------------------------------------------|
| [filename] | [Clarity / Completeness / Boundary / Integration] | [summary] |
```

### All Cons

List every con from every file analysis in a single table. Every con from every individual file analysis must appear exactly once — do not omit cons that appear minor in isolation, because patterns across files may elevate their combined priority.

Sort order: Priority (High first, then Medium, then Low). Within each priority group, sort by file in pipeline execution order.

```
ALL CONS
========

| Priority | File | Dimension | Con (one-line summary) | Impact Type | Improvement (one-line) |
|----------|------|-----------|------------------------|-------------|------------------------|
| [High/Medium/Low] | [filename] | [dimension] | [summary] | [Silent failure / Hard halt / Downstream error / Responsibility conflict] | [improvement] |
```

### Decision Matrix

After the tables, produce a Decision Matrix that groups cons by the action required to fix them. The goal is to batch related fixes across files into single improvement tasks — one action that touches three files is more efficient than three separate single-file actions.

```
DECISION MATRIX
===============

Action Required                  : [description of the fix action — specific enough to assign to a developer]
Files Affected                   : [list of files that need this change]
Cons Addressed                   : [list of con one-liners resolved by this action]
Estimated Impact if Unaddressed  : [Silent failure / Hard halt / Downstream error / Responsibility conflict]
Recommended Fix Cycle            : [Immediate / Next revision / Backlog]

[Repeat for each distinct action]
```

Recommended Fix Cycle definitions — apply these consistently:
- **Immediate** — The con causes pipeline failure, data loss, or incorrect routing in normal operation. Fix before the next pipeline run. Examples: missing output file path, missing Non-UI Waiver checklist path, missing escalation rule.
- **Next revision** — The con causes incorrect behavior in edge cases or degrades output quality. Fix in the next agent file revision cycle. Examples: ambiguous retry cap definition, missing port determination fallback, incomplete async state coverage guidance.
- **Backlog** — The con is a quality or clarity improvement with no immediate failure risk. Address when bandwidth allows. Examples: enriching JSDoc guidance, adding example outputs to a section, clarifying vague transition language.

---

## Analyst Behavior Rules

These rules govern how you write the analysis. They are not guidelines — they are constraints. Violating any rule produces an analysis output that cannot be acted on.

- **Every pro must reference a specific mechanism.** "Well-structured" is not a pro. Name the section, the rule, or the instruction that makes it effective and explain why it works.
- **Every con must have a specific location.** "The constraints section is weak" is not a location. "Constraints section, third bullet" is a location. "Step 3 — Error Analysis, Resolution Rules, Rule 2" is a location.
- **Every con must have a specific improvement action.** "Improve this section" is not an improvement. "Add an instruction stating: [exact instruction text]" is an improvement.
- **Attribute weaknesses to their source, not their symptom.** If a weakness in the Reviewer file is caused by a gap in the Developer file (e.g., the Developer does not guarantee the Completion Report format the Reviewer expects), the con belongs to the Developer file, not the Reviewer file.
- **Do not propose improvements that contradict the pipeline's design intent.** The pipeline is a sequential, agent-delegated SDLC with fixed ordering and formal handoffs. Improvements must work within that design — they must not suggest skipping agents, merging roles, or removing checkpoints.
- **Do not skip the Cross-Pipeline Summary or the Consolidated Action Summary.** Both are required outputs. An analysis that ends after the individual file blocks is incomplete.
- **Every con from every individual file analysis must appear exactly once in the All Cons table.** The Consolidated Action Summary is the primary decision-making reference. Omitting a con from it — even one that appears minor — defeats the purpose of the summary.
- **If a file has more than 3 High-priority cons, list all of them in the Priority Improvements section.** The cap of 3 applies only to Medium and Low priority items. No High-priority con may be omitted.
