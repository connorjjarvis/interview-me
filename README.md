# interview-me

A home for two independent, staff-level [agent skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills) for Claude Code, GitHub Copilot CLI, and OpenAI Codex: one interviews you on the decisions behind a change you just shipped and scores you, the other reviews a branch or PR and drafts the comments for you.

## What it does

Most "interview me" skills run *before* you write code: they grill you into a spec, then hand it off for implementation. This one runs **after**. You finish a piece of work, invoke the skill, and it reads your diff, reconstructs the decisions buried in it, and asks you to defend them — "you did X, why not Y?" — the way a staff engineer would in a design review, not a generic Q&A.

It probes weak answers with a single follow-up, asks one stretch question above the bar you set, and then scores you against a configurable rubric: per-dimension scores, an overall verdict, what you did well, what to work on, and a reading list built from your actual gaps, not boilerplate.

It is not a collaborator. It doesn't suggest fixes, doesn't offer to refactor, doesn't invoke `brainstorming`. It interrogates and it grades.

## Skills in this repo

This repo ships two independent skills. Install either one on its own, or both — they don't depend on each other.

- **[`interview-me-senior`](interview-me-senior/)** — get interviewed and scored on the decisions in a change you just made.
- **[`review-me-senior`](review-me-senior/)** — a review copilot: ranked review points + draftable comments for a branch/PR, learns your conventions, no score.

`review-me-senior` has its own config block at the top of [`review-me-senior/SKILL.md`](review-me-senior/SKILL.md). See a full worked example in [`review-me-senior/examples/sample-review-notes.md`](review-me-senior/examples/sample-review-notes.md), and the format it learns your conventions into in [`review-me-senior/examples/sample-rules.md`](review-me-senior/examples/sample-rules.md).

On top of the core rubric, `review-me-senior` auto-detects which **best-practice packs** apply to the diff and layers their checks in, tagging every finding it sources from a pack with the pack's name — e.g. `[dotnet]`, `[blazor-fsd]` — right in the overview and the saved notes, so you can see where a point came from. It ships two packs: `dotnet` (C#/.NET idioms — null-forgiving `!` overuse, `async void`, sync-over-async, and the like) and `blazor-fsd` (Blazor WASM + MudBlazor + Fluxor + Feature Sliced Design — state mutated outside a reducer, slice boundary violations, undisposed subscriptions). Packs are just Markdown under [`review-me-senior/references/`](review-me-senior/references/), so adding your own is a matter of dropping a new `references/<name>.md` file in the same format — no code to write. Control which packs run with the `best_practice_packs` config key: `auto` (default — detect from the diff's file types and imports), `none` (core rubric only), or an explicit comma list (e.g. `dotnet, blazor-fsd`).

`interview-me-senior` also picks up on this: when your diff leans on a crash-prone idiom or a modern best-practice deviation, it may turn that into a pointed interview question rather than just a design one.

The rest of this README covers `interview-me-senior` in detail.

## Install

This repo ships `interview-me-senior/` and `review-me-senior/`. Install whichever one(s) you want into whichever tool you use — the `SKILL.md` format is the shared [Agent Skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills) standard, so the same folders work across all three.

### Claude Code

Copy the skill folder(s) you want into your `~/.claude/skills/`.

macOS/Linux (both skills):

```bash
cp -r interview-me-senior review-me-senior ~/.claude/skills/
```

Windows (PowerShell, both skills):

```powershell
Copy-Item -Recurse interview-me-senior,review-me-senior "$HOME\.claude\skills\"
```

Only want one? Copy just that folder, e.g. `cp -r review-me-senior ~/.claude/skills/`.

### GitHub Copilot CLI

Copilot CLI discovers Agent Skills from `~/.copilot/skills/` (personal, all projects) or a repo's `.github/skills/` (project-scoped). Two ways to install:

**Option A — `copilot skill add` (recommended).** Point it at whichever skill directory(s) you want — run it once per skill:

```bash
git clone https://github.com/connorjjarvis/interview-me.git
copilot skill add ./interview-me/interview-me-senior
copilot skill add ./interview-me/review-me-senior
```

If you're already in a session, run `/skills reload`, then confirm it loaded with `/skills info interview-me-senior` or `/skills info review-me-senior`.

**Option B — copy it in manually.**

macOS/Linux (both skills):

```bash
cp -r interview-me/interview-me-senior interview-me/review-me-senior ~/.copilot/skills/
```

Windows (PowerShell, both skills):

```powershell
Copy-Item -Recurse interview-me\interview-me-senior,interview-me\review-me-senior "$HOME\.copilot\skills\"
```

Only want one? Run `copilot skill add` (or the copy) for just that folder.

### OpenAI Codex CLI

Codex discovers skills from `~/.codex/skills/` (personal, all projects) or a repo's `.codex/skills/` (project-scoped). Copy the folder(s) in — Codex picks them up on the next session (restart Codex if they don't appear):

macOS/Linux (both skills):

```bash
mkdir -p ~/.codex/skills
cp -r interview-me-senior review-me-senior ~/.codex/skills/
```

Windows (PowerShell, both skills):

```powershell
New-Item -ItemType Directory -Force "$HOME\.codex\skills" | Out-Null
Copy-Item -Recurse interview-me-senior,review-me-senior "$HOME\.codex\skills\"
```

Only want one? Copy just that folder instead.

## Usage

Finish a change, then say **"interview me"**. It also triggers on "grill me on this", "review my decisions", and "score my work". The skill reads your current diff (branch vs. base, falling back to staged changes or your last commit), asks for a one-line "what were you trying to do," then starts asking questions — one at a time. When it's done, it prints a scorecard to the terminal and saves a Markdown + HTML copy outside your repo.

## Demo

```
> interview me

I've reviewed your change. This interview runs at a Staff bar — I'll ask
about the decisions you made and why, not just what you did. There are 5
core questions and one stretch question at the end. Ready?

> ready

You pulled retry logic out of the shared HTTP client and put it directly
in the caller. Walk me through why — what did you weigh against leaving
it in the client, or putting it in some kind of middleware layer instead?

> The client was retrying on every call regardless of whether the
> operation was safe to retry, and that caused duplicate writes on a
> couple of endpoints. Moving it to the caller means each call site
> decides whether retrying makes sense.

What made you choose "retry decision at the call site" over, say, a
decorator or a per-endpoint config flag on the client that says
"retryable: true/false"?

> A config flag would've worked too, honestly — I just didn't want to
> touch the shared client's public API under time pressure. The
> call-site approach was more mechanical: smaller diff, easier to review.

[... three more questions, one bonus question, then:]

================================================================
  Interview — feature/retry-in-caller — 2026-07-07
  Bar level: Staff · Overall: 3.0 / 4
================================================================
  Technical judgment ........... 3/4  Strong
  Trade-off awareness .......... 2/4  Developing
  Risk & edge-case thinking .... 4/4  Exceptional
  Communication/clarity ........ 3/4  Strong
  Business/context alignment ... 3/4  Strong

  Verdict: Solid Senior, leaning Staff. Risk instincts are already at
  the Staff bar — the double-submit catch was unprompted. Trade-off
  framing is the gap: the first answer described the change, not the
  alternatives it was weighed against.

  Saved to: ~/.interview-me/my-repo/2026-07-07-feature-retry-in-caller.md
  Saved to: ~/.interview-me/my-repo/2026-07-07-feature-retry-in-caller.html
================================================================
```

See a full worked example, including the complete transcript, in [`interview-me-senior/examples/sample-scorecard.md`](interview-me-senior/examples/sample-scorecard.md).

## Config

Everything below lives in a single commented block at the top of `interview-me-senior/SKILL.md`. Edit it in place to retune the interview — no logic to touch.

| Key | Default | Purpose |
| --- | --- | --- |
| `bar_level` | `staff` | `mid` / `senior` / `staff` / `principal` — shifts framing + rubric expectations |
| `dimensions` | Technical judgment, Trade-off awareness, Risk & edge-case thinking, Communication/clarity, Business/context alignment | add / remove / reweight |
| `core_questions` | `5` | target count of core questions |
| `probe_depth` | `1` | max follow-ups per question |
| `bonus_question` | `true` | include the stretch question |
| `bonus_offset` | `+1` | levels above `bar_level` for the bonus |
| `tone` | `tough_but_fair` | `warm` / `tough_but_fair` / `brutal` |
| `save_dir` | `~/.interview-me/<repo-name>/` | where scorecards are written — out of repo by default (see Artifact safety) |
| `context_sources` | `diff, work_item, description` | which inputs to use |
| `include_transcript` | `true` | write the full Q/A transcript to the scorecard |
| `include_diff_snippets` | `false` | quote code from the diff in saved artifacts |
| `redact_secrets` | `true` | scrub secrets/tokens/URLs from anything written to disk |
| `auto_open` | `false` | auto-open the HTML report in a browser |

## Artifact safety

This skill runs inside other people's repos and produces two things people are understandably careful about: your source code, quoted in context, and a personal performance score. Both are kept out of your repo unless you explicitly move them in.

- **Out-of-repo by default.** `save_dir` defaults to `~/.interview-me/<repo-name>/`, a per-repo folder under your home directory. Nothing writes into the project you're working in unless you override `save_dir` yourself — and if you do point it inside the repo, the skill checks that the target is gitignored (adding it if it isn't) and warns you before writing if the path is already tracked.
- **`include_transcript`** (default on) controls whether the saved scorecard includes the full question-and-answer log. Turn it off if you only want the scores and verdict.
- **`include_diff_snippets`** (default off) controls whether code from your diff is quoted in the saved Markdown/HTML. With this off, decisions are described in prose — file and function names, not code blocks.
- **`redact_secrets`** (default on) scrubs anything that looks like a secret, token, or URL from anything written to disk before it's saved.
- **`auto_open`** (default off) means the skill prints the path to the HTML report and lets you open it yourself, rather than launching a browser for you.

With the defaults, a saved scorecard is just scores, verdict, and a reading list — nothing proprietary, nothing that reveals your repo, and nothing that ends up in a commit by accident.

## License

MIT — see [LICENSE](LICENSE).
