# Definition of Ready: Allow chapter posts on every channel type

## 1. Document Metadata

| Field | Value |
| --- | --- |
| Task ID | T3 |
| Title | Allow chapter posts on every channel type |
| Task type | Feature — backend / API / docs / Django tests |
| Related feature | Restrict create types, unify channel URLs, and allow chapters on all channels |
| Work folder ID | `2026-10-29-feature-unified-channel-types-and-routes` |
| Source analysis | `analysis-result/2026-10-29-feature-unified-channel-types-and-routes/analysis.md` |
| Parent task-breakdown | `task-result/2026-10-29-feature-unified-channel-types-and-routes/tasks.md` |
| Document status | **READY** |

## 2. Objective and Expected Outcome

Create and update of `post_type=chapter` succeed for any active channel user (`standard`, `sponsor`, `agency`, `book`). Personal accounts still cannot create or update chapters. No `flower_posts` schema change.

After this task, a leftover chapter row on a sponsor (already seeded in tests) becomes updatable instead of rejected. Book chapters keep the same per-channel unique-number rule.

## 3. Context and Existing Behavior

**Confirmed project facts**

- Chapter storage is already per `flower_user` (`flower_posts.post_type`, `chapter_number`, unique constraint `flower_posts_user_chapter_uniq` on `(flower_user, chapter_number)` when number is set). Two channels may both have chapter 1. No extra table is required.
- Application gate: `apps/flowers/services.py` `_assert_chapter_allowed_for_flower_user`, called from chapter create and chapter update.
  - If not `is_channel_user(flower_user)` → `MSG_CHAPTER_CHANNEL_REQUIRED` (`channel_uuid is required for chapter posts.`).
  - If `flower_user.channel_type != User.ChannelType.BOOK` → `MSG_CHAPTER_BOOK_CHANNEL_ONLY` (`Chapter posts are only allowed on book channels.`).
- `is_channel_user` (`apps/channels/validators.py`) is true when `channel_type` is set and `user_type` / `auth_provider` / `social_id` are null. That already includes `sponsor`, `agency`, `book`, and will include `standard` after T1.
- Policy scanning already skips chapters (`_schedule_post_policy_check` returns early for `post_type=chapter`). Extending chapters to all types does not newly scan chapter bodies.
- Attachment posts are already allowed on every channel type.
- Endpoints: `POST /api/v1/flower-posts` and `PUT /api/v1/flower-posts/{uuid}` (`POSTS_URL` in tests).
- Tests: `apps/flowers/tests/test_chapter_posts_api.py`. `test_channel_type_allows_attachment_and_chapter_posts` currently expects sponsor chapter create **and** leftover sponsor chapter update to fail with the book-only message. Personal leftover chapter update still expects `channel_uuid is required for chapter posts.`
- Docs (both trees): `docs/flower-posts/create_flower_post.md` and `update_flower_post.md` still say chapters are book-only and list the book-only warning message.

**Approved requirement:** chapter permission expands to **every** channel type, including existing sponsor/agency rows, not only newly created `standard`/`sponsor`. Personal (non-channel) accounts still cannot create chapters.

## 4. Scope

**Included**

- `_assert_chapter_allowed_for_flower_user`: allow any channel user; still reject personal accounts.
- `MSG_CHAPTER_BOOK_CHANNEL_ONLY` in `apps/flowers/constants.py` — replace, reword, or remove so the public message matches the new rule (the book-only string must not remain as the live rejection for sponsor/agency/standard).
- `apps/flowers/tests/test_chapter_posts_api.py`.
- `create_flower_post.md` and `update_flower_post.md` in `pann-oo-yin-api/docs/flower-posts/` and `pann-oo-yin-web/api_docs/flower-posts/`.

**Excluded**

- Web chapter UI/routes, `ChannelDetailPage` create menu, book-only editor 404 (T4).
- `flower_posts` table shape, uniqueness constraint, chapter cover rules, draft/published, description length, video rejection, `post_type` immutable.
- Policy scanning of chapter bodies (already skipped).
- Channel create/username/sponsorship (T2).
- Username backfill.

## 5. Technical Implementation Requirements

**Proposed gate (confirmed intent, proposed code shape)**

- Keep the personal-account rejection: if not `is_channel_user(flower_user)`, raise `FlowerPostServiceError(MSG_CHAPTER_CHANNEL_REQUIRED, warning=True)`.
- **Remove** the `channel_type != BOOK` branch. Any active channel user may own chapters.
- Inactive / non-channel targets remain rejected by existing channel-load helpers (`_load_channel` / inactive checks already used for channel posts). Do not invent a new type enum check.
- After the type branch is gone, `MSG_CHAPTER_BOOK_CHANNEL_ONLY` is unused. **Proposed:** delete or stop returning it. Do not keep emitting “only allowed on book channels.” If a replacement constant is introduced, it must not re-introduce a type restriction; personal accounts already have `MSG_CHAPTER_CHANNEL_REQUIRED`.

**Unchanged chapter rules**

- `channel_uuid` required for create.
- `chapter_number` required, positive integer, unique per channel user.
- Cover image optional; videos rejected for chapters.
- `post_type` cannot be changed.
- Cannot move a chapter onto another channel the actor does not own (`You do not own this flower post.` — existing test).
- Breaking-policy analysis still not requested for chapters.

**`standard` fixtures**

- Tests should `User.objects.create` a `standard` channel (T1 required). Do not depend on T2’s create API.

## 6. Affected Files and Components

**Modify**

| Path | Responsibility |
| --- | --- |
| `pann-oo-yin-api/apps/flowers/services.py` | `_assert_chapter_allowed_for_flower_user` |
| `pann-oo-yin-api/apps/flowers/constants.py` | Remove or reword `MSG_CHAPTER_BOOK_CHANNEL_ONLY` |
| `pann-oo-yin-api/apps/flowers/tests/test_chapter_posts_api.py` | Invert leftover “book only” assertions; add `standard` success |
| `pann-oo-yin-api/docs/flower-posts/create_flower_post.md` | Chapters allowed on any channel type; still require `channel_uuid`; still reject personal accounts |
| `pann-oo-yin-api/docs/flower-posts/update_flower_post.md` | Same; leftover non-book chapter rows are updatable |
| `pann-oo-yin-web/api_docs/flower-posts/create_flower_post.md` | Mirror API copy |
| `pann-oo-yin-web/api_docs/flower-posts/update_flower_post.md` | Mirror API copy |

No new files. No migration.

## 7. Business Rules and Data Requirements

| Actor / target | Chapter create | Chapter update |
| --- | --- | --- |
| Owner or active member of active `standard` / `sponsor` / `agency` / `book` | Allowed | Allowed |
| Personal flower/bee/florist account (`channel_uuid` omitted or personal leftover chapter) | Rejected `channel_uuid is required for chapter posts.` | Rejected with the same message |
| Duplicate `chapter_number` on the same channel user | Rejected `A chapter with this chapter number already exists in this channel.` | Same uniqueness |
| Same `chapter_number` on a different channel | Allowed | Allowed |

- No `flower_posts` rewrite. Existing book chapters stay. Existing leftover sponsor chapter rows become legally updatable.
- Uniqueness remains `flower_posts_user_chapter_uniq`. No extra constraint.

## 8. API and Integration Contracts

**`POST /api/v1/flower-posts`** with `post_type=chapter` and `channel_uuid` of an active channel of any stored type: 201 for an active owner or member (auth and membership rules unchanged).

**`PUT /api/v1/flower-posts/{uuid}`** on a chapter belonging to any channel type: 200 when the actor may edit that post.

**Removed warning:** `Chapter posts are only allowed on book channels.` must not be returned for sponsor/agency/standard/book.

**Kept warning:** `channel_uuid is required for chapter posts.` for omitted `channel_uuid` / personal accounts.

Auth, multipart, Cloudinary upload, and storage accounting for the channel user stay as today.

## 9. Dependencies and Prerequisites

- **T1** so `User.ChannelType.STANDARD` and the check constraint allow a `standard` chapter fixture.
- **Does not require T2.** Prefer model-seeded channels in tests.
- **T4 depends on this task.** If T4 ships the chapter editor for non-book types while this gate still requires `book`, writers hit API warnings. T4’s editor 404 is a separate web guard; both must be lifted, T3 first.

## 10. Edge Cases and Failure Handling

- Inactive channel users remain ineligible via existing channel post loading (do not special-case type).
- Type change attachment → chapter still rejected (`post_type cannot be changed.`).
- Moving a chapter to another channel the user does not own still rejected.
- Duplicate chapter number on the same channel still unique; race still maps `flower_posts_user_chapter_uniq`.
- Personal leftover chapter rows (test already creates one) stay rejected on update.

## 11. Security and Backward Compatibility

- Additive for non-book channels. Book behavior stays allowed.
- Personal accounts cannot gain chapters by omitting `channel_uuid`.
- Policy analysis remains skipped for chapters; this is existing behavior, not a new scan expansion.
- No schema change, no env change, no auth change.

## 12. Acceptance Criteria

- Owner/member can create and update a chapter on an active `standard`, `sponsor`, `agency`, or `book` channel.
- Omitted `channel_uuid` / personal account still rejected.
- Duplicate `chapter_number` still unique per channel user; two channels may both have chapter 1.
- Sponsor leftover chapter update succeeds (test inversion).
- Docs in both trees no longer say chapters are book-only.

## 13. Testing and Verification

Extend `apps/flowers/tests/test_chapter_posts_api.py` in the existing `APIClient` style:

- Invert `test_channel_type_allows_attachment_and_chapter_posts`: sponsor chapter create and leftover sponsor chapter update must succeed (201/200), not return the book-only message.
- Add `standard` (model-seeded) chapter create and update success. Agency may be included if cheap; not required if sponsor + standard + book already cover “every stored type,” but book and at least one non-book plus `standard` are required by the task.
- Keep personal-account rejection and per-channel duplicate-number coverage (`test_duplicate_chapter_number_is_unique_per_channel`).
- Keep attachment-on-book (and attachment-on-sponsor if present) success.

No Vitest in this task.

## 14. Risks, Assumptions, and Open Questions

**Assumptions**

- “Every channel type” means every **stored** channel type that `is_channel_user` recognizes, including historical agency/book/sponsor and new `standard`.
- Replacement of `MSG_CHAPTER_BOOK_CHANNEL_ONLY` may be deletion; a new type-specific message is not required.

**Risks**

- T4 chapter UI without this gate produces API warnings on non-book editor save.
- Leaving the book-only string in docs or tests will fail CI or mislead clients.

**Open questions**

- None that block implementation.

## 15. Definition of Ready Checklist

- [x] Gate function, message constant, and uniqueness constraint confirmed in source.
- [x] Personal-account rejection preserved.
- [x] Test inversions and `standard` fixture strategy specified.
- [x] Dual-tree flower-post docs listed.
- [x] T1 prerequisite and T4 successor stated.
- [x] Status is READY.
