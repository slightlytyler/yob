---
name: recruiter-warm-intro
description: Draft a warm-intro reply to a recruiter thread from the Job Search - Recruiter Tracker, filling in the template at templates/recruiter-warm-intro.md. Use whenever the user asks to draft, write, or send an intro/warm-up reply to a recruiter or a specific tracker row.
---

# Recruiter warm-intro drafting

Fills in `templates/recruiter-warm-intro.md` for a specific row from the tracker sheet (see
the `recruiter-triage` skill for how that sheet is built and maintained).

**Hard rule: this skill only ever produces a draft. It never sends or transmits anything to
a recruiter, under any phrasing of the request** ("draft and send it," "just send it," "go
ahead," etc. still mean draft-only — if the user wants to override this, that's a separate,
explicit instruction to change the skill, not something a single request should do in the
moment). Tyler reviews every draft and sends it himself. Concretely:

- **Never call** `mcp__claude_ai_Gmail__send_message`, `reply`, or `forward` for a recruiter
  thread from this skill.
- For `Direct Email` sources, use `create_draft` only — it stays in Gmail Drafts for Tyler to
  review and send manually.
- For `LinkedIn InMail` / `LinkedIn Connection Request` sources, there is no draft mechanism
  to use — replying via `inmail-hit-reply@linkedin.com` through Gmail would actually deliver
  the message into LinkedIn's system, so don't call any Gmail send/reply tool for these at
  all. Instead, output the filled-in text directly in the conversation for Tyler to copy and
  paste into LinkedIn himself.

## 1. Pull the row's details and pick a variant

From the tracker: `{{FirstName}}` (first name only, parsed out of Recruiter Name), the
Role(s) Mentioned and Notes (for location/onsite details), and — critically — the **Source**
column, which decides the resume line (step 2).

Check every role the thread mentions against the template's fit criteria (remote or
SoCal-hybrid; frontend/frontend-infra/platform focus) and pick a variant:

- **At least one role fits** → Variant A. Fill `{{MatchingRoles}}` by naming the fitting
  role(s)/company(ies) specifically — don't just say "the role," and don't mention roles from
  the same thread that don't fit (see Josh Markowitz or Victoria/Trope Talent rows for
  threads with several roles to choose from).
- **No role in the thread fits** → Variant B. Fill `{{MismatchReason}}` with the specific,
  factual gap (e.g. "onsite in NYC with no remote option," "SRE-focused rather than
  frontend") — pull this from the Role(s) Mentioned / Notes columns, don't guess.
- **No role mentioned at all** (bare connection request, or one whose note has no role after
  checking per the `recruiter-triage` skill) → neither variant's opening line; start from
  "Quick background" per the template's Notes section.

If unsure which way a role leans (e.g. comp/location ambiguous), ask Tyler rather than
guessing — a wrongly-declined lead is wasted opportunity, and a wrongly-pursued one wastes
both people's time.

## 2. Resume line: attach vs. link

Tyler's resume lives at `~/code/slightlytyler/resume/docs/Tyler_Martinez_CV.pdf` and is also
hosted at `https://slightlytyler.github.io`. Which one to use depends entirely on the
tracker row's **Source**:

| Source | `{{ResumeLine}}` |
|---|---|
| `Direct Email` (real address, replying via Gmail) | `Resume attached.` — actually attach `~/code/slightlytyler/resume/docs/Tyler_Martinez_CV.pdf` to the reply/draft. |
| `LinkedIn InMail` or `LinkedIn Connection Request` | `Here's my resume: https://slightlytyler.github.io` — never claim something is attached; LinkedIn messages (and InMail replies routed through `inmail-hit-reply@linkedin.com`) don't carry attachments the recruiter will actually receive as a file. |

Skip this step entirely for Variant B — it doesn't use `{{ResumeLine}}` (see the template's Notes).

If the resume PDF at that path looks stale (check its modified time against the tracker
row's date, or just ask), regenerate it from the `resume` repo before attaching rather than
sending an outdated version.

## 3. Fill the template and confirm

Substitute the chosen variant's placeholders (`{{FirstName}}` and either `{{MatchingRoles}}`
+ `{{ResumeLine}}`, or `{{MismatchReason}}`) into the template.

Always show the filled draft in the conversation for review — even after calling
`create_draft` for a `Direct Email` row, paste the same text back so Tyler doesn't have to
open Gmail just to see what was drafted.

## 4. After Tyler sends it himself

This skill has no visibility into whether Tyler actually sent a draft — it only creates it
(or hands over text for LinkedIn) and stops. Only update the tracker row's Status to
`Responded` when Tyler explicitly confirms a reply went out; don't infer it from having
created a draft.
