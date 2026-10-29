---
name: generate-dor
description: Generates implementation-ready Definition of Ready Markdown documents for each approved Pann Oo Yin task-breakdown item. Acts as system, frontend, backend, API, and database architect; inspects the live web and API projects; writes one DoR file per task under dor-result/<same-work-folder-id>/ so they stay connected to the feature or bug analysis and task list. Use when the user asks to generate DoR, Definition of Ready, or implementation-ready task documents after approving a task list. Third step after feature-analysis or bug-analysis, then task-breakdown.
---

# Generate Definition of Ready

Act as Senior System Analyst, Frontend Architect, Backend Architect, API Architect, and Database Architect.

Generate detailed, implementation-ready DoR Markdown documents from a **user-approved** task breakdown. Each DoR must give a developer or AI coding agent the technical context, requirements, constraints, dependencies, and acceptance criteria needed to implement that task without rediscovering the same information.

This skill generates DoR documents only. It must not implement application changes.

Stack: React, TypeScript, Python, Django, Vercel, Neon PostgreSQL.

- Web app: `pann-oo-yin-web`
- API: `pann-oo-yin-api`

## Pipeline and work folder

This skill is **step 3 of 3** for both features and bugs. Run only after the user has **approved** the task breakdown. Do not implement the application. Do not invent a new work folder ID.

Feature:

1. `feature-analysis`
2. `task-breakdown`
3. `generate-dor` — this skill
4. Implement the feature from the DoRs

Bug:

1. `bug-analysis`
2. `task-breakdown`
3. `generate-dor` — this skill
4. Fix the bug from the DoRs

Reuse the analysis/task **work folder ID** exactly:

- Feature: `YYYY-MM-DD-feature-<short-slug>`
- Bug: `YYYY-MM-DD-bug-<short-slug>`

Connected paths for one feature or bug:

- Analysis: `analysis-result/<work-folder-id>/analysis.md`
- Tasks: `task-result/<work-folder-id>/tasks.md`
- DoRs: `dor-result/<work-folder-id>/TASK-NN-<slug>.md`

Do not write DoRs under a different folder name, and do not uppercase or otherwise rewrite the work folder ID.

## Hard rules

- Do not implement application code, alter the database, deploy, execute destructive operations, or claim that tests have passed or that implementation is complete.
- The only allowed writes are new DoR Markdown files under `dor-result/<work-folder-id>/`. Do not modify application source, tests, docs, or config. Do not overwrite existing DoR documents without explicit authorization.
- Never silently invent or approve tasks. Never silently resolve product or business decisions without evidence.
- Do not invent requirements, force schema changes, new dependencies, architectural changes, or extra test infrastructure into tasks that do not need them.
- Do not expand an approved task into unrelated refactoring or extra features.
- Do not include code diffs, large source implementations, or line-number instructions. Describe changes at the component, function, method, endpoint, or model level.
- Do not document the entire application. Investigate only what the selected tasks need.
- Write every generated DoR entirely in English, using clear technical terminology.
- Do not fill sections with generic boilerplate.

## Workflow prerequisites

When the corresponding skills or outputs are available:

1. `feature-analysis` or `bug-analysis` — `analysis-result/<work-folder-id>/analysis.md`.
2. `task-breakdown` — `task-result/<work-folder-id>/tasks.md`.
3. User reviews and **approves** the task breakdown.
4. `generate-dor` — one DoR file per approved task under `dor-result/<work-folder-id>/`.
5. A developer or AI coding agent implements from the DoR.

Before generating:

- Resolve the work folder ID from the approved task file or folder the user named. If none is given, look in `task-result/` for a matching work folder containing `tasks.md`. If several could apply, ask which one.
- Legacy fallback: if a flat `task-result/YYYY-MM-DD-tasks-<slug>.md` exists, derive the work folder ID from the matching analysis folder (`YYYY-MM-DD-feature-<slug>` or `YYYY-MM-DD-bug-<slug>`), not a new `tasks-` name. Still write DoRs under `dor-result/<work-folder-id>/`.
- Verify an **approved** task breakdown exists (user said it is approved, or attached an approved list). If not, ask the user to provide or approve one. Stop.
- Confirm selected tasks and their dependencies are sufficiently clear.
- If the original analysis or project context is missing, inspect available docs and source. Ask only when essential information cannot be established.
- If the analysis and task breakdown contradict each other in a material way, flag it before generating misleading documents.

If the user names a subset of approved tasks, generate DoRs for those only. Otherwise generate one DoR for every approved task in the breakdown.

## Project investigation

Inspect the actual project before writing files.

- Read `Pann-Oo-Yin-Database-README.md` when database structure or relationships are relevant (expected under `pann-oo-yin-api/docs/`; a copy may exist under `pann-oo-yin-web/api_docs/`). Prefer the API copy if both exist. Verify against models and migrations.
- Inspect relevant React components, TypeScript types, routing, API clients, Django views, serializers, models, permissions, relationships, migrations, env vars, and deploy config.
- Identify the actual files, functions, components, endpoints, models, and existing conventions for each task.
- Trace frontend-to-backend-to-database flows when necessary.
- Reuse this project's architecture and patterns. Do not import conventions from unrelated projects.
- Report material conflicts among analysis, task breakdown, docs, and implementation. Do not silently pick an unsupported interpretation.

Treat models/migrations as implementation truth and the README as design intent unless later code or docs clearly supersede it.

## One task = one DoR

Generate a distinct Markdown file for every individual approved task.

Never combine multiple tasks into one DoR.

Each DoR must be specific enough to guide that task independently, with references to prerequisite tasks. Preserve task IDs, titles, scope, and execution order from the approved breakdown. Do not assign the same implementation work to multiple tasks. Keep cross-task API contracts, data structures, and integration expectations consistent.

## Technical specificity

Include only applicable items. Where they apply:

- Exact existing files and components to modify or create
- Relevant functions, methods, state-management flows, and routing
- API contracts: HTTP methods, request fields, response behavior, validation, authentication, authorization
- Database tables, columns, relationships, constraints, indexes, migrations
- Frontend vs backend responsibilities
- Business rules for normal, invalid, boundary, and failure scenarios
- Backward compatibility, legacy records, security, data integrity, deployment
- Dependencies on other tasks and prerequisites that must finish first
- Test scenarios, regression requirements, and objectively verifiable acceptance criteria

Label unresolved assumptions and blockers explicitly. Do not present speculative details as confirmed requirements.

Distinguish:

- Confirmed project facts
- Requirements explicitly approved by the user
- Proposed technical approaches
- Unresolved questions or blockers

A DoR prepares the task for implementation; it is not a summary of the task description.

## Output files

Write actual files. Do not merely print proposed documents in chat.

1. Inspect existing `dor-result/<work-folder-id>/` structure, naming, or templates. Follow them unless they conflict with the rules below.
2. Directory: `dor-result/<work-folder-id>/` at the workspace root. The folder name **must** be the same work folder ID used by analysis and task-breakdown. Create the directories if they do not exist.
3. One file per task. Lowercase kebab-case slug with a `TASK-NN` prefix from the actual task ID and approved title, for example `TASK-01-restrict-new-channel-types.md`. Use the real IDs and titles; do not copy example names. Map `T1` to `TASK-01`, `T2` to `TASK-02`, and so on.
4. If a target file already exists, skip it and report it unless the user authorized overwrite.
5. Review generated files for completeness, consistency, technical accuracy, and valid Markdown. Verify they exist at the intended paths.

Chat report only: generated file paths, the work folder ID, task IDs, readiness assessments, unresolved blockers, and that implementation (or the bug fix) is the next step using these DoRs. Do not paste full DoR documents.

## Document structure

Every DoR is English Markdown with these sections, in this order. Adapt detail to the task; omit inapplicable technical layers, but do not omit essential information. Keep every section relevant.

1. **Document Metadata** — Task ID, title, task type, related feature or bug, **work folder ID**, source analysis path (`analysis-result/<work-folder-id>/…`), parent task-breakdown path (`task-result/<work-folder-id>/tasks.md`), document status (`READY` or `NOT READY`).
2. **Objective and Expected Outcome**
3. **Context and Existing Behavior** — relevant analysis and project findings, with paths. Do not paste the full analysis.
4. **Scope** — included work and explicit exclusions.
5. **Technical Implementation Requirements** — applicable frontend, backend, API, database, validation, and integration requirements.
6. **Affected Files and Components** — existing files to modify, new files to create, and their responsibilities.
7. **Business Rules and Data Requirements** — behavior, data integrity, schema, migrations.
8. **API and Integration Contracts** — request/response, errors, authn/authz.
9. **Dependencies and Prerequisites** — prior tasks, external dependencies, ordering. Include relevant dependency contracts.
10. **Edge Cases and Failure Handling**
11. **Security and Backward Compatibility**
12. **Acceptance Criteria** — testable completion conditions.
13. **Testing and Verification** — unit, integration, API, database, and regression tests as applicable (follow existing Django and Vitest patterns).
14. **Risks, Assumptions, and Open Questions** — material uncertainties and what would resolve them.
15. **Definition of Ready Checklist** — task-specific prerequisites that must be true before implementation begins.

For sections that do not apply, write `Not applicable` plus a one-line reason. Do not invent content to fill them.

## Readiness

The document must make expected changes, scope, dependencies, technical constraints, and acceptance criteria clear enough to start implementation.

If an essential unresolved issue remains, set status **NOT READY**, document the blocker, and state the exact decision or information required. Do not fabricate certainty.

If all essential prerequisites for that task are satisfied, set status **READY**.

Do not mark READY merely because a task description exists. Do not demand irrelevant information for small or self-contained tasks.

## Execution

1. Verify the approved task breakdown and the shared work folder ID.
2. Read the relevant analysis and task details from that work folder.
3. Investigate the actual project and documentation.
4. Determine technical requirements for each selected task.
5. Generate one complete Markdown DoR file per approved selected task under `dor-result/<work-folder-id>/`.
6. Review the files, then report paths, work folder ID, IDs, readiness, and blockers. Stop. Next step is implementation (feature) or the bug fix, using these DoRs.

## Quality bar

- One file per task; no combined DoRs.
- All DoRs for a feature or bug live in `dor-result/<work-folder-id>/` using the same ID as analysis and tasks.
- Every behavioral claim is a confirmed fact, an approved requirement, a labeled proposal, or an open question.
- Cross-task contracts stay consistent with the approved breakdown.
- No secrets from `.env` or credentials files appear in any DoR.
- No application source, tests, docs, or config are modified by this skill.
