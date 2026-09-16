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

## 1. Pull the row's details

From the tracker: `{{FirstName}}` (first name only, parsed out of Recruiter Name), `{{Company}}`,
`{{Role}}`, and — critically — the **Source** column, which decides the resume line (step 2).

## 2. Resume line: attach vs. link

Tyler's resume lives at `~/code/slightlytyler/resume/docs/Tyler_Martinez_CV.pdf` and is also
hosted at `https://slightlytyler.github.io`. Which one to use depends entirely on the
tracker row's **Source**:

| Source | `{{ResumeLine}}` |
|---|---|
| `Direct Email` (real address, replying via Gmail) | `Resume attached.` — actually attach `~/code/slightlytyler/resume/docs/Tyler_Martinez_CV.pdf` to the reply/draft. |
| `LinkedIn InMail` or `LinkedIn Connection Request` | `Here's my resume: https://slightlytyler.github.io` — never claim something is attached; LinkedIn messages (and InMail replies routed through `inmail-hit-reply@linkedin.com`) don't carry attachments the recruiter will actually receive as a file. |

If the resume PDF at that path looks stale (check its modified time against the tracker
row's date, or just ask), regenerate it from the `resume` repo before attaching rather than
sending an outdated version.

## 3. Fill the template and confirm

Substitute `{{FirstName}}`, `{{Company}}`, `{{Role}}`, `{{ResumeLine}}` into the template.
If the row has no role/company yet (bare connection request), follow the template's own
note and drop the opening line rather than inventing one.

Always show the filled draft in the conversation for review — even after calling
`create_draft` for a `Direct Email` row, paste the same text back so Tyler doesn't have to
open Gmail just to see what was drafted.

## 4. After Tyler sends it himself

This skill has no visibility into whether Tyler actually sent a draft — it only creates it
(or hands over text for LinkedIn) and stops. Only update the tracker row's Status to
`Responded` when Tyler explicitly confirms a reply went out; don't infer it from having
created a draft.
