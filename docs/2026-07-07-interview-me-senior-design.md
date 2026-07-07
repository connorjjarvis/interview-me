# interview-me-senior — Design Spec

**Date:** 2026-07-07
**Status:** Approved (design), pending implementation
**Repo:** `interview-me` (standalone, GitHub-distributable)
**Skill:** `interview-me-senior`

## 1. Purpose

A standalone, shareable Claude Code skill. You finish a piece of work, invoke
the skill, and it runs a **staff-level engineering interview** on the decisions
you just made — grilling you on *why this and not the alternatives*, probing weak
answers, and scoring you against a configurable rubric.

It runs **after** work is done (post-implementation), which distinguishes it from
existing skills like `interview-me` (Sorbh) that interview you *before* coding to
build a spec. The niche — "grill me on decisions I already made, and score me like
a staff-level interviewer" — does not exist as a published skill today.

**Non-goal:** it is not a collaborative design partner. It must not invoke the
`brainstorming` skill or help you improve the code. It interrogates and scores.

## 2. Flow — five phases

1. **Recon (silent).** Reads the git diff (branch vs base / staged / last-N
   commits), pulls work-item context if the user points at one, and asks the user
   for a one-line "what I was trying to do." No interview questions yet.
2. **Calibrate.** From recon it builds the **decision map** (§4) and selects the
   5-6 sharpest decision points. Announces: *"I've reviewed your change. This runs
   at a Staff bar. Ready?"*
3. **Interview.** One question at a time, each targeting a real decision. A weak or
   hand-wavy answer triggers **one follow-up probe** (config `probe_depth`) before
   moving on. Each answer is scored privately.
4. **Bonus.** One **Principal-level stretch question** (systemic / organisational),
   scored separately.
5. **Report.** Reveals per-dimension scores, overall level verdict, strengths,
   gaps, and a generated reading list → terminal + Markdown + HTML.

## 3. Context sourcing (stack-agnostic)

Designed so anyone can use it, not just the author's Azure DevOps setup.

- **diff** (default on): git diff is the primary source. Skill determines the
  comparison range (current branch vs default base; falls back to staged, then
  last commit).
- **work_item** (on if provided): the user pastes a ticket URL/ID or its text. If
  work-item tooling (ADO, Jira, GitHub issues) happens to be available in the
  user's environment, the skill may use it to fetch parents/context; otherwise it
  uses whatever the user pasted. **No work-item system is hardcoded.**
- **description** (prompted): the user's one-line intent, captured in Recon.

## 4. Decision-map engine (grounds "why not ABC")

During Recon the skill classifies the diff into **decision points**. Each carries:

- **What you did** — the actual code/choice.
- **Plausible alternatives** — 2-3 a senior engineer would expect to have been
  weighed.
- **Trade-off axis** — what it really tests: coupling, failure modes, cost,
  reversibility, team-scaling, etc.

Decision points are ranked by "how much judgment does this reveal," and the top
ones become interview questions. Because the skill already knows the alternatives
before the user answers, it can legitimately challenge the answer rather than ask
from a generic bank.

## 5. Scoring model

**Default dimensions** (configurable): Technical judgment · Trade-off awareness ·
Risk & edge-case thinking · Communication/clarity · Business/context alignment.

**4-point rubric** with calibration anchors:

1. **Poor** — asserts without reasoning.
2. **Developing** — names the *what*, not the *why*.
3. **Strong** — weighs alternatives explicitly.
4. **Exceptional** — weighs alternatives *and* reversibility / second-order
   effects (the Staff+ tell).

- **Per-answer:** each answer scored privately against the relevant dimension(s);
  a follow-up probe can rescue a low score.
- **Bonus question:** scored separately as a Principal-level stretch. Missing it
  does not drag the core score down; nailing it flags "interviewing above the set
  bar."
- **Final verdict:** aggregate plus a plain-English level call, e.g. *"Solid
  Senior; Staff-ready on risk thinking, short of it on trade-off breadth."*

## 6. Config surface (`SKILL.md`, top of file)

A single commented block so anyone can retune without touching logic:

| Key | Default | Purpose |
| --- | --- | --- |
| `bar_level` | `staff` | `mid` / `senior` / `staff` / `principal` — shifts framing + rubric expectations |
| `dimensions` | the five above | add / remove / reweight |
| `core_questions` | `5` | target count of core questions |
| `probe_depth` | `1` | max follow-ups per question |
| `bonus_question` | `true` | include the stretch question |
| `bonus_offset` | `+1` | levels above `bar_level` for the bonus |
| `tone` | `tough_but_fair` | `warm` / `tough_but_fair` / `brutal` |
| `save_dir` | `~/.interview-me/<repo-name>/` | where scorecards are written — **out of repo by default** (§6a) |
| `context_sources` | `diff, work_item, description` | which inputs to use |
| `include_transcript` | `true` | write the full Q/A transcript to the scorecard (§6a) |
| `include_diff_snippets` | `false` | quote code from the diff in saved artifacts (§6a) |
| `auto_open` | `false` | auto-open the HTML report in a browser (§6a) |
| `redact_secrets` | `true` | scrub secrets/tokens/URLs from anything written to disk (§6a) |

## 6a. Artifact safety (privacy boundaries)

The saved outputs quote your code and ticket context and record a **personal
performance score**. Because the skill is designed to run inside *other people's*
repos, artifacts must never leak proprietary code or personal scores into a
shared repository by accident. Requirements:

- **Out-of-repo by default.** `save_dir` defaults to `~/.interview-me/<repo-name>/`
  (per-repo subfolder), so a personal review record is never repo-coupled and
  cannot be accidentally committed. Keyed by repo name so histories stay
  separated.
- **In-repo save is guarded.** If the user overrides `save_dir` to a path inside
  the repo, the skill must, before writing: ensure that directory is listed in
  `.gitignore` (append it if missing), and warn if the directory is already
  git-tracked. Never silently write review artifacts into a tracked path.
- **Transcript / code inclusion are opt-out / opt-in.** `include_transcript`
  (default on) controls whether Q/A text is saved; `include_diff_snippets`
  (default **off**) controls whether code is quoted in artifacts. With both
  trimmed, a saved scorecard is just scores + verdict + reading list — safe to
  share, nothing proprietary.
- **Secret redaction by default.** `redact_secrets` (default on) scrubs obvious
  secrets, tokens, and URLs from any text written to disk (terminal output is
  ephemeral and exempt).
- **No surprise browser launch.** `auto_open` defaults **off**; the skill prints
  the HTML path and opens it only when the user opts in.

These are hard requirements, not suggestions — the implementation plan must cover
each before the skill is considered complete.

## 7. Outputs

Three renderings of one scorecard dataset:

- **Terminal** — per-dimension scores (with anchor names), overall verdict, top
  2-3 strengths, top 2-3 gaps, bonus result.
- **Markdown** — `save_dir/YYYY-MM-DD-<branch-or-topic>.md`: scorecard table,
  verdict, reading list, and — subject to `include_transcript` /
  `include_diff_snippets` (§6a) — the full transcript (question → answer → score
  → what a 4 would have said). The trackable growth record.
- **HTML** — self-contained one-page report (inline CSS/SVG, no external deps;
  same approach as the `devops-pulse` skill): score gauges, per-dimension bars,
  "what went well / what to work on," and a curated **reading list** mapping each
  gap to concrete topics/resources. The skill prints the path; it opens the
  report in a browser only when `auto_open` is set (§6a). Generated from actual
  gaps, not boilerplate.

All saved artifacts respect the §6a privacy boundaries.

## 8. Repo structure

```
Interview-Me/                     # git repo "interview-me"
├── README.md                     # what it is, install, demo transcript, config table
├── LICENSE                       # MIT
├── interview-me-senior/
│   ├── SKILL.md                  # frontmatter config block + 5-phase logic
│   ├── assets/
│   │   └── report-template.html  # HTML scorecard shell
│   └── examples/
│       └── sample-scorecard.md   # sample output
└── docs/
    └── 2026-07-07-interview-me-senior-design.md
```

Install for others: copy `interview-me-senior/` into their `.claude/skills/`
(README documents this). Repo is a family — leaves room for future
`interview-me-mid` / `interview-me-principal` skills.

## 9. Open questions / future

- Additional bar-level skills (`mid`, `principal`) as siblings.
- Trend view across saved scorecards (progress over time).
