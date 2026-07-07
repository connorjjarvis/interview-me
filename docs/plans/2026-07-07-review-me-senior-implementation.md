# review-me-senior Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `review-me-senior`, a review-copilot Claude Code / Copilot CLI / Codex skill that gives a staff-level ranked map of review points on a branch/PR diff, drills into each with reasoning + a draftable comment, and learns your project's conventions locally over time.

**Architecture:** A second skill in the existing `interview-me` repo, authored as runtime instructions (`SKILL.md`) an agent follows with its own tools (git, file read/write). Two example files ship as contracts (saved-notes shape, learned-rules shape). No compiled code, no HTML template, no external dependency. Reuses the interview skill's artifact-safety model.

**Tech Stack:** Markdown (skill + docs), YAML frontmatter. Git for distribution.

## Global Constraints

- **Skill format:** `SKILL.md` MUST have valid YAML frontmatter with `name: review-me-senior` and a `description:` that triggers on "review me", "review this branch/PR", "help me review", "review the code", "help me write a review" — and is distinct enough from `interview-me-senior` (which owns "interview me", "grill me", "score my work", "defend my decisions") that the two don't collide.
- **Stack-agnostic:** no hardcoded Azure DevOps / Jira / GitHub integration. A PR is user-pasted; environment PR tooling is used only if already present.
- **Reviews, never edits:** this skill reviews the diff and drafts *comments*. It MUST NOT modify the code under review, apply fixes, or invoke `brainstorming`. It assists the user's review; it does not do the work.
- **No scoring, no HTML, no PR posting** in v1 (spec §1 non-goals).
- **Artifact safety (spec §7) are HARD requirements** and apply to BOTH `save_dir` (notes) and `rules_dir` (memory): out-of-repo default (`~/.review-me-senior/<repo-name>/`); in-repo overrides gitignored + tracked-check warned via `git rev-parse --show-toplevel`; `include_diff_snippets` (default on) governs code in saved notes; `redact_secrets` (default on); never write a tracked path without warning.
- **Memory writes require explicit confirmation:** the skill shows the exact rule line it will append and writes only on the user's yes. Never silent. Verdicts are exactly `accepted` / `expected` / `forbidden`.
- **Cross-platform:** paths work on Windows, macOS, Linux; `~` expands to home on all three.
- **Existing `interview-me-senior/` is untouched.**
- **Spec is source of truth:** `docs/2026-07-07-review-me-senior-design.md`; section refs (§) below point into it.

---

### Task 1: Example/contract files

**Files:**
- Create: `review-me-senior/examples/sample-rules.md`
- Create: `review-me-senior/examples/sample-review-notes.md`

**Interfaces:**
- Produces: the two output contracts SKILL.md's phases (Task 3) must match — the learned-rules memory shape (§4) and the saved review-notes shape (§6).

- [ ] **Step 1: Write `sample-rules.md`** (the memory format, §4)

A realistic `rules.md` for a fictional repo. Top note: `> Lives at ~/.review-me-senior/<repo-name>/rules.md — local, per-user, out of your repo.` Then 3-4 example rules, each showing all fields: a short **description**, a **verdict** (must use all three across the examples: `accepted`, `expected`, `forbidden`), optional **scope** (a path glob or category), a one-line **rationale**, and a `added: 2026-07-07` date. Use a clear per-rule format, e.g. a `###` heading per rule with a small bullet list of fields, or a table — pick one and be consistent. Include at least one `accepted` (e.g. "service methods return `Result<T>` instead of throwing"), one `expected` (e.g. "new endpoints must have an integration test"), one `forbidden` (e.g. "raw SQL string concatenation").

- [ ] **Step 2: Write `sample-review-notes.md`** (the saved-notes output, §6)

A realistic saved review-notes file for a fictional branch. Structure in order: top privacy note (`> Saved to ~/.review-me-senior/<repo>/ — not committed to your repo.`); `# Review — <branch> — <YYYY-MM-DD>` with base branch + focus line; a **summary line** with counts by tier and category; a `## Review map` section = the full ranked list (each: `path:line` · category · tier · one-line reason, Blocker→Nit, nits last); a `## Detail` section with a block per drilled-into point (location, category/tier/confidence, the observation, the *why*, and the **draft comment kept**); and a `## Rules added this session` note. Use the 8 categories and the four tiers from §3. Include a couple of `include_diff_snippets`-on code snippets so the shape is clear. No placeholders (no TODO/TBD).

- [ ] **Step 3: Verify structure vs spec §3/§4/§6**

Read both files. Confirm: sample-rules.md uses all three verdicts and shows every field; sample-review-notes.md has the map + detail + rules-added sections, uses valid category names and `Blocker/Major/Minor/Nit` tiers. No placeholders.

- [ ] **Step 4: Commit**

```bash
git add review-me-senior/examples/sample-rules.md review-me-senior/examples/sample-review-notes.md
git commit -m "docs: add review-me-senior example contracts (rules + review notes)"
```

---

### Task 2: SKILL.md frontmatter + config block

**Files:**
- Create: `review-me-senior/SKILL.md`

**Interfaces:**
- Produces: the config keys every phase in Task 3 reads. Exact keys (spec §5): `bar_level`, `categories`, `tone`, `context_sources`, `draft_comments`, `comment_style`, `rules_enabled`, `rules_dir`, `save_dir`, `save_notes`, `include_diff_snippets`, `redact_secrets`.

- [ ] **Step 1: Write YAML frontmatter**

```yaml
---
name: review-me-senior
description: A staff-level review copilot for a branch or PR. Reads the diff, gives you a ranked map of review points (correctness, risk, design, tests, security, nits) each with the reasoning and a draftable comment in your voice, then lets you drill into the ones that matter. It does not score you and does not post anything — it helps you understand the code and write the review yourself, and it learns your project's conventions over time. Use when the user says "review me", "review this branch", "review this PR", "help me review", or "help me write a review". (For being interviewed and scored on your own decisions, use interview-me-senior instead.)
---
```

- [ ] **Step 2: Write the CONFIG block (top of body, commented)**

A single fenced, commented block, one key per line, defaults from spec §5:

```
# CONFIG — edit these defaults to retune the review.
bar_level: staff             # mid | senior | staff | principal
categories: correctness, risk, design, maintainability, tests, security, nit, question   # remove any to mute
tone: instructive            # warm | instructive | blunt
context_sources: diff, pr, focus    # diff always on; pr = optional pasted/fetched PR; focus = one-line focus prompt
draft_comments: true
comment_style: concise       # concise | detailed
rules_enabled: true
rules_dir: ~/.review-me-senior/<repo-name>/   # learned project rules — OUT OF REPO
save_dir: ~/.review-me-senior/<repo-name>/    # saved review notes — OUT OF REPO
save_notes: ask              # ask | always | never
include_diff_snippets: true  # quote reviewed code in SAVED notes (terminal always shows it)
redact_secrets: true
```

- [ ] **Step 3: Verify frontmatter parses + is distinct**

Run: `head -20 review-me-senior/SKILL.md`
Expected: frontmatter opens/closes with `---`, `name: review-me-senior`, description contains the review trigger phrases AND the "use interview-me-senior instead" disambiguation.

- [ ] **Step 4: Commit**

```bash
git add review-me-senior/SKILL.md
git commit -m "feat: add review-me-senior frontmatter and config block"
```

---

### Task 3: SKILL.md five-phase logic + memory + artifact safety

**Files:**
- Modify: `review-me-senior/SKILL.md` (append phase instructions after the config block)

**Interfaces:**
- Consumes: config keys (Task 2), the two contract files (Task 1).
- Produces: the complete runtime behavior.

Write explicit, imperative instructions to the executing agent. Cover, in order:

- [ ] **Step 1: Ground rules + Phase 1 Recon (silent)**

Ground rule up top: this skill reviews and drafts comments only — it MUST NOT edit the code under review, apply fixes, or invoke brainstorming; if asked to fix, say it only reviews and drafts, and that they can run a separate session to implement. Then Phase 1: resolve the diff range (default-branch detection → `git merge-base <default> HEAD`...HEAD → staged → last commit) and read it. If `pr` in `context_sources`, let the user paste/point at a PR; use environment PR tooling only if already present (no hardcoded platform), else pasted text. Load the project-rules memory (Step 4). Ask one line: *"anything you want me to focus on, or just a full pass?"* Capture as `focus`.

- [ ] **Step 2: Phase 2 — Analyze → build the review map (silent)**

Instructions per §3: read the diff at `bar_level`, produce review points, each with `file:line`, category (one of the enabled `categories`), importance tier (`Blocker`/`Major`/`Minor`/`Nit`), confidence (hunch vs certain), the observation, the *why* (teaching), and — if `draft_comments` — a draft comment in the user's voice at `comment_style` length. Apply loaded rules: `accepted` matches filtered or greyed "known-OK"; `expected`/`forbidden` become active checks. Rank by tier, then group by category, nits last.

- [ ] **Step 3: Phase 3 Overview + Phase 4 Drill-in**

Overview: print the ranked map — each point `file:line` + category + tier + one-line reason, sorted tier-first, nits last. Drill-in (user-driven): the user selects by number / range / "all correctness" / "skip nits". For each selected point show the code snippet + full reasoning + the draft comment. Support: "why does that matter?", "redraft shorter/longer", and blessing ("that's fine, we do it this way") → hand to Step 5. Never force a march through every point; the user drives order and depth.

- [ ] **Step 4: Phase 1b/5 — Project-rules memory (read + write, §4)**

Read (during recon): if `rules_enabled`, resolve `rules_dir`, read `rules.md` if present, parse rules {description, verdict ∈ accepted/expected/forbidden, optional scope, rationale, date}. Apply in Phase 2. Matching is judgment-based (natural-language), not regex. Support the user asking "what rules do you know for this repo?" → list them.
Write (during drill-in / wrap): only on explicit confirmation. When the user blesses a pattern or states a rule, draft the entry, SHOW THE EXACT LINE(S) to be appended, and append to `rules.md` only on yes (create the file/dir if absent). Never silent. Use the same field format as `examples/sample-rules.md`.

- [ ] **Step 5: Phase 5 — Wrap + save notes**

Per `save_notes` (`ask`/`always`/`never`): build the review notes in the `examples/sample-review-notes.md` shape (summary + full map + detail for drilled points + rules-added note) and save to `save_dir`. Print the saved path.

- [ ] **Step 6: Artifact-safety enforcement (spec §7, HARD) — applies to save_dir AND rules_dir**

In this precedence: (1) resolve the path, expand `~` and `<repo-name>`, default under `~/.review-me-senior/<repo-name>/` (create if missing). (2) If a resolved path is inside the repo (determine via `git rev-parse --show-toplevel` + absolute-path containment): ensure it's in `.gitignore` (append if absent) AND warn if the target is git-tracked before writing. Default out-of-repo → no-op, don't touch `.gitignore`. (3) `include_diff_snippets` false → no code quoted in saved notes (terminal exempt). (4) `redact_secrets` → scrub secrets/tokens/keys/URLs from all disk writes (notes AND rules). (5) never write a tracked path without the (2) warning.

- [ ] **Step 7: Dry-run verification**

Reason through SKILL.md end-to-end against a hypothetical branch with, say, 6 changed points and an existing `rules.md` that blesses one of them. Confirm: recon reads the diff + loads rules; the blessed point is filtered/greyed; the overview is tier-sorted with nits last; drill-in shows snippet+why+draft comment; blessing a new point shows the exact line and writes only on confirmation; notes save to `~/.review-me-senior/<repo>/` (not the repo); no `.gitignore` touched for the out-of-repo default. Fix any SKILL.md wording that fails.

- [ ] **Step 8: Commit**

```bash
git add review-me-senior/SKILL.md
git commit -m "feat: add review-me-senior five-phase logic, rules memory, and artifact safety"
```

---

### Task 4: README — cover both skills

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: both skills. Generalises install to cover copying either skill folder.

- [ ] **Step 1: Add a "Skills in this repo" section**

Near the top (after the intro), add a short section listing the two skills with one line each and what to reach for when:
- **`interview-me-senior`** — get interviewed and scored on the decisions in a change you just made.
- **`review-me-senior`** — a review copilot: ranked review points + draftable comments for a branch/PR, learns your conventions, no score.

- [ ] **Step 2: Generalise the Install section**

Update the existing per-tool install blocks so they install whichever skill folder(s) the user wants (both, or just one). Keep the three tools (Claude Code `~/.claude/skills/`, Copilot CLI `copilot skill add` / `~/.copilot/skills/`, Codex `~/.codex/skills/`). Show copying both folders in the example (e.g. `cp -r interview-me-senior review-me-senior ~/.claude/skills/`) and note you can copy just one.

- [ ] **Step 3: Add a "Config" pointer + link the review examples**

Add a one-line pointer that `review-me-senior` has its own config block at the top of its `SKILL.md`, and link `review-me-senior/examples/sample-review-notes.md` and `sample-rules.md`.

- [ ] **Step 4: Verify + commit**

Run: `grep -c 'review-me-senior' README.md` (expect ≥ 3: skills section, install, examples links).
```bash
git add README.md
git commit -m "docs: document review-me-senior alongside interview-me-senior in README"
```

---

### Task 5: Validation, install check, GitHub metadata refresh

**Files:**
- No new repo files (validation + ops only).

- [ ] **Step 1: Install locally and confirm discoverability**

Copy `review-me-senior/` into `~/.claude/skills/review-me-senior/`. Confirm the skill is discoverable by its `name` and that its description reads distinctly from `interview-me-senior`.

- [ ] **Step 2: Static end-to-end check**

Read the final `SKILL.md` against the spec: all five phases present; taxonomy (8 categories + 4 tiers) usable from the text; memory read/write with the exact-line-before-append rule; artifact safety covering BOTH save_dir and rules_dir; no scoring/HTML/posting crept in; no placeholders. A live interactive run (drives real branch review) is the user's to do — note it as pending, do not fake it.

- [ ] **Step 3: Refresh GitHub description/topics (optional, on user go-ahead)**

If the user wants it published, update the repo description to mention both skills and add a topic like `pr-review`. (Deferred to the user for the actual push; do not push without explicit consent.)

- [ ] **Step 4: Final commit if any fixes were needed**

```bash
git add -A
git commit -m "chore: finalize review-me-senior after validation"
```

---

## Self-Review

**Spec coverage:**
- §1 purpose / non-goals (no score/HTML/posting, reviews-not-edits) → Task 2 description, Task 3 Step 1 ground rule, Global Constraints.
- §2 five phases → Task 3 Steps 1-5.
- §3 taxonomy (8 categories, 4 tiers, confidence) → Task 3 Step 2; Task 1.
- §4 memory (3 verdicts, exact-line-before-append, judgment matching) → Task 3 Step 4; Task 1 sample-rules.md.
- §5 config (12 keys) → Task 2; README pointer Task 4.
- §6 outputs (terminal + Markdown notes) → Task 3 Steps 3/5; Task 1 sample-review-notes.md.
- §7 artifact safety (save_dir + rules_dir) → Task 3 Step 6; Task 5 Step 2.
- §8 repo structure → Tasks 1-2 (folder), Task 4 (README).
No uncovered spec sections.

**Placeholder scan:** No "TBD"/"handle edge cases"/"similar to Task N". Config keys enumerated; example contents specified concretely.

**Type/name consistency:** The 12 config keys are identical across Task 2, Task 3, and the README pointer (Task 4). Category names (`correctness, risk, design, maintainability, tests, security, nit, question`) and tiers (`Blocker/Major/Minor/Nit`) are consistent across Task 1, Task 2's `categories` default, and Task 3. Verdicts (`accepted/expected/forbidden`) consistent across Task 1 and Task 3 Step 4. Paths (`~/.review-me-senior/<repo-name>/`) consistent across Task 2, Task 3 Step 6, Task 5.
