# Merge Profiles Export — Automation Instructions

This file is the source of truth for the "Merge profiles" Intercom → Upfluence export
automation. The scheduled trigger's stored prompt should just point here (see
"Trigger config" at the bottom) so process changes are a git commit, not a
hand-edit of the trigger UI.

## Schedule

Two anchors, both weekdays only:
- **Lyon leg**: 3:50pm Lyon local time (CEST = UTC+2, or CET = UTC+1 — whichever is
  currently in effect).
- **Mexico leg**: 3:50pm Mexico City time, fixed UTC-6, no DST.

Always label the run "4 PM Run" in the Slack summary regardless of leg, to avoid
confusing the team with an odd time. This label is cosmetic only — never let it
change the actual search-window math below.

## 0. Determine which leg this is (do first, silently — no Slack mention)

Check the UTC hour this run fired at:
- Hour 21 → **Mexico leg** ("Mexico 4 PM Run").
- Hour 13 or 14 → **Lyon leg** ("Lyon 4 PM Run").

## 0a. Lyon DST self-check (Lyon leg only; silent; skip on Mexico leg)

Determine today's actual Lyon UTC offset. If the trigger's configured Lyon-leg hour
doesn't match (Lyon crossed a DST boundary since this last ran), the trigger's
schedule needs correcting to the new hour (13 for CEST, 14 for CET) — the Mexico
leg's hour stays fixed. This must currently be done manually in the trigger's
schedule UI; there is no tool access from within a run to edit the trigger itself.

## 1. Search Intercom efficiently

Call `search_conversations` with ALL of these filters in the SAME call:
- `tag_ids: ["10855456"]` — the "Merge profiles" tag ID. Do NOT search by tag name/text
  — the search DSL's `tag` field doesn't exist. This ID narrows results server-side to
  the real matches instead of the whole team queue.
- `team_assignee_id: 5131095`
- `state: "open"`, then repeat with `state: "snoozed"` (the tool takes one state per call).

Never call `search_conversations` with only `team_assignee_id`/`state` and no
`tag_ids` — that returns hundreds of unrelated conversations and forces an oversized
dump into a background agent for no reason.

**Pre-filter using the search results before calling `get_conversation` at all.**
Each search result already includes `statistics.last_contact_reply_at`. Use that to
discard any conversation whose last customer message falls outside this run's
window — do not spend a `get_conversation` call on it. This is the single biggest
lever for keeping the run cheap: `get_conversation` returns full conversation
history (sometimes weeks of it) and is expensive to read; only call it on
conversations that already look in-window from the search metadata.

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

A "Merge profiles"-tagged conversation whose last customer message falls in the
window still needs its content checked before counting as this run's work. Only
treat it as active if the in-window customer message(s) actually relate to
submitting/referencing merge-profile data for a creator (new links, a new file, or
a creator mentioned with zero links — 2a's fallback still handles that case). Pure
status chatter ("any update?", "thanks") or an unrelated issue on the same
historically-tagged thread does NOT count — do not extract data, do not flag it, do
not list it in the Slack "touched" line.

## Window definitions

- **Mexico leg:** since today 3:50pm Lyon time until now. Slack label: "today 4pm Lyon".
- **Lyon leg, Monday:** since last Friday 3:50pm Mexico City time until now (covers
  the weekend backlog). Slack label: "last Friday 4pm Mexico".
- **Lyon leg, Tue–Fri:** since yesterday 3:50pm Mexico City time until now. Slack
  label: "yesterday 4pm Mexico".

These anchors are the real computation boundaries; the display labels (and "4pm" in
general) are cosmetic only.

## 2. Classify each matching conversation

- **Type 1**: creator/social links pasted directly into a message.
- **Type 2**: client attached a CSV/Excel file.

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
- No matching note → default handling: zero links referenced → no CSV row, still
  list conversation in "touched"; one link with no more-complete note → flag
  "only one link sent — follow-up needed" per section 6.

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

## 8. Post to Slack

Post to **#merge-automation** (channel ID `C0BJJFZLRA6`) in this exact format:

```
_Merge profiles export ({leg label}) — {window start} → {actual run time} {weekday} {date}_
Type 1 (links): {N} conversations → {M} creators → [merge_{date}.csv](drive_link)
Type 2 (files): {N} file from {contact name} → reviewed & reformatted → [merge_{contact}_{date}.csv](drive_link)
⚠️ Flags:
- {Contact name} sent a {platform} link — needs follow up
- {Contact name}'s file had unreadable format ({format}) — skipped
```

- Title line italicized. `{leg label}` = "Mexico 4 PM Run" or "Lyon 4 PM Run".
  `{window start}` = the display label above (label only — math is the real anchor).
  `{actual run time} {weekday} {date}` = the real current time/day/date, pulled live,
  never hardcoded.
- Include a Type 1 line only if Type 1 tickets exist; one Type 2 line per Type 2
  ticket only if any exist.
- If flags exist, list them under `⚠️ Flags:`. If none:
  `⚠️ No flags gathered in this attempt` (italicized).
- End with an italicized line: `_Don't forget to add notes in tickets:_` followed by
  one markdown link per conversation actually in-scope per 1a (not every conversation
  the search returned): `[Contact Name](https://app.intercom.com/a/apps/k6viw85x/inbox/conversation/{conversation_id})`.
- If no matching conversations survive the 1a scope check, post just the title line
  followed by: `no conversations found`.

Note: Intercom notes on processed conversations are added manually by the team — do
not attempt to add them as part of this run.

## Trigger config

The trigger's stored prompt should be minimal — a pointer, not a copy of this file:

> Run the Merge Profiles export automation. Read and follow `AUTOMATION.md` in the
> root of the `merge-profiles-export` repo before doing anything else.

Schedule: weekdays, two anchors — Lyon 3:50pm local (13:50 UTC on CEST, 14:50 UTC on
CET) and Mexico 3:50pm fixed (21:50 UTC, no DST). Update the Lyon-leg hour manually
in the trigger UI when Lyon crosses a DST boundary (late March / late October).
