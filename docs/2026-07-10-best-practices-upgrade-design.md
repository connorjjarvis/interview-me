# Best-Practices Upgrade — Design Spec

**Date:** 2026-07-10
**Status:** Approved (design), pending implementation
**Repo:** `interview-me` (standalone, GitHub-distributable)
**Skills touched:** `review-me-senior` (v1.1, main change) + `interview-me-senior` (light nudge)

## 1. Motivation

Real usage feedback: `review-me-senior` raises good points but under-flags
**maintainability** and **language-idiom** problems — e.g. C# null-forgiving `!`
overuse (a crash-prone idiom a rules engine always catches but an LLM skims past).
Two root causes, two fixes:

1. **Recall.** The `maintainability` category (and others) is a bare label with no
   enumerated smells. The LLM has no checklist, so it skims. → Strengthen the core
   rubric.
2. **Stack idioms.** Framework-specific best practices (Blazor WASM, MudBlazor,
   Fluxor, Feature Sliced Design) aren't represented at all. → Add modular,
   auto-detected best-practice packs.

Sonar already runs in the user's PR pipeline, so this skill is deliberately the
**judgment + idiom + teaching** layer, not a replacement for a static analyzer
(approach A chosen over Sonar integration).

## 2. Scope

- **`review-me-senior`:** strengthened universal rubric + modular best-practice
  packs (auto-detected), findings tagged by pack source. Main change.
- **`interview-me-senior`:** a light rubric nudge so it *grills* on clear
  best-practice deviations / crash-prone idioms — question generation, not a
  findings list. No pack machinery.
- Additive / opt-out: with no packs matched and the stronger rubric, behaviour is a
  superset of today's. Five phases, overview→drill-in→learn flow, artifact safety,
  and `rules.md` memory are all unchanged. `rules.md` still overrides packs
  (`accepted` suppresses a pack finding).

## 3. Strengthened universal rubric (`review-me-senior` core)

Bake an explicit **smell checklist** into Phase 2, per category, plus a directive:
*before ranking, do a deliberate mechanical pass for these smells; do not rely on
what jumps out.* Language-general; ships in the core SKILL.md. Representative
checklist content:

- **maintainability** — long methods/params, duplication, deep nesting, primitive
  obsession, magic numbers/strings, unclear names, dead code, misleading comments,
  God classes, boolean params.
- **risk** — swallowed/empty catches, null handling, unvalidated input, resource
  leaks (undisposed), off-by-one, unbounded loops/collections, silent fallbacks.
- **correctness / security / tests** — enumerated similarly (wrong conditions/state;
  injection/authz/secret/unsafe-input; missing/weak tests for the change).

This is the "universal" layer — there is deliberately **no** separate "universal
pack" (it would duplicate the rubric).

## 4. Best-practice packs (`review-me-senior`)

- **Location:** `review-me-senior/references/` — one Markdown file per pack, shipped
  inside the skill folder so it travels with a single-folder install.
- **Pack file format:** a header with `applies_to:` detection signals (extension
  globs and/or telltale imports, e.g. `*.cs`, `*.razor`, `using Fluxor`) followed
  by a list of entries. Each entry: **name**, **what to flag** (the anti-pattern),
  **why** (teaching), **default category** (one of the 8), **default tier**
  (Blocker/Major/Minor/Nit), and a short **example**.
- **Auto-detect:** during recon the skill scans the diff's file extensions +
  telltale imports and enables matching packs. Config key `best_practice_packs`:
  `auto` (default) · `none` (disable) · explicit list (e.g. `dotnet, blazor-fsd`).
- **Merge + transparency:** each pack finding becomes a normal review point
  (category/tier/confidence/why/draft-comment) **tagged with the pack name** (e.g.
  `[dotnet]`) so the user sees the source and can learn the rule. Ranking and
  drill-in are unchanged. `rules.md` memory still wins: an `accepted` rule
  suppresses a matching pack finding.

### 4.1 v1 packs

**`dotnet`** — auto-enables on `*.cs`. Modern C#/.NET idioms & crash-prone
anti-patterns. Representative entries:
- Null-forgiving `!` overuse — silencing nullable warnings instead of handling null
  → NRE at runtime *(risk / Major)*.
- `async void` outside event handlers — unobservable exceptions *(risk / Major)*.
- Sync-over-async (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`) —
  blocking/deadlock *(risk / Major)*.
- Swallowed exceptions / overly broad `catch (Exception)` *(risk / Major)*.
- Undisposed `IDisposable` / missing `using` *(risk / Major)*.
- Fire-and-forget `Task` not awaited; missing `CancellationToken` on async APIs
  *(risk / Minor)*.
- LINQ multiple enumeration; magic strings for config keys *(maintainability /
  Minor)*.
- Idiom nits: `is null` over `== null`, prefer `record`/immutability *(nit)*.

**`blazor-fsd`** — auto-enables on `*.razor` + MudBlazor/Fluxor signals. Blazor
WASM + MudBlazor + Fluxor + Feature Sliced Design. Representative entries:
- **Fluxor** — mutating state outside a reducer; side effects inside a
  `[ReducerMethod]` (reducers must be pure); UI/data work belongs in Effects;
  component reading state without `IState<T>` *(design / Major)*.
- **FSD boundaries** — a feature importing another feature's internals; shared code
  not in the shared layer; imports crossing slice boundaries the wrong way
  *(design / Major)*.
- **Blazor WASM** — undisposed subscriptions / `IDisposable` components; heavy sync
  work on the UI thread; missing `@key` in list rendering; giant components that
  should split; `[Parameter]` mutation; manual `StateHasChanged` misuse
  *(risk / maintainability)*.
- **MudBlazor** — inline `Style=` instead of theme/`Class`; missing `MudForm`
  validation; not using MudBlazor equivalents for consistency
  *(maintainability / nit)*.

Entries above are representative; the pack files hold the full lists and are
user-editable/extensible.

## 5. Interviewer nudge (`interview-me-senior`)

A short addition to the decision-map / question-generation instructions: when the
diff clearly leans on a crash-prone idiom or deviates from a modern best practice,
the interviewer may turn that into a pointed question ("you used null-forgiving `!`
in several spots — what's your runtime guarantee those aren't null?"). It remains a
**questioner**, not a findings list. Examples given are **language-general**
(null-forgiving operators, unobserved async, sync-over-async, swallowed errors,
undisposed resources) so the shared skill stays stack-agnostic. These questions
count within the existing `core_questions` budget and score under existing
dimensions (Technical judgment / Risk & edge-case thinking). No packs, no new
config.

## 6. Config changes

`review-me-senior` gains exactly one key:

| Key | Default | Purpose |
| --- | --- | --- |
| `best_practice_packs` | `auto` | `auto` (detect from diff) · `none` (disable) · explicit list (e.g. `dotnet, blazor-fsd`) |

`interview-me-senior` gains no new config.

## 7. Files & packaging

- **New:** `review-me-senior/references/dotnet.md`, `review-me-senior/references/blazor-fsd.md`.
- **Modified:** `review-me-senior/SKILL.md` (rubric strengthening §3 + pack
  load/auto-detect/merge §4 + `best_practice_packs` config §6);
  `interview-me-senior/SKILL.md` (the §5 grilling nudge).
- **Examples:** update `review-me-senior/examples/sample-review-notes.md` to show a
  couple of pack-tagged findings (e.g. a `[dotnet]` null-forgiving point) so the
  source-tagging is visible.
- **README:** document `best_practice_packs`, the two packs, that packs are
  extensible (drop a `references/*.md`), and that the interviewer now grills on
  best practices.
- **Versioning:** `review-me-senior` v1.1 + a small `interview-me-senior` bump. This
  spec documents the upgrade; the original design specs remain as the v1 record.

## 8. Constraints (unchanged, still binding)

- Stack-agnostic core: no org-specific stack hardcoded into SKILL.md analysis; stack
  knowledge lives in opt-in packs. The `blazor-fsd` pack ships as a first-class
  example, not a core assumption.
- Reviews/drafts only; never edits code; no brainstorming.
- Artifact safety (v1 §7) unchanged; packs are read-only reference files inside the
  skill, no new write paths.
- Portable across Claude Code / Copilot CLI / Codex; packs are plain Markdown inside
  the skill folder.

## 9. Open questions / future

- User-supplied packs from outside the skill folder (a config path) — v1 keeps packs
  inside `references/`.
- More packs (TypeScript/React, Python, Go) as the family grows.
- Auto-seeding a pack's `expected`/`forbidden` entries into `rules.md` — deferred.
