# Definition of Ready: Unify public channel URLs and enable chapter UI on all types

## 1. Document Metadata

| Field | Value |
| --- | --- |
| Task ID | T4 |
| Title | Unify public channel URLs and enable chapter UI on all types |
| Task type | Feature — frontend routing / channel detail / chapter pages / Vitest |
| Related feature | Restrict create types, unify channel URLs, and allow chapters on all channels |
| Work folder ID | `2026-10-29-feature-unified-channel-types-and-routes` |
| Source analysis | `analysis-result/2026-10-29-feature-unified-channel-types-and-routes/analysis.md` |
| Parent task-breakdown | `task-result/2026-10-29-feature-unified-channel-types-and-routes/tasks.md` |
| Document status | **READY** |

## 2. Objective and Expected Outcome

Canonical public URL is `/channels/@username` for every stored channel type. Old singular type-prefixed URLs redirect. Channel detail loads by username (no path-type 404). Writers on every channel type see attachment + chapter create and can open the chapter editor and reader.

After this task, a `standard` channel is reachable even though there is no `/standard/` prefix. Chapter UI is no longer book-only. `isChannelType('standard')` must succeed so list/detail mapping does not drop new rows.

## 3. Context and Existing Behavior

**Confirmed project facts**

Live routes in `pann-oo-yin-web/src/App.tsx` (under the authenticated layout’s parent route):

| Path | Role today |
| --- | --- |
| `channels` | Directory (`ChannelListPage`) |
| `channels/join/:inviteToken` | Invite join |
| `channels/:channelTypeSegment/:userUuid` | Legacy profile-list redirect to `/channels?user_uuid=…` |
| `sponsor/:username` (+ `/chapters/new`, `/chapters/:postUuid`, `/chapters/:postUuid/edit`) | Sponsor detail / chapters |
| `agency/:username` (+ chapter sub-routes) | Agency detail / chapters |
| `book/:username` (+ chapter sub-routes) | Book detail / chapters |
| `sponsors/*`, `agencies/*`, `books/*` | Already `<Navigate to="/channels" replace>` |

The request’s plural `/sponsors/@username` paths are **not** live detail URLs. Product confirmed redirects from **singular** `/sponsor|agency|book/@username` (and chapter sub-routes) to `/channels/@username`.

`channelProfilePath` in `src/utils/channelRoutes.ts` currently returns `/sponsor/@…`, `/agency/@…`, or `/book/@…`. Flower/florist/bee already use `/flowers/@username` via `parseRouteUsername` / `flowerProfilePath`.

`ChannelDetailPage`:

- Reads `expectedChannelType = channelTypeFromPath(location.pathname)`.
- If `!expectedChannelType`, returns `<NotFoundView />` **before** load UI. A type-free `/channels/@username` 404s if this guard remains.
- Also 404s when loaded `channel_type` does not match the path prefix.
- `channelContentPostTypes` returns `['attachment', 'chapter']` only for `book`; otherwise `['attachment']`.
- Writers see `ChannelPostCreateMenu` only when `isBookChannel`; other types get a single Create button (attachment only).
- Empty copy mentions chapters only for book.

`ChapterEditorPage` 404s unless `detail.channel.channel_type === 'book'` (and also on path-type mismatch). Copy says “Write a book chapter” / “Edit a book chapter”. `ChapterReaderPage` 404s on path-type mismatch.

`FlowerPostsGrid` navigates with `channelChapterPath(channelType, username, post.uuid)` and defaults `channelType` to `'book'`. Username comes from `post.username`.

`src/types/channel.ts` `ChannelType = 'sponsor' | 'agency' | 'book'`. `channelService.ts` `isChannelType` accepts only those three; list/detail mapping returns `null` for anything else (`standard` would be dropped).

`pann-oo-yin-web/vercel.json` already SPA-rewrites unknown paths to `index.html`. No rewrite is required for `/channels/@username`. No Vercel env change.

**Approved requirements:** unified URL for **all** types including historical agency/book/sponsor; type is read from the API, not the path; redirects from old singular prefixes; chapter UI on every type.

## 4. Scope

**Included**

- `src/utils/channelRoutes.ts` — canonical `channelProfilePath` is `/channels/@username`; chapter helpers nest under that; query/list-segment parsers accept `standard`; `channelTypeLabel` includes Standard; `channelRequestUserDestination` routes `standard` to the unified path.
- `src/App.tsx` — register unified detail/chapter routes **after** exact `channels` and `channels/join/:inviteToken`; keep `channels/:channelTypeSegment/:userUuid`; replace the three type-prefixed groups with unified routes plus `<Navigate replace>` from old singular prefixes and their chapter sub-routes.
- `ChannelDetailPage.tsx` — drop required `channelTypeFromPath`; always fetch `post_types: ['attachment', 'chapter']`; always show `ChannelPostCreateMenu` for writers; empty copy mentions chapters.
- `ChapterEditorPage.tsx` / `ChapterReaderPage.tsx` — drop book-only and path-type mismatch; fix “book chapter” copy.
- `FlowerPostsGrid.tsx` — chapter URL no longer needs type.
- `channelProfilePath` call sites: `ProfileSponsorsSection.tsx`, `ChannelsPanel.tsx`, `MemberRequestsPanel.tsx`, `ChannelInviteJoinPage.tsx`, `ChannelMembershipCard.tsx` (and the chapter pages). After the helper returns the unified path, call sites that still pass type may keep compiling.
- `src/types/channel.ts` — add `'standard'`; keep `'agency' | 'book'`.
- `src/services/channelService.ts` — `isChannelType` accepts `standard`.
- Vitest: `src/utils/channelRoutes.test.ts`, `src/services/channelService.test.ts`.

**Excluded**

- Create-dialog options and username-field suffix behavior (T5).
- Filter chip labels beyond what routing/query parse needs (`parseChannelTypeQuery` / `parseChannelTypeListSegment` must accept `standard`; adding the Standard chip UI is T5).
- Plural `/sponsors/*` `/agencies/*` `/books/*` (already redirect to the directory).
- Channel Open Graph / crawler support.
- Changing `/channels` directory, `/channels/join/:inviteToken`, or `/florist-dashboard?section=channels`.
- API create/sponsorship (T2) and flowers gate (T3) implementation.

**Docs:** there is no dedicated public-URL document. Inspect channel API docs and both Database READMEs; update only sentences that claim web detail URLs are `/sponsor|agency|book/@username`. If none exist (confirmed: current API docs describe HTTP APIs, not those web paths), T4 does not add a new URL spec file.

## 5. Technical Implementation Requirements

**Canonical paths (confirmed)**

- Detail: `/channels/@{username}` using `parseRouteUsername` and `encodeURIComponent`, same `@` convention as `flowerProfilePath`.
- Chapter reader: `/channels/@{username}/chapters/{postUuid}`
- Chapter create: `/channels/@{username}/chapters/new`
- Chapter edit: `/channels/@{username}/chapters/{postUuid}/edit`
- Empty/invalid username still falls back to `/home` (existing helper behavior).

**Proposed `channelProfilePath`**

- Keep the current `(channelType, username)` signature so existing call sites type-check after `ChannelType` gains `'standard'`, but **ignore type when building the URL**.
- Alternatively a username-only helper is allowed if every call site is updated in this task. Do not leave any in-app link generating `/sponsor/@…` as canonical.

**`App.tsx` route order (required)**

1. Keep `channels/join/:inviteToken` and exact `channels` registered first (already are).
2. Add unified group, still **before** the two-segment legacy list route:
   - `channels/:username/chapters/new` (authenticated, `ChapterEditorPage`)
   - `channels/:username/chapters/:postUuid/edit` (authenticated, `ChapterEditorPage`)
   - `channels/:username/chapters/:postUuid` (`ChapterReaderPage`)
   - `channels/:username` (`ChannelDetailPage`)
3. Keep `channels/:channelTypeSegment/:userUuid` (`LegacyProfileChannelsRedirect`). Two segments (`channel_type=sponsor` + uuid) do not match `/channels/@acme` (one segment after `channels`).
4. Replace the three `sponsor|agency|book/:username` trees with redirects to the matching unified path (`replace`). Chapter sub-routes must redirect too (`/book/@x/chapters/new` → `/channels/@x/chapters/new`, and likewise for reader/edit).
5. Leave `sponsors/*` / `agencies/*` / `books/*` as directory redirects.

**Collision: username `join`**

- Canonical detail is `/channels/@join` (has `@`, one extra segment).
- Invite is `/channels/join/:inviteToken` (no `@`, extra token segment).
- `/channels/join` without a token must not be treated as a channel username. Require `@` on detail (same pattern as flowers). Keep the invite route more specific.

**`ChannelDetailPage`**

- Stop returning `NotFoundView` when `channelTypeFromPath` is null.
- Load by username (existing `getChannelDetail`). Canonicalize missing `@` to `/channels/@username` without needing a path type.
- Stop 404 when API `channel_type` does not match a path prefix (there is no prefix).
- Always request posts with `post_types: ['attachment', 'chapter']`.
- For `canWritePosts`, always render `ChannelPostCreateMenu` (Create post + Write chapter). Remove the book-only single Create button.
- Empty writer copy: mention both a photo/video and a chapter for every type.

**Chapter pages**

- Editor: remove `channel_type !== 'book'` missing-state. Keep owner/member authorization already in the page. Drop path-type mismatch 404.
- Reader: drop path-type mismatch 404. Public read of non-book chapters must work.
- Copy: replace “book chapter” with type-agnostic “chapter”. `channelTypeLabel` must return `Standard` for `standard`.

**`FlowerPostsGrid`**

- Build reader URLs with the unified helper; type is not required for the path. Keep passing type only if still used for display (today it is not).

**Types / mapping**

- `ChannelType` and `ChannelTypeFilter` include `'standard'`.
- `isChannelType` returns true for `'standard'`. `getChannels` must not throw `Invalid Channel Type` when `params.channel_type === 'standard'`. List/detail mappers must not null out `standard` rows.

**Deploy**

- No `vercel.json` change. Ship this web slice in the same release as the first `standard` inserts (T1+T2) so new channels are reachable.

## 6. Affected Files and Components

**Modify**

| Path | Responsibility |
| --- | --- |
| `pann-oo-yin-web/src/utils/channelRoutes.ts` | Canonical paths, labels, query/segment parse, request-user destination |
| `pann-oo-yin-web/src/utils/channelRoutes.test.ts` | Canonical path, redirects helpers if extracted, query parse adds `standard`, agency/book filters unchanged |
| `pann-oo-yin-web/src/App.tsx` | Unified routes, legacy redirects, preserve directory/join/legacy list |
| `pann-oo-yin-web/src/pages/ChannelDetailPage.tsx` | Remove path-type 404; both post types; create menu; empty copy |
| `pann-oo-yin-web/src/pages/ChapterEditorPage.tsx` | Remove book-only and path-type guards; copy |
| `pann-oo-yin-web/src/pages/ChapterReaderPage.tsx` | Remove path-type guard |
| `pann-oo-yin-web/src/components/flower/FlowerPostsGrid.tsx` | Chapter navigation URL |
| `pann-oo-yin-web/src/types/channel.ts` | `'standard'` on `ChannelType` |
| `pann-oo-yin-web/src/services/channelService.ts` | `isChannelType` |
| `pann-oo-yin-web/src/services/channelService.test.ts` | `channel_type: 'standard'` is accepted/sent, not rejected |
| Call sites listed in section 4 | Keep compiling; they should produce `/channels/@username` |

**Create (proposed, optional)**

- A small redirect component or helper used by `App.tsx` to map `/sponsor|agency|book/:username/...` → `/channels/@username/...` so chapter sub-routes stay in sync. Not required if inline `Navigate` elements are clear.

Do not modify `vercel.json`.

## 7. Business Rules and Data Requirements

- Type is never taken from the URL for authorization or 404. Type comes from `GET /api/v1/getchanneldetail/{uuid-or-username}`.
- Historical suffixed usernames (`rose_agency`) are valid path identifiers: `/channels/@rose_agency`.
- New unsuffixed names (`acme`) use the same path shape: `/channels/@acme`.
- Directory, invite join, and florist dashboard query `section=channels` are unchanged.
- No database change in this task.

## 8. API and Integration Contracts

Not applicable to HTTP API shapes. This task consumes existing `getchanneldetail` and flower-post list APIs.

**Web contracts**

| From | To |
| --- | --- |
| `/channels/@acme` | Channel detail (type from API) |
| `/channels/@rose_agency` | Existing agency detail |
| `/agency/@rose_agency` | Redirect → `/channels/@rose_agency` |
| `/book/@story_book` | Redirect → `/channels/@story_book` |
| `/sponsor/@brand_sponsor` | Redirect → `/channels/@brand_sponsor` |
| Matching `/chapters/...` on old prefixes | Redirect to the same chapter suffix under `/channels/@…` |
| `/channels` | Directory |
| `/channels?user_uuid=…` | Profile channel list |
| `/channels/join/:token` | Invite join |
| `/channels/@join` | Channel whose username is `join`, not the invite route |

Chapter editor/reader under `/channels/@username/chapters/...` must load for `standard`, `sponsor`, `agency`, and `book` once T3 has lifted the API gate.

## 9. Dependencies and Prerequisites

- **T3** — chapter editor would 404 or the API would reject without the gate change. Implement T3 before relying on non-book chapter save.
- **T1** — detail can load a `standard` row only if the API/DB can persist it (tests/manual may model-seed).
- **T2 not required** for existing agency/book/sponsor URLs. `isChannelType('standard')` **is** required before T5 creates those channels, so this task includes that mapping.
- T5 depends on this route table; a newly created `standard` channel is only reachable at `/channels/@username`.

**Dependency contract for T5:** `channelProfilePath` returns `/channels/@username`; `ChannelType` includes `standard`; create success navigation must use the unified path.

## 10. Edge Cases and Failure Handling

- If unified routes ship while `expectedChannelType` is still required, every new URL 404s. Removing that guard is mandatory.
- If the editor still requires `book`, writers on standard/sponsor/agency hit a dead editor even after T3.
- UUID lookup on detail already redirects to the canonical username path; that redirect must go to `/channels/@username`, not a type prefix.
- `channelTypeFromPath` returning null for `/channels/@x` is expected; do not use it as a hard requirement. It may still detect old prefixes if redirects are implemented as pages rather than dedicated routes.
- Query parse: `standard` is a valid `channel_type` query; `agencies` (plural word) stays invalid/`null` as today.
- Do not register a greedy `channels/:username` **before** `channels/join/:inviteToken`.

## 11. Security and Backward Compatibility

- Canonical path changes for every existing channel. Redirects are required; hard-delete of old routes without redirects is out of plan.
- Invite tokens stay on `/channels/join/:token`.
- SPA rewrite already covers unknown paths; no new public unauthenticated API.
- Adding `'standard'` to the client union is additive; omitting it hides new channels.

## 12. Acceptance Criteria

- `channelProfilePath` (or simplified helper) is `/channels/@username` for every type; chapter helpers nest under that.
- `/channels/@acme` and `/channels/@rose_agency` render detail (type from API, not path).
- `/agency/@rose_agency`, `/book/@story_book`, `/sponsor/@brand_sponsor`, and matching chapter sub-routes redirect to `/channels/@…`.
- `/channels` directory, `/channels?user_uuid=…`, and `/channels/join/:token` still work. Username `join` is `/channels/@join` and does not collide with invite.
- Owner/member of `standard`, `sponsor`, `agency`, or `book` sees Create post and Write chapter; chapter editor opens for non-book types.
- `isChannelType('standard')` is true; list/detail mapping does not drop those rows.

## 13. Testing and Verification

**Vitest (existing files)**

- `src/utils/channelRoutes.test.ts`
  - `channelProfilePath` is `/channels/@username` for `standard`, `sponsor`, `agency`, and `book`.
  - `channelRequestUserDestination` for those types uses `/channels/@…` (today it expects `/agency/@rose_agency` etc. — those assertions must change).
  - `parseChannelTypeQuery` / `parseChannelTypeListSegment` accept `standard`; agency/book/`all` unchanged; plural junk still null.
- `src/services/channelService.test.ts`
  - Sending/mapping `channel_type: 'standard'` is accepted, not rejected. Extend `getChannels` coverage so `standard` is forwarded in the POST body.

**Manual (after T3)**

- Open an existing agency, book, and sponsor via old and new URLs; confirm replace-redirect and detail render.
- Open chapter editor on a non-book channel; confirm it is not `NotFoundView`.
- Hit `/channels`, `/channels?user_uuid=…`, and `/channels/join/:token`.
- Confirm `/channels/@join` is not captured by the invite route.

No Django tests in this task.

## 14. Risks, Assumptions, and Open Questions

**Assumptions**

- Unified URL applies to historical agency/book/sponsor rows, not only `standard`.
- Redirects are client-side `<Navigate replace>` (confirmed by product). Server 301 is not required because Vercel already serves the SPA.
- Filter **chips** for Standard are T5; parsers and types must already accept `standard` here so T5 can send the query.

**Risks**

- Route order mistakes collide invite or legacy list with detail.
- Leaving any `channelProfilePath` branch that defaults unknown types to `/sponsor/@…` would mis-route `standard`.

**Open questions**

- None that block implementation. Product confirmed redirects. No dedicated public-URL doc file exists to update.

## 15. Definition of Ready Checklist

- [x] Live route table verified in `App.tsx` / `channelRoutes.ts`.
- [x] Path-type 404 and book-only editor guards identified.
- [x] Collision rules for `join`, directory, and two-segment legacy list specified.
- [x] `isChannelType('standard')` included so T5 is not blocked.
- [x] T3/T1 prerequisites stated.
- [x] Status is READY.
