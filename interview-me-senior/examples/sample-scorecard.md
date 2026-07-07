> Saved to ~/.interview-me/<repo>/ by default — not committed to your repo.

# Interview — feature/retry-in-caller — 2026-07-07

**Bar level:** Staff

## Scorecard

| Dimension | Score /4 | Anchor | Justification |
| --- | --- | --- | --- |
| Technical judgment | 3 | Strong | Correctly identified that retry belongs above the transport layer, but didn't spontaneously name the idempotency requirement it creates. |
| Trade-off awareness | 2 | Developing | Explained what moved and where, but only under a follow-up did the coupling-vs-duplication trade-off come out. |
| Risk & edge-case thinking | 4 | Exceptional | Volunteered the double-submit risk on non-idempotent POSTs and described a request-id de-dupe fix without being asked. |
| Communication/clarity | 3 | Strong | Answers were structured and easy to follow, if a little long on the first question. |
| Business/context alignment | 3 | Strong | Tied the change back to the on-call pain (retries hiding as silent duplicate charges) rather than describing it as a pure refactor. |

**Overall: 3.0 / 4**

## Verdict

Solid Senior, leaning Staff. Risk instincts are already at the Staff bar — the
double-submit catch was unprompted and exactly the kind of second-order thinking
that bar expects. Trade-off framing is the gap: the first answer described the
change, not the alternatives it was weighed against, and needed a nudge to get
there.

## What went well

- Spotted the idempotency hazard created by moving retries into the caller before being asked, and had a concrete fix (request-id de-dupe) ready.
- Grounded the change in a real production pain point (duplicate-charge tickets) rather than treating it as a stylistic refactor.
- Answers were direct and organized — no hedging, no padding to fill time.

## What to work on

- Lead with the alternatives considered, not just the choice made — "I put retry in the caller instead of a decorator/middleware because X" should be the opening sentence, not something extracted by a probe.
- Talk about reversibility explicitly: how expensive would it be to walk this back if a future caller needed shared retry policy across five call sites?

## Bonus/stretch result

Asked a Principal-level stretch question: "If every team in the org made this
same call independently, what breaks?" The answer named policy drift (each
caller picking its own backoff and retry count) but stopped there — it didn't
reach for an org-level answer like a shared retry-policy library or a lint rule
enforcing bounds, and didn't mention the operational cost of debugging five
different retry behaviors during an incident. Scored Developing on the stretch;
this is well beyond what's expected at the Staff bar and doesn't count against
the core score.

## Reading list

- **Gap: trade-off framing comes out only under a probe.** Read "A Philosophy of Software Design" ch. 4-6 (Ousterhout) on naming the alternatives you rejected, not just the one you kept.
- **Gap: reversibility isn't part of the default answer shape.** Read Cindy Sridharan's "Testing Microservices, the sane way" section on idempotent retries, and separately, the classic "reversible vs irreversible decisions" framing from Bezos's one-way/two-way-door letter.
- **Gap: retry policy consistency across callers.** Skim the resilience chapter of "Release It!" (Nygard) on bulkheads and backoff — specifically why centralizing retry policy (a shared client wrapper or sidecar) beats letting every caller reinvent it.

## Transcript

### Q1

**Question:** You pulled retry logic out of the shared HTTP client and put it
directly in the caller. Walk me through why — what did you weigh against
leaving it in the client, or putting it in some kind of middleware layer
instead?

**Your answer:** The HTTP client was retrying on every call regardless of
whether the operation was safe to retry, and that was causing duplicate writes
on a couple of endpoints. Moving it to the caller means each call site decides
whether retrying makes sense.

**Follow-up probe:** What made you choose "retry decision at the call site" over,
say, a decorator or a per-endpoint config flag on the client that says
"retryable: true/false"?

**Follow-up answer:** A config flag would've worked too, honestly, I just
didn't want to touch the shared client's public API under time pressure. The
call-site approach was more mechanical — smaller diff, easier to review.

**Score:** Trade-off awareness — 2/4 (Developing). Named what changed and the
immediate symptom it fixed, but the first pass didn't surface the alternatives;
the probe answer landed on "smaller diff" as the deciding factor rather than a
structural reason like blast radius or ownership, which is why it stayed at 2
rather than moving to 3.

**What a 4 would have said:** "I considered three options: keep retry
centralized behind a per-endpoint `retryable` flag, wrap calls in a decorator,
or move the decision to the caller. I picked the caller because retryability
is a property of the *operation's* idempotency, not the transport — and that
property lives with the code that knows whether it's a GET or a
non-idempotent POST. Centralizing it in the client would've meant the client
needing business knowledge it shouldn't have. It's also the most reversible
option: if we later want shared policy back, we can extract a helper without
changing every caller's contract."

### Q2

**Question:** Now that retry lives in the caller, what stops two different
callers from silently disagreeing about how many times to retry, or retrying
something that isn't actually safe to retry twice?

**Your answer:** Right now, nothing enforces that — it's up to whoever writes
the calling code to think it through. I did add a comment on the two endpoints
where retrying is dangerous, warning future callers not to retry blindly. I
also realized while making this change that a retried POST could double-charge
a customer if the first request actually succeeded but the response timed out,
so I added a request-id to de-dupe on the server side.

**Score:** Risk & edge-case thinking — 4/4 (Exceptional). This was volunteered,
not extracted by a probe: unprompted identification of the double-submit /
duplicate-charge risk plus a concrete, correctly-scoped fix (server-side
de-dupe via request-id) rather than a client-side band-aid. This is exactly
the second-order-effects thinking the Staff bar looks for.

**What a 4 would have said:** Effectively what was said. A 4 answer at this
depth might additionally flag that "a comment warning future callers" is a
weak enforcement mechanism long-term, and propose a lint rule or a typed
wrapper (e.g., a `RetryableRequest` marker type) so the safety property is
enforced by the compiler instead of relying on the next engineer reading a
comment — but that's a refinement on an already-strong answer, not a gap that
dragged the score down.
