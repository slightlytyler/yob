---
name: hn-hiring-triage
description: Parse Hacker News "Who's Hiring" threads, filter postings against Tyler's fit criteria, and create or update the "HN Who's Hiring - Leads" Google Sheet. Use whenever the user asks to check the latest Who's Hiring thread, run HN lead generation, or re-triage a specific month's thread.
---

# HN Who's Hiring → lead tracker sheet

Manual (on-demand) pipeline step, same posture as `recruiter-triage` — validate manual runs before ever
proposing to automate/schedule this (see project memory `feedback-verify-before-automating`).

A new "Who's Hiring" thread is posted on HN on the 1st of each month by user `whoishiring`. Tyler will
give a specific thread URL/ID (e.g. `https://news.ycombinator.com/item?id=49522897` for 2026-09) — don't
guess a current thread ID yourself, ask if it's not given.

## 1. Fetch the thread

**Use the Algolia HN API, not the official Firebase API.** Algolia returns the entire thread — root
posting plus every top-level job posting plus their reply threads — as one nested JSON blob in a single
HTTP call:

```
curl -s 'https://hn.algolia.com/api/v1/items/<thread_id>'
```

Confirmed 2026-09-24 on the September 2026 thread (257 postings): this came back as ~495KB, comfortably
fetchable in one shot. The Firebase API (`https://hacker-news.firebaseio.com/v0/item/<id>.json`) only
gives you an `id` list in `kids` — fetching a thread this size that way means one call per posting
(250+ calls) for the same data. Don't use it for this.

The root object's `children` array is the postings — one HN comment per job listing. Each posting has:
- `author` — HN username of the poster (usually the recruiter/founder/hiring manager)
- `created_at` — ISO timestamp
- `id` — use this to build the direct link: `https://news.ycombinator.com/item?id=<id>`
- `text` — HTML string: `<p>` tags separate paragraphs, `<a href="...">` for links, standard HTML
  entities (`&#x2F;` for `/`, `&amp;`, `&#x27;`, etc.)
- `children` — **replies to that specific posting** (people asking questions, sometimes company
  employees responding). These are discussion, not separate job listings — ignore them for extraction.
  Confirmed: ~26% of postings have reply threads.

## 2. Parse each posting

Use Python (via Bash) for this, not manual reading of raw JSON — the volume (250+ postings/month) makes
line-by-line reading impractical for the first pass.

- **Header line**: everything before the first `<p>` in `text`. By convention most posters pipe-delimit
  it (`Company | Role | Location | Remote/Onsite/Hybrid | Comp | URL`), but **field order and presence
  vary a lot and this is not enforced by HN** — confirmed 94% of postings (241/257) have a `|` in the
  header, the rest don't follow the convention at all (a few aren't even job postings — e.g. someone
  linking a third-party thread-browsing tool, someone posting an off-topic rant). Don't parse by fixed
  field position; extract each signal independently:
  - **Links**: regex `href="([^"]+)"` across the *entire* `text`, not just the header — company site and
    a separate "Apply:" link often both appear, and the apply link is sometimes only in the body.
  - **Comp**: regex for `$`/`€`/`£` amounts (`\$\s?[\d,]+k?`), `/hr`, `/mo`. Only ~39% of postings
    (100/257) state a number at all — see step 3 on what to do when it's missing.
  - **Location/arrangement**: keyword search for `remote`/`onsite`/`hybrid`, case-insensitive, across
    the full body. 93% of postings (239/257) have one of these words somewhere, but a few describe an
    arrangement without the word (e.g. "3x/week in office" instead of "hybrid") — don't rely on keyword
    match alone for a final decision, read the posting when the keyword search comes back empty but the
    posting otherwise looks promising.
  - **Contact**: regex for an email address as a fallback signal. Only 25% of postings (63/257) include
    one directly — most rely on an apply-page URL instead (see step 4 on why this matters).
- Decode HTML entities and strip remaining tags for anything you're going to read or write to the sheet.

## 3. Fit criteria (confirmed with Tyler 2026-09-24)

All four must hold. This is a stricter, narrower filter than `recruiter-triage`'s — expect a small
single-digit-percent pass rate (13/257 ≈ 5% on the first real run), not a majority.

1. **Frontend or Platform role.** Match on the role's own title/framing, not just stack keywords
   mentioned in passing — a posting that mentions "React" once while advertising a "Backend Engineer"
   role doesn't count. Counts: Frontend Engineer, Full-Stack Engineer (with real frontend component),
   Product Engineer (React/TypeScript-described), Platform Engineer, DevOps/Infrastructure Engineer,
   SRE. Does **not** count on title alone: Backend Engineer, Data Engineer, AI/ML Engineer, Data
   Scientist, Security Engineer, Compiler/Formal-Verification Engineer, Quant, PM/leadership roles —
   even at a company whose *product* is a platform (e.g. a Kubernetes-orchestration startup's "Backend
   Engineer, Kubernetes controllers" role doesn't count just because the company sells a platform). When
   a single posting lists multiple role variants, evaluate each one independently and only include the
   variants that pass (see the Akkio and Category Labs rows in the tracker for examples — 1 of 3 and 1
   of 4 variants respectively made the cut).
2. **US-based company.** Default to yes — most postings don't state a country at all, and per Tyler's
   explicit instruction (2026-09-24), assume US unless there's a positive signal otherwise. Only exclude
   on an explicit non-US signal: a named non-US city/country as the company's location or office (not
   just a remote-eligible region), an explicit "based in France" (etc.) type statement, or an onsite
   requirement in a non-US city with no US presence mentioned. A broad remote-eligibility statement like
   "remote almost anywhere in the world" is about *where employees can work*, not where the company is
   based — don't treat it as a non-US signal by itself.
3. **Remote-first, OR hybrid specifically in SoCal.** Same location logic as the `recruiter-warm-intro`
   template's fit criteria. Pure onsite (even in SoCal, even at great comp) does **not** count unless the
   posting explicitly says hybrid — confirmed several SF/LA/San Diego onsite-only postings with strong
   comp and role fit were excluded on this basis alone (Steg.AI, Apex Space, Brain Corp, Relativity
   Space, Freeform — all explicitly "Onsite," not "Hybrid").
4. **Cash compensation ~$200k.** Must be an explicit number in the post — don't infer or guess from
   company stage/funding. **If no comp figure is stated at all, exclude the posting**, even if role and
   location are a great fit — confirmed several strong role/location matches (GovStar's explicit
   "Frontend Engineer" role, Valkyrie Aero's explicit "Frontend (HMI)" role, Trax's Angular/Vue/design-
   systems Architect role) were excluded purely because no salary number was given anywhere in the post.
   This is a real, recurring loss — worth mentioning to Tyler each run as a "near misses, comp not
   stated" callout rather than silently dropping them, in case he wants to loosen this rule later. When a
   range is given, "~$200k" means approximate — a range whose *top* reaches roughly $195k+ passes; a wide
   range whose top just barely touches $200k (e.g. $80k–$200k) technically passes but should be flagged
   in Notes as low-confidence, since the real offer for a senior candidate could land well below the
   target.

## 4. A structural note before "triaging" turns into "warming" (future skill)

Unlike the Gmail recruiter pipeline, there's no inbox to draft a reply into. Contact is one of: an
external ATS apply link (Ashby/Greenhouse/Workable/etc. — the large majority), an email address in the
post, or — for postings with neither — replying directly on the public HN thread itself. That last
option would have to go out from Tyler's own HN account; nothing here can draft or post on his behalf the
way `recruiter-warm-intro` drafts into Gmail. When a future "warm this HN lead" skill gets built, it
should hand Tyler either a filled application-form summary or a reply-comment draft as plain text, the
same pattern already used for LinkedIn connection requests — never assume a Gmail-draft-equivalent exists
here.

## 5. Sheet schema and update workflow

Sheet title: **"HN Who's Hiring - Leads"** — a separate file from "Job Search - Recruiter Tracker," since
this is a distinct source with distinct fit criteria and no email-draft warming step.

| Column | Notes |
|---|---|
| Date Posted | The posting's `created_at` date (not the thread's start date) |
| Company | |
| Role(s) | Note which specific variant passed when a posting lists several roles |
| Location / Arrangement | Quote the posting's own wording rather than normalizing it away |
| Cash Comp | As stated in the post; note if it's a floor/ceiling/range and flag low-confidence wide ranges |
| Thread Month | `YYYY-09` style — lets multiple months accumulate in one sheet over time |
| HN Post URL | `https://news.ycombinator.com/item?id=<posting_id>` — the direct comment permalink, not the thread root |
| Contact | Apply URL, or email, or `Reply on HN thread` if neither exists |
| Status | Same three values as `recruiter-triage`: `Not Responded` / `Drafted` (an application or reply text has been prepared but not sent) / `Responded` (Tyler confirms he actually applied/replied) |
| Notes | Why it passed, any caveats (comp-range confidence, which role variant, US-based assumption vs. confirmed, etc.) |

**Same Drive limitation as `recruiter-triage`, same workaround**: the Drive connector can't edit sheet
cells in place. To update: compose the full CSV (all existing rows across all months, plus new ones —
never a diff), validate column counts with `python3 -c "import csv..."` before publishing, `create_file`
with the same title, verify with `read_file_content`, then `trash_file` the previous version. Keep the
same one-row-per-passing-posting design across months — don't split into per-month sheets, so the
accumulated leads stay in one place.

## 6. Re-running for a new month

Each month's thread is a new HN item with a new ID — Tyler will provide it (or the URL). Fetch and triage
it the same way, then append the new month's passing rows to the existing sheet's full row set rather
than starting over. Don't re-fetch or re-triage a month already in the sheet unless Tyler explicitly asks
you to (e.g. because a posting's comp/role got edited after the fact — HN comments are editable for a
window after posting).
