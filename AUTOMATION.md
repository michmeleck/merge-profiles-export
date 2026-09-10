# Merge Profiles Export — Automation Instructions

This file is the source of truth for the "Merge profiles" Intercom → Upfluence export
automation. The scheduled trigger's stored prompt should just point here (see
"Trigger config" at the bottom) so process changes are a git commit, not a
hand-edit of the trigger UI.

## Schedule

One trigger, one cron, two anchors, both weekdays only — cron `58 13,21 * * 1-5`:
- **Lyon leg**: 13:58 UTC — 3:58pm Lyon local time (CEST = UTC+2, or CET = UTC+1,
  whichever is currently in effect; shifts to 14:58 UTC on CET).
- **Mexico leg**: 21:58 UTC fixed — 3:58pm Mexico City time, UTC-6, no DST. This
  value must never change.

Always label the run "4 PM Run" / "4pm" in the Slack summary regardless of leg, to
avoid confusing the team with an odd time. This label is cosmetic only — never let
it change the actual search-window math below.

## 0. Determine which leg this is (do first, silently — no Slack mention)

Check the UTC hour this run fired at:
- Hour 21 → **Mexico leg** ("Mexico 4 PM Run", target 3:58pm Mexico City time).
- Hour 13 or 14 → **Lyon leg** ("Lyon 4 PM Run", target 3:58pm Lyon local time).

## 0a. Lyon DST self-check (Lyon leg only; silent; skip entirely on Mexico leg)

The cron's first value is `13` (CEST, UTC+2) or should be `14` (CET, UTC+1).
Determine today's actual Lyon UTC offset. If it doesn't match what the stored
cron's first hour value assumes (Lyon crossed a DST boundary since this last ran),
the trigger's cron needs correcting to `58 14,21 * * 1-5` (CET) or
`58 13,21 * * 1-5` (CEST) — keep the second value fixed at `21` (Mexico leg). This
must currently be done manually in the trigger's schedule UI; there is no tool
access from within a run to edit the trigger itself. This is routine housekeeping —
do not mention it in the Slack post.

## 1. Search Intercom efficiently

Call `search_conversations` with ALL of these filters in the SAME call:
- `tag_ids: ["10855456"]` — the "Merge profiles" tag ID. Do NOT search by tag name/text
  — the search DSL's `tag` field doesn't exist. This ID narrows results server-side to
  the real matches instead of the whole team queue. If a search using this ID ever
  comes back empty/suspicious across both states, double-check by fetching one known
  "Merge profiles" conversation and confirming its tag id still reads `10855456`
  (tag IDs are workspace-specific and would only change if the tag were deleted and
  recreated).
- `team_assignee_id: 5131095`
- `state: "open"`, then repeat with `state: "snoozed"` (the tool takes one state per call).

Never call `search_conversations` with only `team_assignee_id`/`state` and no
`tag_ids` — that returns hundreds of unrelated conversations and forces an oversized
dump into a background agent for no reason.

**Pre-filter using the search results before calling `get_conversation` at all —
but anchor on `updated_at`, not `last_contact_reply_at` alone.** Each search
result includes both. `last_contact_reply_at` only reflects the customer; it does
NOT bump when a teammate adds an internal note, so filtering on it alone can skip a
conversation before its content is ever read — the exact failure mode 1a(b) exists
to prevent, just one step earlier. `updated_at` bumps on notes, tags, and snoozes
too, so use `updated_at` to decide whether to spend a `get_conversation` call.
This is still the single biggest lever for keeping the run cheap: `get_conversation`
returns full conversation history (sometimes weeks of it) and is expensive to read;
only skip it for conversations whose `updated_at` is outside this run's window.

Once you have the matching in-window IDs, call `get_conversation` on each
individually — this is required to see internal notes and full message bodies,
which search results omit.

### Handling oversized `get_conversation` results

If a `get_conversation` result is too large to read directly and gets saved to a
file, **do NOT spawn a subagent to read the whole file front-to-back.** That
pattern cost ~119,000 tokens on a conversation that only needed its last ~9 parts
plus a scan for internal notes — by far the most expensive step in a typical run.

Instead:
1. `Grep` the saved file for `"part_type": "note"` and for social-URL patterns
   (`instagram\.com|tiktok\.com|youtube\.com|twitch\.tv|pinterest\.com|x\.com|twitter\.com`)
   to locate internal notes and links without reading the whole file.
2. `Read` only the last ~300 lines of the file directly — conversation parts are
   chronological, so anything relevant to *this run's* window is always at the tail.
3. Only fall back to a full-file read if these targeted reads don't answer the
   question.

## 1a. Scope check — surfaced ≠ touched

A "Merge profiles"-tagged conversation still needs its content checked before
counting as this run's work — do not decide inclusion from the customer's
last-message timestamp alone. Treat a conversation as active for this run if
EITHER:

(a) an in-window **customer** message actually relates to submitting/referencing
    merge-profile data for a creator (new links, a new file, or a creator
    mentioned with zero links — 2a's fallback still handles that case); OR
(b) an in-window **internal note** (team-only) contains supported-platform URLs or
    a `software.upfluence.co` profile link for a creator — this counts as active
    work on its own, even if the customer's message(s) in this same window are
    unrelated or there are none at all. Anchor this check to the *note's own*
    `created_at` against the window, not the customer message's timestamp.

If neither (a) nor (b) holds — e.g. the in-window customer message(s) are pure
status chatter ("any update?", "thanks") or an unrelated issue on the same
historically-tagged thread, and no note in that window carries new mergeable data —
do not extract data, do not flag it, do not list it in the Slack "touched" line.

**Why (b) exists:** on 2026-09-10 a teammate staged a full link set for two
creators in an internal note (added ~15:00–15:24 UTC on 2026-09-09), which fell
inside the *prior* Mexico-leg window (13:58–21:58 UTC), but no customer message
landed in that same window to anchor it. Customer-message-only windowing missed it
completely, and it wasn't caught until a human found it manually. Rule (b) closes
that gap: a note is now a valid anchor in its own right, not just a supplement to
an in-window customer message.

## Window definitions

- **Mexico leg:** since today 3:58pm Lyon time (whichever of CEST/CET currently
  applies) until now — i.e. since today's Lyon-leg run. Slack label: "today 4pm Lyon".
- **Lyon leg, Monday:** since last Friday 3:58pm Mexico City time until now (covers
  the weekend backlog — no runs Sat/Sun). Slack label: "last Friday 4pm Mexico".
- **Lyon leg, Tue–Fri:** since yesterday 3:58pm Mexico City time until now. Slack
  label: "yesterday 4pm Mexico".

These anchors are the real computation boundaries; the display labels (and "4pm" in
general) are cosmetic only.

## 2. Classify each matching conversation

- **Type 1**: creator/social links pasted directly into a message.
- **Type 2**: client attached a CSV/Excel file.
- **Type 3**: client reports an existing Upfluence profile has gone blank (lost its
  linked social account, e.g. because the creator changed handle or made security
  changes) and shares the replacement social URL(s) for that SAME creator. This is a
  merge-blank-profile request, not new creator data — handle it per 2b, not per 3/4.

Only extract from CUSTOMER messages sent within the window — ignore older messages
in the same thread (handled by a prior run).

## 2a. Fallback via internal notes

The team sometimes stages a creator's complete set of supported-platform URLs in an
internal note (team-only) — either because the customer sent zero links, or sent
only a partial set and a teammate researched the rest. Before flagging "no
extractable data" or "only one link sent" for any creator, check for a matching
internal note with URLs for that creator:
- Customer message has no links/file for a creator, but a fallback note has URLs →
  extract from the note per section 3, as if the customer had sent them.
- Customer message has a partial set (e.g. one platform), and a note has a more
  complete set for the same creator → use the note's set, don't merge with the
  partial message, don't flag "only one link."
- Notes can also be the ONLY source of a case — don't require the customer to have
  referenced the creator in-window at all. If an internal note added within this
  run's window stages a complete (or partial) set of supported-platform URLs for a
  creator, even one the customer never mentioned in this window (or mentioned only
  in an earlier run's window), treat it as new work for this run per section 3,
  anchored to the note's own `created_at` against the window — see 1a(b).
- No matching note → default handling: zero links referenced → no CSV row, still
  list conversation in "touched"; one link with no more-complete note → flag
  "only one link sent — follow-up needed" per section 6.

## 2b. Type 3 handling — merge blank profile (Mexico leg only)

**Mexico leg only, for now.** This is a trial of the Type 3 auto-ticket flow — skip
this section entirely on the Lyon leg: do not check for it, and do not mention it
in the Slack post for a Lyon-leg run. Revisit this restriction once there's
feedback on how the Mexico-leg tickets are landing.

**Window — separate from the Type 1/Type 2 window above, computed independently,
once per day:** since yesterday's Mexico-leg run (yesterday 3:58pm Mexico City
time, UTC-6 fixed, no DST) until now — a rolling 24h window. Display label if
referenced: "since yesterday 4pm Mexico". This is intentionally wider than the
main Mexico-leg window so this case only needs one pass per day.

**Detection:** within that window, look for a CUSTOMER message referencing an
existing Upfluence profile link of the form
`https://software.upfluence.co/irm/influencers/{id}` that the client is asking to
merge with other creator/social link(s) they've provided (in that same message or
elsewhere in the same in-window exchange). Recognize this by the shape of the
conversation, not a keyword — the client is pointing at an existing (typically
blank) Upfluence profile to combine with new accounts, not just submitting fresh
social links for a new creator entry.

**Notes fallback:** a teammate note (team-only, not customer-visible) can supply
either piece of data exactly as if the customer had sent it:
- If the customer's in-window message(s) give the merge-target social link(s) but
  NOT the `software.upfluence.co` profile link itself, check the conversation's
  internal notes for a team-provided profile link before ruling this case out.
- If the customer's in-window message says a profile is blank/can't be found but
  gives no social link, and a note supplies the replacement social URL(s), use the
  note's set per 2a's logic.
- Only skip 2b entirely for a given creator if neither the customer's message nor a
  note contains a `software.upfluence.co` profile link, or neither contains a
  replacement social link.

For each Type 3 case found (this is "Type 3" in the Slack summary):
1. Do NOT add a row for this creator to the Type 1 CSV — this case is routed to
   Support via a Linear ticket instead of the merge script.
2. Pull the contact's email from the conversation's contact info. Pull the
   Upfluence User ID from the contact's custom attributes (`external_id`); if none
   is present, fall back to the Intercom contact ID and label it
   "(Intercom contact ID — verify)" in the ticket.
3. Pull the company/team name from the conversation's company info, abbreviated,
   for the ticket title — mirror LIB-952's title pattern
   (https://linear.app/upfluence/issue/LIB-952/merge-blank-profile-tla).
4. Do NOT call `mcp__Linear__save_issue` or `mcp__Linear__create_attachment` —
   confirmed (2026-09-10) that both return `No such tool available` in this
   execution environment. They exist in the connector but are set to
   "ask for confirmation," and this session type has no channel to satisfy that
   ask — it's not a pending-approval state, the call fails outright, every time,
   with nothing to wait on. Do not attempt the calls, do not treat a failure here
   as retryable, and do not let this block anything else in the run.
   Instead, prepare the complete ticket content below for a human to paste into
   Linear by hand — this is now the deliverable for Type 3, reported per 8b:
   - Team: Support (`3f272a17-cc3b-4e90-b74d-16fac7701c18`)
   - Title: `Merge blank profile - {short client/company identifier}`
   - Labels: `ICP`, `Service`
   - Priority: High
   - Assignee: none (unassigned)
   - State: **Triage** — set this explicitly. Leaving the issue merely unassigned
     does NOT route it to Triage on its own (confirmed: it defaults to Backlog) —
     the state must be set.
   - Description, mirroring LIB-952's exact shape (one numbered sub-item per
     additional social link if more than one was provided):
     ```
     ## Description

     Client is asking us to merge a blank creator profile with their new social media account(s):

     1. Blank profile: [{profile_url}]({profile_url}) → merge with:
        1. [{social_url_1}]({social_url_1})
        2. [{social_url_2}]({social_url_2})
        (one numbered sub-item per additional link provided)

     ## Account information

     * Email: {email}
     * User ID: {user_id}
     ```
   - Conversation link to attach (a teammate adds this manually when creating the
     issue): `https://app.intercom.com/a/apps/k6viw85x/conversations/{conversation_id}`.
5. This conversation counts as "touched" for the 1a scope check and belongs in the
   Slack "touched" list, but it does NOT go into either CSV.
6. Track the fully-formatted content for each case (title, labels, priority,
   description, conversation link) to report per section 8b. Nothing here is
   "pending approval" anymore — there's no tool call in flight to wait on — so 8b
   can go out in the same run as section 8, not as a later follow-up.

## 3. Type 1 extraction rules

- One row per creator. Supported platforms only: instagram_url, x_url, twitch_url,
  tiktok_url, pinterest_url, youtube_url, blog_url.
- name: from the message if present, else Instagram handle, else TikTok handle.
- email: only if explicitly shared per-creator in the message text; else blank.
- Combine ALL Type 1 tickets from this run into ONE combined CSV,
  `merge_{YYYY-MM-DD}.csv` (today's date).

## 4. Type 2 extraction rules

- Download the attached CSV/Excel file, validate its structure against the required
  format (section 5), fix it if off (headers, column order, misplaced values).
- Output each Type 2 ticket as its OWN CSV,
  `merge_{contact_name}_{YYYY-MM-DD}.csv` — never combine Type 2 tickets together or
  with the Type 1 CSV.

## 5. CSV format — exact headers, in order

```
name,email,instagram_url,x_url,twitch_url,tiktok_url,pinterest_url,youtube_url,blog_url
```

## 6. Flag conditions

Do NOT include a flagged row/file in the CSV. Track flags for the Slack summary:
- Unsupported platform sent (Facebook, Snapchat, etc.)
- Only one link sent for a creator, with no more-complete internal note found — follow-up needed
- Malformed/invalid URL
- Unreadable file (Type 2)
- URLs in wrong columns (Type 2)

## 7. Upload CSVs to Google Drive

Upload each CSV (combined Type 1, plus one per Type 2 ticket) into the "Merge
files" Drive folder, folder ID `1QLkaORi9cB8TWeRh02qgjYd87ecXERgS`. Get each file's
shareable view URL (`https://drive.google.com/file/d/FILE_ID/view`).

## 8. Post to Slack — main summary (send immediately, do NOT wait on Linear)

As soon as Type 1/Type 2 processing and the Drive uploads in section 7 are done,
post to **#merge-automation** (channel ID `C0BJJFZLRA6`) in this exact format.
Section 2b no longer calls a tool that could hang or need approval, so there's
nothing to wait on — this message and 8b's follow-up can both go out in the same
run, back to back.

```
**_{leg label}_** → from {window start}
Type 1 (links): {N} conversations → {M} creators → [merge_{date}.csv](drive_link)
Type 2 (files): {N} file from {contact name} → reviewed & reformatted → [merge_{contact}_{date}.csv](drive_link)
Type 3 (blank profile): {N} case(s) — manual Linear ticket, see next message
:warning: Flags:
- {Contact name} sent a {platform} link — needs follow up
- {Contact name}'s file had unreadable format ({format}) — skipped
```

- Title line: only `{leg label}` is bold AND italicized (`**_{leg label}_**`).
  Follow it with plain text ` → from {window start}` — a literal arrow, not a dash.
  No parentheses around the leg label. `{leg label}` = "Mexico 4 PM Run" or "Lyon 4
  PM Run" per step 0. `{window start}` = the display label from the Window section
  above (label only — math is the real anchor). Do NOT include the actual run
  time, weekday, or date in the title line.
- Include a Type 1 line only if Type 1 tickets exist; one Type 2 line per Type 2
  ticket only if any exist.
- Include the `Type 3 (blank profile): {N} case(s) — manual Linear ticket, see next message`
  line only if section 2b found at least one blank-profile case this run
  (Mexico-leg only, never on Lyon-leg). Omit entirely if section 2b found zero cases.
- Include the `:warning: Flags:` section, with the list beneath it exactly as
  above, ONLY when at least one real flag exists this run. If there are no
  flags, omit the entire Flags section — no heading, no emoji, and no "no
  flags" line of any kind. The section should simply not appear at all when
  nothing was flagged.
- End with an italicized line: `_Don't forget to add notes in tickets:_` followed by
  one markdown link per conversation actually in-scope per 1a (not every conversation
  the search returned): `[Contact Name](https://app.intercom.com/a/apps/k6viw85x/inbox/conversation/{conversation_id})`.
- If no matching conversations survive the 1a scope check, post just the title line
  followed by: `no conversations found`.

## 8b. Post Type 3 manual-ticket content — separate message, same run (Mexico leg only)

Only relevant on a Mexico-leg run where section 2b found at least one blank-profile
case. Immediately after posting section 8's summary (no waiting on anything), post
a SECOND, separate message to the same **#merge-automation** channel with the
ready-to-paste content for each case, so a teammate can create the Linear issue(s)
by hand in under a minute:

```
Type 3 (blank profile) — {N} case(s), paste into Linear manually:

**Case 1 — {short client/company identifier}**
Team: Support · Labels: ICP, Service · Priority: High · State: Triage
Title: Merge blank profile - {short client/company identifier}
Description:
​```
## Description

Client is asking us to merge a blank creator profile with their new social media account(s):

1. Blank profile: [{profile_url}]({profile_url}) → merge with:
   1. [{social_url_1}]({social_url_1})

## Account information

* Email: {email}
* User ID: {user_id}
​```
Attach: https://app.intercom.com/a/apps/k6viw85x/conversations/{conversation_id}

(repeat as **Case 2**, **Case 3**, etc. for each additional case this run)
```

`{N}` = count of cases found this run. One `Case` block per distinct blank-profile
case per section 2b (not per conversation — a single conversation can produce more
than one). Skip this section entirely (post nothing) if section 2b found zero
blank-profile cases this run, or if this is a Lyon-leg run. Keep this as its own
message rather than folding it into section 8 — the main summary should stay
short and scannable; the paste-ready content is bulkier and belongs separately.

Note: Intercom notes on processed conversations are added manually by the team — do
not attempt to add them as part of this run.

## Trigger config

The trigger's stored prompt should be minimal — a pointer, not a copy of this file:

> Run the "Merge Profiles" export from Intercom to Upfluence. Read and follow
> `AUTOMATION.md` in the root of the `merge-profiles-export` repo before doing
> anything else — it is the source of truth for this automation. Do not rely on
> memory of past runs' instructions; always re-read the file fresh, since it is
> updated independently via git commits.

Schedule: weekdays, one cron covering both anchors — `58 13,21 * * 1-5` (13:58 UTC
Lyon leg on CEST, shifts to 14:58 on CET; 21:58 UTC Mexico leg, fixed, no DST).
Update the Lyon-leg hour manually in the trigger UI when Lyon crosses a DST
boundary (late March / late October).
