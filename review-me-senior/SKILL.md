---
name: review-me-senior
description: A staff-level review copilot for a branch or PR. Reads the diff, gives you a ranked map of review points (correctness, risk, design, tests, security, nits) each with the reasoning and a draftable comment in your voice, then lets you drill into the ones that matter. It does not score you and does not post anything — it helps you understand the code and write the review yourself, and it learns your project's conventions over time. Use when the user says "review me", "review this branch", "review this PR", "help me review", or "help me write a review". (For being interviewed and scored on your own decisions, use interview-me-senior instead.)
---

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
