# Job search pipeline

This workspace supports Tyler's job search pipeline: identify opportunities from inbound recruiter
communications (Gmail + LinkedIn), track them, and eventually drive applying and communications.

Current stage: Gmail-based recruiter triage feeding a tracking spreadsheet. Manual/on-demand for now —
scheduling is the eventual goal but only after the manual flow is validated.

For the actual search/classify/tracker-update procedure, see the `recruiter-triage` skill
(`.claude/skills/recruiter-triage/SKILL.md`) — invoke it rather than re-deriving the approach.

## Known constraints

- The Gmail connector supports one authenticated account at a time; switching accounts requires
  disconnecting the current one. See the skill for which accounts are in play and what each contains.
- The Google Drive connector can create/replace files but cannot edit an existing spreadsheet's cells —
  the skill's update workflow (recreate + trash old version) works around this.
