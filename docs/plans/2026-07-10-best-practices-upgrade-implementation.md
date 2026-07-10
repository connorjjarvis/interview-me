# Best-Practices Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `review-me-senior` catch maintainability and language-idiom issues it currently skims past, via a strengthened universal rubric plus modular, auto-detected best-practice packs (C#/.NET and Blazor/MudBlazor/Fluxor/FSD), and give `interview-me-senior` a light best-practice-grilling nudge.

**Architecture:** Both skills are prose `SKILL.md` runtime instructions. The reviewer gains a `references/` folder of Markdown "packs" (curated anti-pattern lists) that it auto-detects from the diff and applies as extra checks, tagging findings by pack source. No compiled code, no new dependency, packs travel inside the skill folder.

**Tech Stack:** Markdown (skills + packs + docs), YAML-ish pack headers. Git for distribution.

## Global Constraints

- **Stack-agnostic core:** no org/framework stack hardcoded into `SKILL.md` analysis. Stack knowledge lives ONLY in opt-in packs under `references/`. The `blazor-fsd` pack ships as a first-class example, not a core assumption.
- **Reviewer reviews/drafts only:** `review-me-senior` MUST NOT edit code, apply fixes, or invoke brainstorming. Unchanged.
- **Additive / opt-out:** with no packs matched and the stronger rubric, behaviour is a superset of today. Do not remove or weaken existing phases, memory, or artifact safety.
- **`rules.md` memory still wins:** an `accepted` rule suppresses a matching pack finding.
- **New config key is exactly `best_practice_packs`**, default `auto`, values: `auto` (detect from diff) · `none` (disable) · explicit comma list of pack names (e.g. `dotnet, blazor-fsd`).
- **Pack finding tag format:** each pack finding is tagged with its pack name in square brackets, e.g. `[dotnet]`, `[blazor-fsd]`, visible in the overview, drill-in, and saved notes.
- **Interviewer nudge is language-general** (no Blazor/Fluxor/org stack named), counts within the existing `core_questions` budget, scores under existing dimensions, and adds NO new config.
- **Artifact safety unchanged:** packs are read-only reference files; no new write paths.
- **Portable** across Claude Code / Copilot CLI / Codex; packs are plain Markdown inside the skill folder.
- **Commit straight to `main`.** Existing `interview-me-senior` v1 behaviour otherwise untouched.
- **Spec is source of truth:** `docs/2026-07-10-best-practices-upgrade-design.md`; section refs (§) below point into it.

---

### Task 1: Best-practice pack files

**Files:**
- Create: `review-me-senior/references/dotnet.md`
- Create: `review-me-senior/references/blazor-fsd.md`

**Interfaces:**
- Produces: the two packs and the shared pack-file FORMAT that Task 2's SKILL.md loads and parses. Format is fixed here and referenced by later tasks.

Pack file format (use verbatim for both files):
- Line 1: `# <pack-name> pack` (pack-name = `dotnet` / `blazor-fsd`).
- An `**Applies to:**` line naming detection signals — file extensions and optional telltale imports/usings. This is what auto-detect (Task 2) keys on.
- Then one `## <entry name>` section per anti-pattern, each with exactly these bullet fields:
  - `- **Flag:** <the anti-pattern to detect>`
  - `- **Why:** <the teaching reason it matters / the consequence>`
  - `- **Category:** <one of: correctness, risk, design, maintainability, tests, security, nit>`
  - `- **Tier:** <Blocker | Major | Minor | Nit>`
  - `- **Example:** <a short code example of the smell>`

- [ ] **Step 1: Write `references/dotnet.md`**

Header: `# dotnet pack`, then `**Applies to:** C#/.NET — files matching \`*.cs\`.` Then one `##` entry for each of these (use the category/tier in parentheses), each with Flag/Why/Category/Tier/Example filled concretely:
- Null-forgiving `!` overuse (risk / Major) — silencing nullable warnings instead of handling null → NRE at runtime. Example: `var name = user!.Profile!.Name;`
- `async void` outside event handlers (risk / Major) — exceptions are unobservable and crash the context.
- Sync-over-async: `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` (risk / Major) — blocking/deadlock.
- Swallowed exceptions / overly broad `catch (Exception)` with no rethrow/log (risk / Major).
- Undisposed `IDisposable` / missing `using` (risk / Major).
- Fire-and-forget `Task` not awaited (risk / Minor).
- Missing `CancellationToken` on async APIs (risk / Minor).
- LINQ multiple enumeration of the same `IEnumerable` (maintainability / Minor).
- Magic strings for config keys (maintainability / Minor).
- `== null` instead of `is null` pattern; prefer `record`/immutability where a DTO is mutated (nit / Nit).

- [ ] **Step 2: Write `references/blazor-fsd.md`**

Header: `# blazor-fsd pack`, then `**Applies to:** Blazor WebAssembly + MudBlazor + Fluxor + Feature Sliced Design — files matching \`*.razor\` (and \`*.cs\` alongside them), or code with \`using Fluxor\` / \`using MudBlazor\`.` Then one `##` entry each, Flag/Why/Category/Tier/Example filled:
- Fluxor: mutating state outside a reducer (design / Major).
- Fluxor: side effects inside a `[ReducerMethod]` — reducers must be pure; do I/O in an Effect (design / Major).
- Fluxor: component reads state directly instead of via `IState<T>` (design / Major).
- FSD: a feature importing another feature's internals; shared code not placed in the shared layer; imports crossing slice boundaries the wrong way (design / Major).
- Blazor WASM: component subscribes/allocates `IDisposable` but does not implement `IDisposable`/`Dispose` (risk / Major).
- Blazor WASM: heavy synchronous work on the UI thread / in `OnInitialized` instead of async (risk / Major).
- Blazor WASM: missing `@key` on a rendered list (maintainability / Minor).
- Blazor WASM: giant component that should be split; `[Parameter]` mutated internally; manual `StateHasChanged` misuse (maintainability / Minor).
- MudBlazor: inline `Style=` instead of theme/`Class`; missing `MudForm` validation; hand-rolled control where a MudBlazor equivalent exists (maintainability / Nit).

- [ ] **Step 3: Verify both packs**

Read both files. Confirm: header line present; `**Applies to:**` line present with extensions/signals; every `##` entry has all five bullet fields (Flag/Why/Category/Tier/Example); only valid categories (correctness, risk, design, maintainability, tests, security, nit) and tiers (Blocker/Major/Minor/Nit) used; no placeholders (TODO/TBD).

- [ ] **Step 4: Commit**

```bash
git add review-me-senior/references/dotnet.md review-me-senior/references/blazor-fsd.md
git commit -m "feat: add dotnet and blazor-fsd best-practice packs for review-me-senior"
```

---

### Task 2: review-me-senior SKILL.md — rubric + pack engine + config

**Files:**
- Modify: `review-me-senior/SKILL.md`

**Interfaces:**
- Consumes: the pack files + format from Task 1 (`references/*.md`, `**Applies to:**` header, the five per-entry fields).
- Produces: the `best_practice_packs` config key and the runtime behaviour that loads/auto-detects/applies packs and strengthens the rubric.

- [ ] **Step 1: Add the `best_practice_packs` config key**

In the fenced CONFIG block, add one line (keep every existing key unchanged):
```
best_practice_packs: auto    # auto (detect from the diff) | none | explicit list e.g. dotnet, blazor-fsd (packs live in references/)
```

- [ ] **Step 2: Strengthen the Phase 2 rubric (spec §3)**

In Phase 2, after the category list, add an explicit per-category **smell checklist** and a directive to sweep for them mechanically BEFORE ranking (not just what jumps out). Add checklist content for at least `maintainability` (long methods/params, duplication, deep nesting, primitive obsession, magic numbers/strings, unclear names, dead code, misleading comments, God classes, boolean params) and `risk` (swallowed/empty catches, null handling, unvalidated input, undisposed resources, off-by-one, unbounded loops/collections, silent fallbacks), plus a short line for correctness/security/tests. Language-general — no framework named here.

- [ ] **Step 3: Add pack loading + auto-detect to Phase 1 Recon**

Add a recon step: resolve `best_practice_packs`. If `none`, load no packs. If an explicit list, load those named files from `references/`. If `auto` (default), scan the diff's changed-file extensions and telltale imports and enable every pack in `references/` whose `**Applies to:**` signals match (e.g. `*.cs` → `dotnet`; `*.razor` or `using Fluxor`/`using MudBlazor` → `blazor-fsd`). Read each enabled pack file. State (silently, for its own use) which packs are active; it may mention the active packs once in the Phase 2 → overview handoff.

- [ ] **Step 4: Apply packs in Phase 2 with source tagging (spec §4)**

Add to Phase 2: for each enabled pack, treat every entry as an active check against the diff. When an entry matches, emit a normal review point using the entry's **Category** and **Tier** (the skill may adjust the tier by judgment for this specific diff), with the entry's **Why** as the teaching, and **tag the point with its pack name in square brackets** (e.g. prefix the observation with `[dotnet]`). Pack findings rank and drill in like any other point. Reassert that `rules.md` memory still wins: an `accepted` rule suppresses a matching pack finding (count it among filtered points as today).

- [ ] **Step 5: Surface the tag in overview + saved notes**

Ensure the instructions say the pack tag (`[dotnet]` etc.) appears in the overview line and carries through into the saved review-notes detail block, so the source is visible end to end.

- [ ] **Step 6: Dry-run verification**

Reason through SKILL.md end-to-end for a hypothetical branch touching a `.razor` and a `.cs` file with a null-forgiving `!` and a direct Fluxor state mutation, `best_practice_packs: auto`. Confirm: recon auto-enables `dotnet` + `blazor-fsd`; Phase 2 emits `[dotnet]` and `[blazor-fsd]` tagged points at the packs' categories/tiers; the strengthened rubric's mechanical pass is invoked; an `accepted` rule matching one of them suppresses it; tags appear in overview + notes; with `best_practice_packs: none` no packs load and behaviour is the old behaviour plus the stronger rubric. Fix any wording that fails.

- [ ] **Step 7: Commit**

```bash
git add review-me-senior/SKILL.md
git commit -m "feat: strengthen review-me rubric and add auto-detected best-practice pack engine"
```

---

### Task 3: interview-me-senior SKILL.md — best-practice grilling nudge

**Files:**
- Modify: `interview-me-senior/SKILL.md`

**Interfaces:**
- Produces: a language-general nudge in the interviewer's question generation. Independent of the reviewer changes.

- [ ] **Step 1: Add the nudge to the decision-map / question phase (spec §5)**

In the phase that builds decision points / questions, add a short paragraph: when the diff clearly leans on a crash-prone idiom or deviates from a modern best practice, the interviewer MAY turn that into a pointed question (e.g. "you used the null-forgiving operator in several spots — what's your runtime guarantee those aren't null?"). Keep it a QUESTION generator, not a findings list. Give a handful of LANGUAGE-GENERAL examples only (null-forgiving operators, unobserved async, sync-over-async, swallowed errors, undisposed resources) — do NOT name Blazor/Fluxor/MudBlazor or any org stack. These questions count within the existing `core_questions` budget and are scored under the existing dimensions (Technical judgment / Risk & edge-case thinking). Add NO new config key.

- [ ] **Step 2: Verify scope**

Read the change. Confirm: it's within the existing phase, adds no config key, names no specific framework/org stack, and doesn't turn the interviewer into a reviewer (still questions, no findings list / no scoring changes).

- [ ] **Step 3: Commit**

```bash
git add interview-me-senior/SKILL.md
git commit -m "feat: add language-general best-practice grilling nudge to interview-me-senior"
```

---

### Task 4: Examples + README

**Files:**
- Modify: `review-me-senior/examples/sample-review-notes.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: the pack tag format (`[dotnet]`) from Task 2 and pack names from Task 1.

- [ ] **Step 1: Add pack-tagged findings to the sample notes**

In `review-me-senior/examples/sample-review-notes.md`, add (or adapt) at least two review points that are pack-sourced and tagged — e.g. a `[dotnet]` null-forgiving `!` point and a `[blazor-fsd]` Fluxor-state-mutation point — appearing in both the `## Review map` and `## Detail` sections, consistent with the existing shape. Update the summary counts if needed. Keep it realistic; no placeholders.

- [ ] **Step 2: Document the packs + config in README**

In `README.md`: under review-me-senior's coverage, add a short note that it applies auto-detected best-practice packs (shipping `dotnet` and `blazor-fsd`), that findings are tagged by pack, that packs are extensible (drop a `references/*.md`), and document the `best_practice_packs` config key (`auto` / `none` / explicit list). Also add a one-liner that `interview-me-senior` now grills on best-practice deviations. Keep existing content intact.

- [ ] **Step 3: Verify + commit**

Run: `grep -c 'best_practice_packs' README.md` (expect ≥ 1) and confirm `[dotnet]` and `[blazor-fsd]` both appear in `review-me-senior/examples/sample-review-notes.md`.
```bash
git add review-me-senior/examples/sample-review-notes.md README.md
git commit -m "docs: document best-practice packs and show pack-tagged findings in the sample"
```

---

### Task 5: Validation & re-install

**Files:**
- No new repo files (validation/ops only).

- [ ] **Step 1: Static end-to-end check**

Read the final `review-me-senior/SKILL.md` against the spec: `best_practice_packs` key present; strengthened rubric with per-category smell checklists; pack auto-detect in recon; pack application with `[pack]` tagging in Phase 2; tag surfaced in overview + notes; `rules.md`-wins reasserted; no scoring/HTML/posting/edit crept in. Read `interview-me-senior/SKILL.md`: nudge present, language-general, no new config. Confirm the two pack files parse against the Task 1 format. No placeholders.

- [ ] **Step 2: Re-install both skills locally**

Copy `review-me-senior/` and `interview-me-senior/` into `~/.claude/skills/` (overwriting), so the installed copies match. Confirm the `references/` folder came across with `review-me-senior`.

- [ ] **Step 3: Note pending user items**

A live interactive run (review a real Blazor/.NET branch; confirm packs fire and tags show) is the user's to do — note as pending, do not fake. Publishing to GitHub is the user's call.

- [ ] **Step 4: Final commit if any fixes were needed**

```bash
git add -A
git commit -m "chore: finalize best-practices upgrade after validation"
```

---

## Self-Review

**Spec coverage:**
- §1 motivation / approach A (no Sonar) → Global Constraints + Task 1/2.
- §2 scope (both skills, additive, memory-wins) → Task 2 Step 4, Task 3, Global Constraints.
- §3 strengthened rubric → Task 2 Step 2.
- §4 pack system (location, format, auto-detect, merge+tag) → Task 1 (format), Task 2 Steps 3-5.
- §4.1 v1 packs (dotnet, blazor-fsd entries) → Task 1 Steps 1-2.
- §5 interviewer nudge → Task 3.
- §6 config (`best_practice_packs`) → Task 2 Step 1; README Task 4 Step 2.
- §7 files/examples/README → Task 1 (references), Task 4 (examples + README).
- §8 constraints → Global Constraints.
No uncovered spec sections.

**Placeholder scan:** No "TBD"/"handle edge cases"/"similar to Task N". Pack entries and config line are concrete.

**Type/name consistency:** Pack names (`dotnet`, `blazor-fsd`), the tag format (`[dotnet]`/`[blazor-fsd]`), the config key (`best_practice_packs` with values `auto`/`none`/list), and the pack per-entry fields (Flag/Why/Category/Tier/Example) are consistent across Task 1, Task 2, and Task 4. Categories (correctness, risk, design, maintainability, tests, security, nit) and tiers (Blocker/Major/Minor/Nit) match the existing skill and the packs.
