# Retire `agency` and `book` channel types after backfilling them to `standard`

Work folder ID: `2026-10-29-bug-retire-agency-book-channel-types`

### Work folder

Work folder ID: `2026-10-29-bug-retire-agency-book-channel-types`

Later skills must reuse these paths:

- Analysis: `analysis-result/2026-10-29-bug-retire-agency-book-channel-types/analysis.md`
- Tasks (next): `task-result/2026-10-29-bug-retire-agency-book-channel-types/tasks.md`
- DoRs (after task approval): `dor-result/2026-10-29-bug-retire-agency-book-channel-types/`

Related prior work (do not reuse that folder): `2026-10-29-feature-unified-channel-types-and-routes` added `standard`, restricted **create** to `standard`/`sponsor`, and **explicitly kept** historical `agency`/`book` rows and filters.

### Bug summary and expected versus actual behavior

**Who/what is affected:** Every `users` row with `channel_type` `agency` or `book` (active and inactive), plus every list/filter/detail/sponsorship/docs/UI path that still treats those values as first-class stored types.

**Expected (this request):**

1. Existing stored `agency` and `book` channels become `standard`.
2. After that rewrite, `agency` and `book` are removed from the database allow-list and from product support (create already rejected; list/filter, model choices, sponsorship special-cases, UI chips, and docs must stop treating them as live types).

**Actual (current tree, after the unified-types feature):**

- Create already rejects `agency`/`book` (`POST /api/v1/channel` → 400 `Invalid Channel Type`).
- Stored types, check constraint, Django choices, list/filter enums, sponsorship join, and web filters still include `agency` and `book`.
- No data migration rewrites existing rows. Historical agency/book channels still load, filter, edit, post chapters, and join sponsors as those types.

**Environment / severity:** Product follow-on, not a runtime crash. Severity is data + contract: production rows and public APIs still emit `agency`/`book`. This is a breaking read/filter change once those values are removed.

This is **not** a defect in the previous feature. That work’s approved scope was “keep historical types; no row backfill.” The gap is that product now wants the excluded next step.

### Reproduction steps and investigation findings

No crash to reproduce. Confirmed by inspecting the live API, web, and schema (read-only).

**Database / model (implementation truth)**

- Channels are `users` rows with `channel_type` set and `user_type` / `auth_provider` / `social_id` null (`pann-oo-yin-api/apps/users/models.py`, `apps/channels/validators.py` `is_channel_user`).
- `User.ChannelType` choices today: `sponsor`, `agency`, `book`, `standard`.
- `users.channel_type` is `varchar(10)`. Check constraint `users_channel_type_check` allows `NULL | sponsor | agency | book | standard` (`models.py` `Meta.constraints`; live SQL in `apps/users/migrations/0017_user_channel_type_standard_check.py`).
- Latest users migration is **0017**. Next unused number is **0018**.
- 0017 **does not** `UPDATE` rows. It only expands the check.
- Both Database README copies still list stored values including `agency`/`book` and say the create restriction is application-enforced while the database still stores historical rows:
  - `pann-oo-yin-api/docs/Pann-Oo-Yin-Database-README.md`
  - `pann-oo-yin-web/api_docs/Pann-Oo-Yin-Database-README.md`

**Create vs stored vs filter (API)**

- Create allow-list is already `standard`/`sponsor` (`apps/channels/constants.py` `CHANNEL_TYPE_CREATE_VALUES`; `validators.py` `validate_channel_type`). Tests assert POST `agency`/`book`/`partner` → 400 (`apps/channels/tests/test_channel_api.py`).
- Filter allow-list still includes `agency`/`book` (`CHANNEL_TYPE_FILTER_VALUES`). `GET`/`POST` list with `channel_type=agency|book` is valid and returns those rows.
- `CHANNEL_TYPE_VALUES` and `CHANNEL_USERNAME_SUFFIXES` in `constants.py` still mention agency/book. Neither name is imported by other Python modules after the unified-types split (dead stored-type leftovers).
- `_sponsorship_join_allowed` still allows requesters `standard` **or** `agency` **or** `book` onto a `sponsor` (`apps/channels/services.py` ~429–441, also used at join ~1286 and accept-batch ~1879 / ~1956).

**Chapters / membership (why backfill is safe for those features)**

- Chapter create/update already allows any channel user (`apps/flowers/services.py` `_assert_chapter_allowed_for_flower_user` uses `is_channel_user`, not `BOOK`). Converting a book channel to `standard` does not drop chapters; uniqueness remains `(flower_user, chapter_number)`.
- `channel_members` and `channel_requests` key off UUIDs, not `channel_type`. Existing agency/book → sponsor memberships and pending join requests remain valid after the type column changes to `standard`, because `standard` is already a legal sponsorship requester.
- Edit still cannot change `channel_type` (`docs/channels/edit_channel.md`). Conversion must be a migration, not the edit API.

**Frontend still treats agency/book as live types**

- `src/types/channel.ts` `ChannelType` = `'sponsor' | 'agency' | 'book' | 'standard'`.
- `src/services/channelService.ts` `isChannelType` accepts `agency`/`book`. If those values are removed from the union **before** rows are rewritten, list/detail mapping returns `null` and hides leftover rows.
- Filter chips still offer Agency / Book: `ChannelsPanel.tsx`, `ChannelListPage.tsx`, `ProfileChannelsSection.tsx`.
- `profileChannelsVisibility.ts` `SPECIFIC_CHANNEL_TYPES` includes them.
- `channelInviteJoin.ts` `isSponsorshipEligibleOwnedChannel` includes them.
- Copy still promises historical listing: Florist dashboard “Existing agency and book channels remain listed.”; Channels panel help; Flower dashboard “Sponsor and agency channels”; i18n `noOwnedJoinChannelsMessage` “Standard, Agency, or Book”.
- Canonical URL is already `/channels/@username`. Legacy singular prefixes `/sponsor|agency|book/@username` (and chapter sub-routes) **redirect** (`src/App.tsx` `LegacyTypePrefixedChannelRedirect`, `channelRoutes.ts` `legacyPrefixedChannelRedirectPath`). Plural `/agencies/*` `/books/*` already bounce to `/channels`.

**Usernames (not part of the type column)**

- Create/rename already use normal-user rules (no underscore). Unchanged historical `_agency` / `_book` / `_sponsor` usernames still save (`validators.py` `validate_channel_username` stored-username bypass).
- Reserved suffixes for **normal users** still include `_agency` and `_book` (`apps/users/constants.py` `RESERVED_USERNAME_SUFFIXES`; web `src/utils/usernameRules.ts`). Tests in florist/flower/user suites reject `myname_agency` / `myname_book`.
- The request does not ask to rewrite usernames.

**Tests that still seed `User.ChannelType.AGENCY` / `BOOK`**

These would violate `users_channel_type_check` after the constraint shrinks, unless fixtures change:

- `apps/channels/tests/test_channel_api.py` (list/filter agency/book, edit of `talent_agency` / `story_book`)
- `apps/channels/tests/test_channel_request_api.py`
- `apps/channels/tests/test_channel_invite_check_and_sponsorship.py` (`house_agency`, `house_book`, accept-agency-and-book)
- `apps/channels/tests/test_channel_limits.py` (`mix3_agency`)
- `apps/flowers/tests/test_chapter_posts_api.py`, `test_flower_posts_api.py`, `test_juice_posts_api.py`
- `apps/content_analysis/tests/test_ban_content_api.py`
- `apps/florists/tests/test_florist_api.py`

Web Vitest: `channelRoutes.test.ts`, `channelService.test.ts`, `channelInviteJoin.test.ts`, `profileChannelsVisibility.test.ts` still assert agency/book as stored/filter types. `usernameRules.test.ts` only uses `_book` as an **underscore** example, not as a live type.

Row counts in Neon were **not** queried (analysis must not write to or depend on production credentials). The migration must update **all** matching `users` rows, including `inactive=true`.

### Root cause with supporting evidence, or clearly labeled hypotheses

**Confirmed cause of the remaining `agency`/`book` support:** the previous approved design kept those values as historical stored types. Evidence:

- Constraint and model choices still list them (`users/models.py`, migration 0017).
- Filter validator still accepts them (`validators.py` + `CHANNEL_TYPE_FILTER_VALUES`).
- Sponsorship still names them (`_sponsorship_join_allowed`).
- Web union, chips, labels, and invite eligibility still include them.
- Docs and both Database READMEs still describe them as stored types.

There is **no** separate bug that prevents an `UPDATE` of `channel_type` to `standard`. The check already allows `standard`. The edit API simply does not accept `channel_type`, so a Django data migration is required.

**Not a root cause:** chapter permission, unified URLs, or create-API rejection. Those already match the unified-types plan. Retiring the types does not require reversing T3/T4.

**Hypothesis (does not block the plan):** some clients or bookmarks still call list with `channel_type=agency|book`. After filter removal those requests should 400 `Invalid Channel Type` (same as `partner` today). Unfiltered list will return the rewritten rows as `standard`.

### Affected files and end-to-end execution flow

Trigger is a **one-time data rewrite + constraint shrink**, then application code that no longer accepts or displays the old types.

1. **Deploy/migration (Neon with API):** `0018` (suggested name `0018_user_channel_type_retire_agency_book`) `UPDATE` then replace `users_channel_type_check`.
2. **API reads/writes:** create stays `standard`/`sponsor`; list/filter/detail/join no longer accept or emit `agency`/`book`.
3. **Web:** chips, `isChannelType`, labels, invite eligibility, copy. Public path stays `/channels/@username`.
4. **Docs:** API `docs/` + web `api_docs/` copies and both Database READMEs.

**Database**

- `pann-oo-yin-api/apps/users/models.py` — drop `AGENCY`/`BOOK` choices; check `channel_type__in=["sponsor", "standard"]`; comment “channel user (sponsor/standard)”.
- New `pann-oo-yin-api/apps/users/migrations/0018_user_channel_type_retire_agency_book.py` (or next unused number after 0017).
- README copies as above.

**API / backend**

- `apps/channels/constants.py` — stored/filter types = `standard`, `sponsor` (+ `all` for filters). Remove agency/book constants and unused `CHANNEL_USERNAME_SUFFIXES` / unused `CHANNEL_TYPE_VALUES` if nothing imports them.
- `apps/channels/services.py` — `_sponsorship_join_allowed` requester allow-list = `standard` only (target remains `sponsor`).
- `apps/channels/validators.py` — no logic change beyond constants.
- Tests listed above: stop seeding `AGENCY`/`BOOK`; keep POST-create `agency`/`book` → 400; keep reserved-suffix tests `_agency`/`_book` for **normal users**.
- Flower chapter tests can keep a chapter fixture on a **standard** (or sponsor) channel; rename `garden_book` usernames is optional (username ≠ type).

**Frontend**

- `src/types/channel.ts` — `ChannelType` = `'sponsor' | 'standard'`.
- `src/services/channelService.ts` — `isChannelType` matches.
- `src/utils/channelRoutes.ts` — labels/parse/query only `standard`/`sponsor`; **keep** `/agency/@x` and `/book/@x` redirects (URL compatibility, not type support). `channelTypeFromPath` may still map those prefixes for redirect helpers only.
- Filter option arrays in `ChannelsPanel.tsx`, `ChannelListPage.tsx`, `ProfileChannelsSection.tsx`.
- `profileChannelsVisibility.ts`, `channelInviteJoin.ts`.
- Copy: `FloristDashboardPage.tsx`, `FlowerDashboardPage.tsx`, `ChannelsPanel.tsx`, `src/i18n/en.ts` `noOwnedJoinChannelsMessage`, `src/i18n/my.ts` matching string.
- `usernameRules.ts`: keep `RESERVED_USERNAME_SUFFIXES` including `_agency`/`_book`. Historical suffix **detector** map may keep `_agency`/`_book` as reserved-name knowledge or drop `ChannelType` that still lists agency/book if unused after chips go away. Do not start requiring suffixes again.
- Vitest files listed above.
- `FlowerPostsGrid.tsx` default `channelType = 'book'` is a leftover default for chapter URL building; change the default to `'standard'` (or drop the unused default) so the web type union compiles.

**Deploy**

- `vercel.json`: no change. SPA rewrite already covers `/channels/@username`.
- Env vars: none.
- Neon: apply 0018 with the API that shrinks filters/choices.

**None:** auth/token, Cloudinary, `MAX_CHANNEL_COUNT`, `flower_posts` schema, `channel_members` schema, invite-token URLs, chapter uniqueness, Django admin (`apps/users/admin.py` does not list `channel_type`).

### Proposed fix and justification

Smallest reliable fix: **one users migration that rewrites then shrinks**, plus the application/docs/tests that still name `agency`/`book` as stored/filter types.

**Migration 0018 (same `SeparateDatabaseAndState` + `RunSQL` pattern as 0016/0017), in one database operation sequence:**

1. `UPDATE users SET channel_type = 'standard' WHERE channel_type IN ('agency', 'book');`  
   Do not touch `sponsor` or `NULL`. Do not rewrite `username`. Include inactive rows.
2. `DROP CONSTRAINT IF EXISTS users_channel_type_check` then `ADD CONSTRAINT`  
   `channel_type IS NULL OR channel_type::text = ANY (ARRAY['sponsor', 'standard']::text[])`.

Django state in the same file: `AlterField` choices to sponsor/standard; replace check constraint values.

**Reverse:** data rewrite is **not** restorable (agency vs book is lost). Reverse SQL must **not** attempt to set rows back to agency/book. Reverse only restores the **0017 forward** check (`NULL | sponsor | agency | book | standard`) so a code rollback can still insert those values if needed. Use `RunSQL.noop` (or equivalent) for the `UPDATE`. This matches existing irreversible backfills in the repo (`RunPython.noop` / `RunSQL.noop` on florists/flowers/juices migrations).

**Application after 0018:**

- Create: unchanged (`standard`/`sponsor` only; `agency`/`book` still 400).
- Filter: `all` / `standard` / `sponsor` only; `agency`/`book` → 400 `Invalid Channel Type`.
- Detail/list responses: only `standard`/`sponsor` (after backfill, no remaining rows of the old types; constraint enforces it).
- Sponsorship: `standard` → `sponsor` only in code. Converted former agency/book channels keep working because they **are** `standard`.
- Web: Standard / Sponsor / All filters; invite dropdown owned `standard` only (plus exclude `sponsor` as today).
- Keep `/agency/@username` and `/book/@username` **redirects** so old shared links still reach `/channels/@username`. That is not “supporting the type”; the loaded `channel_type` will be `standard`.
- Keep reserved suffixes `_agency` and `_book` so new people cannot register names that collide with historical channel usernames.

**Deliberately not in this fix:**

- Renaming `rose_agency` / `story_book` usernames.
- Removing `_sponsor` reserved suffix or sponsor type.
- Dropping legacy URL redirects.
- Changing chapter rules, channel limit, profile image, or membership table shape.
- A mapping table to remember which rows were agency vs book.

**Deploy coupling:** ship API + 0018 + web together. If web drops `agency` from `isChannelType` before 0018 runs, leftover agency/book rows disappear from the UI. If 0018 runs while old API still filters/displays those types, filters return empty and labels would still say Agency/Book for zero rows. One release avoids that window.

### Regression risks and edge cases

- **Irreversible rewrite.** Rollback of 0018 cannot restore which channels were agency vs book. Constraint reverse can re-allow the values; data stays `standard`.
- **Filter bookmarks.** `/channels?channel_type=agency` (and profile list segments `channel_type=agency`) become invalid. `parseChannelTypeQuery('agency')` should return `null` and the page should fall back to `all` (today `?? 'all'` on `ChannelListPage`). API callers that still POST `channel_type: "agency"` get 400. That is the intended “stop support” behavior.
- **Edit of converted channels.** Type is `standard`; unchanged `_agency`/`_book` username still saves via the stored-username bypass. A rename still cannot use underscores.
- **Sponsorship.** Existing `channel_members` where the member user was an agency/book channel remain; that user is now `standard`, which is already allowed. Pending `channel_requests` likewise. Do not delete memberships.
- **Chapters on former books.** Remain on the same `flower_user` UUID. No `flower_posts` update.
- **Direct ORM/admin inserts** of `agency`/`book` must fail the check after 0018. That is required to “delete the types.”
- **Tests** that still `User.objects.create(channel_type=AGENCY|BOOK)` will error on IntegrityError after the constraint change.
- **i18n `common.agencies` / `common.books`.** No `t.common.agencies` call sites found in `src/`. Optional cleanup; not required for behavior.
- **Username `join` and `/channels/join/:token`.** Unrelated; do not touch route order.
- **Normal users.** Unchanged. Reserved `_agency`/`_book` suffixes stay blocked.

### Tests and verification steps

**Acceptance**

1. After 0018, `SELECT DISTINCT channel_type FROM users` contains only `NULL`, `sponsor`, and `standard` (no `agency`, no `book`).
2. Every row that was `agency` or `book` is `standard`; `username`, `uuid`, memberships, requests, and flower posts are unchanged.
3. Check constraint rejects insert of `agency` or `book`.
4. `POST /api/v1/channel` with `standard`/`sponsor` still 201; with `agency`/`book` still 400 `Invalid Channel Type`.
5. List/filter `channel_type=standard|sponsor|all` works; `agency`/`book` returns 400 `Invalid Channel Type`.
6. Converted channel detail still loads at `/channels/@<existing-username>` (including `_agency` / `_book` usernames).
7. `/agency/@rose_agency` and `/book/@story_book` still redirect to `/channels/@…`.
8. Owner/member can still create attachment and chapter posts on a converted channel.
9. A converted (now `standard`) channel can still request/be accepted on a `sponsor`; `sponsor` still cannot join another channel.
10. Web filters show All / Standard / Sponsor only. Invite dropdown includes owned `standard`, not `sponsor`, and does not list Agency/Book as types.
11. Reverse of 0018 restores the 0017 check (allows agency/book in the constraint) and does **not** rewrite `standard` rows back.

**API tests**

- Add a migration/data assertion (local migrate or a small Django test): seed agency+book, apply 0018, both rows are `standard`, inserting `agency` fails.
- `test_channel_api.py`: keep create-reject agency/book; change list/filter cases from agency/book to standard/sponsor; edit of a **legacy suffixed username** can use a seeded `standard` (or sponsor) named `talent_agency` to prove the unchanged-username bypass is about the **name**, not the old type.
- `test_channel_invite_check_and_sponsorship.py` / `test_channel_request_api.py`: drop agency/book requester fixtures; keep `standard` → sponsor success and sponsor-rejected cases.
- Flower/content_analysis/florist tests: replace `ChannelType.BOOK`/`AGENCY` seeds with `STANDARD` (or `SPONSOR` where the test needs a sponsor). Keep reserved-username tests `_agency`/`_book`.

**Web tests**

- `channelRoutes.test.ts`: parse/query no longer accept `agency`/`book` as **filters**; redirects from `/agency/@x` still exist.
- `channelService.test.ts`: `channel_type: 'standard'` still accepted; `'agency'` is not a channel type.
- `channelInviteJoin.test.ts`: only `standard` eligible; `sponsor` not; drop agency/book eligible assertions.
- `profileChannelsVisibility.test.ts`: specific types are `standard`/`sponsor` only.

**Manual**

- Local migrate 0018; count rows before/after; open a former agency URL (redirect) and confirm type label Standard; filter Agency absent; join that channel to a sponsor.

Do not run 0018 against Neon from analysis. Apply it with the API release.

### Remaining uncertainties and final recommendation

**Assumptions (proceed unless product contradicts)**

- Usernames are **not** backfilled. Only `users.channel_type` changes.
- `_agency` / `_book` remain reserved suffixes for normal users.
- Legacy `/agency/` and `/book/` **URL redirects stay**.
- Filter `channel_type=agency|book` becomes invalid (stop support), not an alias for `standard`.
- Data rewrite is one-way; no mapping table.

**Unknown (does not block if the assumptions above are accepted)**

- Exact Neon row counts of `agency` vs `book`. The `UPDATE … WHERE channel_type IN ('agency','book')` does not need the count.
- Whether any external client still POSTs list filters `agency`/`book`. They will get 400 after this change.

**Recommendation:** Proceed with the proposed fix. Backfill `agency`/`book` → `standard` in users migration 0018, shrink `users_channel_type_check` to `NULL | sponsor | standard`, then remove those types from API filters, sponsorship code, web unions/chips/copy, tests, and docs in the same release. Do not rewrite usernames and do not remove old prefixed URL redirects.

Next step after this analysis is `task-breakdown` on work folder `2026-10-29-bug-retire-agency-book-channel-types`.
