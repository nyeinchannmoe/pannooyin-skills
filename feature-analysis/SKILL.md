---
name: feature-analysis
description: Analyzes Pann Oo Yin feature requests and produces a detailed implementation plan without modifying code. Inspects pann-oo-yin-web (React/TypeScript) and pann-oo-yin-api (Django), verifies schema against Pann-Oo-Yin-Database-README.md, and reports impact, risks, migrations, and acceptance criteria. Writes analysis-result/<YYYY-MM-DD-feature-slug>/analysis.md and creates the work folder reused by task-breakdown and generate-dor. Use when the user asks for a new feature, feature analysis, an implementation plan, or to analyze requirements before coding. First step of the feature pipeline.
---

# Feature Analysis

Analyze the requested Pann Oo Yin change and produce an implementation plan. Do not write or edit application code in this pass.

Stack to account for: React, TypeScript, Python, Django, Vercel, Neon PostgreSQL.

- Web app: `pann-oo-yin-web`
- API: `pann-oo-yin-api`

## Pipeline and work folder

This skill is **step 1 of 3** for a feature. Run only this step. Do not start `task-breakdown` or `generate-dor` unless the user asks.

1. `feature-analysis` — this skill; creates the work folder.
2. `task-breakdown` — same work folder.
3. `generate-dor` — same work folder, after the user approves the task list.
4. A developer or AI coding agent implements from the DoRs.

Every feature uses one **work folder ID** that later skills must reuse exactly:

`YYYY-MM-DD-feature-<short-slug>`

`<short-slug>` is lowercase hyphenated text from the request (max ~60 characters).

Create that folder under `analysis-result/` and write the analysis inside it:

`analysis-result/<work-folder-id>/analysis.md`

Do not write a flat file such as `analysis-result/YYYY-MM-DD-feature-<slug>.md`. Later skills look up this work folder ID and write:

- `task-result/<work-folder-id>/tasks.md`
- `dor-result/<work-folder-id>/TASK-NN-<slug>.md`

If a matching work folder already exists (user named it, or `analysis-result/<work-folder-id>/` is present), reuse that ID. If `analysis.md` already exists in that folder, append `-2`, `-3` to the filename (`analysis-2.md`) unless the user authorized overwrite.

## Hard rules

- Do NOT implement changes, execute destructive operations, or generate a code-only answer during analysis.
- Do not run migrations, deletes, resets, deploys, or any command that writes to the database or remote systems.
- The only allowed write is the analysis result file under `analysis-result/<work-folder-id>/`. Do not modify application source, tests, docs, or config.
- Discover relevant behavior from the current repo. Do not invent project behavior or make unsupported assumptions.
- Do not document the whole app. Report only what is relevant to this request.
- Distinguish verified facts from assumptions and unresolved questions.
- Prefer reusing existing architecture and utilities. Avoid unnecessary dependencies, refactoring, schema changes, and scope expansion.
- When requirements are ambiguous or critical information is missing, identify the specific issue and ask only essential clarification questions.
- If the user also asks to implement: complete this analysis only, then stop and wait for approval of the plan.

## Discovery

Inspect only what the request can affect. Use the live tree; paths below are starting points, not a complete inventory.

### Database

1. Locate and read `Pann-Oo-Yin-Database-README.md` (expected under `pann-oo-yin-api/docs/`; a copy may exist under `pann-oo-yin-web/api_docs/`). Prefer the API copy if both exist.
2. Verify relevant tables, columns, constraints, and relationships against Django models, migrations, and serializers.
3. Record discrepancies. Treat the running models/migrations as implementation truth and the README as design intent unless code comments or later docs clearly supersede it.

### Backend (`pann-oo-yin-api`)

Inspect as applicable:

- Routing: `config/urls.py`, app `urls.py`
- Views, serializers, services, validators, permissions, authentication
- Models, managers, migrations under `apps/*/migrations/`
- Shared utilities in `apps/common/` (auth, admin restriction, pagination, errors, tokens, storage)
- Settings: `config/settings/`, `.env.example` (never print secret values from `.env`)
- Endpoint docs under `docs/`
- Tests under `apps/*/tests/`
- Deploy: `vercel.json`, `requirements.txt`, README

### Frontend (`pann-oo-yin-web`)

Inspect as applicable:

- Routing and guards: `src/App.tsx`, `src/components/auth/`, `src/utils/*Routes*`
- Pages, components, hooks, types, utils
- API clients in `src/services/` (especially `apiClient.ts`)
- Copied API docs under `api_docs/`
- `.env.example` (`VITE_*` must stay in sync with backend settings they mirror)
- `vercel.json`, `package.json`, colocated `*.test.ts` / `*.test.tsx`

### Cross-cutting checks

Analyze the requested change against existing behavior, backward compatibility, authentication, authorization, data integrity, security, performance, and deployment constraints where relevant.

Identify affected files, API contracts, database changes, migration requirements, dependencies, edge cases, risks, and potential regressions.

## Workflow

1. Choose the work folder ID (`YYYY-MM-DD-feature-<short-slug>`). Reuse an existing matching folder when one already exists for this feature.
2. Restate the request as testable requirements. Mark gaps instead of filling them in.
3. Search the web and API code, docs, tests, env examples, and schema README for current behavior.
4. Compare README schema vs models/migrations. List only relevant mismatches.
5. Map the change onto existing endpoints, UI, and data. Prefer the smallest reuse path.
6. Decide whether schema, API contract, env, or deploy config must change. Default to no schema change.
7. If a missing fact blocks a safe plan, ask the essential questions in chat and still write the analysis with those items unresolved.
8. Create `analysis-result/<work-folder-id>/` if needed. Write the full report to `analysis.md` using the required sections below. Do not paste the report into chat. Stop. Tell the user the work folder ID and that the next step is `task-breakdown`.

## Analysis result file

Write the complete analysis as a markdown file. Do not print the analysis result in chat.

- Directory: `analysis-result/<work-folder-id>/` at the workspace root. Create the directories if they do not exist.
- Filename: `analysis.md`.
- File contents: a title, the work folder ID, then the required analysis sections in the order below. No secrets.
- Chat: only the result file path, the work folder ID, a one-sentence recommendation, any essential clarification questions, and that the next step is `task-breakdown`. Do not restate the report.

## Required analysis output

Use these headings, in this order:

### Work folder

State the work folder ID and the paths later skills must reuse:

- Analysis: `analysis-result/<work-folder-id>/analysis.md`
- Tasks (next): `task-result/<work-folder-id>/tasks.md`
- DoRs (after task approval): `dor-result/<work-folder-id>/`

### Feature requirements and intended behavior

What the user asked for, who it is for, and the expected before/after behavior. Call out out-of-scope items.

### Existing implementation findings

Verified current behavior, with file paths. Include README-vs-code schema discrepancies that matter.

### Affected frontend, backend, API, and database components

Concrete files, endpoints, types, tables/fields, env vars, and docs. Note “none” where a layer is untouched.

### Proposed implementation steps

An actionable, ordered plan. Each step should name the area (web / API / DB / docs / deploy), the files or contracts to change, and what “done” means. Include migration steps only when schema or data backfill is required.

### Backward compatibility and migration strategy

API contract compatibility, existing rows, clients, env vars, and Vercel/Neon constraints. If no migration is needed, say so and why.

### Risks, edge cases, and security considerations

Authz, data leaks, validation, integrity, performance, and likely regressions.

### Testing and acceptance criteria

Checkable acceptance criteria plus tests to add or extend, matching existing Django and Vitest patterns. Name the suites/files to follow.

### Assumptions, unresolved questions, and final recommendation

Split **Verified**, **Assumptions**, and **Unresolved questions**. End with one recommendation: proceed, proceed with listed caveats, or pause until answers are provided.

## Quality bar

- Every behavioral claim is backed by a path or labeled as an assumption.
- The plan is the smallest change that satisfies the requirements.
- The full report is written only to `analysis-result/<work-folder-id>/analysis.md`; chat does not contain the analysis.
- The work folder ID is stable so `task-breakdown` and `generate-dor` can find this feature.
- No application source, tests, docs, or config are modified by this skill.
- No secrets from `.env` or credentials files appear in the report.
