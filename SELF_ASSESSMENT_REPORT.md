# Self-assessment — IA#1

Submitted by: 23120205 — Ngô Gia An

Total I claim: 70 / 100

| Criterion | Max | I claim | Evidence |
|---|---:|---:|---|
| Contract | 25 | 25 | `spec.md` §Contract defines three endpoints, complete request/response examples, Socket.IO payloads, idempotent replay, status codes, and named `ApiError`; §Data defines fields, states, uniqueness, and monetary invariants. |
| Acceptance criteria | 25 | 25 | `spec.md` AC1–AC10 contain concrete inputs and observable results. They cover exact and rounded arithmetic, empty state, payment failure, simultaneous requests, duplicates, pending reconciliation, permissions, limits, completion, and order locking. |
| Edge cases and non-goals | 20 | 20 | `spec.md` §Errors explicitly covers all five thin places: empty state, partial failure, permissions, concurrency and duplicates, and limits. §Out of scope is explicit. |
| Implementability | 20 | 0 | Not assessable before the 1 October swap: no implementer question list exists yet. This row will be rescored from `questions.md`. |
| Revision | 10 | 0 | No revision is due before the swap. This row will be rescored after every received question is addressed visibly in `spec-revised.md`. |

## What I did not manage

The implementation handoff has not happened yet, so I cannot provide evidence for implementability or revision. I have deliberately not invented a question list or a revised specification. The current total therefore measures only the 70 points that can be evidenced from the initial `spec.md`.

## What I would do differently

After the swap, I will classify each implementer question as behavioural or editorial before revising the specification. I will make each answer visible in `spec-revised.md`, then update this report and rename the submission archive so its filename matches the new total.
