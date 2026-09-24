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
  review and send manually. Always pass `replyToMessageId` (the thread's most recent message
  ID) so the draft threads under the existing conversation.
- **Never use `update_draft` to fix a reply draft's text.** Confirmed 2026-09-24: `update_draft`
  has no `replyToMessageId` field, and calling it on a reply draft silently detaches it into a
  new standalone thread — Gmail's threading breaks even though the subject still says "Re:".
  If a draft needs correcting after creation, call `create_draft` again from scratch with the
  same `replyToMessageId` and corrected body — this leaves the old draft behind as a duplicate
  in the same thread (Gmail's Drafts list collapses same-thread drafts into one row, so it
  won't visibly multiply, but the stale one is still there underneath).
- **Cleaning up a stale/duplicate draft:** try `trash_message` on its `messageId` first. It
  needs `gmail.modify` scope, which this connector may or may not have at a given moment (it
  didn't on 2026-09-24, then did after Tyler widened it) — if it errors with "Insufficient
  scope," don't retry it blind; tell Tyler exactly which draft to delete manually (recipient +
  approximate timestamp, since Gmail's Drafts list won't show a clean single identifier per
  duplicate) and move on. If a stale draft sits in the same thread as a duplicate that gets
  deleted through the Gmail UI (not via this tool), re-run `list_drafts` afterward rather than
  assuming which one survived — deleting via the UI can behave unexpectedly (e.g. removing both
  drafts in a thread at once, or resaving the remaining one with a new revision/messageId) and
  this was observed live on 2026-09-24.
- **`LinkedIn InMail`** (sender `inmail-hit-reply@linkedin.com`): **use `create_draft` here
  too.** Corrected 2026-09-24 — earlier versions of this skill claimed no draft mechanism
  existed for LinkedIn threads at all; that was never actually tested and was wrong. A draft
  is inert regardless of recipient: `create_draft` with `to: ["inmail-hit-reply@linkedin.com"]`
  and `replyToMessageId` set to the InMail message's ID produces a normal Gmail draft
  (`labelIds: ["DRAFT"]`, correctly threaded, "Re:" subject, original message quoted below —
  confirmed via `get_draft` on a live test) that sits in Drafts exactly like a Direct Email
  reply until Tyler sends it himself. Only an actual **send** would relay through LinkedIn and
  deliver to the recruiter — never call `send_message`/`reply`/`forward` on it, per the hard
  rule above, but `create_draft` is exactly as safe here as it is for Direct Email.
- **`LinkedIn Connection Request`** (sender `invitations@linkedin.com`): still **no Gmail
  draft** — this is a different, unverified mechanism. `invitations@linkedin.com` handles
  connection invites, not messaging; there's no evidence replying to it delivers a message to
  the sender the way InMail's reply-relay does, and accepting + messaging a new connection
  only happens inside LinkedIn itself. Keep outputting the filled-in text directly in the
  conversation for Tyler to copy and paste into LinkedIn after accepting.
- **Gmail can silently merge unrelated connection-request emails into one thread.** Confirmed
  2026-09-24: Steve Bonomo (SCALRR) and Brandon Roosevelt (Hex) — two different people, one
  day apart — landed in the identical thread ID because both came from
  `invitations@linkedin.com` with the generic subject "I want to connect." Don't assume a
  shared thread ID means duplicate/related rows for connection requests; check the sender
  name inside each message, not just the thread grouping.

## 1. Pull the row's details and pick a variant

From the tracker: `{{FirstName}}` (first name only, parsed out of Recruiter Name), and the
Role(s) Mentioned and Notes (for location/onsite details). The **Source** column still
determines how the draft gets delivered (step 3), just not the resume line anymore.

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

## 2. Resume line: link only, never an attachment

`{{ResumeLine}}` is always `You can find my resume at https://slightlytyler.github.io` — the
same wording regardless of Source/channel, **with no trailing punctuation directly after the
bare URL**. Confirmed 2026-09-24: Gmail's compose-time auto-linker swallows a period sitting
right after a bare URL into the link target itself (verified via `get_draft`'s rendered
`htmlBody`, which showed the href ending in `...github.io.` — the period baked into the
click target). If a period is grammatically needed, put something after the URL instead of
right after it, or just end the sentence at the URL with no punctuation, as above. Never
attach the PDF and never claim one is attached.

**Why not attach:** `create_draft`'s `attachments[].content` field takes a literal base64
string in the tool call — there's no file-path passthrough. Getting Tyler's resume PDF
(`~/code/slightlytyler/resume/docs/Tyler_Martinez_CV.pdf`) into that field means reading it
through `Read` (which returns it as numbered lines, since it's effectively one giant line)
and retyping ~40K characters into the call by hand, with no way to verify afterward that the
reconstruction was byte-exact. A single dropped or altered character silently corrupts the
PDF a recruiter would receive. That risk isn't worth it for what a link accomplishes just as
well — confirmed via direct instruction from Tyler (2026-09-24) after this was flagged.

If the hosted resume at `https://slightlytyler.github.io` looks stale relative to a tracker
row's date, regenerate it from the `resume` repo (see that repo's README) before drafting,
rather than sending an outdated version — just don't attach it.

**Confirmed limitation, don't re-litigate (2026-09-24):** every draft this connector creates
via `create_draft`/`update_draft` wraps the resume link in Gmail's Safe Browsing redirect
(`https://www.google.com/url?q=https://slightlytyler.github.io&source=gmail&ust=...&sa=E`),
regardless of whether the URL comes from plain-text auto-linking or an explicit `<a href>` in
`htmlBody` — both were tested directly on a live draft and produced the identical wrapped
link. This is applied server-side at draft-creation time, not something the request content
controls, so don't spend another round trying a different way to author the link. **The link
still works** — it correctly redirects to the resume — this is cosmetic, not broken. The one
observed way it resolves to a clean plain href is when Tyler opens/touches the draft in
Gmail's own compose UI himself (seen on the Ben Herman draft); that can't be triggered from
this tool. If Tyler flags this again, point him at this note rather than attempting another
fix — the trailing-punctuation bug above is a separate, real, and already-fixed issue; this
redirect-wrapping is not that, and isn't fixable from here.

## 3. Fill the template and confirm

Substitute the chosen variant's placeholders — `{{FirstName}}`, `{{ResumeLine}}`, and either
`{{MatchingRoles}}` or `{{MismatchReason}}` — into the template.

Always show the filled draft in the conversation for review — even after calling
`create_draft` for a `Direct Email` row, paste the same text back so Tyler doesn't have to
open Gmail just to see what was drafted.

## 4. Update the tracker's Status column

As soon as a draft is created (Gmail) or text is handed over (LinkedIn), set that row's
Status to `Drafted` in the tracker (see the `recruiter-triage` skill's update workflow) — this
is what distinguishes "nothing done yet" from "ready for Tyler to review and send."

This skill has no visibility into whether Tyler actually sent a draft — it only creates it
(or hands over text for LinkedIn) and stops. Only move a row's Status to `Responded` when
Tyler explicitly confirms a reply went out; don't infer it from having created a draft, and
don't leave it at `Not Responded` once a draft exists — that undercounts real progress.
