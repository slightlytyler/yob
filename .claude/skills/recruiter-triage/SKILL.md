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

Run both queries with `search_threads` (`pageSize: 50`, `view: THREAD_VIEW_MINIMAL`, `newer_than` tuned to how far back you need):

```
in:inbox newer_than:90d (recruiter OR recruiting OR "talent acquisition" OR "reaching out" OR "open role" OR "open position" OR "job opportunity" OR hiring) -category:promotions

in:inbox newer_than:90d (from:linkedin.com OR from:indeed.com OR from:greenhouse.io OR from:lever.co OR from:myworkday.com OR from:ziprecruiter.com)
```

Adjust the keyword list if the results look thin — but keep queries concise (Gmail search penalizes long literal phrases).

## 3. Classify: real outreach vs. noise

**Keep** (these are trackable opportunities):
- Direct emails from a named person at a recruiting agency or company (has a real reply-able email address).
- LinkedIn InMail (`sender: inmail-hit-reply@linkedin.com`) — a real recruiter pitching a real role.
- LinkedIn connection requests from someone whose title is recruiting-related (`sender: invitations@linkedin.com`), even with no role pitched yet.

**Discard as noise** (do not add rows for these):
- `messages-noreply@linkedin.com` salary/market digests ("Senior Software Engineer insights: $XXXk/yr..."), "X is a top company hiring near you," "X people viewed your profile," "You're getting noticed."
- `updates-noreply@linkedin.com` feed activity ("X recently posted," "X reacted to this post," "X and others share their thoughts").
- `editors-noreply@linkedin.com` — LinkedIn News editorial content.
- Automated job-board match digests (e.g. `donotreply@match.indeed.com` "your background could be a match," `no-reply@indeed.com` "jobs based on your skills") — algorithmic suggestions, not recruiter outreach.
- Anything clearly unrelated (Nextdoor, receipts, newsletters) that happened to match a keyword.

If genuinely unsure whether something counts, ask the user rather than guessing — cheap to check, expensive to pollute the tracker.

## 4. Critical extraction rule: LinkedIn InMail sender name

**The `from` header and subject line are useless for identifying who sent an InMail** — every InMail comes from `inmail-hit-reply@linkedin.com` regardless of recruiter, and the subject is just the pitch's headline. The `search_threads` snippet also won't reliably show the name.

To get the real name, agency, and full pitch: call `get_thread` with `messageFormat: PLAIN_TEXT` on that thread. The recruiter's LinkedIn display name is the **first line of the body, directly above the repeated subject line and the "Reply" / `linkedin.com/messaging/thread/...` link**, e.g.:

```
Let's Chat! Senior Full Stack Engineer @ Airbyte
Let's Chat! Senior Full Stack Engineer @ Airbyte

      Abby R.
        Reply
        https://www.linkedin.com/messaging/thread/...

Hey Tyler!
...
```

Their title/agency is usually in the signature block at the end of the body (e.g. "Abby — Senior Technical Recruiter @ Airbyte", "Griffin Lewin — Director of Recruiting @ Quantum"). Always fetch the full body for every InMail thread before writing a tracker row — never leave a row as "(unnamed recruiter)" without having tried `get_thread` first. While in there, also grab: comp range, location/remote policy, and whether the recruiter is in-house or a third-party agency (phrases like "I work with 250+ VC-backed startups" or "partnering with [company]" signal agency/embedded recruiter, not in-house).

LinkedIn connection-request emails (`invitations@linkedin.com`) already include the name/title/company in the snippet — no extra fetch needed there.

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
| Status | Only two values: `Responded` / `Not Responded` |
| Gmail Thread Link | `https://mail.google.com/mail/u/0/#all/<threadId>` |
| Source Inbox | Which connected Gmail account this came from (see step 1) |
| Notes | Comp, location, flags (e.g. duplicate/lookalike-domain outreach, cross-references between recruiters at the same agency), anything time-sensitive |

## 6. Creating / updating the sheet

**Confirmed limitation (checked 2026-09-15, accepted — do not re-litigate this each run):** the Google Drive connector (`mcp__claude_ai_Google_Drive__*`) **cannot edit an existing spreadsheet's cells** — `update_file` only changes title/parent, and there is no separate Google Sheets connector available to this account (checked in Settings → Connectors; not offered). Real in-place cell/range updates are not possible with the current toolset. Recreate-and-replace is the accepted permanent approach, not a stopgap — don't re-propose adding a Sheets connector or re-check for one unless the user brings it up.

To "update" the tracker:

1. Compose the full CSV (all existing rows + new/corrected ones — always the complete dataset, not a diff).
2. `create_file` with the same `title` ("Job Search - Recruiter Tracker"), `contentMimeType: text/csv`, and the full CSV as `textContent`. Drive auto-converts CSV to a native Google Sheet.
3. `trash_file` the previous version's fileId (reversible — do not hard-delete).
4. Give the user the new file's `viewUrl`. **The URL changes on every run** — always hand over the fresh link rather than assuming the old one still resolves to current data (the old one still resolves, but to stale/trashed content).
5. Keep row order stable (append/update in place within the CSV you compose, don't re-sort) so diffs between runs stay easy to eyeball.

Always `read_file_content` on the newly created sheet once to confirm the data landed correctly before reporting success.
