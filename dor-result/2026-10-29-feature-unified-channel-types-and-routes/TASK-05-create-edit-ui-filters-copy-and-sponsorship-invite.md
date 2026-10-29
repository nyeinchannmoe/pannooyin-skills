# Definition of Ready: Create/edit UI, filters, copy, and sponsorship invite eligibility

## 1. Document Metadata

| Field | Value |
| --- | --- |
| Task ID | T5 |
| Title | Create/edit UI, filters, copy, and sponsorship invite eligibility |
| Task type | Feature — frontend forms / filters / i18n / Vitest |
| Related feature | Restrict create types, unify channel URLs, and allow chapters on all channels |
| Work folder ID | `2026-10-29-feature-unified-channel-types-and-routes` |
| Source analysis | `analysis-result/2026-10-29-feature-unified-channel-types-and-routes/analysis.md` |
| Parent task-breakdown | `task-result/2026-10-29-feature-unified-channel-types-and-routes/tasks.md` |
| Document status | **READY** |

## 2. Objective and Expected Outcome

Florists can create only Standard and Sponsor channels with signup-like usernames. Filters still find historical agency/book rows and add Standard. Invite/join UI lets owned `standard` (and agency/book) join a sponsor, not a sponsor joining another channel. Create/edit hints no longer demand `_sponsor` suffixes.

After this task, creating `acme` as Standard lands on `/channels/@acme` (via T4 helpers) and that channel can be selected in the florist invite dropdown.

## 3. Context and Existing Behavior

**Confirmed project facts**

- `ChannelFormDialog.tsx` create options are Sponsor / Agency / Book (`CREATE_CHANNEL_TYPE_OPTIONS`). Hints always describe a required type suffix (`channelTypeHint` + `getRequiredChannelUsernameSuffix`).
- `UsernameField` + `useUsernameValidation`: when `channelType` is set, the field **requires** that type’s suffix and **skips** the normal-user underscore ban. `allowUsername` already treats the current stored username as available (edit of `brand_sponsor` can pass client validation **if** the suffix rule also passes — today it does because the stored name has the suffix). After T2, a rename to an unsuffixed name is valid on the API; the client suffix requirement would block that rename unless removed.
- `usernameRules.ts` `ChannelType` is `'sponsor' | 'agency' | 'book'` with `CHANNEL_USERNAME_SUFFIXES`. `normalUserUsernameContainsUnderscore` already exists for signup.
- `POST /api/v1/user/validateusername` remains suffix-agnostic and allows underscores (`john_01`). The create form must enforce the underscore ban client-side, as signup already does.
- Filters without Standard:
  - `ChannelsPanel.tsx` `CHANNEL_TYPE_OPTIONS`: All types, Sponsor, Agency, Book
  - `ChannelListPage.tsx` `CHANNEL_TYPE_OPTIONS`: All, Sponsor, Agency, Book
  - `ProfileChannelsSection.tsx` `CHANNEL_TYPE_FILTERS`: All, Sponsor, Agency, Book
  - `profileChannelsVisibility.ts` `SPECIFIC_CHANNEL_TYPES` = `sponsor`, `agency`, `book`
- `FloristDashboardPage.tsx` channels tab description: `Sponsor, agency, and book channels`.
- Invite: `channelInviteJoin.ts` `isSponsorshipEligibleOwnedChannel` is `agency` or `book` only. i18n `channels.noOwnedJoinChannelsMessage`: `You need an agency or book channel to request sponsorship.`
- T4 adds `'standard'` to `src/types/channel.ts` and `isChannelType`. This task must not regress that.

**Approved requirements:** create UI only Standard/Sponsor; create and rename usernames look like normal users; unchanged stored suffixed username remains valid via `allowUsername`; filters add Standard and keep Agency/Book; invite dropdown includes owned `standard`/`agency`/`book`, excludes `sponsor`.

## 4. Scope

**Included**

- `ChannelFormDialog.tsx` — create options Standard / Sponsor only; hints stop telling users to append `_sponsor` / `_agency` / `_book`, including on edit.
- `usernameRules.ts`, `useUsernameValidation.ts`, `UsernameField.tsx` — create and rename: no suffix required, underscore banned; `allowUsername` treats the **current** stored username as valid so a no-op save of `brand_sponsor` succeeds.
- Filter option arrays: `ChannelsPanel.tsx`, `ChannelListPage.tsx`, `ProfileChannelsSection.tsx`.
- `FloristDashboardPage.tsx` channels description / any help text that still promises agency/book **creation**.
- `src/utils/profileChannelsVisibility.ts` — `SPECIFIC_CHANNEL_TYPES` add `standard`.
- `src/utils/channelInviteJoin.ts` — `isSponsorshipEligibleOwnedChannel` includes `standard` (keep `agency`/`book`; still exclude `sponsor`).
- i18n `src/i18n/en.ts` and `my.ts` — suffix-required strings, invite empty-state, and any create copy that promises agency/book creation.
- Vitest listed in section 13.

**Excluded**

- Public route table (T4).
- API create/sponsorship implementation (T2).
- Backfill of stored usernames.
- Changing `validateusername`.
- Changing `/channels` directory route, invite-token path, or `MAX_CHANNEL_COUNT` UI beyond copy that mentions types.
- Re-adding Agency/Book as create options.

## 5. Technical Implementation Requirements

**Create type options**

- Dropdown values: `standard` and `sponsor` only (plus the existing empty “Select channel type” placeholder if still used).
- Labels: `Standard` and `Sponsor`.
- Edit mode still **displays** historical types (agency/book/sponsor/standard) as read-only context if shown today; it must not offer a type change control (API cannot change type).

**Username field (create and rename)**

- Channel create/edit usernames use the same client rules as flower/bee/florist signup: length 3–30, charset lowercase letters/numbers, **underscore banned**, reserved exact `pannooyin` still blocked by `validateusername` plus existing reserved-name handling.
- **Do not** call `hasRequiredChannelUsernameSuffix` for validity on create or on a **changed** username.
- **Do** apply `normalUserUsernameContainsUnderscore` for channel usernames (today the hook skips that when `channelType` is set).
- `allowUsername`: if the trimmed value equals the currently stored username, status is available **even when** that stored value contains `_` or a legacy suffix. This is the only underscore exception, and only for that exact current value.
- If the user edits `brand_sponsor` to `acme_sponsor` or any other underscore name, the field is invalid (do not wait for the API).
- `channelUsername && !channelType` “Select a channel type” remains appropriate on **create** until a type is chosen; it must not re-introduce suffix checks after a type is selected.
- Proposed: `CHANNEL_USERNAME_SUFFIXES` may remain as a detector for legacy stored names (tests/docs) but must not drive the required-suffix error path for create/rename.

**Hints / copy**

- Replace suffix hints in `ChannelFormDialog` (`channelTypeHint`) with signup-like guidance (no underscore, 3–30, letters and numbers). Edit hint must not say the username “must end with `_sponsor`”.
- `t.auth.usernameChannelSuffixRequired` should no longer be used on this form. Either stop referencing it or retarget it; do not leave the suffix message on create/edit.
- `FloristDashboardPage` description must not imply florists create agency/book channels. Proposed: mention Standard and Sponsor, and that existing agency/book channels remain listed.
- `t.channels.noOwnedJoinChannelsMessage` must include Standard (and still agency/book), not only agency/book.
- Myanmar (`my.ts`) keys that mirror these English strings must stay in sync.

**Filters**

- Add Standard to each filter list. Keep Sponsor, Agency, Book, and All so historical rows remain findable.
- Proposed order: All, Standard, Sponsor, Agency, Book (All first matches current pattern; Standard beside Sponsor is proposed, not a product mandate — do not hide Agency/Book).
- `isSpecificChannelType('standard')` must be true so profile empty-state logic treats Standard like other specific filters.

**Invite eligibility**

- `isSponsorshipEligibleOwnedChannel`: `standard` **or** `agency` **or** `book`. `sponsor` false.
- `shouldLoadOwnedChannelsForInvite` stays florist + target sponsor (no change required unless it hard-codes eligible owned types).
- `filterSponsorshipOwnedChannels` follows the helper.

**Navigation after create**

- `ChannelFormDialog` / panel success handlers that navigate to the new channel must use T4 `channelProfilePath` (`/channels/@username`). There is no `/standard/` prefix.

## 6. Affected Files and Components

**Modify**

| Path | Responsibility |
| --- | --- |
| `pann-oo-yin-web/src/components/channel/ChannelFormDialog.tsx` | Create options; hints; still pass `allowUsername` on edit |
| `pann-oo-yin-web/src/utils/usernameRules.ts` | Suffix map only for legacy detection; channel username helpers aligned with normal-user rules |
| `pann-oo-yin-web/src/hooks/useUsernameValidation.ts` | Underscore ban on channel create/rename; suffix not required; keep `allowUsername` bypass |
| `pann-oo-yin-web/src/components/auth/UsernameField.tsx` | Only if props/comments still say suffix is required for any `channelType` |
| `pann-oo-yin-web/src/components/channel/ChannelsPanel.tsx` | Filter options; any empty/help text that promises agency/book create |
| `pann-oo-yin-web/src/pages/ChannelListPage.tsx` | Filter options include Standard |
| `pann-oo-yin-web/src/components/channel/ProfileChannelsSection.tsx` | Filter chips include Standard |
| `pann-oo-yin-web/src/pages/FloristDashboardPage.tsx` | Channels tab description |
| `pann-oo-yin-web/src/utils/profileChannelsVisibility.ts` | `SPECIFIC_CHANNEL_TYPES` add `standard` |
| `pann-oo-yin-web/src/utils/channelInviteJoin.ts` | Eligible owned types include `standard` |
| `pann-oo-yin-web/src/i18n/en.ts` | Create/invite/username copy |
| `pann-oo-yin-web/src/i18n/my.ts` | Matching Myanmar copy |
| Tests in section 13 | |

**Create**

| Path | Responsibility |
| --- | --- |
| `pann-oo-yin-web/src/utils/usernameRules.test.ts` (and/or hook tests) | Create, unchanged legacy suffix, rename-to-normal-user |

Do not modify API source in this task.

## 7. Business Rules and Data Requirements

| Surface | Rule |
| --- | --- |
| Create dialog types | Standard and Sponsor only |
| Create username | Normal-user rules; no suffix; underscore invalid |
| Edit username unchanged | Stored `_sponsor` / `_agency` / `_book` is valid |
| Edit username changed | Normal-user rules; underscore invalid |
| Filters | Standard, Sponsor, Agency, Book, All |
| Invite dropdown | Owned standard, agency, book; never sponsor |
| Create copy | Must not promise agency/book creation |

No schema or stored-username backfill.

## 8. API and Integration Contracts

This task is a client of T2:

- `POST /api/v1/channel` with `agency`/`book` or suffixed usernames returns 400; the UI must not send those.
- `PUT /api/v1/channel` accepts unchanged legacy usernames and rejects underscore renames.
- Join/accept: `standard` → `sponsor` allowed; `sponsor` → anything not allowed. If the dropdown and API diverge, florists either cannot pick the new channel or they get `Don't allow to join this channel`.

`validateusername` is unchanged. Client-side underscore ban is mandatory.

Create success should navigate to T4 `/channels/@username`.

## 9. Dependencies and Prerequisites

- **T2** — create/sponsorship/username API contracts. Do not ship this UI against the old suffix-required API.
- **T4** — a newly created `standard` channel is only reachable at `/channels/@username`; `ChannelType` / `isChannelType` already include `standard`.
- T1 is implied through T2/T4 for persisting `standard`.
- T3 is not required for create/filter/invite; chapter buttons are T4.

**Ordering:** T1 → T2 and T3 in parallel after T1 → T4 after T3 (and T1) → **T5 last** after T2 and T4.

## 10. Edge Cases and Failure Handling

- `validateusername` returning success for `john_01` must not mark the channel field available; local underscore check runs first (signup already does this when `channelType` is unset).
- `allowUsername` bypass is **exact current stored username only**. Pasting a different suffixed name is invalid.
- Create blocked by `MAX_CHANNEL_COUNT` stays as today (`isChannelLimitError`); type change is not a workaround.
- Filter Standard against a florist who only has agency/book shows the existing empty state, not a crash.
- Invite page empty state when the florist owns only sponsor channels: updated copy, still no sponsor in the dropdown.

## 11. Security and Backward Compatibility

- Historical agency/book rows remain editable and filterable.
- Unchanged-legacy bypass prevents owners from being locked out of the edit dialog.
- Client must not send `agency`/`book` on create (breaking API in T2).
- No new secrets or env vars.

## 12. Acceptance Criteria

- Create dialog offers only Standard and Sponsor.
- Create username field rejects underscores, does not require a suffix, and still blocks reserved `pannooyin` client-side (`validateusername` remains suffix-agnostic).
- Edit of a channel whose stored username is `brand_sponsor` can save without changing the field; changing the field to an underscore name is invalid.
- Filters include Standard / Sponsor / Agency / Book / All.
- Florist invite dropdown includes owned `standard`, `agency`, and `book`, and excludes `sponsor`.
- Create/help copy no longer promises agency/book creation.

## 13. Testing and Verification

**Vitest**

- Extend `src/utils/profileChannelsVisibility.test.ts` — `isSpecificChannelType('standard')` is true; `all` still unfiltered.
- Extend `src/utils/channelInviteJoin.test.ts` — `standard`/`agency`/`book` eligible; `sponsor` not. Today’s test that expects only agency/book from `filterSponsorshipOwnedChannels` must include `standard`.
- Add `src/utils/usernameRules.test.ts` and/or hook tests for:
  - create: `acme` valid shape; `acme_sponsor` / any `_` invalid; no required suffix
  - unchanged legacy: stored `brand_sponsor` allowed via `allowUsername`
  - rename to normal-user: `acme` valid; `acme_sponsor` invalid

**Manual**

- Create `standard` with username `acme`, land on `/channels/@acme`.
- Join that channel to a sponsor from the invite UI.
- Save an existing `_agency` channel without renaming.
- Confirm filters still list an existing book/agency row.

## 14. Risks, Assumptions, and Open Questions

**Assumptions**

- Suffix map may remain in `usernameRules.ts` for detecting legacy names; it must not be required on create/rename.
- Filter chip order is proposed as All / Standard / Sponsor / Agency / Book.

**Risks**

- Leaving `useUsernameValidation` suffix-required when `channelType` is set blocks unsuffixed create and rename even if T2 is done.
- Changing only the invite dropdown without T2 (or vice versa) splits web and API.
- Creating Standard before T4 ships sends users to a 404 (no `/standard/` prefix).

**Open questions**

- None that block implementation. Product already resolved no suffix on rename and Standard sponsorship.

## 15. Definition of Ready Checklist

- [x] Create options, username modes, filters, and invite eligibility specified against live files.
- [x] `allowUsername` legacy bypass and client-side underscore ban specified.
- [x] i18n keys that still promise agency/book create or agency/book-only sponsorship identified.
- [x] T2 and T4 prerequisites stated; this is the last task in the breakdown.
- [x] Tests and manual checks listed.
- [x] Status is READY.
