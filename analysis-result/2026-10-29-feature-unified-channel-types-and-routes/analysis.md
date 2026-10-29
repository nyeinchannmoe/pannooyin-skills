# Restrict create types, unify channel URLs, and allow chapters on all channels

### Work folder

Work folder ID: `2026-10-29-feature-unified-channel-types-and-routes`

- Analysis: `analysis-result/2026-10-29-feature-unified-channel-types-and-routes/analysis.md`
- Tasks (next): `task-result/2026-10-29-feature-unified-channel-types-and-routes/tasks.md`
- DoRs (after task approval): `dor-result/2026-10-29-feature-unified-channel-types-and-routes/`

### Feature requirements and intended behavior

**Who it is for:** Florists (and other normal users) who create channels; anyone who opens, lists, edits, or posts on a channel, including existing agency/book/sponsor rows.

**What was asked:**

1. Newly created channels may use only `standard` and `sponsor`.
2. Keep every existing stored type (`sponsor`, `agency`, `book`) and historical rows unchanged.
3. Existing channels must keep working (detail, edit, list, membership, invite join, and existing agency/book sponsorship).
4. Channel usernames should look like normal-user usernames (no `_sponsor` / `_agency` / `_book` suffix) on **create and on rename**. Existing stored usernames stay until the owner changes them.
5. Channel detail becomes a single route: `/channels/@username`. Remove type-prefixed public URLs; keep redirects from the old singular prefixes.
6. On channel details, **every** channel type may upload both attachment posts and chapter posts (today only `book` can create both).
7. A new `standard` channel may join a `sponsor` channel (same sponsorship path that `agency`/`book` use today).

**Before:**

- Create accepts `sponsor`, `agency`, and `book`, each with a required username suffix.
- Public detail URLs are type-prefixed: `/sponsor/@username`, `/agency/@username`, `/book/@username` (plus matching `/chapters/...` sub-routes). Plural `/sponsors/*`, `/agencies/*`, `/books/*` already redirect to the `/channels` directory.
- Chapter create/update is allowed only when the stored `channel_type` is `book`. Other types may create attachment posts only.
- Channel detail UI shows the post+chapter create menu only for book channels.
- Only `agency`/`book` channels may join a `sponsor`.

**After (intended):**

- `POST /api/v1/channel` accepts only `standard` and `sponsor`. `agency` and `book` on create return `Invalid Channel Type`.
- Create **and rename** usernames use the same rules as flower/bee/florist: 3–30, `a-z0-9` only, no underscore, reserved `pannooyin` blocked, unique across all `users` rows. Saving an **unchanged** legacy `_sponsor` / `_agency` / `_book` username still succeeds so historical rows are not force-rewritten.
- Canonical public URL is `/channels/@username` for every type, including historical agency/book/sponsor rows. Type is read from the API, not from the path. `/sponsor/@username`, `/agency/@username`, `/book/@username` (and their chapter sub-routes) redirect to the unified path.
- Create and update of `post_type=chapter` succeed for any active channel user (including `standard`, `sponsor`, `agency`, `book`). Personal (non-channel) accounts still cannot create chapters.
- List/filter/detail still return historical `agency` and `book` values. Filters keep those types so old rows remain findable.
- Channel-to-channel sponsorship: `standard`, `agency`, and `book` may request membership in a `sponsor`. `sponsor` still cannot join another channel.

**Out of scope (unless product later says otherwise):**

- Backfilling or renaming existing `agency`/`book` rows to `standard`.
- Removing `agency`/`book` from the database check constraint or from list/detail responses.
- Changing `MAX_CHANNEL_COUNT`, profile image, description, or owner-membership create flow.
- Changing normal-user signup username rules.
- Backfilling existing suffixed usernames to drop `_sponsor` / `_agency` / `_book`.
- Letting a `sponsor` join another channel.
- Adding channel Open Graph / social-preview crawler support (flower/florist/bee have it; channel URLs do not today).
- Changing `/channels` (directory), `/channels/join/:inviteToken`, or `/florist-dashboard?section=channels`.

### Existing implementation findings

Channels are `users` rows with `channel_type` set and `user_type` / `auth_provider` / `social_id` null (`pann-oo-yin-api/apps/users/models.py`, `pann-oo-yin-api/apps/channels/validators.py` `is_channel_user`).

**Stored types (README and live constraint agree):** `NULL`, `sponsor`, `agency`, `book`.

- README: `pann-oo-yin-api/docs/Pann-Oo-Yin-Database-README.md` (web copy: `pann-oo-yin-web/api_docs/Pann-Oo-Yin-Database-README.md`).
- Model choices and `users_channel_type_check`: `pann-oo-yin-api/apps/users/models.py`.
- Live constraint last expanded in `pann-oo-yin-api/apps/users/migrations/0016_user_channel_type_book_check.py`.
- Column is `varchar(10)`, not a PostgreSQL enum. `standard` (8 chars) fits.

**`standard` does not exist** in models, constraints, API constants, frontend types, or routes.

**Create validation** (`pann-oo-yin-api/apps/channels/serializers.py` `ChannelCreateSerializer`, `validators.py` `validate_channel_type`):

- Allowed types = `CHANNEL_TYPE_VALUES` = `sponsor`, `agency`, `book` (`constants.py`).
- Username must end with `CHANNEL_USERNAME_SUFFIXES` for that type.
- Duplicate check is case-insensitive across all users (`users.validators.username_exists`).

**Edit validation** (`services.py` `edit_channel`):

- Does not accept `channel_type`; type cannot be changed.
- Re-runs `validate_channel_username` against the **existing** type, so suffix remains required even when the username is unchanged.

**List/filter validation** (`validate_channel_type_filter`): `all`, `sponsor`, `agency`, `book`. Unknown values 400.

**Public web routes (verified in `pann-oo-yin-web/src/App.tsx` and `src/utils/channelRoutes.ts`):**

| Path | Role today |
| --- | --- |
| `/channels` | Directory (`ChannelListPage`) |
| `/channels/join/:inviteToken` | Invite join |
| `/channels/:channelTypeSegment/:userUuid` | Legacy profile-list redirect to `/channels?user_uuid=…` |
| `/sponsor/@username` (+ `/chapters/...`) | Sponsor detail / chapters |
| `/agency/@username` (+ `/chapters/...`) | Agency detail / chapters |
| `/book/@username` (+ `/chapters/...`) | Book detail / chapters |
| `/sponsors/*`, `/agencies/*`, `/books/*` | Already `<Navigate to="/channels">` |

The request named `/sponsors/@username`, `/agencies/@username`, `/books/@username`. Those **plural** paths are not the live detail URLs; they already bounce to the directory. The URLs that must be retired as canonical detail routes are the **singular** prefixes `/sponsor/`, `/agency/`, `/book/`.

`ChannelDetailPage` currently 404s unless `channelTypeFromPath` returns a type (`if (!expectedChannelType) return <NotFoundView />`) and also 404s when the loaded `channel_type` does not match the path prefix. A type-free `/channels/@username` cannot keep that check.

`ChapterEditorPage` additionally 404s unless `detail.channel.channel_type === 'book'`.

**Username rules today:**

| Account | Format | Underscore | Suffix |
| --- | --- | --- | --- |
| Normal user | 3–30, `^[a-z0-9_]+$` then extra ban | Forbidden anywhere | Reserved suffixes blocked (`users/constants.py`) |
| Channel | Same charset | Required in suffix | `_sponsor` / `_agency` / `_book` |
| `POST /api/v1/user/validateusername` | Charset/length/uniqueness/reserved name only | Does not know user vs channel | Does not enforce suffix or underscore ban (`docs/user/validate_username.md`) |

Frontend create/edit uses `UsernameField` + `useUsernameValidation`, which **always requires the type suffix** when `channelType` is set. Create options are Sponsor / Agency / Book (`ChannelFormDialog.tsx`).

**Chapter posts today:**

- Schema already supports chapters on any `flower_user` (`flower_posts.post_type`, `chapter_number`, unique `(flower_user, chapter_number)` when number is set). No extra table is required to let sponsor/standard/agency own chapters.
- Application gate: `pann-oo-yin-api/apps/flowers/services.py` `_assert_chapter_allowed_for_flower_user` requires `User.ChannelType.BOOK`. Message: `Chapter posts are only allowed on book channels.` (`apps/flowers/constants.py` `MSG_CHAPTER_BOOK_CHANNEL_ONLY`). Used on create and update.
- Docs: `docs/flower-posts/create_flower_post.md`, `update_flower_post.md`.
- Tests encode the book-only rule: `apps/flowers/tests/test_chapter_posts_api.py` (`test_channel_type_allows_attachment_and_chapter_posts` expects sponsor chapter create/update to fail).
- Web: `channelContentPostTypes` returns `['attachment', 'chapter']` only for `book`; otherwise `['attachment']`. Create UI uses `isBookChannel` to choose `ChannelPostCreateMenu` vs a single Create button (`ChannelDetailPage.tsx`).

**Sponsorship today:** only `agency`/`book` may join a `sponsor` (`services.py` `_sponsorship_join_allowed`; `docs/channels/join_channel.md`, `act_channel_request.md`). Frontend: `src/utils/channelInviteJoin.ts` `isSponsorshipEligibleOwnedChannel`. There is **no** database check that forbids `standard` as `channel_members.user_uuid`; the gate is application code. README membership text still says `agency`/`book` only (`docs/Pann-Oo-Yin-Database-README.md`).

**README vs code (relevant only):**

- Allowed stored types match (`sponsor`, `agency`, `book`). No `standard`.
- README still says channel usernames **must** use suffixes; that is still true in code.
- `User` model comment says “NULL user_type = channel user (sponsor/agency)” and omits `book`; implementation already includes `book`.
- Request’s plural URL names do not match live singular detail routes.

### Affected frontend, backend, API, and database components

**Database**

- Table/column: `users.channel_type` (`varchar(10)`).
- Constraint: `users_channel_type_check` must **add** `standard`. Do **not** remove `agency` or `book`. No row backfill.
- Follow the 0016 pattern: Django state + `RunSQL` `DROP`/`ADD CONSTRAINT`.
- New migration after `users.0016_user_channel_type_book_check` (suggested `0017_user_channel_type_standard_check`).
- Also update Django `User.ChannelType` choices in `apps/users/models.py`.
- `flower_posts`: no schema change. Chapter uniqueness is already per channel user.
- Env vars: none.
- `created_channel_count`, `channel_members`, `channel_requests`: no schema change.

**API / backend**

- `apps/channels/constants.py` — split **create** types (`standard`, `sponsor`) from **stored/filter** types (`standard`, `sponsor`, `agency`, `book`). Suffix map is only for recognizing legacy stored names, not for requiring them on create or rename.
- `apps/channels/validators.py` — create-type validator; username validator that uses normal-user rules for create and for any **changed** username. Unchanged legacy suffixed usernames skip the underscore ban.
- `apps/channels/serializers.py` — `ChannelCreateSerializer.validate_channel_type` uses create allow-list; list/filter serializers add `standard`.
- `apps/channels/services.py` — `edit_channel` applies normal-user username rules when the username **changes**; if the submitted username equals the stored one, accept it (including `_agency` / `_book` / `_sponsor`). `_sponsorship_join_allowed` (and the accept-batch checks around lines 1281 / 1874 / 1951) add `standard` alongside `agency`/`book`; still require the target to be `sponsor`.
- `apps/users/models.py` — `ChannelType.STANDARD = "standard"`; expand check constraint values.
- `apps/flowers/services.py` — `_assert_chapter_allowed_for_flower_user`: allow any channel user (`is_channel_user`), still reject personal accounts. Update `MSG_CHAPTER_BOOK_CHANNEL_ONLY` (or replace it) so the public message matches the new rule.
- Tests that **POST-create** `agency`/`book`: `apps/channels/tests/test_channel_api.py`, `test_channel_limits.py` (fills max count with `two_agency` / `three_book` via the create endpoint). Seed historical types with `User.objects.create(...)`.
- Chapter tests: `apps/flowers/tests/test_chapter_posts_api.py` — sponsor (and new `standard`) chapter create/update should succeed; leftover “book only” assertions must be inverted. Keep the “no channel_uuid / personal account” rejection.
- Django admin: `apps/users/admin.py` does not list `channel_type`; no required admin change.

**API contract / docs (API copy + web `api_docs/` copy)**

- `docs/channels/create_channel.md` — create types `standard`/`sponsor`; username = normal-user rules; constraint now includes `standard`.
- `docs/channels/edit_channel.md` — create/rename usernames use normal-user rules; unchanged legacy suffixed usernames remain valid.
- `docs/channels/get_channels.md`, `get_my_channels.md`, `get_channel_detail.md` — responses/filters include `standard` while still listing `agency`/`book`.
- `docs/Pann-Oo-Yin-Database-README.md` (both copies) — stored values add `standard`; create vs historical username rules; note create restriction is application-enforced.
- `docs/user/validate_username.md` — still identity-agnostic; note channel create now uses normal-user underscore rules.
- `docs/flower-posts/create_flower_post.md`, `update_flower_post.md` — chapters allowed on any channel type; still require `channel_uuid` and still reject personal accounts.
- `docs/channels/join_channel.md`, `act_channel_request.md`, `get_channels.md`, `get_channel_requests.md` — sponsorship requesters are `standard`, `agency`, or `book`; target remains `sponsor`.
- Database README (both copies) — membership `user_uuid` may be a normal user or `standard` / `agency` / `book` (still never a `sponsor`).

**Frontend**

- `src/types/channel.ts` — `ChannelType` adds `'standard'`; keep `'agency' | 'book'`.
- `src/utils/usernameRules.ts` — suffix map only to detect legacy stored names; create and rename use normal-user username helpers.
- `src/hooks/useUsernameValidation.ts`, `src/components/auth/UsernameField.tsx` — create and rename: no suffix, underscore banned; `allowUsername` still treats the **current** stored username as valid so a no-op save of `brand_sponsor` succeeds.
- `src/components/channel/ChannelFormDialog.tsx` — create options only Standard / Sponsor; hints stop telling users to append `_sponsor`, including on edit.
- `src/utils/channelRoutes.ts` + `src/App.tsx` — canonical `channelProfilePath` becomes `/channels/@username` (same `@` convention as `flowerProfilePath`). Chapter helpers become `/channels/@username/chapters/...`. Declare these **after** `channels/join/:inviteToken` and the exact `channels` list route. Replace the three type-prefixed route groups with one group. Keep `/channels/:channelTypeSegment/:userUuid` (two segments) so it does not collide with `/channels/@username` (one segment).
- Redirect old singular type-prefixed URLs (`/sponsor/@x`, `/agency/@x`, `/book/@x` and their chapter sub-routes) to the unified path (**confirmed**). Plural `/sponsors/*` etc. already go to the directory; leave that behavior.
- `ChannelDetailPage.tsx` — stop requiring `channelTypeFromPath`; load by username; always request `post_types: ['attachment', 'chapter']`; always show `ChannelPostCreateMenu` for writers; empty copy always mentions chapters.
- `ChapterEditorPage.tsx` — remove `channel_type !== 'book'` 404; drop path-type mismatch. Copy currently says “book chapter”.
- `ChapterReaderPage.tsx` — drop path-type mismatch.
- `src/components/flower/FlowerPostsGrid.tsx` — `channelChapterPath` no longer needs `channelType` for the URL (type stays optional for display if still used).
- `src/services/channelService.ts` — `isChannelType` must accept `standard` or list/detail mapping returns `null`.
- Filters still include Agency / Book for historical rows, and add Standard: `ChannelsPanel.tsx`, `ChannelListPage.tsx`, `ProfileChannelsSection.tsx`.
- Copy: `FloristDashboardPage.tsx`, `ChannelsPanel.tsx` empty/help text, `src/i18n/en.ts` and `my.ts`.
- `src/utils/profileChannelsVisibility.ts` — `SPECIFIC_CHANNEL_TYPES` add `standard`.
- Call sites of `channelProfilePath`: `ProfileSponsorsSection.tsx`, `ChannelsPanel.tsx`, `MemberRequestsPanel.tsx`, `ChannelInviteJoinPage.tsx`, `ChannelMembershipCard.tsx`.
- `src/utils/channelInviteJoin.ts` — `isSponsorshipEligibleOwnedChannel` includes `standard` (keep `agency`/`book`; still exclude `sponsor`).
- Tests: `channelRoutes.test.ts`, `channelService.test.ts`, `profileChannelsVisibility.test.ts`, `channelInviteJoin.test.ts`. Add coverage for username create/edit modes.

**Deploy**

- Apply the new users migration on Neon with the API deploy (same class of change as `0016`).
- Web route change should ship in the same release as the first `standard` inserts.
- `pann-oo-yin-web/vercel.json` already SPA-rewrites unknown paths to `index.html`. No rewrite is required for `/channels/@username`. Optional later: crawler OG for channels (not present for `/sponsor/@…` today).
- No Vercel env change.

**None:** auth/token, Cloudinary, storage quota, chapter table shape, channel limit counter schema, invite-token URLs.

### Proposed implementation steps

1. **DB / API — persist `standard` without touching existing rows.**  
   Files: `apps/users/models.py`, new `apps/users/migrations/0017_user_channel_type_standard_check.py` (name as next number).  
   Done: constraint allows `NULL | sponsor | agency | book | standard`; inserting `standard` succeeds; existing `agency`/`book`/`sponsor` still satisfy the check; reverse SQL restores the 0016 constraint.

2. **API — split create allow-list from stored types.**  
   Files: `apps/channels/constants.py`, `validators.py`, `serializers.py`.  
   Done: `POST /api/v1/channel` with `agency` or `book` → 400 `Invalid Channel Type`; `standard` and `sponsor` accepted; list/filter `channel_type=agency|book|standard|sponsor|all` still valid.

3. **API — create usernames like normal users.**  
   Files: `validators.py` (`validate_channel_username` or a create-specific helper), `serializers.py` `ChannelCreateSerializer.validate`. Reuse `validate_normal_user_username` + reserved-name check, but keep channel API message `Invalid Username`.  
   Done: `username=acme` is valid for `standard`/`sponsor`; `acme_sponsor` and any underscore username rejected on create; collision with a flower/bee/florist username still `Duplicate Username`.

4. **API — edit usernames like normal users, without rewriting stored names.**  
   Files: `services.py` `edit_channel`, validators.  
   Done:  
   - Unchanged username on an existing `_sponsor` / `_agency` / `_book` channel succeeds.  
   - Any **changed** username (all types, including legacy agency/book/sponsor) uses normal-user rules: no underscore, no required suffix, reserved `pannooyin` blocked.  
   - Type still cannot be changed.

5. **API — `standard` may join a `sponsor`.**  
   Files: `apps/channels/services.py` `_sponsorship_join_allowed` and accept-batch checks; docs listed above.  
   Done: a `standard` channel owner can request join on a `sponsor`; accept creates membership. `agency`/`book` still can. `sponsor` → any channel still `Don't allow to join this channel`. `standard` → non-sponsor still rejected.

6. **API — chapters on every channel type.**  
   Files: `apps/flowers/services.py`, `apps/flowers/constants.py`.  
   Done: create/update `post_type=chapter` with `channel_uuid` of an active `standard`, `sponsor`, `agency`, or `book` channel succeeds for an owner/member; omitted `channel_uuid` still rejected; duplicate `chapter_number` still unique per channel user.

7. **Web — unified public URL.**  
   Files: `channelRoutes.ts`, `App.tsx`, pages listed above.  
   Done: `channelProfilePath(_, 'acme')` (or a simplified helper) is `/channels/@acme`; chapter paths nest under that; `/channels` list and `/channels/join/:token` still work; visiting `/sponsor/@brand_sponsor` redirects to `/channels/@brand_sponsor` (same for agency/book and chapter sub-routes); `ChannelDetailPage` no longer 404s for missing path type.

8. **Web — create/edit UI and username field.**  
   Files: `ChannelFormDialog.tsx`, `UsernameField.tsx`, `useUsernameValidation.ts`, `usernameRules.ts`, i18n.  
   Done: create dropdown is Standard / Sponsor; username field behaves like signup (no suffix, no underscore) on create and on rename; current stored suffixed username remains valid via `allowUsername`.

9. **Web — channel detail posting.**  
   Files: `ChannelDetailPage.tsx`, `ChapterEditorPage.tsx`, `ChapterReaderPage.tsx`, `FlowerPostsGrid.tsx`.  
   Done: every channel shows both Create post and Write chapter; post list fetches both types; chapter editor opens for a `standard` or `sponsor` channel.

10. **Web — filters, copy, and sponsorship invite UI.**  
   Files: `ChannelsPanel.tsx`, `ChannelListPage.tsx`, `ProfileChannelsSection.tsx`, `FloristDashboardPage.tsx`, `profileChannelsVisibility.ts`, `channelInviteJoin.ts`.  
   Done: users can filter Standard / Sponsor / Agency / Book / All; create copy no longer promises agency/book creation; `isChannelType` accepts `standard`; florist invite dropdown includes owned `standard` channels (and historical agency/book), not `sponsor`.

11. **Docs.**  
    Files: API `docs/` listed above and matching `pann-oo-yin-web/api_docs/` copies, plus both Database READMEs.  
    Done: create vs stored types, create vs rename username rules, unified public URL with redirects, chapters on any channel, and `standard`/`agency`/`book` → `sponsor` sponsorship are explicit.

12. **Tests.**  
    API: extend `test_channel_api.py` (create allow-list, unsuffixed success, suffixed create failure, edit of seeded agency/book/sponsor including rename to an unsuffixed name). Update `test_channel_limits.py` so the three-channel fill uses `standard`/`sponsor` (or model-seeded agency/book) instead of POST-creating `agency`/`book`. Flip chapter tests in `test_chapter_posts_api.py`. Extend `test_channel_invite_check_and_sponsorship.py` and `test_channel_request_api.py` so a model-seeded `standard` channel can request and be accepted on a sponsor; keep agency/book cases.  
    Web: extend `channelRoutes.test.ts` (canonical `/channels/@…`, redirects from old prefixes); `channelService.test.ts`; `profileChannelsVisibility.test.ts`; `channelInviteJoin.test.ts` (`standard` eligible, `sponsor` not); add username-rule tests for create vs unchanged-legacy vs rename.  
    Done: create agency/book is red; create standard/sponsor unsuffixed is green; chapter on sponsor/standard is green; `standard` → sponsor join is green; existing-type fixtures still pass.

13. **Deploy.**  
    Ship API code + 0017 together so `standard` inserts cannot hit the old check. Ship web in the same release so `/channels/@username` exists when the first `standard` channel is created and so chapter UI matches the API.

### Backward compatibility and migration strategy

**No data migration.** Existing `users.channel_type` values stay `sponsor` / `agency` / `book`. No username rewrite. No `flower_posts` rewrite.

**Schema change is constraint expansion only**, same approach as `0016_user_channel_type_book_check.py`: drop/re-add `users_channel_type_check`. Existing rows remain valid. Reverse must not drop `book`.

**Create contract is intentionally breaking** for `agency` and `book` and for suffixed usernames. Clients that still POST those values get 400. The in-repo web app is the client to update in the same release.

**Read/list/detail/join contracts stay compatible** for historical types. Adding `standard` to filter enums is additive. Clients that hard-code a three-value union (current web `isChannelType`) must be updated or they will hide new channels.

**Public URLs:** canonical path changes for every existing channel. Smallest compatible approach is client-side `<Navigate replace>` from `/sponsor|agency|book/@username` (and chapter sub-routes) to `/channels/@username`. Hard-deleting those routes without redirects would break shared links, in-app history, and any external bookmark of the current URLs. Invite links (`/channels/join/:token`) are already on the unified prefix and stay.

**Route collision to preserve:**

- `/channels` = directory (exact).
- `/channels/join/:inviteToken` = invite (must stay registered before `channels/:username`).
- `/channels/:channelTypeSegment/:userUuid` = two-segment legacy list redirect; does not match `/channels/@acme`.
- A channel whose username is literally `join` would be `/channels/@join`, which does not match `/channels/join/:token` because of the `@` and the extra segment.

**Edit of legacy channels:** do not backfill usernames. An unchanged `_sponsor` / `_agency` / `_book` username remains valid. Any **new** username on any type must look like a normal user (no underscore, no type suffix).

**Sponsorship:** join/accept contracts expand additively so `standard` may join `sponsor`. Existing `agency`/`book` → `sponsor` memberships and requests keep working. `sponsor` still cannot be a member of another channel. No `channel_members` schema change.

**Chapters:** allowing chapters on non-book channels is additive for those rows. Existing book chapters keep the same unique-number rule. Existing leftover chapter rows on sponsor (the test already seeds one) become updatable instead of rejected.

**Env:** no new `VITE_*` or API settings. `MAX_CHANNEL_COUNT` unchanged.

**Vercel/Neon:** apply 0017 on Neon before or with API traffic that inserts `standard`. SPA rewrite already covers `/channels/@username`. No enum rewrite, no table rewrite.

**Normal users:** unchanged. They still cannot use underscores or reserved suffixes, so they cannot impersonate old channel names like `brand_sponsor`. New channels share the same username namespace and uniqueness index (`users_username_lower_uniq`).

### Risks, edge cases, and security considerations

- **Old URLs.** Product confirmed redirects. Shared `/book/@story_book` (and `/sponsor/@…`, `/agency/@…`, chapter sub-routes) must `<Navigate replace>` to `/channels/@username`. Hard 404 is out of plan.
- **`ChannelDetailPage` path-type guard.** If `/channels/@username` ships while `expectedChannelType` is still required, every unified URL 404s (`if (!expectedChannelType) return <NotFoundView />`).
- **Chapter editor book-only guard.** If the API allows chapters on all types but `ChapterEditorPage` still requires `channel_type === 'book'`, writers on standard/sponsor/agency hit a dead editor.
- **Edit form vs legacy suffixes.** Product wants all **new** names to look like normal users. If that rule is applied without `allowUsername` / an unchanged-value bypass, owners cannot save existing `_agency`/`_book`/`_sponsor` usernames. Bypass only the current stored username; any edit of the field must pass normal-user rules.
- **Sponsorship for `standard`.** `_sponsorship_join_allowed`, accept-batch checks, join/act docs, README membership text, and `isSponsorshipEligibleOwnedChannel` must all add `standard` together. If only the web dropdown changes, the API still returns `Don't allow to join this channel`. If only the API changes, florists cannot pick the new channel on an invite.
- **Username harvest / impersonation.** Unsuffixed channel names collide with people. That is required by “look like a normal user” plus the existing global unique index. Duplicate handling already exists.
- **`validateusername` is still suffix-agnostic.** The create form must enforce the underscore ban client-side (as signup already does) because the availability endpoint allows `john_01`.
- **Length:** dropping `_sponsor` (8 chars) slightly increases the usable base (max 30). Empty-base suffix cases go away for create.
- **Username `join`.** Canonical `/channels/@join` is safe. `/channels/join` without a token must not be treated as a channel username; keep the invite route more specific and require `@` on detail (same pattern as flowers).
- **Direct ORM/admin inserts** can still write `agency`/`book`; only the create API is restricted. That matches “keep types in the database.”
- **Integrity:** do not add a DB check that forbids `agency`/`book`; that would break historical rows and tests.
- **Chapter uniqueness** is per channel, so two channels may both have chapter 1. No extra constraint is needed.
- **Policy analysis** already skips chapters (`_schedule_post_policy_check`). Extending chapters to all types does not newly scan chapter bodies.

### Testing and acceptance criteria

**Acceptance**

1. `POST /api/v1/channel` with `channel_type=standard` or `sponsor` and username `acme` (unused, 3–30, `[a-z0-9]+`) returns 201 and stores that type/username.
2. Create with `agency`, `book`, `partner`, blank, or missing type returns 400 `Invalid Channel Type`.
3. Create with `acme_sponsor`, `acme_agency`, any `_`, or `pannooyin` returns 400 `Invalid Username`.
4. Create username equal to an existing flower/bee/florist/channel username (any case) returns 400 `Duplicate Username`.
5. Seeded existing `agency`/`book`/`sponsor` rows still load on list, detail, edit, delete, invite, and (for agency/book/`standard`) sponsor join.
6. Edit of a legacy `brand_sponsor` (or `_agency` / `_book`) channel with the same username succeeds; changing it to `acme` succeeds; changing it to `acme_sponsor` or any underscore name returns 400 `Invalid Username`.
7. `GET`/`POST` list filters accept `standard` and still accept `agency`/`book`.
8. Web create dialog offers only Standard and Sponsor; create username field rejects underscores and does not require a suffix.
9. Visiting `/channels/@acme` shows the new channel. Visiting `/channels/@rose_agency` shows the existing agency channel. `/agency/@rose_agency`, `/book/@story_book`, `/sponsor/@brand_sponsor`, and matching chapter sub-routes redirect to `/channels/@…`.
10. `/channels` directory, `/channels?user_uuid=…`, and `/channels/join/:token` still work.
11. Owner/member of a `standard`, `sponsor`, `agency`, or `book` channel can create an attachment post and a chapter post; chapter numbers remain unique per channel.
12. Personal accounts still cannot create chapters (`channel_uuid` required).
13. A `standard` channel can request and be accepted as a member of a `sponsor`. Existing `agency`/`book` sponsorship still works. A `sponsor` still cannot join another channel.
14. No existing `users` row is updated by migration 0017.

**API tests to extend (existing Django `APIClient` / `TestCase` style):**

- `apps/channels/tests/test_channel_api.py` — replace POST-create agency/book success with 400; add standard/unsuffixed sponsor success; keep edit tests but seed legacy channels via `User.objects.create` where create is no longer allowed.
- `apps/channels/tests/test_channel_limits.py` — fill the max count without POST-creating `agency`/`book` (today `test_limit_blocks_fourth_channel` creates `two_agency` / `three_book`).
- `apps/flowers/tests/test_chapter_posts_api.py` — sponsor (and `standard`) chapter create/update succeed; keep personal-account rejection; keep duplicate chapter-number per channel.
- `apps/channels/tests/test_channel_invite_check_and_sponsorship.py`, `test_channel_request_api.py` — keep agency/book cases; add `standard` request + accept on a sponsor; keep `sponsor` requesting join rejected.

**Web tests (Vitest, follow existing files):**

- `src/utils/channelRoutes.test.ts` — `channelProfilePath` is `/channels/@username` for every type; old helper paths either redirect or are removed; query parse adds `standard`; agency/book filters unchanged.
- `src/services/channelService.test.ts` — `channel_type: 'standard'` is sent, not rejected.
- `src/utils/profileChannelsVisibility.test.ts` — `standard` is a specific filter.
- `src/utils/channelInviteJoin.test.ts` — `standard`, `agency`, and `book` eligible; `sponsor` not.
- New: `src/utils/usernameRules.test.ts` (and/or hook tests) for create, unchanged legacy suffix, and rename-to-normal-user rules.

### Assumptions, resolved questions, and final recommendation

**Verified**

- Live stored types are only `sponsor`, `agency`, `book` (plus `NULL`).
- Create and edit both require suffixes today.
- Create, list, and UI all treat agency/book as first-class types.
- `standard` is absent from schema, API, and web.
- Live detail URLs are singular `/sponsor|agency|book/@username`, not the plural paths named in the request. Plural prefixes already redirect to `/channels`.
- `/channels` is already the directory; `/channels/join/:token` is already invite join. A detail route `/channels/@username` fits the flower/florist/bee `@username` pattern if those more-specific routes stay registered first.
- `users.channel_type` is `varchar(10)` with a replaceable check constraint; adding a value does not rewrite rows.
- Channel and normal-user usernames already share one uniqueness index.
- Chapter storage is already per `flower_user`; the book-only rule is application code plus UI, not a DB constraint.
- Type-specific sponsorship still keys off stored `channel_type`.

**Assumptions**

- `standard` is a **new** stored value, not a rename of `agency` or `book`.
- Restriction is **create-API + create-UI only**; historical types remain writable via ORM and remain fully readable.
- “Username look like normal user” applies to create of `standard`/`sponsor` and to **any username change** on any stored type. Existing suffixed names are not backfilled.
- Unified URL applies to **all** types, including historical agency/book/sponsor rows (type is not in the path).
- Chapter permission expands to **every** channel type, including existing sponsor/agency rows, not only newly created `standard`/`sponsor`.
- Edit does not change `channel_type`.
- List filters should add `standard` and keep Agency/Book so old rows remain findable.
- `standard` joining a non-sponsor channel stays disallowed, matching today’s agency/book rule.

**Resolved product answers**

1. **Yes.** A new `standard` channel may join a `sponsor`. Keep existing `agency`/`book` → `sponsor`. `sponsor` still cannot join another channel.
2. **No suffix on rename.** All channel usernames, including changes to legacy rows, must look like a normal user. Unchanged stored `_sponsor` / `_agency` / `_book` names remain valid until the owner edits them.
3. **Redirects.** `/sponsor/@username`, `/agency/@username`, `/book/@username`, and their chapter sub-routes redirect to `/channels/@username`.

**Recommendation:** Proceed. Expand the check constraint to allow `standard`, restrict **create** to `standard` and `sponsor`, use normal-user username rules on create and on rename (with an unchanged-legacy bypass), allow `standard`/`agency`/`book` to join `sponsor`, unify public URLs to `/channels/@username` with redirects from the old singular prefixes, and allow chapter create/update on every channel type. Do not migrate existing rows.
