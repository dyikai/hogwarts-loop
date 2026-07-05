# The Loop — process and data handoff

How the orchestrator (**Albus** — the strategist who plans and dispatches, i.e.
you/the main agent), the implementer (**Harry** — `implementer.md`), and the
reviewer (**Severus** and the specialist bench — `reviewer.md`) fit together, and
— the important part — exactly what data crosses each edge between them. If a
payload isn't specified here, it tends to get dropped; the handoffs are where a
loosely-run loop leaks.

## The state machine

```
                          ┌──────────── PLAN — Albus (once) ────────────┐
                          │ read source of truth → decompose into       │
                          │ checkpoints → write living plan doc →        │
                          │ define each checkpoint's gate                │
                          └──────────────────────┬──────────────────────┘
                                                 │
        ┌────────────────── PER CHECKPOINT (in dependency order) ───────────────────┐
        │                                                                            │
        │   ALBUS (orchestrator) ─[A: implementer brief]─▶ HARRY (implementer)      │
        │        ▲                                      │                            │
        │        │                              builds + runs gates                  │
        │        └──────[B: delivery report]───────────┘                            │
        │        │                                                                   │
        │   orchestrator ──[C: review brief]──▶ SEVERUS (reviewer)                   │
        │        ▲                                 │                                 │
        │        │                        verifies + reruns gates + probes           │
        │        └────────[D: review report]──────┘                                 │
        │        │                                                                   │
        │   verdict?                                                                 │
        │     ├── REQUEST-CHANGES ──[E: findings relay]──▶ HARRY                     │
        │     │        ▲                                       │                     │
        │     │        └──────[F: fix report]─────────────────┘                     │
        │     │        └──────▶ back to [C] (Severus re-reviews)                       │
        │     │                                                                      │
        │     └── APPROVE ──▶ orchestrator commits (code + plan update) +            │
        │                     folds P3s/memos forward + advances to next CP          │
        └────────────────────────────────────────────────────────────────────────────┘
                                                 │
                          ┌──────────────────────┴──────────────────────┐
                          │ CLOSEOUT: acceptance mapping · reconcile      │
                          │ deviations into spec · closeout doc ·         │
                          │ name residual/manual-only risk explicitly     │
                          └──────────────────────────────────────────────┘
```

A checkpoint exits (edge → APPROVE) only when Severus approves **and** the gate is
green. Both, every time.

## The handoff payloads

Each lettered edge above carries a specific payload. Treat these as the contract
between roles.

### A — orchestrator → Harry: *implementer brief*
- Checkpoint scope (this checkpoint's deliverables only).
- Exact spec section references (the standard to build to).
- Gate commands (the done-condition).
- Hard constraints (frozen surfaces, layering/dependency rules, etc.).
- **Carried memos**: constraints Severus flagged in earlier checkpoints that this
  one must honor.
Full template in `implementer.md`.

### B — Harry → orchestrator: *delivery report*
- What was delivered, against the brief's checklist.
- Full gate results (real command output, including any ugly parts).
- Every deviation: ID + what changed + why.
- Files changed / added.
- Self-flagged risk spots ("review this hardest").
- Confirmation: not committed, stayed in scope.

### C — orchestrator → Severus: *review brief*
- How to see the diff + the baseline commit (only the unpassed changes).
- Spec sections to check against.
- Harry's declared deviation list (for rulings).
- Focus areas, highest-risk-first.
Full template in `reviewer.md`.

### D — Severus → orchestrator: *review report*
- Verdict: APPROVE or REQUEST-CHANGES.
- Findings, each [P1 / P2 / P3] + file:line + concrete failure scenario +
  suggested fix.
- A ruling on each declared deviation (accept/reject + reason).
- Gate rerun results — and, honestly, what was only static-analyzed.
- On APPROVE: downstream memos for later checkpoints.

### E — orchestrator → Harry: *findings relay*
- The P1/P2 findings to fix (P3s are recorded, not blocking).
- Which deviations were rejected and must change.
- Any orchestrator decision on contested points.

### F — Harry → orchestrator: *fix report*
- Per-finding: what the fix was.
- Reran gates (whole gate, not just the touched module).
- New tests added to lock each bug against recurrence.
Then loop back to edge C for Severus's re-review.

## What the orchestrator does at each verdict

- **REQUEST-CHANGES:** relay the P1/P2 findings (edge E) → Harry fixes (edge F) →
  Severus re-reviews (edge C). Never advance with an open P1/P2.
- **APPROVE:** commit the checkpoint as one traceable unit (the code **and** the
  plan-doc status update in the same commit). Stamp the checkpoint ✅ with date +
  commit hash in the plan. Then fold forward — see below.

## Carry-forward: how data survives between checkpoints

Two things must persist across checkpoint boundaries or the loop leaks knowledge:

- **Deviation registry** (in the plan doc). Every accepted deviation, with ID,
  reason, and scope. This is the honest record of "what we built vs. what the doc
  said," and it's what you reconcile into the spec/contract at closeout.
- **Carry-forward memos** (in the plan doc, written into the *target* checkpoint's
  section). When Severus's APPROVE report (edge D) contains a downstream memo — a
  constraint the next stage must honor, a limitation to work around, a test the
  finale must add — the orchestrator writes it *now* into that future
  checkpoint's brief-to-be. A reviewer who has read the whole run will often
  predict a hazard several checkpoints early; capture the prediction or you'll
  rediscover it the hard way.

If it isn't written into the plan doc, it didn't happen. The living doc is the
loop's memory between stages.

## Closeout (after the last checkpoint)

- Compile the acceptance mapping: every requirement ↔ how it was verified
  (automated test / headless assertion / **manual or user acceptance** for
  anything the environment couldn't execute — stated as such, never faked).
- Reconcile the deviation registry back into the spec/contract as errata.
- Write the closeout doc: acceptance check-off, known limitations (recorded, not
  hidden), items explicitly deferred to a future milestone.
- The residual risk the automated nets can't catch is handled by *naming it
  explicitly*, not by pretending it's covered.

## Anti-patterns (the ways the loop rots)

- **The orchestrator writes code.** No independent judge left; you're reviewing
  your own work by proxy.
- **Same agent implements and reviews.** Collapses the one property that makes
  this work.
- **Severus trusts Harry's gate report.** Rerun it, always.
- **Advancing with open P1/P2.** The whole point is not shipping the known bug.
- **Faking verification you couldn't run.** State the boundary; route to manual.
- **Losing a memo between checkpoints.** If it's not in the plan doc, it's gone.
- **Checkpoints too big to review deeply.** If Severus can't hold it in his head,
  split finer.
