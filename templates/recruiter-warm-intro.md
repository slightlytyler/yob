# Recruiter warm-intro template

Used for the "warm the thread" stage of the pipeline: the goal at this stage is not to
evaluate or commit to a specific role, just to (1) confirm interest/availability, (2)
introduce background, and (3) state clearly what kind of role and setup Tyler's looking
for, so the recruiter can self-select whether to keep going. Works as an email reply
(direct agency contacts) or pasted into a LinkedIn message (InMail / connection requests
— those never expose a real email, see `recruiter-triage` skill).

Fill in the placeholders per-thread from the tracker sheet, then read it back once before
sending — this is a template, not a mail-merge; a couple of sentences (what specifically
caught your interest, or an explicit gap like SRE vs. product) usually make it land better
than the generic version alone.

## Fit criteria

Before picking a variant, check every role the thread mentions against both of these:

- **Location/setup**: remote, OR hybrid/onsite for a SoCal-based team. Onsite/hybrid for a non-SoCal team is *not* automatically a mismatch — Tyler's last role did quarterly onsites for a non-local team — but a role that's explicitly full-time onsite outside SoCal with no remote option is a mismatch.
- **Focus**: frontend, frontend infrastructure, or platform engineering. A role that's exclusively backend/SRE/DevOps/non-engineering with no frontend or platform-adjacent angle is a mismatch (see the Teema Group row — SRE only, already declined).

A thread can mention several roles (e.g. Josh Markowitz's four, Victoria's five) with a mix of fits and mismatches — call out the ones that fit by name and simply don't mention the ones that don't; no need to explicitly reject the mismatched ones when at least one role in the same thread is worth pursuing.

## Placeholders

- `{{FirstName}}` — recruiter's first name
- `{{MatchingRoles}}` — Variant A only: the specific role(s)/company name(s) that fit, referenced by name (e.g. "the Staff Frontend Engineer role at Fun.xyz" or, for several, "the Fun.xyz and Fomo roles")
- `{{MismatchReason}}` — Variant B only: a brief, neutral description of the specific gap (e.g. "fully onsite in Chicago with no remote option" or "backend/SRE-focused rather than frontend") — state it factually, not apologetically
- `{{ResumeLine}}` — channel-dependent; see the `recruiter-warm-intro` skill for which line to use (direct email vs. LinkedIn)

## Variant A — at least one role fits

Use when the thread mentions a role (or roles) matching both fit criteria above.

> Hi {{FirstName}},
>
> Thanks for reaching out — {{MatchingRoles}} sound like a good fit, I'd love to hear more.
>
> Quick background: frontend infrastructure engineer, 10+ years experience, most recently 5 years at Coinbase owning the GraphQL data-layer platform that powers ~95% of UI surfaces and is used daily by 700+ engineers. I've been on sabbatical since May.
>
> What I'm looking for more broadly:
>
> - Remote-first (I'm based in San Diego), open to hybrid for a SoCal-based team, or roughly quarterly onsites for a non-local one — that's how my last role worked (2-4 onsites/year)
> - Frontend infrastructure / platform engineering, though in practice I work full-stack. My last role included building and maintaining a production GraphQL service end to end.
>
> Happy to set up a quick call to discuss further. {{ResumeLine}}
>
> Best,
> Tyler

## Variant B — no role in the thread fits

Use when every role mentioned fails at least one fit criterion (and skip entirely for threads with no role mentioned at all — see Notes).

> Hi {{FirstName}},
>
> Thanks for reaching out. Based on what you've described, {{MismatchReason}} — doesn't sound like the right fit for me right now.
>
> For context in case something else on your desk fits better: I'm a frontend infrastructure engineer, 10+ years experience, most recently 5 years at Coinbase owning their GraphQL data-layer platform. I'm remote-first (San Diego based), open to hybrid for a SoCal team or roughly quarterly onsites otherwise, and focused on frontend infrastructure / platform engineering, though in practice full-stack.
>
> If anything along those lines comes up, I'd love to hear about it.
>
> Best,
> Tyler

## Notes

- No role/company mentioned at all yet (e.g. a bare LinkedIn connection request with no personal note, or one whose note doesn't mention a role — see the `recruiter-triage` skill on checking for hidden notes first)? Use neither variant's opening line — start straight from "Quick background" / "I'm a frontend infrastructure engineer" instead of claiming a fit or mismatch that hasn't been stated.
- If the thread already shows Tyler declined for a mismatched reason (see the Teema Group row — SRE, not frontend), don't reuse either variant verbatim there; it already has a targeted reply.
- Where two recruiters at the same agency show up in the tracker (e.g. Glocomms, Aspen Search), it's fine to mention that you've heard from the agency before if it feels natural — no need to pretend it's the first contact.
- Variant B doesn't include {{ResumeLine}} — don't push a resume on a role being declined; it only makes sense in the "something else on your desk" framing if the recruiter asks.
