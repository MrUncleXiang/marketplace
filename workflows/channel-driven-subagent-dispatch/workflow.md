# Channel-Driven Sub-Agent Dispatch Workflow

---

## Core Principles

1. **Plan before code** — define the task, planning artifacts, and acceptance criteria before implementation.
2. **The main session coordinates** — the main session clarifies requirements, plans the task, dispatches workers, updates specs, commits, and finishes the work.
3. **Implementation and checking run in channel workers** — use `trellis channel spawn` for implement/check workers by default instead of host-native sub-agents.
4. **Pass context explicitly** — worker context order is `jsonl entries -> prd.md -> design.md -> implement.md`.
5. **Keep results auditable** — use `trellis channel messages --raw` for worker events; pretty output is an operator dashboard and may truncate progress.
6. **Persist decisions** — requirements, research, plans, and review conclusions belong in task files.

---

## Trellis System

### Developer Identity

Initialize your identity on first use:

```bash
python3 ./.trellis/scripts/init_developer.py <your-name>
```

### Spec System

`.trellis/spec/` stores project engineering guidelines. Before writing code, load the package/layer specs relevant to the task:

```bash
python3 ./.trellis/scripts/get_context.py --mode packages
```

### Task System

Each task has its own directory under `.trellis/tasks/{MM-DD-name}/` with `task.json`, `prd.md`, `design.md`, `implement.md`, optional `research/`, and `implement.jsonl` / `check.jsonl`.

Common commands:

```bash
python3 ./.trellis/scripts/task.py create "<title>" [--slug <name>] [--parent <dir>]
python3 ./.trellis/scripts/task.py start <name>
python3 ./.trellis/scripts/task.py current --source
python3 ./.trellis/scripts/task.py finish
python3 ./.trellis/scripts/task.py archive <name>
python3 ./.trellis/scripts/task.py validate <name>
```

### Channel System

Channels are the worker collaboration and event-audit layer. Use `--ephemeral` for temporary implementation/check channels. Use `--type forum` for durable discussion boards; a `thread` is an item inside a forum.

Stable worker handles:

- `implement` — implementation worker
- `check` — default check worker
- `check-cc` — Claude check worker
- `check-cx` — Codex check worker

---

<!--
  WORKFLOW-STATE BREADCRUMB CONTRACT

  [workflow-state:STATUS] blocks are the single source for per-turn prompt injection.
  Do not delete tags or change tag syntax. The body can change; parsers should not.
-->

## Phase Index

```
Phase 1: Plan    -> classify, get task-creation consent, then write planning artifacts
Phase 2: Execute -> implement/check through trellis channel workers
Phase 3: Finish  -> verify, update spec, commit, and wrap up
```

### Request Triage

- Simple conversation or small task: ask only whether this turn should create a Trellis task. If the user says no, skip Trellis for this session.
- Complex task: ask whether you may create a Trellis task and enter planning. If the user says no, do not do broad inline implementation.
- User approval to create a task is not approval to start implementation. Implementation waits until artifacts are reviewed and `task.py start` has run.

### Planning Artifacts

- `prd.md` — requirements, constraints, and acceptance criteria.
- `design.md` — technical design for every new task.
- `implement.md` — execution plan, validation commands, review gates, and rollback points for every new task.
- `implement.jsonl` / `check.jsonl` — worker context manifests. Put spec and research files here, not code files.

Every new task must have concise `prd.md`, `design.md`, and `implement.md` before first `task.py start`; no fixed sections or word counts are required.

### Parent / Child Task Trees

Use a parent task when one request contains several independently verifiable deliverables. Child tasks own deliverables that can be planned, implemented, checked, and archived independently. Parent/child structure is not a dependency system; dependencies must be written in the child `prd.md` / `implement.md`.

[workflow-state:no_task]
No active task. First classify the current turn and ask for task-creation consent before creating any Trellis task.
Simple conversation / small task: ask only whether this turn should create a Trellis task. If the user says no, skip Trellis for this session.
Complex task: ask the user if you can create a Trellis task and enter the planning phase. If the user says no, explain, clarify scope, or suggest a smaller split.
[/workflow-state:no_task]

### Phase 1: Plan

- 1.0 Create task `[required · once]`
- 1.1 Requirement exploration `[required · repeatable]`
- 1.2 Research `[optional · repeatable]`
- 1.3 Configure context `[required · once]`  (sub-agent-dispatch platforms only)
- 1.4 Activate task `[required · once]`
- 1.5 Completion criteria

[workflow-state:planning]
Load `trellis-brainstorm`; stay in planning.
Finish all three concise planning files, attest and validate, show the reviewed version, and obtain confirmation before version-specific `task.py start`.
Multi-deliverable scope: consider a parent task plus independently verifiable child tasks; dependencies must be written in child artifacts, not implied by tree position.
Channel-worker mode: curate `implement.jsonl` and `check.jsonl` as spec/research manifests before start.
Main session: save decision-relevant external evidence promptly and link it in `prd.md` / `## Research`; explain no research or applicable reuse. Research workers stay in their assigned research scope.
[/workflow-state:planning]

[workflow-state:planning-inline]
Load `trellis-brainstorm`; stay in planning.
Finish all three concise planning files, attest and validate, show the reviewed version, and obtain confirmation before version-specific `task.py start`.
Multi-deliverable scope: consider a parent task plus independently verifiable child tasks; dependencies must be written in child artifacts, not implied by tree position.
Inline mode: skip jsonl curation; Phase 2 reads artifacts/specs via `trellis-before-dev`.
Main session: save decision-relevant external evidence promptly and link it in `prd.md` / `## Research`; explain no research or applicable reuse. Research workers stay in their assigned research scope.
[/workflow-state:planning-inline]

### Phase 2: Execute

- 2.1 Implement `[required · repeatable]`
- 2.2 Quality check `[required · repeatable]`
- 2.3 Rollback `[on demand]`

Channel-driven sub-agent dispatch is the default execution model for this workflow. The main session uses `trellis channel create`, `trellis channel spawn`, `trellis channel send`, and `trellis channel wait` to coordinate workers. Fall back to native host sub-agents only when the user explicitly asks for native dispatch or a host-only capability is required.

[workflow-state:in_progress]
Flow: channel-driven `implement` worker -> channel-driven `check` worker -> `trellis-update-spec` -> commit (Phase 3.4) -> `/trellis:finish-work`.
Main-session default: use `trellis channel spawn` with `.trellis/agents/implement.md` and `.trellis/agents/check.md`; do not use native Claude Task / Codex sub_agent unless explicitly requested or host-only tools require it.
Worker context order: jsonl entries -> `prd.md` -> `design.md` -> `implement.md`. Use stable worker handles such as `implement`, `check`, `check-cx`, `check-cc`; read results with `trellis channel messages --raw` when precision matters.
Main session: save and integrate decision-changing research before adopting it; return to planning for changed agreed results. Implement/check workers report source, finding and impact; this reminder does not authorize rewriting task planning. Research workers save only assigned research and return its path.
[/workflow-state:in_progress]

[workflow-state:in_progress-inline]
Flow: `trellis-before-dev` -> edit -> channel-driven `check` worker -> validation -> `trellis-update-spec` -> commit (Phase 3.4) -> `/trellis:finish-work`.
Inline implementation is allowed only when the user asked for it or the change is too small to justify a worker. After editing, prefer `trellis channel spawn --agent check` for independent review.
Read context before editing: `prd.md` -> `design.md` -> `implement.md`, plus relevant spec/research loaded by skills.
Main session: save and integrate decision-changing research before adopting it; return to planning for changed agreed results. Implement/check workers report source, finding and impact; this reminder does not authorize rewriting task planning. Research workers save only assigned research and return its path.
[/workflow-state:in_progress-inline]

### Phase 3: Finish

- 3.2 Debug retrospective `[on demand]`
- 3.3 Spec update `[required · once]`
- 3.4 Commit changes `[required · once]`
- 3.5 Wrap-up reminder

> Note: step 3.1 was folded into 2.2 (last-iteration full-scope check) and 3.4 (commit preamble). Numbering kept stable to avoid breaking external references.

[workflow-state:completed]
Code committed. Run `/trellis:finish-work`; if dirty, return to Phase 3.4 first.
[/workflow-state:completed]

---

## Rules

1. Identify the current Phase, then continue from the next step in that Phase.
2. Run steps in order inside each Phase; `[required]` steps cannot be skipped.
3. Phase 2 uses channel workers by default. Do not implement large changes directly in the main session unless the user asked for inline work or the task is small enough.
4. Worker briefs must state the active task, goal, editable scope, validation commands, and forbidden actions.
5. `trellis channel messages --raw` is the precise audit path; pretty output is only for quick status checks.
6. After a worker completes, the main session integrates the result and runs check workers when needed. Final judgment stays with the main session.

### Active Task Routing

[Claude Code, Cursor, OpenCode, codex-sub-agent, Kiro, Gemini, Qoder, CodeBuddy, Copilot, Droid, Pi]

- Planning or unclear requirements -> `trellis-brainstorm`.
- `in_progress` implementation -> `trellis channel spawn --agent implement`.
- `in_progress` quality check -> `trellis channel spawn --agent check`.
- Repeated debugging -> `trellis-break-loop`; spec updates -> `trellis-update-spec`.

[/Claude Code, Cursor, OpenCode, codex-sub-agent, Kiro, Gemini, Qoder, CodeBuddy, Copilot, Droid, Pi]

[codex-inline, Kilo, Antigravity, Devin]

- Planning or unclear requirements -> `trellis-brainstorm`.
- Before editing -> `trellis-before-dev`; after editing -> prefer a channel-driven `check` worker.
- Repeated debugging -> `trellis-break-loop`; spec updates -> `trellis-update-spec`.

[/codex-inline, Kilo, Antigravity, Devin]

---

## Phase 1: Plan

Goal: clarify requirements, get task-creation consent, and produce planning artifacts that must be reviewed before implementation.

#### 1.0 Create task `[required · once]`

Create the task directory only after task-creation consent:

```bash
python3 ./.trellis/scripts/task.py create "<task title>" --slug <name>
```

Run only `create` here. Do not also run `start`. `start` switches status to `in_progress`, which moves the breadcrumb into execution.

#### 1.1 Requirement exploration `[required · repeatable]`

Load `trellis-brainstorm` and write user requirements into `prd.md`. Complex tasks also need `design.md` and `implement.md`.

Requirements:

- Ask one question at a time.
- Prefer researching over asking for information that can be discovered.
- Update task artifacts immediately when requirements change.
- Split broad work into parent task + child tasks.
- Keep `prd.md` focused on requirements and acceptance criteria, not implementation checklists.

#### 1.2 Research `[optional · repeatable]`

When research is needed, write results to `{TASK_DIR}/research/`. Research files must be usable by later workers.

**Decision evidence and handoff**:
- Save only necessary facts and accessible source identifiers. Remove credentials and unrelated personal data, respect project access boundaries, and do not persist unrelated raw request/response payloads.
- Record external evidence when it supports, rejects, or changes a task decision, whether from web sources, local third-party dependencies, user-provided material, or experiments. Routine project-code reading and unrelated browsing do not require a research log.
- Save useful findings before presenting or acting on the conclusion, not just at session end. Reuse a topic file under `research/`; no fixed filename, headings, length, or operation count is required.
- Include the question, locatable sources, conclusion and limits, decision impact, and unresolved issues. Distinguish observed facts, inference, and unknowns; record alternatives and rejection reasons only when actually compared. For version-sensitive claims, identify the relevant version/commit or access date. Local dependencies need identity/version and file/symbol locations; user material must be attributed, and experiments need reproducible conditions, steps, and observations.
- Keep key conclusions and links in `prd.md` under `## Research`; cite the same record beside affected design decisions. State why no external research is needed, or why existing research remains applicable, rather than creating empty files. Not yet researched, no reliable answer found, and no research needed are different states.
- Record source conflicts and unknowns with their effect and next action; not finding an answer does not prove a capability absent. If an unknown changes scope, behavior, compatibility, risk, or acceptance, return to planning rather than claim readiness. When new evidence replaces a conclusion, update the record, decisions, and references so the current conclusion is clear.
- Task-document links resolve relative to their document; manifest `file` paths resolve from the repository root. Select research for each role, not every report for every worker. Missing or stale critical evidence must be surfaced, not silently skipped.
- These are model instructions and review requirements, not automatic capture, semantic scoring, or new `task.py start` refusal conditions. A file or link alone does not establish evidence quality.

When using a research worker, follow this dispatch contract:

- **Dispatch prompt guard**: Start with `Active task: <task path>`; provide the specific question, assigned `research/<topic>.md` path, and minimum content: locatable sources, conclusion/limits, decision impact, and facts/inferences/unknowns. If task identity is missing or ambiguous, return to the main session rather than guess.
- **Return and integration**: Research workers write only within their assigned research scope and return a path plus summary. The main session must actually read the file, check the question, sources, conclusion and limits, decision impact, and distinction between facts, inferences and unknowns; request missing content before integrating the conclusion into `prd.md` / `design.md`.


#### 1.3 Configure context `[required · once]`

Curate worker context manifests:

- `implement.jsonl` — specs and research needed by the implementation worker.
- `check.jsonl` — quality specs, test specs, and research needed by the check worker.

Do not put code files in jsonl. Workers read code during execution. Select the decision-relevant research each role needs and verify the paths; manifest `file` paths are repository-root relative. A file existing under `research/` does not mean a worker received or read it.

#### 1.4 Activate task `[required · once]`

Before the existing start review, check `prd.md` / `## Research`: required evidence is saved or reusable, citations are locatable, facts/inferences/unknowns are distinguishable, and result-affecting unknowns have been handled. Explain why research is unnecessary when applicable. This is a review requirement, not an additional script gate.

After artifact review, start the task:

```bash
python3 ./.trellis/scripts/task.py start <task-dir>
```

#### 1.5 Completion criteria

| Condition | Required |
| --- | :---: |
| `prd.md` exists | yes |
| Decision-relevant evidence is saved and linked, or no research / reuse is explained; critical unknowns are handled | yes |
| user confirms task should enter implementation | yes |
| `task.py start` has run | yes |
| `design.md` exists for every new task | yes |
| `implement.md` exists for every new task | yes |
| `implement.jsonl` and `check.jsonl` each contain at least one real curated entry (seed row does not count) | yes |

Runtime consumers tolerate missing or seed-only manifests for compatibility, but that tolerance is not a planning-ready state.

---

## Phase 2: Execute

Goal: the main session turns reviewed planning artifacts into checked code through channel workers.

#### 2.1 Implement `[required · repeatable]`

Before this step, if requirements/design/plan materially changed, pause affected work, run `task.py replan`, update the three planning files and repeat 1.4 confirmation. Normal coding, tests and fixes within the approved requirements continue. Record progress separately from the reviewed plan.

Before implementation, actually read the decision-relevant research and its limits, through the role manifest or task links; merely finding a path is not enough. Before checking, compare key decisions and acceptance claims with their cited evidence and unresolved issues. Implementation/check workers report new decision-changing evidence (source, finding, impact) to the main session; a general reminder does not authorize rewriting task planning. The main session saves and integrates it, or assigns follow-up research, before adopting it. Preserve each worker's explicitly assigned code/test write scope.

[Claude Code, Cursor, OpenCode, codex-sub-agent, Kiro, Gemini, Qoder, CodeBuddy, Copilot, Droid, Pi]

Use channel-driven implement dispatch:

```bash
TASK=.trellis/tasks/<active-task>
trellis channel create impl-<topic> --task "$TASK" --by main --ephemeral
trellis channel spawn impl-<topic> \
  --agent implement \
  --as implement \
  --jsonl "$TASK/implement.jsonl" \
  --file "$TASK/prd.md" \
  --file "$TASK/design.md" \
  --file "$TASK/implement.md" \
  --cwd "$PWD" \
  --timeout 60m
trellis channel send impl-<topic> --as main --to implement --text-file /tmp/implement-brief.md
trellis channel wait impl-<topic> --as main --kind done --from implement --timeout 60m
trellis channel messages impl-<topic> --raw --from implement --last 20
```

Every new task must provide all three files. If any is missing, return to planning before dispatching affected work. Only a verified legacy in-progress/review task with all planning metadata keys absent may use the available historical files without backfilling. The brief must state the worker goal, forbidden actions, validation commands, and expected completion summary.

Native sub-agent fallback is allowed only when the user explicitly asks for it or a host-only capability is required.

[/Claude Code, Cursor, OpenCode, codex-sub-agent, Kiro, Gemini, Qoder, CodeBuddy, Copilot, Droid, Pi]

[codex-inline, Kilo, Antigravity, Devin]

1. Load `trellis-before-dev`.
2. Read `prd.md`, then `design.md`, then `implement.md`.
3. Read relevant research.
4. Small changes may be implemented inline; larger changes should still use a channel worker.
5. After implementation, enter channel-driven check.

[/codex-inline, Kilo, Antigravity, Devin]

#### 2.2 Quality check `[required · repeatable]`

Before this step, if requirements/design/plan materially changed, pause affected work, run `task.py replan`, update the three planning files and repeat 1.4 confirmation. Normal coding, tests and fixes within the approved requirements continue. Record progress separately from the reviewed plan.

Actually read the check-role research; review key decisions against locatable sources, versions/limits and unresolved issues. Missing or stale evidence that can change the agreed result returns to planning; do not pass merely because a file exists.

[Claude Code, Cursor, OpenCode, codex-sub-agent, Kiro, Gemini, Qoder, CodeBuddy, Copilot, Droid, Pi]

Use channel-driven check dispatch:

```bash
TASK=.trellis/tasks/<active-task>
trellis channel create cr-<topic> --task "$TASK" --by main --ephemeral
trellis channel spawn cr-<topic> \
  --agent check \
  --as check \
  --jsonl "$TASK/check.jsonl" \
  --file "$TASK/prd.md" \
  --file "$TASK/design.md" \
  --file "$TASK/implement.md" \
  --cwd "$PWD" \
  --timeout 30m
trellis channel send cr-<topic> --as main --to check --text-file /tmp/check-brief.md
trellis channel wait cr-<topic> --as main --kind done --from check --timeout 30m
trellis channel messages cr-<topic> --raw --from check --last 40
```

For independent cross-provider review, spawn `check-cc` and `check-cx` in the same channel:

```bash
trellis channel spawn cr-<topic> --agent check --provider claude --as check-cc --cwd "$PWD" --timeout 30m
trellis channel spawn cr-<topic> --agent check --provider codex --as check-cx --cwd "$PWD" --timeout 30m
trellis channel send cr-<topic> --as main --to check-cc --text-file /tmp/check-brief.md
trellis channel send cr-<topic> --as main --to check-cx --text-file /tmp/check-brief.md
trellis channel wait cr-<topic> --as main --kind done --from check-cc,check-cx --all --timeout 30m
```

Check workers should directly fix clear issues. The main session reads raw events and makes the final judgment.

[/Claude Code, Cursor, OpenCode, codex-sub-agent, Kiro, Gemini, Qoder, CodeBuddy, Copilot, Droid, Pi]

[codex-inline, Kilo, Antigravity, Devin]

Load `trellis-check` or use a channel-driven check worker. If issues are found, fix and re-check until green.

[/codex-inline, Kilo, Antigravity, Devin]

#### 2.3 Rollback `[on demand]`

When follow-up research changes a decision, save the findings and update the research entry, affected decisions, citations and role manifests. Mark replaced conclusions; changes to agreed scope, behavior, compatibility, risk or acceptance return to planning and review.

- If check finds a PRD defect -> return to Phase 1, fix artifacts, then execute again.
- If an implement worker goes off-track -> narrow the brief, redispatch, or revert that work.
- If more research is needed -> write to `{TASK_DIR}/research/`, then redispatch.

---

## Phase 3: Finish

Goal: verify quality, capture lessons, and commit the work.

#### 3.2 Debug retrospective `[on demand]`

If the same class of issue recurred, load `trellis-break-loop` and record root cause plus prevention.

#### 3.3 Spec update `[required · once]`

Load `trellis-update-spec` and decide whether new patterns, pitfalls, or technical decisions should be written back to `.trellis/spec/`.

#### 3.4 Commit changes `[required · once]`

**Spec-sync preamble**: before drafting commits, ask: did this task fix a bug or surface non-obvious knowledge that should land in `.trellis/spec/` so future-you (or future-AI) doesn't repeat the mistake? If yes, return to Phase 3.3 first — spec writes belong in the same task's commit batch, not as a forgotten follow-up.

The main session commits work changes. Before committing, separate AI-edited files from unknown dirty files.

```bash
git status --porcelain
git log --oneline -5
```

Do not amend. Do not push.

#### 3.5 Wrap-up reminder

After committing, remind the user to run `/trellis:finish-work` to archive the task and record the session.

---

## Customizing Trellis

This workflow is customized through `.trellis/workflow.md`. Scripts parse tags and headings; they do not store fallback prose.

### Change a step

Edit the corresponding Phase 1 / 2 / 3 step body.

### Change per-turn prompt text

Edit the body of the matching `[workflow-state:STATUS]` block. Do not change tag names or syntax.

### Add a custom status

Add:

```text
[workflow-state:my-status]
...
[/workflow-state:my-status]
```

A lifecycle hook or script must write `task.json.status` to that value, otherwise the block is never read.


## Reviewed planning version and material changes

For every new task, prepare readable, nonempty `prd.md` (requirements), `design.md` (solution), and `implement.md` (implementation and verification order). Keep each concise; the topics above are guidance, not required headings or word counts. Read all three together and resolve contradictions. File presence or a successful script cannot assess meaning or prove user identity.

Before first start or start after replanning:

1. Check the three files and decision-relevant research, then run `task.py attest <task-dir> --note "what was reviewed"`.
2. Run `task.py validate <task-dir>`. A waiting version may validate successfully but is still not approved.
3. Show the user the complete `reviewId`, explanation, and a plain-language summary of scope, solution, verification and limits. Append the summary, original three fingerprints, changes and real response/source to `<task-dir>/research/planning-review-log.md`; do not overwrite older rounds or invent message identifiers. Preserve the old review log before clearing records with replan; if saving fails, address that first. Full historical text requires version control; fingerprints cannot restore overwritten text.
4. Only after the user's explicit response to that version, recheck and run `task.py start <task-dir> --review <reviewId> --approval-note "actual response" --approval-source "locatable conversation source"`. Ordinary attest always creates a new version and clears prior approval; recomputing hashes cannot grant permission.

If requirements, design or the implementation plan materially change, including a change agreed in conversation, pause affected work and dispatched dependencies. Run `task.py replan <task-dir> --reason "what changed" --affected "work to pause" --independent "work that does not depend on the change, or none"`, update the three files and repeat review/confirmation. State why any continuing work is independent; that field does not grant new authorization. Keep normal coding, testing and fixes within the approved requirements moving. Record normal progress separately without checking off the frozen reviewed plan.

For a verified wording-only correction after start, use `task.py attest <task-dir> --editorial --note "specific correction; behavior and scope unchanged"`; preserve the reviewed version and confirmation. If the user has already responded but first start is not recorded, add `--editorial-note "specific correction"` to that same version-specific start. Inspect the actual diff; neither command can determine whether a purported wording correction changes meaning.

Only `in_progress`/`review` tasks with all three planning metadata keys absent may resume through the legacy path without backfilling the files. Partial, null or malformed records are not legacy. Completed tasks cannot start again. Same-task writes are serialized by the main session; concurrency and exactly-once hooks are not promised. Session hints query `task.py planning-status <task-dir> --json`; unavailable checks never imply permission to start. New records must match the current files on resume. Archive planning problems warn while existing archive checks remain in force.
