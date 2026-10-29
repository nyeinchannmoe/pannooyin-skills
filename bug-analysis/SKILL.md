---
name: bug-analysis
description: Investigates Pann Oo Yin bug reports, traces the failing flow, and produces a root-cause analysis and fix plan without modifying code. Inspects pann-oo-yin-web (React/TypeScript) and pann-oo-yin-api (Django), checks schema against Pann-Oo-Yin-Database-README.md when data is involved, and reports evidence, risks, and verification steps. Writes analysis-result/<YYYY-MM-DD-bug-slug>/analysis.md and creates the work folder reused by task-breakdown and generate-dor. Use when the user reports a bug, regression, incorrect behavior, error, or asks to debug before coding a fix. First step of the bug pipeline.
---

# Bug Analysis

Investigate the reported Pann Oo Yin bug and produce a reliable debugging and resolution plan. Do not write or edit application code in this pass.

Stack to account for: React, TypeScript, Python, Django, Vercel, Neon PostgreSQL.

- Web app: `pann-oo-yin-web`
- API: `pann-oo-yin-api`

## Pipeline and work folder

This skill is **step 1 of 3** for a bug. Run only this step. Do not start `task-breakdown` or `generate-dor` unless the user asks.

1. `bug-analysis` — this skill; creates the work folder.
2. `task-breakdown` — same work folder.
3. `generate-dor` — same work folder, after the user approves the task list.
4. A developer or AI coding agent fixes the bug from the DoRs.

Every bug uses one **work folder ID** that later skills must reuse exactly:

`YYYY-MM-DD-bug-<short-slug>`

`<short-slug>` is lowercase hyphenated text from the report (max ~60 characters).

Create that folder under `analysis-result/` and write the analysis inside it:

`analysis-result/<work-folder-id>/analysis.md`

Do not write a flat file such as `analysis-result/YYYY-MM-DD-bug-<slug>.md`. Later skills look up this work folder ID and write:

- `task-result/<work-folder-id>/tasks.md`
- `dor-result/<work-folder-id>/TASK-NN-<slug>.md`

If a matching work folder already exists (user named it, or `analysis-result/<work-folder-id>/` is present), reuse that ID. If `analysis.md` already exists in that folder, append `-2`, `-3` to the filename (`analysis-2.md`) unless the user authorized overwrite.

## Hard rules

- Do NOT implement fixes, execute destructive operations, or present an unverified workaround as a confirmed solution during analysis.
- Do not run migrations, deletes, resets, deploys, or any command that writes to the database or remote systems.
- The only allowed write is the analysis result file under `analysis-result/<work-folder-id>/`. Do not modify application source, tests, docs, or config.
- Investigate the actual project whenever the skill is used. Do not invent behavior or treat guesses as facts.
- Distinguish the confirmed root cause from hypotheses. If evidence is insufficient, say what is missing and propose the next diagnostic steps instead of guessing.
- Prefer the smallest reliable fix that addresses the root cause. Avoid speculative fixes, unnecessary refactoring, unrelated changes, and new dependencies.
- Report only what is relevant to this bug. Do not catalog unrelated features, tables, or defects.
- If the user also asks to fix: complete this analysis only, then stop and wait for approval of the plan.

## Discovery

Inspect only the flow implicated by the report. Use the live tree; paths below are starting points, not a complete inventory.

### Reproduce and trace

1. Restate expected vs actual behavior from the report. Note environment (local, Vercel, preview), user type, and inputs if given.
2. Reproduce or trace the reported behavior where feasible using code paths, existing tests, request/response shapes, and logs. Do not mutate shared or production data.
3. Follow the complete affected flow across frontend, API, backend, database, and deployment configuration where necessary.

### Frontend (`pann-oo-yin-web`)

Inspect as applicable:

- Routing and guards: `src/App.tsx`, `src/components/auth/`, `src/utils/*Routes*`
- Components, pages, hooks, types, client state
- API requests/responses in `src/services/` (especially `apiClient.ts`) and client logging
- Copied API docs under `api_docs/`
- `.env.example` (`VITE_*` vs backend mirrors), `vercel.json`, colocated `*.test.ts` / `*.test.tsx`

### Backend (`pann-oo-yin-api`)

Inspect as applicable:

- Routing: `config/urls.py`, app `urls.py`
- Django views, serializers, services, validators, authentication, permissions
- Models, managers, migrations under `apps/*/migrations/`
- Shared utilities in `apps/common/` (auth, errors, request logging, pagination, tokens, storage)
- Settings: `config/settings/`, `.env.example` (never print secret values from `.env`)
- Endpoint docs under `docs/`, tests under `apps/*/tests/`
- Deploy: `vercel.json`, `requirements.txt`, README

### Database

Consult `Pann-Oo-Yin-Database-README.md` when database behavior is relevant (expected under `pann-oo-yin-api/docs/`; a copy may exist under `pann-oo-yin-web/api_docs/`). Prefer the API copy if both exist.

Verify relevant schema, constraints, and relationships against models and migrations. Treat models/migrations as implementation truth and the README as design intent unless later code or docs clearly supersede it. Record only discrepancies that could explain the bug.

### Cross-cutting checks

Check regression risks, edge cases, data integrity, security, backward compatibility, and environment-specific behavior (local vs Vercel, env vars, CORS, cron, serverless/Neon connection limits).

## Workflow

1. Choose the work folder ID (`YYYY-MM-DD-bug-<short-slug>`). Reuse an existing matching folder when one already exists for this bug.
2. Capture the bug as expected vs actual, plus any reproduction notes and missing facts.
3. Trace the failing path through UI, client state, HTTP, Django, data, and deploy config.
4. Collect concrete evidence: file paths, stack traces, status codes, payloads, logs, tests, or schema mismatches.
5. Decide: confirmed root cause, or ranked hypotheses with the next diagnostic step for each.
6. Propose the smallest reliable fix and the tests/verification needed to prove it. Do not apply it.
7. Create `analysis-result/<work-folder-id>/` if needed. Write the full report to `analysis.md` using the required sections below. Do not paste the report into chat. Stop. Tell the user the work folder ID and that the next step is `task-breakdown`.

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

### Bug summary and expected versus actual behavior

What was reported, who/what is affected, and the expected vs actual result. Call out environment and severity if known.

### Reproduction steps and investigation findings

Minimal reproduction if known; otherwise the trace performed. Cite files, requests, responses, and logs actually inspected.

### Root cause with supporting evidence, or clearly labeled hypotheses

State a confirmed root cause only with concrete code paths, logs, errors, or reproducible evidence. Otherwise list hypotheses, each with supporting and contradicting evidence, and what would confirm it.

### Affected files and end-to-end execution flow

The failing sequence from UI (or trigger) through API, backend, database, and deploy config. List implicated files. Note “none” where a layer is untouched.

### Proposed fix and justification

The smallest reliable fix, why it addresses the root cause, and what it deliberately does not change. Label any workaround as unverified, not as the solution.

### Regression risks and edge cases

Nearby behavior, data integrity, authz, compatibility, and environment-specific failure modes the fix could disturb.

### Tests and verification steps

Regression tests to add or extend (follow existing Django and Vitest patterns), plus manual verification and measurable acceptance criteria.

### Remaining uncertainties and final recommendation

What is still unknown, the next diagnostic steps if any, and one recommendation: proceed with the proposed fix, gather listed evidence first, or pause.

## Quality bar

- A cause is “confirmed” only when evidence from this investigation supports it.
- Every behavioral claim is backed by a path, log, error, or labeled as a hypothesis.
- The proposed fix is the smallest change that addresses the root cause.
- The full report is written only to `analysis-result/<work-folder-id>/analysis.md`; chat does not contain the analysis.
- The work folder ID is stable so `task-breakdown` and `generate-dor` can find this bug.
- No application source, tests, docs, or config are modified by this skill.
- No secrets from `.env` or credentials files appear in the report.
