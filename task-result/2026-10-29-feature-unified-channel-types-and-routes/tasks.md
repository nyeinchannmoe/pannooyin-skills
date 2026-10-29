# Task breakdown: Restrict create types, unify channel URLs, and allow chapters on all channels

### Work folder

Work folder ID: `2026-10-29-feature-unified-channel-types-and-routes`

- Source analysis: `analysis-result/2026-10-29-feature-unified-channel-types-and-routes/analysis.md`
- This breakdown: `task-result/2026-10-29-feature-unified-channel-types-and-routes/tasks.md`
- DoRs (next, after approval): `dor-result/2026-10-29-feature-unified-channel-types-and-routes/`

### Scope and complexity assessment

**In scope:** Add stored channel type `standard` (constraint expansion only). Restrict `POST /api/v1/channel` to `standard` and `sponsor`. Use normal-user username rules on create and on any username **change**; leave unchanged legacy `_sponsor` / `_agency` / `_book` names valid. Unify public URLs to `/channels/@username` with redirects from singular `/sponsor|agency|book/@username` (and chapter sub-routes). Allow chapter create/update on every channel type. Allow `standard` (and existing `agency`/`book`) to join a `sponsor`. Update list filters, create UI, copy, invite eligibility, API docs, and both Database README copies. Fold Django and Vitest coverage into the task that delivers each behavior.

**Effort:** Medium-large. Touches `users` schema, `channels` and `flowers` APIs, public web routing, create/edit forms, and docs. Create contract is intentionally breaking for `agency`/`book` and suffixed usernames; read/list/detail stay compatible for historical types. No row backfill.

**Main dependencies:** Migration 0017 (or next number after `users.0016`) must exist before any `standard` insert. Web create of `standard` is unreachable until `/channels/@username` exists (there is no `/standard/` prefix today). Chapter UI is a dead editor unless the flowers gate is lifted first. Sponsorship invite dropdown and `_sponsorship_join_allowed` must change together. Ship API + 0017 + web in one release (same class of Neon deploy as 0016). `pann-oo-yin-web/vercel.json` already SPA-rewrites unknown paths; no rewrite or env change.

**Risks:** Unified detail 404s if `ChannelDetailPage` still requires `channelTypeFromPath`. Chapter editor 404s if it still requires `book`. Edit save fails on legacy suffixes without an unchanged-username bypass. Route order: keep `/channels`, `/channels/join/:inviteToken`, and `/channels/:channelTypeSegment/:userUuid` from colliding with `/channels/@username`. `isChannelType` must accept `standard` or list/detail mapping hides new rows.

**Out of scope (from analysis):** Backfilling types or usernames; dropping `agency`/`book` from the constraint or responses; changing `MAX_CHANNEL_COUNT`, profile image, description, owner-membership create, or normal-user signup rules; letting a `sponsor` join another channel; channel Open Graph; changing `/channels` directory, `/channels/join/:inviteToken`, or `/florist-dashboard?section=channels`.

**Gaps that affect tasking:** None material. Product already resolved sponsorship for `standard`, no suffix on rename, and redirects from singular prefixes. Public URL docs are not a dedicated file in the analysis; T4 should update any listed API/README text that still describes type-prefixed detail paths.

### Tasks

#### T1 — Persist `standard` on `users.channel_type`

- **Objective and expected outcome:** Expand the stored channel-type allow-list so `standard` can be inserted without rewriting existing rows. After this task, the check constraint and Django `User.ChannelType` choices are `NULL | sponsor | agency | book | standard`.
- **Scope:**
  - Included: `pann-oo-yin-api/apps/users/models.py` (`ChannelType.STANDARD`, check constraint values, model comment that currently omits `book`/`standard`); new migration after `0016_user_channel_type_book_check` following that file’s Django state + `RunSQL` `DROP`/`ADD CONSTRAINT` pattern (suggested `0017_user_channel_type_standard_check`); both Database README copies so stored values include `standard`.
  - Excluded: create-API allow-list, username rules, web types, sponsorship, chapters, row backfill, removing `agency`/`book`.
- **Dependencies:** None.
- **Acceptance criteria:**
  - Constraint allows `standard` (8 chars; column is `varchar(10)`).
  - Existing `sponsor` / `agency` / `book` / `NULL` rows still satisfy the check.
  - Reverse SQL restores the 0016 constraint and does not drop `book`.
  - No `users` row is updated by the migration.
- **Verification:** Run the new migration against a local DB (or migration test) and insert a `User` with `channel_type=standard`; confirm a pre-existing agency/book/sponsor fixture still loads. Reverse and re-apply. Do not run this against Neon from this task.

#### T2 — Restrict channel create, unify username rules, and let `standard` join a sponsor

- **Objective and expected outcome:** Create accepts only `standard` and `sponsor` with normal-user usernames. Rename uses the same rules; an unchanged legacy suffixed username still saves. List/filter includes `standard` and still accepts `agency`/`book`. `standard`, `agency`, and `book` may request/be accepted on a `sponsor`; `sponsor` still cannot join another channel.
- **Scope:**
  - Included: `apps/channels/constants.py` (split create types from stored/filter types; suffix map only to recognize legacy names); `validators.py`; `serializers.py` (`ChannelCreateSerializer`, list/filter); `services.py` `edit_channel`, `_sponsorship_join_allowed`, and accept-batch checks (~1281 / 1874 / 1951); channel API tests and docs listed below.
  - Excluded: flowers chapter gate; web UI/routes; changing `channel_type` on edit; DB constraint (T1); letting a sponsor join another channel.
- **Dependencies:** T1 (so `standard` inserts do not hit the old check).
- **Acceptance criteria:**
  - `POST /api/v1/channel` with `standard` or `sponsor` and username `acme` (unused, 3–30, `[a-z0-9]+`) returns 201.
  - Create with `agency`, `book`, `partner`, blank, or missing type returns 400 `Invalid Channel Type`.
  - Create with `acme_sponsor`, any `_`, or `pannooyin` returns 400 `Invalid Username` (keep that channel API message; reuse `validate_normal_user_username` + reserved-name check).
  - Create username equal to an existing flower/bee/florist/channel username (any case) returns 400 `Duplicate Username`.
  - Edit of a seeded `brand_sponsor` (or `_agency` / `_book`) with the same username succeeds; change to `acme` succeeds; change to `acme_sponsor` or any underscore name returns 400 `Invalid Username`.
  - Type cannot be changed on edit.
  - List/filter accepts `standard`, `sponsor`, `agency`, `book`, `all`.
  - A model-seeded `standard` owner can request join on a `sponsor` and accept creates membership. `agency`/`book` still can. `sponsor` → any channel still `Don't allow to join this channel`. `standard` → non-sponsor still rejected.
- **Verification:**
  - Extend `apps/channels/tests/test_channel_api.py`: POST-create agency/book becomes 400; add unsuffixed `standard`/`sponsor` success; seed historical types with `User.objects.create` for edit/list/detail.
  - Update `apps/channels/tests/test_channel_limits.py` so the three-channel fill uses create of `standard`/`sponsor` or model-seeded agency/book (today `test_limit_blocks_fourth_channel` POST-creates `two_agency` / `three_book`).
  - Extend `test_channel_invite_check_and_sponsorship.py` and `test_channel_request_api.py` with `standard` request + accept; keep agency/book and sponsor-rejected cases.
  - Docs (API copy + `pann-oo-yin-web/api_docs/`): `create_channel.md`, `edit_channel.md`, `get_channels.md`, `get_my_channels.md`, `get_channel_detail.md`, `join_channel.md`, `act_channel_request.md`, `get_channel_requests.md`, `docs/user/validate_username.md`, both Database READMEs (create vs historical usernames; create restriction is application-enforced; membership `user_uuid` may be a normal user or `standard` / `agency` / `book`, never a `sponsor`).

#### T3 — Allow chapter posts on every channel type

- **Objective and expected outcome:** Create and update of `post_type=chapter` succeed for any active channel user (`standard`, `sponsor`, `agency`, `book`). Personal accounts still cannot create chapters. No `flower_posts` schema change.
- **Scope:**
  - Included: `apps/flowers/services.py` `_assert_chapter_allowed_for_flower_user` (allow `is_channel_user`; still reject personal accounts); `apps/flowers/constants.py` `MSG_CHAPTER_BOOK_CHANNEL_ONLY` (replace or reword to match the new rule); `apps/flowers/tests/test_chapter_posts_api.py`; `docs/flower-posts/create_flower_post.md` and `update_flower_post.md` (both API and web `api_docs/` copies).
  - Excluded: web chapter UI/routes (T4); chapter table shape; policy scanning of chapter bodies (already skipped).
- **Dependencies:** T1 (for `standard` chapter fixtures). Does not require T2, but `standard` create via API is only available after T2; tests should model-seed `standard` if needed.
- **Acceptance criteria:**
  - Owner/member can create and update a chapter on an active `standard`, `sponsor`, `agency`, or `book` channel.
  - Omitted `channel_uuid` / personal account still rejected.
  - Duplicate `chapter_number` still unique per channel user; two channels may both have chapter 1.
- **Verification:** Invert leftover “book only” assertions in `test_chapter_posts_api.py` (including `test_channel_type_allows_attachment_and_chapter_posts`, which currently expects sponsor chapter create/update to fail). Keep personal-account rejection and per-channel duplicate-number coverage. Add `standard` chapter create/update success.

#### T4 — Unify public channel URLs and enable chapter UI on all types

- **Objective and expected outcome:** Canonical public URL is `/channels/@username` for every stored type. Old singular type-prefixed URLs redirect. Channel detail loads by username (no path-type 404). Writers on every channel type see attachment + chapter create and can open the chapter editor/reader.
- **Scope:**
  - Included: `src/utils/channelRoutes.ts`, `src/App.tsx` (register unified routes after exact `/channels` and `/channels/join/:inviteToken`; keep `/channels/:channelTypeSegment/:userUuid`; replace the three type-prefixed groups; `<Navigate replace>` from `/sponsor|agency|book/@username` and chapter sub-routes); `ChannelDetailPage.tsx` (drop `channelTypeFromPath` requirement; always fetch `post_types: ['attachment', 'chapter']`; always show `ChannelPostCreateMenu` for writers; empty copy mentions chapters); `ChapterEditorPage.tsx` and `ChapterReaderPage.tsx` (drop book-only and path-type mismatch; fix “book chapter” copy); `FlowerPostsGrid.tsx` (`channelChapterPath` no longer needs type for the URL); `channelProfilePath` call sites (`ProfileSponsorsSection.tsx`, `ChannelsPanel.tsx`, `MemberRequestsPanel.tsx`, `ChannelInviteJoinPage.tsx`, `ChannelMembershipCard.tsx`); `src/types/channel.ts` and `src/services/channelService.ts` `isChannelType` so `standard` does not map to `null`; Vitest for routes and channelService.
  - Excluded: create-dialog options and username-field suffix behavior (T5); filter chips copy beyond what routing needs; plural `/sponsors/*` `/agencies/*` `/books/*` (already redirect to the directory); OG/crawler support.
- **Dependencies:** T3 (chapter editor would 404 or the API would reject without the gate change). T1 so detail can load a `standard` row. T2 not required for existing agency/book/sponsor URLs, but `isChannelType('standard')` must work before T5 creates those channels.
- **Acceptance criteria:**
  - `channelProfilePath` (or simplified helper) is `/channels/@username` for every type; chapter helpers nest under that.
  - `/channels/@acme` and `/channels/@rose_agency` render detail (type from API, not path).
  - `/agency/@rose_agency`, `/book/@story_book`, `/sponsor/@brand_sponsor`, and matching chapter sub-routes redirect to `/channels/@…`.
  - `/channels` directory, `/channels?user_uuid=…`, and `/channels/join/:token` still work. Username `join` is `/channels/@join` and does not collide with invite.
  - Owner/member of `standard`, `sponsor`, `agency`, or `book` sees Create post and Write chapter; chapter editor opens for non-book types.
- **Verification:** Extend `src/utils/channelRoutes.test.ts` (canonical path, redirects, query parse adds `standard`, agency/book filters unchanged) and `src/services/channelService.test.ts` (`channel_type: 'standard'` is accepted). Manually: open an existing agency/book/sponsor via old and new URLs; open chapter editor on a non-book channel after T3.

#### T5 — Create/edit UI, filters, copy, and sponsorship invite eligibility

- **Objective and expected outcome:** Florists can create only Standard and Sponsor channels with signup-like usernames. Filters still find historical agency/book rows and add Standard. Invite/join UI lets owned `standard` (and agency/book) join a sponsor, not a sponsor joining another channel. Create/edit hints no longer demand `_sponsor` suffixes.
- **Scope:**
  - Included: `ChannelFormDialog.tsx` (create options Standard / Sponsor only); `usernameRules.ts`, `useUsernameValidation.ts`, `UsernameField.tsx` (create and rename: no suffix, underscore banned; `allowUsername` treats the **current** stored username as valid); `ChannelsPanel.tsx`, `ChannelListPage.tsx`, `ProfileChannelsSection.tsx`, `FloristDashboardPage.tsx`; `src/utils/profileChannelsVisibility.ts`; `src/utils/channelInviteJoin.ts` `isSponsorshipEligibleOwnedChannel`; i18n `src/i18n/en.ts` and `my.ts`; Vitest listed below.
  - Excluded: public route table (T4); API create/sponsorship (T2); backfill of stored usernames.
- **Dependencies:** T2 (create/sponsorship contracts) and T4 (a newly created `standard` channel is only reachable at `/channels/@username`).
- **Acceptance criteria:**
  - Create dialog offers only Standard and Sponsor.
  - Create username field rejects underscores, does not require a suffix, and still blocks reserved `pannooyin` client-side (`validateusername` remains suffix-agnostic).
  - Edit of a channel whose stored username is `brand_sponsor` can save without changing the field; changing the field to an underscore name is invalid.
  - Filters include Standard / Sponsor / Agency / Book / All.
  - Florist invite dropdown includes owned `standard`, `agency`, and `book`, and excludes `sponsor`.
  - Create/help copy no longer promises agency/book creation.
- **Verification:** Extend `src/utils/profileChannelsVisibility.test.ts` and `src/utils/channelInviteJoin.test.ts` (`standard`/`agency`/`book` eligible; `sponsor` not). Add `src/utils/usernameRules.test.ts` (and/or hook tests) for create, unchanged legacy suffix, and rename-to-normal-user. Manually: create `standard` with username `acme`, land on `/channels/@acme`, join that channel to a sponsor; save an existing `_agency` channel without renaming.

### Split rationale

One task would mix a constraint expansion, a breaking create contract, a flowers permission change, and a high-risk public-route swap that 404s if any page guard is left behind. Five tasks keep a verifiable schema prerequisite, the channels API contract (create, usernames, filters, sponsorship) as one Django-testable unit, chapter permission as its own flowers suite, unified URLs plus chapter UI as an atomic web slice (standard has no type-prefixed URL), and create/filter/invite UI as a follow-on that depends on both the API and the new route.
