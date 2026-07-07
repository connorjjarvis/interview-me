# review-me-senior — Design Spec

**Date:** 2026-07-07
**Status:** Approved (design), pending implementation
**Repo:** `interview-me` (standalone, GitHub-distributable) — second skill in the family
**Skill:** `review-me-senior`

## 1. Purpose

A review copilot. You point it at a branch (or optionally a PR); it does a
staff-level read of the diff and returns a **ranked map of review points**, then
lets you **drill into any one** for the full reasoning and a **draft comment in
your voice**. It is calm and teaching-oriented — no grilling, no personal score.

It helps you two ways: review your own work before pushing, and write better
reviews for others — learning from the reasoning either way. Over repeated runs it
**learns your project's conventions** (stored locally) so it stops re-flagging
things you have blessed.

**Relationship to `interview-me-senior`:** sibling skill, same repo. The
interviewer *interrogates and scores the person*; this one *assesses the work and
helps you articulate a review*. Shared machinery (diff/PR recon, staff-level
knowledge, artifact-safety model) but distinct intent, tone, and output.

**Relationship to `/code-review` and linters:** those *hunt and report*; this one
*teaches and helps you articulate*. Every finding carries the reasoning and a
draftable comment, and it is interactive. Overlap on "find the bugs" is expected
and acceptable.

**Non-goals (v1):** no posting comments back to a PR; no scoring; no HTML report;
no shared/committed rules file (local-only memory in v1).

## 2. Flow — five phases

1. **Recon (silent).** Resolve the diff range exactly as the interview skill does:
   default branch detection → `git merge-base <default> HEAD`...HEAD; fall back to
   staged (`git diff --staged`), then last commit. If `pr` is in
   `context_sources`, accept a PR the user pastes or points at — fetch its diff +
   context only if PR tooling (ADO/Jira/GitHub-issue MCP or CLI) already exists in
   the environment, otherwise use pasted text. Never hardcode a platform. Load this
   repo's project-rules memory (§4). Ask one line: *"anything you want me to focus
   on, or just a full pass?"*
2. **Analyze → build the review map (silent).** Read the diff at `bar_level` and
   produce **review points**. Each carries: location (`file:line`), category (§3),
   importance tier (§3), confidence, the observation, the *why* (teaching), and a
   draftable comment. Apply loaded rules — `accepted` patterns are filtered or
   greyed as "known-OK"; `expected`/`forbidden` become active checks. Rank.
3. **Overview.** Print the ranked map: each point as `file:line` + category tag +
   tier + one-line reason, sorted by tier then grouped by category, nits last. The
   user sees the whole landscape at a glance.
4. **Drill-in (interactive, user-driven).** The user selects what to explore — by
   number, a range, "all the correctness ones," "skip the nits," etc. For each
   selected point, show the **code snippet + full reasoning + a draft comment** (if
   `draft_comments`). The user can push back ("why does that matter?"), request a
   redraft, or say *"that's fine, we do it this way here"* → the skill offers to
   remember it (§4). No forced march; the user drives depth and order.
5. **Learn + wrap.** Any blessed pattern or stated rule is written to the local
   project-rules memory (only on confirmation, §4). At the end, per `save_notes`,
   optionally save the session's review notes (§6) out of repo.

## 3. Finding taxonomy + ranking

**Categories** (each toggle-able via `categories`, default all on):

- **Correctness / bug risk** — logic errors, off-by-one, wrong conditions.
- **Risk & edge-cases** — failure modes, error handling, nulls, concurrency,
  boundaries.
- **Design & decision** — abstraction boundaries, coupling, "why this shape" calls.
- **Maintainability** — naming, duplication, complexity, readability.
- **Test coverage** — missing or weak tests for the change.
- **Security** — injection, authz, secret handling, unsafe input.
- **Nit / style** — formatting, minor polish (always sinks to the bottom).
- **Question for author** — genuinely open things to *ask* rather than assert
  (useful reviewing others' PRs; learning by asking).

**Importance tiers:** `Blocker` → `Major` → `Minor` → `Nit`. Overview sorts by
tier first, then groups by category.

**Confidence:** each point notes when it is a hunch vs. a certainty, so the user
does not post something they are unsure of ("understand before you commit to a
comment").

**Per-point fields:** `file:line` · category · tier · confidence · observation ·
why · draft comment.

Points matching an `accepted` rule are filtered (or greyed "known-OK") before
ranking, so the map only shows what is genuinely worth attention.

## 4. Project-rules memory

**Location:** `~/.review-me-senior/<repo-name>/rules.md` — local, per-user, out of
repo, keyed by repo directory name. Plain Markdown; human-readable and editable.

**Rule entry captures:**
- **description** of the pattern (e.g. "service layer returns `Result<T>` rather
  than throwing"),
- **verdict:** `accepted` (fine here — stop flagging), `expected` (we do it this
  way — flag deviations), or `forbidden` (always flag this),
- optional **scope** (path glob or category),
- one-line **rationale** and the date added.

**Read:** Recon loads every rule for the repo. Analyze applies them: `accepted`
filters/greys matching points; `expected`/`forbidden` become active checks.

**Written:** only during drill-in / wrap, and only on explicit confirmation. When
the user blesses something, the skill drafts the rule entry, **shows the exact line
it will append**, and writes only on yes. The user can also state a rule directly
("always flag raw SQL string concatenation") and the skill captures it. Never
silent.

**Matching is judgment-based:** the skill reads rules as natural language and
decides whether a finding matches — no brittle regex required (rules may name
files/symbols). The user can ask *"what rules do you know for this repo?"* any time
to list them.

Local + out-of-repo, so conventions and code never land in the repo by accident.
(Shared/committed rules are a possible future extension, deliberately out of scope
for v1.)

## 5. Config surface (`SKILL.md`, top of file)

| Key | Default | Purpose |
| --- | --- | --- |
| `bar_level` | `staff` | `mid` / `senior` / `staff` / `principal` — depth of reasoning |
| `categories` | all 8 on | toggle any category off |
| `tone` | `instructive` | `warm` / `instructive` / `blunt` — how findings are phrased |
| `context_sources` | `diff, pr, focus` | `diff` always on; `pr` = optional pasted/fetched PR; `focus` = the one-line focus prompt |
| `draft_comments` | `true` | generate a draftable comment per point in drill-in |
| `comment_style` | `concise` | `concise` / `detailed` — length/shape of drafted comments |
| `rules_enabled` | `true` | load + apply the project-rules memory |
| `rules_dir` | `~/.review-me-senior/<repo-name>/` | where learned rules live (out of repo) |
| `save_dir` | `~/.review-me-senior/<repo-name>/` | where review notes are written (out of repo) |
| `save_notes` | `ask` | `ask` / `always` / `never` — save the session's review notes at wrap |
| `include_diff_snippets` | `true` | quote reviewed code in **saved notes** (terminal always shows it) |
| `redact_secrets` | `true` | scrub secrets/tokens/URLs from anything written to disk |

Two deliberate differences from the interview skill: `include_diff_snippets`
defaults **on** (local review notes are useless without the code, and they are
out-of-repo), and there is no scoring config.

## 6. Outputs

- **Terminal (primary).** The ranked overview map, then the interactive drill-in
  (code snippet + reasoning + draft comment per point). A working session, not a
  cold report.
- **Saved Markdown review notes** (out of repo, `save_dir`, governed by
  `save_notes`). Structure: title / branch / date / base / focus; a summary line
  (counts by tier + category); the **full ranked map** (nothing lost); then a
  detailed block per drilled-into point (location, category/tier/confidence,
  observation, why, and the **draft comment kept**); plus a note of any rules added
  this session. Paste-ready into a PR.

**No HTML report in v1** (deliberate; easy to add later). Saved artifacts respect
the §7 safety rules.

## 7. Artifact safety (privacy boundaries)

Same model as `interview-me-senior` §6a. Enforce in this order before writing any
file (applies to both `save_dir` notes and the `rules_dir` memory):

1. **Resolve the path.** Expand `~` and `<repo-name>`. Defaults resolve under
   `~/.review-me-senior/<repo-name>/`, outside the repo. Create if missing.
2. **If a resolved path is inside the current repo** (only via user override):
   determine via `git rev-parse --show-toplevel` + absolute-path containment. If
   inside, ensure the directory is in `.gitignore` (append if absent) and warn if
   the target is already git-tracked before writing. Default out-of-repo paths make
   this a no-op — do not touch `.gitignore`.
3. **`include_diff_snippets`** governs code quoting in **saved notes** (default on
   here). Terminal output is exempt (always shows code).
4. **`redact_secrets`** (default on) scrubs secrets/tokens/keys/URLs from all
   disk-written text (notes and rules). Terminal exempt.
5. **Never** write to a git-tracked path without the step-2 warning.

## 8. Repo structure

```
Interview-Me/
├── README.md                  # updated: "Skills in this repo" covering both
├── LICENSE
├── interview-me-senior/       # existing — untouched
│   └── ...
├── review-me-senior/          # new
│   ├── SKILL.md               # config block + five-phase logic
│   └── examples/
│       ├── sample-review-notes.md   # saved-notes output contract
│       └── sample-rules.md          # example learned rules.md
└── docs/
    └── 2026-07-07-review-me-senior-design.md
```

- No `assets/` (no HTML template in v1).
- README gains a short "Skills in this repo" section; install instructions
  generalised to cover copying either skill folder. GitHub description/topics
  refreshed to reflect two skills.

## 9. Open questions / future

- Shared/committed project-rules file (team-wide), layered with the local file.
- Optional HTML review summary.
- Optional post-to-PR integration (deliberately excluded from v1).
- A `principal`-level sibling, or a `mid` bar, as the family grows.
