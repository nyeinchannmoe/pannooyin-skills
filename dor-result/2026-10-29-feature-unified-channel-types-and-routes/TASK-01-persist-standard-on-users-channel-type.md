# Definition of Ready: Persist `standard` on `users.channel_type`

## 1. Document Metadata

| Field | Value |
| --- | --- |
| Task ID | T1 |
| Title | Persist `standard` on `users.channel_type` |
| Task type | Feature — database / Django model |
| Related feature | Restrict create types, unify channel URLs, and allow chapters on all channels |
| Work folder ID | `2026-10-29-feature-unified-channel-types-and-routes` |
| Source analysis | `analysis-result/2026-10-29-feature-unified-channel-types-and-routes/analysis.md` |
| Parent task-breakdown | `task-result/2026-10-29-feature-unified-channel-types-and-routes/tasks.md` |
| Document status | **READY** |

## 2. Objective and Expected Outcome

Expand the stored channel-type allow-list so a `users` row can persist `channel_type=standard` without rewriting existing rows.

After this task:

- Django `User.ChannelType` choices are `sponsor`, `agency`, `book`, and `standard`.
- Check constraint `users_channel_type_check` allows `NULL | sponsor | agency | book | standard`.
- Inserting `standard` succeeds locally.
- Existing `sponsor` / `agency` / `book` / `NULL` rows still satisfy the check.
- No `users` row is updated by the migration.
- Reverse SQL restores the **0016 forward** constraint (still includes `book`) and does not drop `book`.

This task does not change the create API, username rules, web types, sponsorship, or chapters.

## 3. Context and Existing Behavior

**Confirmed project facts**

- Channels are `users` rows with `channel_type` set and `user_type` / `auth_provider` / `social_id` null (`pann-oo-yin-api/apps/users/models.py`, `pann-oo-yin-api/apps/channels/validators.py` `is_channel_user`).
- Stored values today: `NULL`, `sponsor`, `agency`, `book`. `standard` is absent from models, constraints, API constants, and frontend types.
- `users.channel_type` is `varchar(10)` (migration `0011_user_channel_nullable_fields_db`). `standard` is 8 characters and fits.
- Live constraint last expanded in `pann-oo-yin-api/apps/users/migrations/0016_user_channel_type_book_check.py`. That file is currently the latest users migration. Suggested next name: `0017_user_channel_type_standard_check`.
- 0016 uses `SeparateDatabaseAndState`: Django state `AddConstraint` plus `RunSQL` `DROP CONSTRAINT IF EXISTS` / `ADD CONSTRAINT` with `ARRAY['sponsor', 'agency', 'book']`. Reverse SQL of **0016** restores only `sponsor` and `agency` (drops `book`). The new migration must not copy that reverse; T1 reverse must restore the 0016 **forward** allow-list including `book`.
- Model `Meta.constraints` currently encodes `NULL | sponsor | agency | book` as `users_channel_type_check`.
- Model comment on `user_type` still says “NULL user_type = channel user (sponsor/agency)” and omits `book` / `standard`.
- Django admin (`apps/users/admin.py`) does not list `channel_type`; no admin change is required.
- Database README copies agree with the live constraint today:
  - `pann-oo-yin-api/docs/Pann-Oo-Yin-Database-README.md`
  - `pann-oo-yin-web/api_docs/Pann-Oo-Yin-Database-README.md`

**Approved requirement:** add stored value `standard`. Do not remove `agency` or `book`. No row backfill.

## 4. Scope

**Included**

- `User.ChannelType.STANDARD = "standard"` and updated `channel_type` choices.
- Expand `users_channel_type_check` in model `Meta` and in a new users migration after `0016`.
- Update the model comment so channel users are described as sponsor / agency / book / standard.
- Update both Database README copies so stored `channel_type` values and the check-constraint text include `standard` and mention migration 0017.

**Excluded**

- Create-API allow-list, username suffix rules, list/filter enums (T2).
- Flowers chapter gate (T3).
- Web types, routes, create UI, filters, invite eligibility (T4, T5).
- Row backfill or renaming existing `agency` / `book` / `sponsor` rows.
- Removing `agency` or `book` from the constraint.
- Applying this migration to Neon from this task.
- Changing `MAX_CHANNEL_COUNT`, profile image, description, owner-membership create, or env vars.

## 5. Technical Implementation Requirements

**Database**

- Table/column: `users.channel_type` (`varchar(10)`, nullable).
- Constraint name: `users_channel_type_check` (keep the name; replace the expression).
- New allowed values: `NULL`, `sponsor`, `agency`, `book`, `standard`.
- No `UPDATE` of existing rows. No enum type. No new index.

**Proposed migration shape** (follow 0016, adapted because the constraint already exists in Django state):

1. `AlterField` on `User.channel_type` so Django choices include `standard` (same class of change as `0015_user_channel_type_book`).
2. `SeparateDatabaseAndState`:
   - **State:** `RemoveConstraint` `users_channel_type_check`, then `AddConstraint` with `Q(channel_type__isnull=True) | Q(channel_type__in=["sponsor", "agency", "book", "standard"])`.
   - **Database:** `RunSQL` that `DROP CONSTRAINT IF EXISTS users_channel_type_check` and `ADD CONSTRAINT` checking `channel_type IS NULL OR channel_type::text = ANY (ARRAY['sponsor', 'agency', 'book', 'standard']::text[])`.
   - **Reverse SQL:** drop the constraint if it exists, then re-add the **0016 forward** check: `NULL | sponsor | agency | book`. Reverse must still allow `book`.

**Backend**

- `pann-oo-yin-api/apps/users/models.py`: add `ChannelType.STANDARD`; add `"standard"` to the check-constraint `channel_type__in` list; fix the `user_type` comment so it no longer implies only sponsor/agency.
- Do not change `apps/channels/*` or `apps/flowers/*` in this task.

**Frontend / API contract**

- Not applicable for behavior. README stored-value text only.

**Validation**

- Application create validation stays on T2. After T1, ORM / tests may insert `standard`; `POST /api/v1/channel` still rejects it until T2.

## 6. Affected Files and Components

**Modify**

| Path | Responsibility |
| --- | --- |
| `pann-oo-yin-api/apps/users/models.py` | `ChannelType` choices, check constraint values, channel-user comment |
| `pann-oo-yin-api/docs/Pann-Oo-Yin-Database-README.md` | Stored `channel_type` values and `users_channel_type_check` / migration note |
| `pann-oo-yin-web/api_docs/Pann-Oo-Yin-Database-README.md` | Same stored-value text as the API README |

**Create**

| Path | Responsibility |
| --- | --- |
| `pann-oo-yin-api/apps/users/migrations/0017_user_channel_type_standard_check.py` (or the next unused number after 0016) | Django state + `RunSQL` constraint expansion and reverse |

If 0017 already exists when implementation starts, use the next unused number and record that name in the migration file and README. Do not invent a second constraint name.

## 7. Business Rules and Data Requirements

- `standard` is a **new** stored value, not a rename of `agency` or `book`.
- Historical `sponsor` / `agency` / `book` / `NULL` rows remain valid.
- Direct ORM/admin inserts of `agency` / `book` remain possible; this task does not restrict writes at the database layer.
- Column length 10 is sufficient; do not widen the column.
- Unique username index `users_username_lower_uniq` is unchanged.
- `flower_posts`, `channel_members`, `channel_requests`, and `created_channel_count` are unchanged.

## 8. API and Integration Contracts

Not applicable. This task does not change HTTP endpoints, request fields, or response shapes. Create still uses `CHANNEL_TYPE_VALUES` (`sponsor`, `agency`, `book`) until T2.

## 9. Dependencies and Prerequisites

- No prior feature task. This is the first task in the breakdown.
- Local PostgreSQL (or the project’s existing test database) must be able to apply users migrations through 0016 before 0017.
- Do not apply 0017 to Neon from this task. Neon deploy happens with the API release that also ships T2 (same class of change as 0016).

**Contract for later tasks:** after T1, `User.ChannelType.STANDARD` and the live check allow inserting `standard`. T2, T3, and T4 may model-seed `standard` rows. T2 is the first task that may insert `standard` through `POST /api/v1/channel`.

## 10. Edge Cases and Failure Handling

- Reverse of 0017 must not restore the pre-book 0016 reverse (`ARRAY['sponsor', 'agency']`). That would make existing `book` rows fail the check.
- `DROP CONSTRAINT IF EXISTS` must be used so local databases that already match Django state and Neon-style databases both apply cleanly (same reason 0016 used `IF EXISTS`).
- Do not add `standard` by dropping `agency` or `book` from the `ANY` array.
- Inserting an unknown value such as `partner` must still fail the check.

## 11. Security and Backward Compatibility

- Constraint expansion is additive. Existing rows stay valid.
- No data migration, no username rewrite, no secret/env change.
- Application create remains restricted to historical types until T2, so production traffic that does not insert `standard` is unaffected by the new allow-list.

## 12. Acceptance Criteria

- Constraint allows `standard` (8 characters; column is `varchar(10)`).
- Existing `sponsor` / `agency` / `book` / `NULL` rows still satisfy the check.
- Reverse SQL restores the 0016 forward constraint (`NULL | sponsor | agency | book`) and does not drop `book`.
- No `users` row is updated by the migration.
- `User.ChannelType.STANDARD` exists and is `"standard"`.
- Both Database README copies list `standard` among stored `channel_type` values and document that `users_channel_type_check` allows `NULL`, `sponsor`, `agency`, `book`, and `standard` via 0017.
- Username-suffix bullets and create-API restriction wording are left for T2 unless a stored-value sentence would otherwise claim `standard` does not exist.

## 13. Testing and Verification

- Apply the new migration against a **local** database (or a Django migration test). Do not run it against Neon from this task.
- Insert a `User` with `channel_type=standard` (channel-shaped: `user_type` / `auth_provider` / `social_id` null). Confirm it saves.
- Confirm a pre-existing agency, book, sponsor, and normal-user (`channel_type` NULL) fixture still loads / still satisfies the check.
- Reverse the migration and re-apply. After reverse, inserting `standard` must fail and inserting `book` must still succeed.
- Query or inspect `pg_constraint` / `\d users` if needed to confirm the check expression includes `standard` after apply.

No new Vitest. No requirement to add a dedicated Django test module if the local migrate / insert / reverse cycle is executed and recorded; a small migration test is acceptable if the API repo already has that pattern.

## 14. Risks, Assumptions, and Open Questions

**Assumptions**

- 0016 remains the latest users migration at implementation time. If another users migration lands first, use the next number and keep the 0016-style `RunSQL` pattern.
- README username-must-suffix sentences stay until T2; T1 only adds `standard` to stored values and the constraint/migration note.

**Risks**

- Copying 0016’s reverse SQL verbatim would drop `book` on rollback.
- Shipping T2 inserts before this migration is on the target database causes a check-constraint violation.

**Open questions**

- None that block implementation.

## 15. Definition of Ready Checklist

- [x] Approved task T1 exists with clear schema-only scope.
- [x] Live constraint, column type, and latest migration number confirmed in source.
- [x] Reverse behavior specified (restore 0016 forward, keep `book`).
- [x] No row backfill; no Neon apply from this task.
- [x] README copies and model comment identified.
- [x] Dependencies for T2/T3/T4 (`ChannelType.STANDARD` insertable) stated.
- [x] Status is READY.
