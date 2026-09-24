---
name: recruiter-triage
description: Search Gmail for inbound recruiter/opportunity messages, classify them, and create or update the "Job Search - Recruiter Tracker" Google Sheet. Use whenever the user asks to check for new recruiter emails, refresh the job tracker, or re-run recruiter triage.
---

# Recruiter triage → tracker sheet

Manual (on-demand) pipeline step. Do NOT wrap this in a scheduled/cron routine unless the user explicitly asks — this project's stated preference is to validate manual runs before automating (see project memory `feedback-verify-before-automating`).

## 1. Confirm which Gmail account is connected

The Gmail connector (`mcp__claude_ai_Gmail__*`) only supports **one account at a time** — connecting a second account requires disconnecting the first. Before running searches, check the `toRecipients` field on any result to confirm which inbox you're actually searching, and say so if it's ambiguous. Known accounts for this user:
- `tylercmartinez@gmail.com` — the LinkedIn-linked address; primary source of recruiter InMail/connection-request traffic.
- `slightlytyler@gmail.com` — current daily-driver inbox; occasional direct agency recruiter emails and Indeed digests.

## 2. Search

**Don't default to `newer_than:90d`.** On 2026-09-23, a 90-day window missed 15 real recruiter relationships spanning back to 2026-05-05 — including the single highest-value lead found so far (Josh Markowitz, three roles up to $1.5M comp) — because the sabbatical (and this pipeline) started in May, before any triage was happening, and a 90-day rolling window keeps sliding past that start date. **Use `after:2026/05/01` (Tyler's sabbatical start) instead of a relative `newer_than`, every run, until the user says otherwise.** A relative window is fine only once the backlog is confirmed clear and the user explicitly wants to move to incremental runs.

Run both queries with `search_threads` (`pageSize: 50`, `view: THREAD_VIEW_MINIMAL`):

```
in:inbox after:2026/05/01 (recruiter OR recruiting OR "talent acquisition" OR "reaching out" OR "open role" OR "open position" OR "job opportunity" OR hiring) -category:promotions

in:inbox after:2026/05/01 (from:linkedin.com OR from:indeed.com OR from:greenhouse.io OR from:lever.co OR from:myworkday.com OR from:ziprecruiter.com)
```

The first query is full-text and can exceed the tool's output size over a long date range — if it errors on size, split it by date into smaller chunks (e.g. `after:X before:Y` windows a few weeks wide) rather than narrowing the date range back down, and also add `-from:linkedin.com` to it once you've separately covered LinkedIn via the second query, since direct-agency-email senders (their own company domains, e.g. `@aspensearch.com`, `@ingreatco.com`) are exactly what the keyword query catches that the LinkedIn-sender query cannot.

Adjust the keyword list if results look thin, but keep queries concise (Gmail search penalizes long literal phrases). If the user references a specific message, name, or company you don't have in the tracker, don't assume you mis-scanned what you fetched — search by that exact name/phrase directly (drop the date filter entirely, `includeTrash: true`) before concluding it doesn't exist. That's how both the Josh Markowitz gap and this whole May–June backlog were found.

## 3. Classify: real outreach vs. noise

**Keep** (these are trackable opportunities):
- Direct emails from a named person at a recruiting agency or company (has a real reply-able email address).
- LinkedIn InMail (`sender: inmail-hit-reply@linkedin.com`) — a real recruiter pitching a real role.
- LinkedIn connection requests (`sender: invitations@linkedin.com`) whose personalized note — fetched in full, see step 4 — references a role, hiring, or a company. **Don't gate this on job title.** Founders, hiring managers, and generic "I work with 250+ startups" senders all pitch real roles this way just as often as people with a "recruiter" title; the title alone is not a reliable signal (see the Josh Markowitz / Hits Only example in step 4).

**Discard as noise** (do not add rows for these):
- `messages-noreply@linkedin.com` salary/market digests ("Senior Software Engineer insights: $XXXk/yr..."), "X is a top company hiring near you," "X people viewed your profile," "You're getting noticed."
- `updates-noreply@linkedin.com` feed activity ("X recently posted," "X reacted to this post," "X and others share their thoughts").
- `editors-noreply@linkedin.com` — LinkedIn News editorial content.
- Automated job-board match digests (e.g. `donotreply@match.indeed.com` "your background could be a match," `no-reply@indeed.com` "jobs based on your skills") — algorithmic suggestions, not recruiter outreach.
- Anything clearly unrelated (Nextdoor, receipts, newsletters) that happened to match a keyword.

If genuinely unsure whether something counts, ask the user rather than guessing — cheap to check, expensive to pollute the tracker.

## 4. Critical extraction rule: always fetch the full thread body — for InMail *and* connection requests

**Never classify or write a row from the `search_threads` snippet alone, for either message type.** The snippet is unreliable in different ways for each:

**InMail** (`sender: inmail-hit-reply@linkedin.com`): the `from` header and subject line are useless for identifying who sent it — every InMail comes from the same relay address regardless of recruiter, and the subject is just the pitch's headline.

**Connection requests** (`sender: invitations@linkedin.com`): the snippet usually shows only generic boilerplate — "X is waiting for your response," "X wants to connect" — even when the sender wrote a real personalized note pitching a specific role. That note is invisible until you fetch the thread. Example: a connection request from "Josh Markowitz, Founder @ Hits Only" had subject "I want to connect" and a snippet that said only "waiting for your response" — but the actual note read *"I might have an epic frontend role for you @ Circle or some other fantastic positions if you open to NYC."* Skipping the fetch on connection requests (as this skill used to instruct) misses exactly this kind of message, and title-based filtering would have excluded it too since "Founder" isn't a recruiting title.

To get the real name, agency, full pitch, or personal note: call `get_thread` with `messageFormat: PLAIN_TEXT` on **every** InMail and every connection request before deciding whether to keep it or how to fill its row. Use `PLAIN_TEXT`, not `FULL_CONTENT`'s `html_body` — LinkedIn's HTML rendering can drop the personalized note entirely (confirmed on the Josh Markowitz thread: `html_body` only had the generic invite template plus "people you may know," while `plaintext_body` had the actual note) even though it's present in plaintext.

For InMail, the recruiter's LinkedIn display name is the **first line of the body, directly above the repeated subject line and the "Reply" / `linkedin.com/messaging/thread/...` link**, e.g.:

```
Let's Chat! Senior Full Stack Engineer @ Airbyte
Let's Chat! Senior Full Stack Engineer @ Airbyte

      Abby R.
        Reply
        https://www.linkedin.com/messaging/thread/...

Hey Tyler!
...
```

Their title/agency is usually in the signature block at the end of the body (e.g. "Abby — Senior Technical Recruiter @ Airbyte", "Griffin Lewin — Director of Recruiting @ Quantum"). Never leave a row as "(unnamed recruiter)" without having tried `get_thread` first. While in there, also grab: comp range, location/remote policy, and whether the sender is in-house or a third-party agency (phrases like "I work with 250+ VC-backed startups" or "partnering with [company]" signal agency/embedded recruiter, not in-house).

For connection requests, the personal note (if any) appears after the sender's name/title, before the "Accept" / "View profile" links — read it in full and use it to decide both inclusion (step 3) and the Role(s) Mentioned field, rather than defaulting to "Unspecified - no role pitched yet" without checking.

## 5. Tracker sheet schema

One row per **recruiter relationship**, not per role — a single recruiter/company thread may branch into multiple roles over time, which goes in Notes rather than becoming a new row. Columns:

| Column | Notes |
|---|---|
| Date First Contact | |
| Recruiter Name | Include title/agency inline where known, e.g. `Thomas Clark - Principal Recruitment Consultant @ Iterate (agency)` |
| Company | The end client/employer, not the agency, when known |
| Source | `Direct Email` / `LinkedIn InMail` / `LinkedIn Connection Request` |
| Contact Info | Real email for direct outreach; `Reply via LinkedIn thread (no email exposed)` for InMail/connection requests |
| Role(s) Mentioned | |
| Status | Three values: `Not Responded` (default) / `Drafted` (a `recruiter-warm-intro` reply exists — as a Gmail draft or LinkedIn text handed to Tyler — but he hasn't confirmed sending it) / `Responded` (Tyler confirmed a reply actually went out) |
| Gmail Thread Link | `https://mail.google.com/mail/u/0/#all/<threadId>` |
| Source Inbox | Which connected Gmail account this came from (see step 1) |
| Notes | Comp, location, flags (e.g. duplicate/lookalike-domain outreach, cross-references between recruiters at the same agency), anything time-sensitive |

## 6. Creating / updating the sheet

**Confirmed limitation (checked 2026-09-15, accepted — do not re-litigate this each run):** the Google Drive connector (`mcp__claude_ai_Google_Drive__*`) **cannot edit an existing spreadsheet's cells** — `update_file` only changes title/parent, and there is no separate Google Sheets connector available to this account (checked in Settings → Connectors; not offered). Real in-place cell/range updates are not possible with the current toolset. Recreate-and-replace is the accepted permanent approach, not a stopgap — don't re-propose adding a Sheets connector or re-check for one unless the user brings it up.

To "update" the tracker:

1. Compose the full CSV (all existing rows + new/corrected ones — always the complete dataset, not a diff). **Wrap every field containing a comma in double quotes** (and double up any literal `"` inside it) — an unquoted comma anywhere (a title like `"Founder @ X, Y"`, a note like `"(checked, none found)"`) silently shifts every column after it for that row. After composing, re-scan every field you just wrote for a bare `,` outside quotes before calling `create_file` — this has broken rows in practice (2026-09-23 run).
2. `create_file` with the same `title` ("Job Search - Recruiter Tracker"), `contentMimeType: text/csv`, and the full CSV as `textContent`. Drive auto-converts CSV to a native Google Sheet.
3. `trash_file` the previous version's fileId (reversible — do not hard-delete).
4. Give the user the new file's `viewUrl`. **The URL changes on every run** — always hand over the fresh link rather than assuming the old one still resolves to current data (the old one still resolves, but to stale/trashed content).
5. Keep row order stable (append/update in place within the CSV you compose, don't re-sort) so diffs between runs stay easy to eyeball.

Always `read_file_content` on the newly created sheet once to confirm the data landed correctly before reporting success.
