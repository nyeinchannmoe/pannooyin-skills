---
name: task-breakdown
description: Converts a Pann Oo Yin Feature Analysis or Bug Analysis result into implementation-ready, appropriately sized tasks with scope, dependencies, acceptance criteria, and verification. Reuses the analysis work folder ID and writes task-result/<same-folder>/tasks.md so generate-dor can find the same feature or bug. Use when the user asks to break down an analysis into tasks, create an implementation task list, or split a planned feature or bugfix before coding. Second step after feature-analysis or bug-analysis.
---

# Task Breakdown

Turn an existing Feature Analysis or Bug Analysis result into a numbered, execution-ordered task list. Do not implement anything.

Stack to account for: React, TypeScript, Python, Django, Vercel, Neon PostgreSQL.

- Web app: `pann-oo-yin-web`
- API: `pann-oo-yin-api`

## Pipeline and work folder

This skill is **step 2 of 3** for both features and bugs. Run only this step. Do not start `generate-dor` unless the user asks. Do not invent an analysis if step 1 is missing.

Feature:

1. `feature-analysis`
2. `task-breakdown` — this skill
3. `generate-dor` (after the user approves this task list)
4. Implement from the DoRs

Bug:

1. `bug-analysis`
2. `task-breakdown` — this skill
3. `generate-dor` (after the user approves this task list)
4. Fix from the DoRs

Reuse the analysis **work folder ID** exactly. Do not invent a new date or slug.

- Feature IDs look like `YYYY-MM-DD-feature-<short-slug>`
- Bug IDs look like `YYYY-MM-DD-bug-<short-slug>`

Write the breakdown to the matching folder:

`task-result/<work-folder-id>/tasks.md`

`generate-dor` will later write:

`dor-result/<work-folder-id>/TASK-NN-<slug>.md`

Do not write a flat file such as `task-result/YYYY-MM-DD-tasks-<slug>.md`.

## Hard rules

- This skill generates task breakdowns only. Do not implement changes, modify application code, or execute database operations.
- Do not run migrations, deletes, resets, deploys, or any command that writes to the database or remote systems.
- The only allowed write is the task-breakdown file under `task-result/<work-folder-id>/`.
- Use the provided analysis result as the primary input. Do not invent requirements missing from the analysis.
- Consult the actual project, `Pann-Oo-Yin-Database-README.md`, and related docs only when needed to resolve dependencies, implementation boundaries, or technical uncertainties.
- If essential information is missing, identify the uncertainty instead of silently inventing a requirement.
- Avoid duplicate tasks, unrelated refactoring, unnecessary technical details, and scope expansion.
- Never change the work folder ID from the source analysis.

## Input

Resolve the source analysis and work folder ID in this order:

1. Use the analysis file or work folder the user attached or named.
2. If none is given, look in `analysis-result/` for a matching work folder (`YYYY-MM-DD-feature-<slug>` or `YYYY-MM-DD-bug-<slug>`) containing `analysis.md` (or `analysis-2.md`, …). If several could apply, ask which one.
3. Legacy fallback: if a flat file `analysis-result/YYYY-MM-DD-feature-<slug>.md` or `analysis-result/YYYY-MM-DD-bug-<slug>.md` exists, treat the filename without `.md` as the work folder ID. Prefer moving/reading that content as the analysis; still write tasks under `task-result/<work-folder-id>/`.
4. If no analysis exists, stop after asking for it. Do not invent a plan from a vague request.

The work folder ID is the analysis parent folder name, or the legacy filename without `.md`. Use that string unchanged for `task-result/<work-folder-id>/`.

## Split decision

First assess overall complexity, scope, dependencies, risks, and implementation effort.

Create **exactly ONE task** when the work can be implemented, tested, and verified as one cohesive unit. Do not split work unnecessarily.

Split into smaller tasks only when doing so improves implementation reliability, testing, review, or delivery. Keep each task independently understandable and verifiable.

Prefer vertical slices or meaningful implementation milestones over arbitrary file-based divisions. Keep tightly coupled changes together when splitting them would create incomplete or untestable intermediate states.

Cover relevant frontend, backend, API, database, migrations, security, compatibility, and testing work without overlooking necessary integration. Fold testing into the task that delivers the behavior unless a separate verification task is the only reliable way to prove the whole slice.

Identify task dependencies and an explicit execution order. Separate prerequisites from dependent work, and avoid unnecessary dependencies.

Keep tasks small enough to implement and verify reliably, but not so small that they create unnecessary overhead.

## Workflow

1. Read the analysis. Record the work folder ID. Extract intended behavior, proposed steps, affected layers, risks, tests, and unresolved questions.
2. Inspect the repo only where the analysis is incomplete on boundaries or dependencies.
3. Decide one task vs a split. Record the reason.
4. Write tasks in execution order with IDs `T1`, `T2`, …
5. Create `task-result/<work-folder-id>/` if needed. Write the full breakdown to `tasks.md`. Do not paste it into chat. Stop. Tell the user the work folder ID and that the next step is to approve the list, then run `generate-dor`.

## Result file

Write the complete breakdown as a markdown file. Do not print the task list in chat.

- Directory: `task-result/<work-folder-id>/` at the workspace root. Create the directories if they do not exist.
- Filename: `tasks.md`. If that path exists, append `-2`, `-3` (`tasks-2.md`) unless the user authorized overwrite.
- File contents: a title, the work folder ID, a link to the source analysis file, then the required sections below. No secrets.
- Chat: only the result file path, the work folder ID, the task count, a one-sentence split rationale, any essential clarification questions, and that the next step is user approval then `generate-dor`. Do not restate the tasks.

## Required output

Use these headings, in this order:

### Work folder

State the work folder ID and the connected paths:

- Source analysis: `analysis-result/<work-folder-id>/analysis.md` (or the actual analysis filename)
- This breakdown: `task-result/<work-folder-id>/tasks.md`
- DoRs (next, after approval): `dor-result/<work-folder-id>/`

### Scope and complexity assessment

Brief statement of what is in scope, effort, main dependencies, and risks. Call out analysis gaps that affect tasking.

### Tasks

A numbered list in execution order. For each task include:

- **Task ID and title** — `T<n>` plus a concise title.
- **Objective and expected outcome**
- **Scope** — included work, and explicit exclusions where necessary.
- **Dependencies** — `None` or the Task IDs that must complete first.
- **Acceptance criteria** — testable checks.
- **Verification** — tests and manual checks, matching existing Django and Vitest patterns where the analysis names them.

### Split rationale

A short explanation of why the work should be split, or why a single task is sufficient.

## Quality bar

- Every task traces to the source analysis; gaps are labeled as uncertainties.
- One task unless a split clearly improves reliability, testing, review, or delivery.
- The work folder ID matches the source analysis exactly.
- No application source, tests, docs, or config are modified by this skill.
- The full breakdown is written only to `task-result/<work-folder-id>/tasks.md`; chat does not contain the task list.
- No secrets from `.env` or credentials files appear in the report.
