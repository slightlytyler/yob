---
name: hn-hiring-apply
description: Work through rows Tyler has marked "Interested" in the "HN Who's Hiring - Leads" sheet — fill out the ATS application form directly via the Playwright browser bridge, or draft an outreach email, per row. Never submits or sends anything. Use whenever the user asks to apply to, follow up on, or work through HN Who's Hiring leads.
---

# HN Who's Hiring → application prep

Manual (on-demand) pipeline step, same posture as `hn-hiring-triage` and `recruiter-triage` —
validate manual runs before ever proposing to automate/schedule this.

This picks up after `hn-hiring-triage`: Tyler reviews the triaged "HN Who's Hiring - Leads"
sheet, marks specific rows `Interested: Yes`, and updates that row's **Contact** column to point
directly at the thing to act on for that specific role — either an ATS application URL for that
exact posting, or an email address pulled from the HN post. This skill only works rows Tyler has
explicitly marked interested; don't second-guess his calls by re-applying the original triage
fit criteria (e.g. a row `hn-hiring-triage` flagged as "low confidence" or a strong match doesn't
override whatever Tyler actually marked in `Interested` — that column is his decision, not a
derived value).

## 1. Browser bridge: Playwright MCP, set up 2026-09-30

This Claude Code session has no native browser tool — "Claude for Chrome" (what Tyler calls "the
Claude browser plugin") is a separate product running inside his actual Chrome browser with no
bridge from here. But a real bridge exists: the **Playwright MCP server**
(`claude mcp add playwright -- npx -y @playwright/mcp@latest`, added to this project's local
config), which exposes real `browser_navigate` / `browser_snapshot` / `browser_fill_form` /
`browser_click` / `browser_type` / `browser_file_upload` / `browser_take_screenshot` tools against
an actual (headed, not headless) Chromium instance. **New Claude Code sessions need to be
started/resumed for these tools to show up** — they attach at session start, not live.

This is an isolated browser context with no saved logins (confirmed acceptable to Tyler
2026-09-30 — none of these ATS forms need his existing sessions). If a future flow needs logins,
that would mean reconfiguring the server with a persistent `--user-data-dir`, not something to
assume works by default.

**File-path sandboxing:** `browser_file_upload` takes real absolute paths (no base64 through the
tool call, unlike Gmail attachments) but Playwright MCP restricts them to the project's working
directory and its own `.playwright-mcp/` output folder — a path elsewhere (e.g. under
`~/code/slightlytyler/resume/docs/`) is rejected as "outside allowed roots." Work around this by
copying the file into `.playwright-mcp/` first (that directory is gitignored — confirmed added to
`.gitignore` 2026-09-30, don't recreate that entry) and uploading from there.

**Hard rule, unchanged despite having real browser control: never click Submit / Send / Apply,
under any phrasing of the request.** Fill the form, upload the resume, take a screenshot showing
the filled state, and stop. Tyler reviews the live browser tab himself and submits it. This isn't
a capability gap being routed around anymore — it's a deliberate policy, identical in spirit to
`recruiter-warm-intro`'s draft-only rule for Gmail. Don't revisit this without Tyler explicitly
asking to change it.

For **email** rows there's still no bridge needed or wanted — those go through `create_draft`
exactly as before (step 4).

## 2. Which rows to work

Read the "HN Who's Hiring - Leads" sheet (`search_files` for the title if you don't have a fresh
fileId — the ID changes on every recreate-and-replace update, same as the recruiter tracker).
Filter to rows where `Interested` = `Yes`. For each one, look at `Contact`:

- If it's a URL pointing at an ATS platform (Ashby, Greenhouse, Workable, Lever, a company's own
  `careers.*` domain, etc.) → step 3.
- If it's an email address → step 4.
- **Sanity-check the contact against the row's own Company/Notes before acting on it.** Tyler
  types these in by hand while going through the sheet, and a domain mismatch (e.g. a `Cora AI`
  row with a contact at `@core.ai` instead of `@cora.ai`) is an easy typo to make and an easy one
  to carry forward unnoticed. If something looks off, flag it to Tyler and confirm before
  drafting or filling against it — wasted work either way, but worth catching early.

Don't touch rows where `Interested` is `No` or blank.

## 3. Filling out the ATS form

1. `browser_navigate` to the row's `Contact` URL. Some postings (e.g. an Ashby company careers
   page with an `ashby_jid` query param) land on a generic listing first — `browser_snapshot` to
   confirm you're on the right specific posting, and click through (e.g. an "Apply for this Job"
   button, or an "Application" tab alongside "Overview") if not.
2. `browser_snapshot` to see the actual form fields and their current `ref`s — refs shift after
   every action that changes the DOM (a click, a combobox selection, a new snapshot in general),
   so re-snapshot before referencing an element you haven't just gotten a ref for in the same
   response.
3. Fill every field this skill has a confident, known answer for for (see the table below) using
   `browser_fill_form` for a batch of plain text fields, or `browser_type`/`browser_click`
   individually for comboboxes, radios, and buttons that need their own handling.
4. **Do dropdowns/selects and other non-file fields before the file upload, not after.**
   Confirmed 2026-09-30 on the Confido Legal form: filling text fields, then uploading the resume,
   then calling `browser_select_option` on two `<select>` dropdowns caused a runaway file-chooser
   loop — each call left an open, unhandled file-chooser modal state, stacking up over a dozen
   deep, and `browser_take_screenshot` and further actions started failing with "does not handle
   the modal state." Closing the page (`browser_close`) didn't clear it either; only navigating to
   a fresh URL (`browser_navigate` to `about:blank` worked) reset the browser state enough to start
   over. Redoing the same form with dropdowns filled first and the resume upload as the last action
   completed with zero issues. Cause unconfirmed (possibly a Playwright MCP or Chromium bug
   specific to native `<select>` elements combined with a recent file input), but the ordering
   workaround is real and cheap — always upload files last.
5. Upload the resume via `browser_file_upload` per the sandboxing note in step 1 — a real file
   upload beats a link when the form supports one.
6. For anything the table below doesn't cover, leave it blank/unanswered — don't guess.
7. `browser_take_screenshot` (full page) so Tyler can see the exact filled state without needing
   to already have the tab focused, and paste it into the conversation.
8. **Do not click Submit / Apply.** Tell Tyler plainly what's filled and what's left for him,
   labeled clearly with company/role/HN post URL so he can match it to the right tab.

### Known facts to fill in

| Field | Value |
|---|---|
| Full name | Tyler Martinez |
| Email | slightlytyler@gmail.com |
| Location | San Diego, CA |
| LinkedIn | https://www.linkedin.com/in/tyler-martinez-a148a090 |
| GitHub | https://github.com/slightlytyler |
| Resume | Upload the real PDF (`~/code/slightlytyler/resume/docs/Tyler_Martinez_CV.pdf`, copied into `.playwright-mcp/` first) via `browser_file_upload` when the form has a native upload field. For an email draft instead (step 4), use the link https://slightlytyler.github.io only — never attach a PDF to an email, per `recruiter-warm-intro`'s hard-won rule. |
| Location preference | Remote-first; open to hybrid for a SoCal-based team, or roughly quarterly onsites (2-4/year) for a non-local one |
| Work authorization / visa sponsorship | No, does not require visa sponsorship now or in the future |
| Background summary | Frontend infrastructure engineer, 10+ years experience, most recently 5 years at Coinbase owning the GraphQL data-layer platform (powers ~95% of UI surfaces, used daily by 700+ engineers). On sabbatical since May 2026. |
| Why interested in this role | 2-3 sentences pulled from the row's `Notes`/`Role(s)` — name the specific thing about *this* posting that fits (stack, remote policy, comp, mission), not a generic paragraph |
| "How did you hear about us" | Hacker News "Who's Hiring" thread |

### Fields to leave for Tyler — don't guess

- Phone number
- Desired salary, if a form demands a single number rather than accepting "see range" — the row's
  own `Cash Comp` figure is a reference point, not an answer to fill in on his behalf
- Availability / start date
- Self-rated experience scales, "years of experience with X" sliders, or any other subjective
  self-assessment a form asks the candidate to make about themselves — Tyler's own judgment call
  about himself, not a fact this skill can look up or infer, even when his background makes an
  answer obvious. Confirmed 2026-09-30 on the Ashby application: left the 1-4
  programming-experience radio unanswered for him to pick.
- Optional diversity/EEO surveys — leave entirely untouched, don't answer on Tyler's behalf even
  with a "prefer not to answer" option selected. **Confirmed 2026-09-30 on the Ashby
  application: don't even try to interact with these fields, including to clear them.** After
  uploading the resume, Ashby's own "Ashby may use Artificial Intelligence with this application"
  feature silently populated age/gender/ethnicity/community answers and reworded the free-text
  essay answer, none of which came from anything Tyler or this skill provided — apparently
  inferred from the resume/name. Attempting to uncheck the fabricated checkboxes got silently
  reverted back to checked on a later snapshot, i.e. **the page fights you and wins.** Don't spend
  turns re-clicking these — leave them exactly as encountered (fabricated or not) and tell Tyler
  the diversity section needs his own review as the very last thing before he submits, since
  earlier fixes don't reliably stick. This is a platform behavior specific to Ashby (or possibly
  other ATSes with similar "AI-assisted" disclosures) — check whether it recurs on non-Ashby forms
  rather than assuming every ATS does this.

### Free-text essay questions

A "describe your background / an interesting project" style question **can** be drafted from the
background summary above plus the row's specific role fit — done successfully on the Ashby
application, describing the Coinbase GraphQL platform work in ~2 paragraphs. Call out clearly in
the report to Tyler that it's a draft in his voice he should read and edit before submitting, not
a final answer — an unedited AI-sounding essay response reads badly in a personal application in
a way a filled-in name field doesn't.

## 4. Drafting the email

For a row whose `Contact` is an email address, follow the same shape as `recruiter-warm-intro`'s
drafting but as **new outreach, not a reply** — do not pass `replyToMessageId` (there's no
existing thread to reply into; this is a cold application, not a warm recruiter relationship).

- `create_draft` with `to: [<contact email>]`, a subject like `Interest in <Role> at <Company>
  (via HN Who's Hiring)`, and a body that: opens by referencing the specific HN posting, gives
  the background summary above, states why this specific role fits (same sourcing rule as the
  form-filling step — pull from that row's Notes/Role(s), not a generic pitch), and closes with
  the resume link (never an attachment) and an offer to talk further.
- Paste the same body back into the conversation after creating the draft, so Tyler can review it
  without opening Gmail.
- Never call `send_message`, `reply`, or `forward` on it, under any phrasing of the request — same
  hard rule as `recruiter-warm-intro`.

## 5. Update the sheet's Status column

Same three-value semantics as the recruiter tracker: `Not Responded` (default) / `Drafted` (a
form has been filled out live in the browser, or a Gmail draft created) / `Responded` (Tyler
confirms he actually submitted the application or sent the email). Only move a row to `Responded`
on Tyler's explicit confirmation — don't infer it from having filled a form or created a draft,
and definitely don't infer it just because the browser session touched the Submit button's
presence on the page (it should never have been clicked).

## 6. Sheet update workflow

Same Drive limitation and recreate-and-replace workflow as `hn-hiring-triage` (see that skill for
the full rationale — Drive can't edit cells in place):

1. Compose the full CSV (all existing rows, `Interested`/`Contact`/`Status`/`Notes` columns
   included, updating only the `Status` for rows just worked — never a diff).
2. Wrap every field containing a comma in double quotes (double up any literal `"`), and re-scan
   for a bare unquoted `,` before publishing — this has broken rows in the recruiter tracker in
   the past.
3. Validate column counts with a `python3 -c "import csv..."` check before calling `create_file`.
4. `create_file` with the same title ("HN Who's Hiring - Leads"), `contentMimeType: text/csv`.
5. `read_file_content` on the new file to confirm the table range and row count look right.
6. `trash_file` the previous version's fileId (reversible, never hard-delete).
7. Hand Tyler the fresh `viewUrl` — it changes on every run.
