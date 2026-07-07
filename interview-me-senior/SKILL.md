---
name: interview-me-senior
description: Runs a staff-level engineering interview on a piece of work you just finished — reads the git diff (plus any work-item context you provide), grills you on why you made the decisions you did versus the alternatives, probes weak answers, and scores you against a configurable rubric. Use when the user says "interview me", "grill me on this", "review/defend my decisions", or "score my work" after completing a change.
---

```
# CONFIG — edit these defaults to retune the interview.
bar_level: staff            # mid | senior | staff | principal
core_questions: 5
probe_depth: 1              # max follow-ups per question
bonus_question: true
bonus_offset: +1            # levels above bar_level for the stretch question
tone: tough_but_fair        # warm | tough_but_fair | brutal
save_dir: ~/.interview-me/<repo-name>/   # OUT OF REPO by default (see Artifact Safety)
context_sources: diff, work_item, description
include_transcript: true
include_diff_snippets: false
redact_secrets: true
auto_open: false
# dimensions (score each 1-4): Technical judgment, Trade-off awareness,
#   Risk & edge-case thinking, Communication/clarity, Business/context alignment
```
