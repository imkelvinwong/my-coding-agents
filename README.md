# SDLC Agent Pipeline

A structured, multi-agent software development lifecycle (SDLC) pipeline. It drives a feature request from architecture through testing using formal handoffs, quality gates, machine-readable outcome contracts, and Session Recovery at every phase.

---

## What This Is

The pipeline orchestrates eight specialized AI agents through the complete software delivery process. Each agent executes a defined role, produces a specific artifact, and initiates handoffs via the **Orchestrator**. Agents cannot skip phases. Artifacts are mandatory. Phases require successful upstream validation checkpoints to proceed.

The architecture enforces continuous, compounding improvement. Preceding artifacts establish strict machine-readable outcome contracts for subsequent agents. The system classifies every failure by root cause and routes it to the responsible agent for resolution, strictly prohibiting ad-hoc downstream patching.

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
 Orchestrator   ──── Pre-Execution Planning
      │               • Scan md_docs/*/active/ for prior artifacts (3-branch logic)
      │               • Invoke Session Recovery if prior artifacts exist
      │               • Map agent sequence and artifact dependencies
      │               • Record effort gate thresholds
      ▼
   Planner      ──── Produces ARCHITECTURE.md (11 sections)
      │               • Fenced JSON scheduling payload in Section 10
      │               • Orchestrator parses payload programmatically
      ▼
 UI Scope Gate  ──── Orchestrator decision (evidence-based, not name-based)
      │               • UI_REQUIRED  → invoke Designer
      │               • UI_NOT_REQUIRED → Orchestrator writes Non-UI Waiver
      ▼
  Designer      ──── Produces DESIGN.md (10 sections + Section 8.1)
 (or Orchestrator     • Component specs, async states, streaming states, accessibility
  writes waiver)      • Structured handoff notification to Orchestrator
      ▼
  Researcher    ──── Produces TECHNICAL_VERIFICATION_REPORT.md
 (skipped if no       • Validates tensor shapes, VRAM budget, framework API compatibility
  ML components)      • Verified PyTorch/TensorFlow modules + Pydantic v2 I/O schemas
      ▼
  Developer     ──── Produces source code + DEVELOPER_COMPLETION.md
      │               • Implements all architecture + design contracts
      │               • Logs specification gaps via Specification Gap Protocol
      │               • Executes self-review checklist
      ▼
  Reviewer      ──── Produces REVIEW_CYCLE_N.md
      │               • decision_code: APPROVED → proceed to Builder
      │               • decision_code: REJECTED → defect report → route by root cause
      ▼
   Builder      ──── Produces BUILDER_COMPLETION.md
      │               • Compiles, starts dev server, runs smoke test
      │               • "Ready for Tester: Yes" required to proceed
      ▼
   Tester       ──── Produces TEST_PLAN.md + TEST_COMPLETION.md + LLM_EVAL_RESULTS.md
      │               • Derives test plan first; counts total targets
      │               • < 12 targets  → sequential execution
      │               • ≥ 12 targets  → Parallel Fan-Out (Agent A + Agent B + Agent C)
      │               • Agent A: Spec-Independent tests (always)
      │               • Agent B: UI-Dependent tests (skipped if Non-UI Waiver exists)
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
    researcher/active/     <FEATURE_NAME>_TECHNICAL_VERIFICATION_REPORT.md (if ML components present)
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
The **Planner** emits a strictly structured JSON block inside Section 10 of the architecture document. The **Orchestrator** extracts and parses this block programmatically to enforce effort-aware operational modes based on `total_story_points`:
* **≤ 5 story points**: Single-run execution; runs the full pipeline without pausing.
* **6–13 story points**: Milestone confirmation required; pauses after **Planner** and **Designer** to present the phase breakdown.
* **≥ 14 story points**: Phased approval required; hard halts after **Planner** and **Designer** for explicit written user approval.

### Machine-Readable Outcome Contracts
Phase-completion routing decisions strictly evaluate dedicated machine-readable outcome contracts, ignoring surrounding human prose entirely.
* **Reviewer Phase**: Evaluates code quality and outputs `decision_code` (`APPROVED` or `REJECTED`) within `REVIEW_CYCLE_N.md`.
* **Tester Phase**: Executes the test suites and outputs `status_code` (`PASSING` / `BLOCKED`) within `TEST_COMPLETION.md`.
* **LLM Eval Gate**: For features with LLM inference components, the **Tester** additionally outputs `eval_status` (`PASS` / `FAIL`) within `LLM_EVAL_RESULTS.md`. A `FAIL` result blocks pipeline completion regardless of `status_code` and routes back to the **Developer** with the specific `eval_score` and `eval_threshold` values.

### UI Scope Gate
The **Orchestrator** classifies features as `UI_REQUIRED` or `UI_NOT_REQUIRED` based on architectural evidence. When `UI_NOT_REQUIRED`, the **Orchestrator** writes a formal 4-element Non-UI Waiver to the design track. This waiver triggers the **Reviewer** and **Tester** to automatically bypass UI-conformance paths and aborts the spawning of Fan-Out Agent B.

### Streaming Data State Designs (Section 8.1)
The **Designer** specification includes Section 8.1 (Streaming Data State Designs) for components rendering streaming data (token-by-token LLM output, server-sent events, or WebSocket-pushed content). Section 8.1 defines four mandatory streaming states: **Connecting**, **Streaming**, **Interrupted**, and **Complete**. Section 9 (Accessibility) maps `aria-busy` state transitions, announcement batching strategies for high-frequency updates, and keyboard-accessible interruption affordances. Section 10 (Design System Consistency Checklist) enforces the completeness of Section 8.1 entries prior to handoff.

### Parallel Fan-Out (Tester Phase)
When a derived test plan dictates 12 or more distinct test targets, the **Tester** activates Parallel Fan-Out mode instead of sequential execution. Up to three parallel agents run under strict filesystem-level I/O isolation:
* **Agent A**: Generates Spec-Independent tests (unit, integration, edge cases). *Always spawned.*
* **Agent B**: Generates UI-Dependent tests (component, interaction, accessibility). *Skipped if the design document is a Non-UI Waiver.*
* **Agent C**: Generates LLM Eval tests (prompt-response scoring, threshold-gated assertions, regression tests). *Skipped if the feature contains no LLM inference components.*

A Heartbeat Monitor safety protocol validates the initialization of every spawned agent before output polling begins. A missing start signal triggers an immediate abort and fallback to sequential execution. Agent A and Agent B evaluate against a fixed 10-second window. Agent C evaluates against a dynamic window derived from `max_inference_latency_ms` in architecture Section 8, utilizing the formula `max(10, ceil(max_inference_latency_ms / 1000) × 3)` (defaulting to 90 seconds if absent). This prevents false aborts on inference-heavy evaluation suites.

### Defect Classification & Specification Gaps
The **Reviewer** classifies defects by root cause to dictate routing: **Class A** (Developer Error; routes back to **Developer**), **Class B** (Specification Ambiguity; routes back to **Planner** or **Designer**), or **Class C** (Systemic Pattern; triggers an immediate global pipeline halt). To establish a Class B classification, Developers must preemptively log ambiguities in their completion report using the Specification Gap Protocol (defined in Conventions).

### Escalation Caps
Self-remediating agents (**Builder** and **Tester**) enforce a strict 3-attempt retry cap per error classification. If remediation fails on the third attempt, the agent writes a detailed escalation report, notifies the **Orchestrator**, and globally halts the pipeline to prevent infinite loops.

### Session Recovery
If an interruption halts a session mid-pipeline, `session-recovery.prompt.md` executes a 10-step state reconstruction workflow. It cleans up orphaned staging files, inventories complete contract artifacts via field-presence criteria (including `eval_status` in `LLM_EVAL_RESULTS.md`), identifies stale upstream contracts via timestamp analysis, re-verifies effort gates, and resumes from the earliest valid re-entry point.

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

* **`<FEATURE_NAME>`**: Enforces PascalCase, utilizing no spaces or special characters. It is globally broadcast by the **Orchestrator** and never altered.
* **Active vs. Archive**: Active contracts reside exclusively in `active/`. Only the **Orchestrator** moves files to `archive/` upon completion or halt, appending a synchronized UTC timestamp (`<FEATURE_NAME>_<DOC_TYPE>_YYYYMMDDHHMMSS.md`).
* **Section References**: Architecture document sections number 1–11. Design document sections number 1–10 plus Section 8.1 (Streaming Data State Designs). **Reviewer** defect reports mandate citations to these specific sections for exact traceability.
* **Outcome Fields**: `decision_code` (`APPROVED` / `REJECTED`) and `status_code` (`PASSING` / `BLOCKED`) must enforce uppercase strings and accompany detailed requirement blocks to validate as complete. For LLM-bearing features, `eval_status` (`PASS` / `FAIL`) dictates an additional outcome gate evaluated prior to pipeline completion.
* **Non-UI Waiver**: Must satisfy all four elements of the canonical Non-UI Waiver schema defined in `SKILL.md`. Agents author, validate, and consume the waiver strictly against that central definition.
* **Specification Gap Protocol**: Developers must document gaps utilizing exactly five snake_case keys (`gap_document`, `specification_gap_description`, `chosen_interpretation`, `rejected_alternatives`, and `risk_assessment`) to avoid rejection as a distinct minor defect.

---

## Local Development Workflow

GitHub Copilot (custom agents, usable in both VS Code and IntelliJ) and Claude Code (sub-agents) are parallel implementations of the same eight-agent SDLC pipeline. Every role definition, workflow step, artifact contract, escalation rule, and constraint in each agent's system prompt is identical across both environments. The implementations differ only in frontmatter syntax, file locations, and tool-name conventions. Body content — the system prompt below each frontmatter block — is always copied verbatim between formats.

### Agent Format Comparison

| Dimension | GitHub Copilot | Claude Code |
| --- | --- | --- |
| File location | Project root — `orchestrator.agent.md` | `.claude/agents/orchestrator.md` |
| `name` value | PascalCase — `Orchestrator` | Lowercase kebab — `orchestrator` |
| Tool list format | Python-list — `['read', 'agent', 'search']` | Comma-separated string — `Read, Grep, Glob` |
| Sub-agent delegation | Separate `agents: ['Planner', 'Designer', ...]` field | `Agent(planner, designer, ...)` inside `tools:` |
| GitHub tools | Inline in `tools:` — `github/issue_read` | `mcpServers: [github]` block in frontmatter |
| Context7 MCP | Inline in `tools:` — `io.github.upstash/context7/*` | `mcpServers: [context7]` block in frontmatter |
| `execute` tool | `'execute'` in Python-list | `Bash` in comma-separated string |
| Per-agent model | Not configurable | `model: sonnet \| opus \| haiku \| inherit` |
| Preloaded context | Workspace-level `.github/copilot-instructions.md` | Per-agent `skills: [markdown-file-management]` |
| Session start | Copilot Chat — `@orchestrator` | Terminal — `claude --agent orchestrator` |
| Body content | **Identical across both implementations** | **Identical across both implementations** |

### Configuring GitHub Copilot

GitHub Copilot reads `.github/copilot-instructions.md` as workspace context before every Copilot Chat session. This is the functional equivalent of Claude Code's `skills:` frontmatter field — it loads the pipeline's contracts and conventions into Copilot's working context without requiring them to be embedded in each agent file individually. Create the file at the repository root with the following directives, sourced directly from the agent files:

```markdown
This is a multi-agent SDLC pipeline. Each `.agent.md` file is a strict contract with downstream
consumers. Edits require cross-pipeline impact verification before committing.

## File-Naming Convention
Contract artifacts follow `<FEATURE_NAME>_<DOC_TYPE>.md`. `<FEATURE_NAME>` is PascalCase and
immutable within a run. Read and write exclusively to `md_docs/*/active/`. Never reference or
suggest reading from `md_docs/*/archive/`. Only the Orchestrator may move files to `archive/`.

## Machine-Readable Outcome Fields
These fields are evaluated by exact uppercase string match. Paraphrases and lowercase variants
break pipeline routing and are invalid in all contexts.
- `decision_code`: `APPROVED` or `REJECTED` — written by Reviewer to `REVIEW_CYCLE_N.md`
- `status_code`: `PASSING` or `BLOCKED` — written by Tester to `TEST_COMPLETION.md`
- `eval_status`: `PASS` or `FAIL` — written by Tester to `LLM_EVAL_RESULTS.md` (LLM features only)

## Specification Gap Protocol
Gaps must use exactly five snake_case keys: `gap_document`, `specification_gap_description`,
`chosen_interpretation`, `rejected_alternatives`, `risk_assessment`. Non-standard key names or
aliases cause Class B gaps to be reclassified as Class A defects.

## Agent Tool Boundaries
- Orchestrator: the only agent holding `agent`; delegates to all other pipeline agents.
- Reviewer: holds `github/get_file_contents`, `github/list_commits`, `github/search_code`,
  `github/issue_read`, `github/issue_write`, `github/pull_request_read` for PR-aware review.
- Builder and Tester: hold `execute` — all build, run, and test operations are terminal tasks.
- Designer, Planner, Developer, Researcher: hold `read/search/edit/web/todo` only.
```

### Configuring GitHub Copilot in IntelliJ

IntelliJ's GitHub Copilot plugin uses a different tool vocabulary than VS Code, and does not natively expose `web` or `browser` tools. The `web` gap is covered by the agents' training knowledge and context7 (which handles all library-specific documentation lookups). The `browser` gap is resolved by adding the Playwright MCP, which Builder and Tester require for smoke testing and component verification. No web search API key is needed. Agent body content, pipeline ordering, and all artifact contracts remain identical across VS Code and IntelliJ — only the `tools:` list values differ.

#### VS Code → IntelliJ Tool Name Mapping

| VS Code Tool | IntelliJ Equivalent | Resolved by |
|---|---|---|
| `read` | `read_file`, `open_file`, `list_dir` | Built-in (split into three tools) |
| `search` | `file_search`, `grep_search`, `semantic_search` | Built-in (split into three tools) |
| `edit` | `create_file`, `insert_edit_into_file`, `replace_string_in_file` | Built-in (split into three tools) |
| `execute` | `run_in_terminal`, `get_terminal_output` | Built-in (split into two tools) |
| `agent` | `run_subagent` | Built-in (direct rename) |
| `todo` | `ask_questions` | Built-in |
| `web` | *(no direct mapping)* | Training knowledge + context7 for library docs |
| `browser` | `io.github.microsoft/playwright-mcp/*` | Playwright MCP |
| — | `apply_patch` | IntelliJ addition — code patching |
| — | `get_errors` | IntelliJ addition — error introspection |
| — | `validate_cves` | IntelliJ addition — dependency security validation |

#### Agent Tool Lists

The eight agents use the following `tools:` values in pipeline execution order:

| Agent | IntelliJ `tools:` |
|---|---|
| Orchestrator | `['read_file', 'open_file', 'list_dir', 'file_search', 'grep_search', 'semantic_search', 'create_file', 'insert_edit_into_file', 'replace_string_in_file', 'run_subagent', 'ask_questions', 'io.github.github/github-mcp-server/issue_read', 'io.github.github/github-mcp-server/issue_write', 'io.github.github/github-mcp-server/pull_request_read']` |
| Planner | `['read_file', 'open_file', 'list_dir', 'file_search', 'grep_search', 'semantic_search', 'create_file', 'insert_edit_into_file', 'replace_string_in_file', 'ask_questions', 'io.github.upstash/context7/*']` |
| Designer | `['read_file', 'open_file', 'list_dir', 'file_search', 'grep_search', 'semantic_search', 'create_file', 'insert_edit_into_file', 'replace_string_in_file', 'ask_questions', 'io.github.upstash/context7/*']` |
| Researcher | `['read_file', 'open_file', 'list_dir', 'file_search', 'grep_search', 'semantic_search', 'create_file', 'insert_edit_into_file', 'replace_string_in_file', 'apply_patch', 'get_errors', 'validate_cves', 'ask_questions', 'io.github.upstash/context7/*']` |
| Developer | `['read_file', 'open_file', 'list_dir', 'file_search', 'grep_search', 'semantic_search', 'create_file', 'insert_edit_into_file', 'replace_string_in_file', 'apply_patch', 'get_errors', 'validate_cves', 'ask_questions', 'io.github.upstash/context7/*']` |
| Reviewer | `['read_file', 'open_file', 'list_dir', 'file_search', 'grep_search', 'semantic_search', 'create_file', 'insert_edit_into_file', 'replace_string_in_file', 'validate_cves', 'ask_questions', 'io.github.upstash/context7/*', 'io.github.github/github-mcp-server/get_file_contents', 'io.github.github/github-mcp-server/list_commits', 'io.github.github/github-mcp-server/search_code', 'io.github.github/github-mcp-server/issue_read', 'io.github.github/github-mcp-server/issue_write', 'io.github.github/github-mcp-server/pull_request_read']` |
| Builder | `['read_file', 'open_file', 'list_dir', 'file_search', 'grep_search', 'semantic_search', 'create_file', 'insert_edit_into_file', 'replace_string_in_file', 'apply_patch', 'run_in_terminal', 'get_terminal_output', 'get_errors', 'ask_questions', 'io.github.microsoft/playwright-mcp/*']` |
| Tester | `['read_file', 'open_file', 'list_dir', 'file_search', 'grep_search', 'semantic_search', 'create_file', 'insert_edit_into_file', 'replace_string_in_file', 'apply_patch', 'run_in_terminal', 'get_terminal_output', 'get_errors', 'run_subagent', 'ask_questions', 'io.github.microsoft/playwright-mcp/*']` |

> [!IMPORTANT]
> **Note on IntelliJ Wildcard Limitations:** Replace `/*` with the explicit tool configurations of that MCP server. Because wildcard annotations fail in IntelliJ, explicitly defining or adding each tool is the cleanest required workaround.

### Engineering Rules for Editing Pipeline Files

Every agent file acts as a strict contract consumed by downstream agents. Editing a pipeline file without verifying downstream consumer impact introduces systemic defects equivalent to implementing code against a stale specification.

* **Classify the gap before opening any file.** Before modifying an agent file to address an observed failure, classify the gap against the four Pipeline Analysis dimensions. A Dimension 1 gap (Clarity) dictates tightening instructions. A Dimension 2 gap (Completeness) dictates defining missing scenarios. A Dimension 3 gap (Boundary Enforcement) dictates auditing responsibility conflicts across agents. A Dimension 4 gap (Integration Fitness) requires auditing both producing and consuming agents before modifying either.
* **Route to the root, not the symptom.** The agent discovering a failure is rarely responsible for the fix. If the **Tester** produces unexpected output, the root cause frequently exists within the **Planner's** architecture template, not `tester.agent.md`. Correct the upstream contract; do not implement downstream symptom patches.
* **Plan the edit prior to execution.** Before modifying an agent file, strictly identify: the downstream agents consuming the modified section, the artifact fields dependent upon it, and the potential impact on cross-pipeline handoffs. This mirrors the **Orchestrator's** mandate to construct a Pipeline Execution Plan prior to agent invocation.
* **Machine-readable outcome contracts are structural, not prose.** `decision_code`, `status_code`, and `eval_status` are evaluated by exact uppercase string matching. Do not alter their valid states (defined in Conventions) in any file without synchronously updating `orchestrator.agent.md`, `reviewer.agent.md`, and `session-recovery.prompt.md`. This structural constraint applies equally to the Specification Gap Protocol's five keys — the **Reviewer's** Class B verification protocol relies on exact snake_case strings.
* **Apply the 3-attempt retry cap to edit cycles.** If three manual edit iterations fail to resolve an agent file gap, halt manual intervention and execute Pipeline Analysis Refinement mode. Three failed manual attempts indicate a systemic cross-pipeline integration flaw, mirroring the threshold that triggers **Builder** and **Tester** escalations.
* **Inspect and classify before applying fixes.** Pipeline Analysis executes in Analysis-only mode by default to separate identification from resolution (analogous to the **Reviewer's** restriction against fixing code). Execute the Analysis pass, evaluate the Decision Matrix, and apply only `Immediate`-priority fixes. Schedule `Next revision` and `Backlog` items for formal revision cycles.

### When to Run Pipeline Analysis

| Trigger | Mode |
| --- | --- |
| Following modifications to any agent, skill, or prompt file | Analysis — Verifies the edit resolves the targeted gap without introducing regressions |
| Prior to initiating a new project run | Analysis — Confirms pipeline integrity before committing to a full delivery cycle |
| When any agent generates unexpected output | Analysis — Determines whether the failure isolates to the local agent or originates from stale upstream contracts |
| After every 3–5 pipeline runs | Analysis — Surfaces accumulated technical drift before it manifests as a systemic pattern |
| Upon identifying a Dimension 3 or Dimension 4 gap | Refinement — Applies `Immediate`-priority structural fixes across all affected files in a single pass |

### Guardrails

These non-negotiable constraints apply regardless of the active development tool:

* **Never read from `md_docs/*/archive/`.** Archived files function strictly as historical records of completed or superseded runs. Every agent reads exclusively from `active/`. Referencing archived artifacts during active development introduces obsolete specifications.
* **Only the Orchestrator archives files.** Agents never archive their own outputs, and manual development actions must never move files to `archive/` mid-pipeline. Premature archiving destroys the contract map required by Session Recovery.
* **Execute Pipeline Analysis before committing changes.** Single-file edits frequently sever cross-pipeline integrations. Analysis exposes these integration fractures prior to committing the file.
* **Write the diagnosis before escalating.** Both the **Builder** and **Tester** mandate a comprehensive escalation report prior to alerting the **Orchestrator**. Apply the identical standard during development: document the root cause fully before initiating a review or escalating a pipeline failure. Undocumented escalations force diagnostic resets.
* **Session interruption is a recoverable state, not data loss.** The **Orchestrator's** three-branch logic and the 10-step Session Recovery workflow ensure phase preservation. Never restart a pipeline from the **Planner** when `md_docs/*/active/` artifacts exist. Invoke the **Orchestrator** to trigger automated Session Recovery.
* **`<FEATURE_NAME>` remains immutable within a run.** Manual file creation or renaming within `md_docs/` must strictly preserve the `<FEATURE_NAME>_<DOC_TYPE>.md` format. Session Recovery relies entirely on this structural pattern; deviations render the artifact an untrackable orphan.

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