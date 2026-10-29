# Definition of Ready: Restrict channel create, unify username rules, and let `standard` join a sponsor

## 1. Document Metadata

| Field | Value |
| --- | --- |
| Task ID | T2 |
| Title | Restrict channel create, unify username rules, and let `standard` join a sponsor |
| Task type | Feature — backend / API / docs / Django tests |
| Related feature | Restrict create types, unify channel URLs, and allow chapters on all channels |
| Work folder ID | `2026-10-29-feature-unified-channel-types-and-routes` |
| Source analysis | `analysis-result/2026-10-29-feature-unified-channel-types-and-routes/analysis.md` |
| Parent task-breakdown | `task-result/2026-10-29-feature-unified-channel-types-and-routes/tasks.md` |
| Document status | **READY** |

## 2. Objective and Expected Outcome

Make the channels API match the approved create, username, list-filter, and sponsorship contracts.

After this task:

- `POST /api/v1/channel` accepts only `standard` and `sponsor`.
- Create usernames follow normal-user rules (3–30, `a-z0-9` only, no underscore, reserved `pannooyin` blocked, globally unique).
- Edit uses the same rules when the username **changes**. An **unchanged** stored `_sponsor` / `_agency` / `_book` username still saves.
- `channel_type` cannot be changed on edit (already true; must stay true).
- List/filter accepts `standard` and still accepts `agency`, `book`, `sponsor`, and `all`.
- `standard`, `agency`, and `book` may request and be accepted on a `sponsor`. A `sponsor` still cannot join another channel. `standard` → non-sponsor remains rejected.

## 3. Context and Existing Behavior

**Confirmed project facts**

- Create endpoint: `POST /api/v1/channel` (`ChannelCollectionView.post` → `ChannelCreateSerializer` → `create_channel`). Auth: bearer access token, active normal user.
- Create type allow-list is `CHANNEL_TYPE_VALUES` = `sponsor`, `agency`, `book` (`apps/channels/constants.py`). `validate_channel_type` rejects blank/missing/`partner` with `Invalid Channel Type`.
- Create username: `validate_channel_username` requires `CHANNEL_USERNAME_SUFFIXES` for that type (`_sponsor` / `_agency` / `_book`). Format comes from `validate_username_format` (`^[a-z0-9_]+$`, length 3–30). Reserved exact name `pannooyin` uses `is_reserved_username`. Duplicate check is `username_exists` (case-insensitive) in `create_channel`, message `Duplicate Username`.
- Edit: `PUT /api/v1/channel` (`ChannelEditSerializer` has no `channel_type`). `edit_channel` always re-runs `validate_channel_username` against the **existing** stored type, so a suffix is still required even when the username is unchanged.
- List filters: `ChannelListQuerySerializer` / `GetChannelsQuerySerializer` → `validate_channel_type_filter` using `CHANNEL_TYPE_FILTER_VALUES` = `all`, `sponsor`, `agency`, `book`. Unknown values 400 `Invalid Channel Type`. Used by `POST /api/v1/getmychannels` and `POST /api/v1/getchannels`.
- Sponsorship: `_sponsorship_join_allowed` allows only requesting `agency` or `book` against a target `sponsor`. Callers:
  - `join_channel` when `acting_channel_uuid` is set (`_assert_sponsorship_join_allowed`)
  - `_apply_channel_request_act` on accept of a channel-user join
  - `act_on_channel_request` accept-batch precheck
  - Failure messages: `Don't allow to join this channel` (request) and `Some channels can't process member acception` (accept of a disallowed pairing)
- There is no database check forbidding `standard` as `channel_members.user_uuid`; the gate is application code.
- `validate_normal_user_username` (`apps/users/validators.py`) applies format plus the underscore ban. Its docstring currently says channel usernames must not use that helper. T2 **must** reuse it for create and for changed usernames, but keep the channel API message `Invalid Username` (do not leak `Username must not contain underscores for normal user accounts.`).
- `POST /api/v1/user/validateusername` stays identity-agnostic (allows underscore). Do not change that endpoint’s behavior.
- Tests that POST-create `agency` / `book`: `apps/channels/tests/test_channel_api.py` (including `test_create_sponsor_and_agency_channels` and book-create cases) and `test_channel_limits.py` (`two_agency` / `three_book`). Sponsorship tests already model-seed channels via `_create_owned_channel`.

**Approved requirements**

- Create allow-list is `standard` and `sponsor` only.
- Create **and rename** usernames look like normal users. Unchanged legacy suffixed names remain valid.
- `standard` may join a `sponsor`. Keep `agency` / `book` → `sponsor`. `sponsor` still cannot join another channel.

## 4. Scope

**Included**

- `apps/channels/constants.py` — split create types from stored/filter types; keep suffix map only to **recognize** legacy names, not to require them on create or rename.
- `apps/channels/validators.py` — create-type validator; username validator for create and for changed edit usernames; unchanged-legacy bypass.
- `apps/channels/serializers.py` — `ChannelCreateSerializer.validate_channel_type` uses the create allow-list; list/filter serializers accept `standard`.
- `apps/channels/services.py` — `edit_channel` username rules; `_sponsorship_join_allowed` adds `standard` (call sites that already use the helper must keep using it).
- Channel API tests listed in section 13.
- Docs (API copy **and** `pann-oo-yin-web/api_docs/` copy) listed in section 6.
- Optional comment fix on `validate_normal_user_username` so it no longer forbids channel reuse.

**Excluded**

- Database constraint / `User.ChannelType` (T1).
- Flowers chapter gate (T3).
- Web UI, routes, create-dialog options, client username field (T4, T5).
- Changing `channel_type` on edit.
- Letting a sponsor join another channel.
- Changing `MAX_CHANNEL_COUNT`, profile image, description, owner-membership create, or `validateusername` rules.
- Username backfill.

## 5. Technical Implementation Requirements

**Constants (proposed)**

- Add `CHANNEL_TYPE_STANDARD = "standard"`.
- `CHANNEL_TYPE_CREATE_VALUES` = `standard`, `sponsor`.
- Stored/filter types include historical values: `standard`, `sponsor`, `agency`, `book`.
- `CHANNEL_TYPE_FILTER_VALUES` = `all` plus those stored types.
- `CHANNEL_USERNAME_SUFFIXES` remains for `_agency` / `_sponsor` / `_book` so legacy names can be recognized if needed; **do not** require a suffix on create or rename. Do not add `_standard`.

**Create type validation**

- `validate_channel_type` (used only by `ChannelCreateSerializer`) must accept only create values, case-insensitive, trimmed.
- `agency`, `book`, `partner`, blank, missing, and any other value → 400 `Invalid Channel Type` (`MSG_INVALID_CHANNEL_TYPE`). Serializer field `error_messages` for required/blank/null already use that string; keep them.

**Create username validation**

- Reuse `validate_normal_user_username` plus `is_reserved_username`.
- Catch those validation errors and re-raise `MSG_INVALID_USERNAME` (`Invalid Username`).
- Valid example: type `standard` or `sponsor`, username `acme` (unused).
- Invalid examples: `acme_sponsor`, any `_`, `pannooyin`.
- Duplicate still happens in `create_channel` via `username_exists` → `Duplicate Username`. Do not weaken case-insensitive uniqueness.

**Edit username validation**

- Normalize the submitted username (trim + lowercase, same as today).
- If it equals the channel’s **currently stored** username, accept it, including `_sponsor` / `_agency` / `_book`. Skip the underscore ban and suffix requirement.
- If it differs, apply the same normal-user + reserved-name rules as create, still mapped to `Invalid Username`.
- Changing `brand_sponsor` → `acme` succeeds (if unused). Changing to `acme_sponsor` or any underscore name fails.
- Duplicate against other users still `Duplicate Username` (`username_exists(..., exclude_uuid=channel.uuid)`).
- Do not add `channel_type` to `ChannelEditSerializer`.

**List / filter**

- `validate_channel_type_filter` must accept `standard` (and still `all`, `sponsor`, `agency`, `book`).
- `null` / omitted / blank / `all` still mean no type restriction.
- Unknown values still 400 `Invalid Channel Type`.
- Detail/list responses already return stored `channel_type`; they will start returning `standard` once such rows exist. No response-schema code change beyond filter validation unless a hardcoded three-value tuple exists in channels code (today the filter tuple is the gate).

**Sponsorship**

- `_sponsorship_join_allowed`: requesting channel type in `{standard, agency, book}` **and** target type `sponsor`. Both must be channel users (`is_channel_user`).
- `sponsor` → any channel: still false → `Don't allow to join this channel`.
- `standard` / `agency` / `book` → non-sponsor: still false.
- Accept path: a valid `standard` → `sponsor` pending request, when accepted, creates membership (`_ensure_membership`), same as agency/book today.
- Do not change invite-token URLs, owner-rejoin, or normal-user join.

**`create_channel` persistence**

- Continues to `User.objects.create(..., channel_type=channel_type, ...)`. After T1 this can persist `standard`. Do not add extra suffix logic in the service.

## 6. Affected Files and Components

**Modify**

| Path | Responsibility |
| --- | --- |
| `pann-oo-yin-api/apps/channels/constants.py` | Create vs stored/filter type sets; optional `CHANNEL_TYPE_STANDARD` |
| `pann-oo-yin-api/apps/channels/validators.py` | Create-type allow-list; create/rename username rules; unchanged-legacy bypass |
| `pann-oo-yin-api/apps/channels/serializers.py` | Create type field uses create validator; list/filter already call `validate_channel_type_filter` |
| `pann-oo-yin-api/apps/channels/services.py` | `edit_channel`; `_sponsorship_join_allowed` (and keep the three existing callers on that helper) |
| `pann-oo-yin-api/apps/users/validators.py` | Optional docstring update on `validate_normal_user_username` only |
| `pann-oo-yin-api/apps/channels/tests/test_channel_api.py` | Create allow-list, unsuffixed success, suffixed create failure, edit of seeded historical types |
| `pann-oo-yin-api/apps/channels/tests/test_channel_limits.py` | Fill max count without POST-creating `agency`/`book` |
| `pann-oo-yin-api/apps/channels/tests/test_channel_invite_check_and_sponsorship.py` | `standard` request + accept on a sponsor; keep agency/book and sponsor-rejected cases |
| `pann-oo-yin-api/apps/channels/tests/test_channel_request_api.py` | Same sponsorship coverage as needed for request/act APIs |

**Docs — update both trees** (`pann-oo-yin-api/docs/` and `pann-oo-yin-web/api_docs/`):

| Doc | Required contract text |
| --- | --- |
| `channels/create_channel.md` | Create types `standard`/`sponsor`; username = normal-user rules; constraint includes `standard`; create restriction is application-enforced |
| `channels/edit_channel.md` | Rename uses normal-user rules; unchanged legacy suffixed usernames remain valid; type cannot be changed |
| `channels/get_channels.md` | Filters/responses include `standard`; historical `agency`/`book` remain |
| `channels/get_my_channels.md` | Same filter enum |
| `channels/get_channel_detail.md` | Response `channel_type` includes `standard` while still listing `agency`/`book` |
| `channels/join_channel.md` | Requesters `standard`/`agency`/`book`; target `sponsor` |
| `channels/act_channel_request.md` | Same sponsorship pairing |
| `channels/get_channel_requests.md` | Stored `channel_type` on user_info/channel_info may be `standard` |
| `user/validate_username.md` | Still identity-agnostic; note that channel **create/rename** now apply the normal-user underscore ban in the channel API, not in this endpoint |
| `Pann-Oo-Yin-Database-README.md` (both copies) | Create vs historical usernames; create restriction is application-enforced; membership `user_uuid` may be a normal user or `standard` / `agency` / `book`, never a `sponsor`; channel-to-channel membership is `standard`/`agency`/`book` → `sponsor` |

**Do not create** new endpoints or new serializers unless a tiny helper in `validators.py` is clearer than overloading `validate_channel_username`.

## 7. Business Rules and Data Requirements

| Action | Allowed types | Username |
| --- | --- | --- |
| Create | `standard`, `sponsor` | Normal-user rules; no suffix |
| Edit (username unchanged) | Stored type unchanged | Stored value accepted, including `_sponsor` / `_agency` / `_book` |
| Edit (username changed) | Stored type unchanged | Normal-user rules; no suffix |
| List/filter | `all`, `standard`, `sponsor`, `agency`, `book` | N/A |
| Sponsorship request/accept | Requester `standard`/`agency`/`book`; target `sponsor` | N/A |

- ORM may still insert `agency`/`book` (tests and historical data). Only the create API is restricted.
- No `users` row rewrite. No `channel_members` schema change.
- Global uniqueness index `users_username_lower_uniq` is the duplicate source of truth.
- `MAX_CHANNEL_COUNT` unchanged. Limit tests must still fill three channels, then fail the fourth.

## 8. API and Integration Contracts

**`POST /api/v1/channel`** (intentionally breaking for `agency`/`book` and suffixed names)

| Request | Result |
| --- | --- |
| `channel_type=standard` or `sponsor`, username `acme` unused, 3–30 `[a-z0-9]+`, valid name | 201, stores that type/username |
| `agency`, `book`, `partner`, blank, missing type | 400 `Invalid Channel Type` |
| `acme_sponsor`, any `_`, `pannooyin` | 400 `Invalid Username` |
| Username equal to an existing flower/bee/florist/channel username (any case) | 400 `Duplicate Username` |

Auth, profile image, description, owner-membership create, and limit behavior stay as today.

**`PUT /api/v1/channel`**

| Request | Result |
| --- | --- |
| Seeded `brand_sponsor` (or `_agency` / `_book`) with the same username | Success (`Channel Edition Successfully`) |
| Change username to unused `acme` | Success |
| Change to `acme_sponsor` or any underscore name | 400 `Invalid Username` |
| Body includes a new `channel_type` | Ignored / not accepted (field is not on the serializer) |

**`POST /api/v1/getchannels` and `POST /api/v1/getmychannels`**

- `channel_type=standard` is valid and filters to those rows.
- `agency` / `book` / `sponsor` / `all` remain valid.

**`POST /api/v1/channels/join/{uuid}`** with body `channel_uuid` = owned `standard` channel, path UUID = active sponsor: success path same as agency/book (`Join Channel Requested` / membership lifecycle). Sponsor acting channel → warning `Don't allow to join this channel`.

**Accept:** `standard` join request on a sponsor creates membership. Invalid pairings still `Some channels can't process member acception` on batch accept.

## 9. Dependencies and Prerequisites

- **T1 must be applied** on the database used by these tests before any `standard` insert. Otherwise `User.objects.create(channel_type="standard")` and create-API success hit `users_channel_type_check`.
- T3 is independent of this contract except that it may model-seed `standard` (also needs T1).
- T5 must not ship create-UI for `standard` until this API exists; T5 lists T2 as a dependency.
- Web routes for `/channels/@username` are T4; this task does not require them.

**Dependency contract for T5:** create only `standard`/`sponsor` with unsuffixed usernames; edit unchanged-legacy bypass; sponsorship eligibility `standard`/`agency`/`book` → `sponsor`.

## 10. Edge Cases and Failure Handling

- Whitespace/case: types and usernames are trimmed and lowercased (existing create type behavior). Unchanged-username compare must use the normalized form so `Brand_Sponsor` on stored `brand_sponsor` still counts as unchanged.
- Empty base suffix cases (`_sponsor`) are invalid on **create** and on **changed** usernames because of the underscore ban and/or format rules.
- `validateusername` still allows `john_01`. The channel create/edit API must reject it. The web field (T5) must also reject it client-side; this task is API-only.
- Reserved suffixes `_bot` / `_partner` are invalid for new channel names because they contain `_`.
- Concurrent duplicate username still maps `IntegrityError` to `Duplicate Username` (existing `create_channel` / `edit_channel` handling).
- Accept-batch must keep using `_sponsorship_join_allowed` so a `standard` request is not treated as `MSG_SOME_CHANNELS_CANT_ACCEPT`.

## 11. Security and Backward Compatibility

- Create contract is **intentionally breaking** for clients that POST `agency`/`book` or suffixed usernames. The in-repo web app is updated in T5 in the same release.
- Read/list/detail/join stay compatible for historical types. Adding `standard` to filters is additive.
- Unchanged-legacy bypass is required so owners can save other fields without being forced to rename.
- Do not add a DB check that forbids `agency`/`book`.
- Username namespace is already shared with people; duplicates remain 400. Normal users still cannot use reserved suffixes, so they cannot impersonate `brand_sponsor`.
- No new env vars. No token/auth change.

## 12. Acceptance Criteria

- `POST /api/v1/channel` with `standard` or `sponsor` and username `acme` (unused, 3–30, `[a-z0-9]+`) returns 201.
- Create with `agency`, `book`, `partner`, blank, or missing type returns 400 `Invalid Channel Type`.
- Create with `acme_sponsor`, any `_`, or `pannooyin` returns 400 `Invalid Username`.
- Create username equal to an existing flower/bee/florist/channel username (any case) returns 400 `Duplicate Username`.
- Edit of a seeded `brand_sponsor` (or `_agency` / `_book`) with the same username succeeds; change to `acme` succeeds; change to `acme_sponsor` or any underscore name returns 400 `Invalid Username`.
- Type cannot be changed on edit.
- List/filter accepts `standard`, `sponsor`, `agency`, `book`, `all`.
- A model-seeded `standard` owner can request join on a `sponsor` and accept creates membership. `agency`/`book` still can. `sponsor` → any channel still `Don't allow to join this channel`. `standard` → non-sponsor still rejected.
- Docs listed above are updated in **both** trees.

## 13. Testing and Verification

Follow existing Django `APIClient` / `TestCase` style.

- `test_channel_api.py`
  - Change `_create_channel` defaults so success paths use unsuffixed `standard`/`sponsor` (today default `abc_sponsor` would 400).
  - POST-create `agency`/`book` becomes 400 `Invalid Channel Type`.
  - Add unsuffixed `standard`/`sponsor` 201 cases.
  - Seed historical types with `User.objects.create` (channel-shaped fields) for edit, list, detail, delete, and filter tests that still need agency/book rows.
  - Keep suffix-mismatch and reserved-name coverage, but as **create 400 Invalid Username**, not as wrong-suffix-for-type.
  - Edit tests: unchanged `_sponsor` save; rename to unsuffixed; rename to underscore fails.
- `test_channel_limits.py`
  - `test_limit_blocks_fourth_channel` must not POST-create `two_agency` / `three_book`. Fill with create of `standard`/`sponsor` and/or model-seeded agency/book.
- `test_channel_invite_check_and_sponsorship.py` and `test_channel_request_api.py`
  - Extend `_create_owned_channel` usage with a `standard` channel.
  - Assert `standard` can request and be accepted on a sponsor.
  - Keep agency/book success and sponsor-rejected cases.

Do not run migrations against Neon from this task. Local tests assume T1’s 0017 is applied.

## 14. Risks, Assumptions, and Open Questions

**Assumptions**

- Channel API messages stay `Invalid Username` / `Invalid Channel Type` / `Duplicate Username` even when the underlying helper is `validate_normal_user_username`.
- Suffix map is retained only for documentation/recognition, not for requiring suffixes.
- `CHANNEL_TYPE_VALUES` if left as the create list would wrongly keep allowing `agency`/`book`; it must be split.

**Risks**

- Leaving any POST-create agency/book fixture in tests will fail after the allow-list change.
- Applying username rules on edit without the unchanged-value bypass blocks saving legacy channels.
- Updating `_sponsorship_join_allowed` but documenting only agency/book in join/act docs / README leaves T5 and operators with a split brain.

**Open questions**

- None that block implementation. Product already resolved sponsorship for `standard` and no suffix on rename.

## 15. Definition of Ready Checklist

- [x] T1 is the schema prerequisite; this DoR states that inserts of `standard` require it.
- [x] Create vs stored/filter types specified.
- [x] Create and rename username rules specified, including unchanged-legacy bypass and message mapping.
- [x] Sponsorship helper and both join/accept outcomes specified.
- [x] Breaking create contract and additive read/filter contract documented.
- [x] Tests and dual-tree docs listed.
- [x] Status is READY.
