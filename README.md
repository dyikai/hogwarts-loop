> 🌐 **Language:** **English (current)** · [中文](README-ZH.md)

# Hogwarts Loop

> A "plan → build → review → commit" discipline that takes a large engineering effort all the way from a plan to **committed, reviewed** code — by decomposing it into a sequence of **small, independently-verifiable** checkpoints.

This is a Claude Code **Skill**. It is not a runnable program but a **working method and role protocol** for an orchestrating agent: when you take on an effort where a mistake is expensive and the blast radius is high, use it to organize several agents into a division of labor that checks itself.

---

## Table of contents

- [In one sentence](#in-one-sentence)
- [Why it earns its cost](#why-it-earns-its-cost)
- [The three roles](#the-three-roles)
- [When to use it, and when not to](#when-to-use-it-and-when-not-to)
- [The mechanism: the core loop](#the-mechanism-the-core-loop)
- [The handoffs: what crosses each edge](#the-handoffs-what-crosses-each-edge)
- [Memory between checkpoints: the plan doc](#memory-between-checkpoints-the-plan-doc)
- [Reviewer variants: the professor bench](#reviewer-variants-the-professor-bench)
- [Closeout](#closeout)
- [Anti-patterns: how the loop rots](#anti-patterns-how-the-loop-rots)
- [Adapting to your project](#adapting-to-your-project)
- [File structure](#file-structure)

---

## In one sentence

Take a large, vague effort and decompose it into small, independently-verifiable **checkpoints**; each checkpoint is **built by an implementer agent (Harry)**, then an **independent reviewer agent (Severus, plus a bench of specialists)** tries to break it, and the two loop "fix → re-review" until it is clean; then the **orchestrator (Albus — you / the main agent)** commits it and advances to the next checkpoint.

The core engine is **builder / reviewer separation**: the agent who writes the code and the agent who hunts for its bugs are never the same one.

---

## Why it earns its cost

Running two agents plus a loop is more expensive than just implementing it yourself. It pays off precisely when **a mistake is expensive**, because it defends against the single biggest failure mode in real work:

> **An author cannot see the bug in code they just convinced themselves is correct.**

The implementer's own tests pass because they only test what the implementer **thought about**. An independent reviewer, holding an adversarial mindset and re-deriving correctness from the spec, walks the paths the author skipped. **That gap is where the worst bugs live — the ones that ship green.**

So the value is not "more review." It is the combination of three things — keep all three or the pattern degrades into theater:

1. **Structural independence** — the reviewer is not the author.
2. **Bounded blast radius** — checkpoints are small.
3. **A full-workspace safety net rerun on every change** — the gates.

---

## The three roles

The cast is a mnemonic: Dumbledore (Albus) plans and deploys everyone from above; Harry does the work; Severus scrutinizes it. **Use the names when dispatching** so each agent stays distinct and accumulates its own context.

| Role | Name | Who it is | Responsibilities | Never does |
|------|------|-----------|------------------|------------|
| **Planner / orchestrator** | **Albus** | **You** (the main agent) — *not* a spawned agent | Read the source of truth → decompose into a checkpoint plan → dispatch, judge, commit, carry memos and deviations forward | **Never writes feature code** — the moment the planner also implements, there is no independent judge left |
| **Implementer** | **Harry** | One persistent agent across the whole run | Build **one** checkpoint to spec, run the gates, report honestly on everything done and every deviation, stay in scope | Does not commit, does not exceed scope |
| **Reviewer** | **Severus** (+ specialist bench: Alastor / Filius / Minerva) | One persistent agent across the whole run | Independently verify against the spec, **rerun the gates himself**, probe adversarially to break it, return APPROVE or REQUEST-CHANGES with categorized findings | **Only rules, never fixes** (Harry fixes) |

> **Keep Harry and Severus as two persistent, context-carrying, independent agents** so they accumulate project memory across checkpoints (Severus remembers what he flagged; Harry remembers his own decisions). **Reuse the same two agents** each checkpoint rather than spawning fresh ones. Only spawn them when the user has asked for this workflow — it is **opt-in**.

---

## When to use it, and when not to

### ✅ Use it for

- **Planning and building** milestone / phase work against a design or spec
- Large refactors, engine / framework internals
- Anything spanning many files or multiple sessions
- Anything where a single silent regression would be costly

> Reach for it at the **planning** stage — when the user wants to lay out a big piece of work **and then actually build it** — not once the code is already being written.

### ❌ Don't use it for

- Small changes you can do and verify in one pass
- Exploratory spikes where the goal is learning, not shipping
- Tasks with **no objective gate** (no way to say "this checkpoint passed" — you'll just spin)
- A one-off review of an existing PR
- Authoring the design / requirements spec itself (that is the **input** to this loop, not its job)

---

## The mechanism: the core loop

The whole mechanism is a state machine: one **plan** up front, then a small **"build → review → fix"** loop per checkpoint, then a **closeout** once all pass.

```
                          ┌──────────── PLAN — Albus (once) ────────────┐
                          │ read source of truth → decompose into        │
                          │ checkpoints → write living plan doc →         │
                          │ define each checkpoint's gate                 │
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

**A checkpoint passes only when both conditions hold: Severus APPROVES** and **the gate is green.** Both, every time.

### The three stages, unpacked

**1. Plan (Albus does this once — and it is the highest-leverage decision in the run)**

- **Read the source of truth.** The design / spec / roadmap is the contract everything downstream cites. If it doesn't exist, that's a red flag — this loop needs an objective standard to review against.
- **Decompose into checkpoints.** Slice by **independently-verifiable capability** (not by file, not by layer); put a foundation / infra checkpoint first; lay a regression baseline before a risky rewrite; concentrate breaking / risky work into its own checkpoint; order by dependency; end with a closeout checkpoint. **A typical milestone lands around 5–9 checkpoints.**
- **Define each checkpoint's gate up front** — make "passed" an objective set of commands + assertions, not a vibe.
- **Write the living plan doc** (see `assets/plan-template.md`) and **commit it first** — it is the spine of the run and its memory between stages.

**2. The per-checkpoint loop**

1. Dispatch the **implementer brief** to Harry (edge A).
2. Receive Harry's **delivery report** (edge B).
3. Dispatch the **review brief** to Severus / a specialist (edge C).
4. Receive the **review report** (edge D).
5. If **REQUEST-CHANGES** → relay the blocking findings (edge E) → Harry fixes (edge F) → re-review (back to edge C). Loop until clean. **Never advance with an open P1/P2.**
6. If **APPROVE and the gate is green** → commit the checkpoint as **one traceable unit** (the code **and** the plan-doc status update in the same commit) → stamp the checkpoint ✅ + date + commit hash → fold memos forward → advance.

**3. Closeout** (see [Closeout](#closeout) below).

---

## The handoffs: what crosses each edge

**The handoffs are where a loosely-run loop leaks.** Each lettered edge above carries a specific payload. Treat them as the contract between roles — **if a payload isn't specified, it tends to get dropped.**

| Edge | Direction | Payload | Key point |
|------|-----------|---------|-----------|
| **A** | orchestrator → Harry: **implementer brief** | this checkpoint's scope, exact spec-section references, gate commands, hard constraints, **memos carried from earlier checkpoints** | Naming the exact spec sections matters more than restating requirements — it keeps implementer and reviewer citing the same standard |
| **B** | Harry → orchestrator: **delivery report** | what was delivered (against the brief's checklist), **full real gate output (including the ugly parts)**, every deviation (ID + what + why), files changed, self-flagged high-risk spots, confirmation "not committed / stayed in scope" | Self-flagging "review this hardest" makes you look *better* — the reviewer disproportionately finds real bugs exactly where the author already sensed unease |
| **C** | orchestrator → Severus: **review brief** | how to see the diff + the baseline commit, spec sections to check against, Harry's declared deviation list, focus areas highest-risk-first | Only the unpassed changes |
| **D** | Severus → orchestrator: **review report** | verdict (APPROVE / REQUEST-CHANGES), each finding ([P1/P2/P3] + file:line + concrete failure scenario + suggested fix), a ruling on each deviation, gate-rerun results (honestly noting what was only static-analyzed), and on APPROVE the **downstream memos** for later checkpoints | — |
| **E** | orchestrator → Harry: **findings relay** | the P1/P2 findings to fix (P3s recorded, not blocking), which deviations were rejected and must change, the orchestrator's decision on contested points | — |
| **F** | Harry → orchestrator: **fix report** | per-finding fix, **rerun the whole gate** (not just the touched module), new tests added to lock each bug against recurrence | Then back to edge C for Severus's re-review |

### Severity scheme (P1 / P2 / P3)

Categorizing findings is what keeps the loop from either stalling on nits or shipping real bugs:

- **P1 — blocking, correctness.** A real defect. State a **concrete failure scenario** (specific inputs / state → wrong output or crash), not a general worry. Must be fixed to pass.
- **P2 — blocking, should-fix.** A spec violation, a missing required test, a contract mismatch, a tautological assertion that verifies nothing. Fixed to pass unless explicitly waived.
- **P3 — non-blocking.** Style, doc wording, future-proofing, an observation for a later checkpoint. Record it and carry it forward; do not block on it.

**A checkpoint passes when P1 and P2 are cleared to zero** and the gate is green. P3s ride along in the plan doc's carry-forward memos.

---

## Memory between checkpoints: the plan doc

Two things must persist across checkpoint boundaries or the loop leaks knowledge. Both live in the **plan doc**:

- **Deviation registry.** Every accepted deviation, with ID, reason, and scope. This is the honest record of "what we built vs. what the doc said," and it's what you reconcile into the spec / contract at closeout.
- **Carry-forward memos.** When Severus's APPROVE report (edge D) contains a downstream memo — a constraint the next stage must honor, a limitation to work around, a test the finale must add — the orchestrator writes it **now** into that future checkpoint's brief-to-be. A reviewer who has read the whole run will often predict a hazard several checkpoints early; capture the prediction or you'll rediscover it the hard way.

> **If it isn't written into the plan doc, it didn't happen.** The living doc is the loop's memory between stages.

---

## Reviewer variants: the professor bench

The reviewer is a **role**, not a fixed identity. The only invariant is **independence: the reviewer must not be the author.** As long as that holds, you can vary who reviews, how many review, and what tools they use. **Heterogeneous reviewers are a feature** — different reviewers have different blind spots, so a panel widens coverage rather than duplicating it.

### The specialist bench

Severus is the default lead (correctness / adversarial). For a checkpoint whose risk matches a specific lens, add (or swap in) a specialist. Each is named after a Hogwarts professor whose subject / character maps to the lens, so the name itself is the mnemonic (Albus / Dumbledore sits *above* the bench as the orchestrator, not on it):

| Reviewer | Lens | Watches for |
|----------|------|-------------|
| **Severus** (Snape, Potions) | correctness / adversarial (default lead) | the paths tests don't cover; the bug the author can't see — nothing escapes him |
| **Alastor** (Mad-Eye Moody, Auror) | security | injection, authz, secrets, unsafe deserialization, untrusted input, attack surface — "constant vigilance" |
| **Filius** (Flitwick, Charms; dueling champion) | performance | hot paths, allocations, complexity, N+1, wasted work — fast and elegant |
| **Minerva** (McGonagall, Transfiguration) | concurrency | races, deadlocks, reentrancy, state-transition ordering — order and discipline |

Highest value on the checkpoint whose risk matches the lens (e.g. Alastor on one that touches auth or parses untrusted input; Minerva on one that adds threading or a scheduler; Filius on a hot path). The bench is extensible — add another professor for another lens as needed.

### Other forms

- **Review panel (multiple reviewers, one checkpoint).** Dispatch 2–3 professors on the same diff; each returns its own report (edge D), and the orchestrator de-duplicates and merges before relaying to Harry. **Reserve this for the riskiest checkpoints.**
- **Built-in review tooling as a reviewer or an extra gate.** If the host offers review commands / skills — a working-diff bug review (`/code-review`, with higher effort tiers), a security pass (`/security-review`), a graph-backed structural review — run one as part of a checkpoint's gate, or as an independent second opinion beside Severus.
- **Heavyweight cloud review (ultrareview).** A multi-agent deep review (`/code-review ultra`) is the strongest gate for a make-or-break checkpoint. **Caveat: it is user-triggered and billed — the orchestrator cannot launch it.** So the human runs it on a checkpoint they want extra-hardened, and the orchestrator folds its findings into the loop like any other review report.

> Whatever the mix, preserve the invariant and the discipline: **independent from the author, categorized findings, gates rerun, boundaries stated honestly.**

---

## Closeout

(after the last checkpoint)

- **Compile the acceptance mapping:** every requirement ↔ how it was verified (automated test / headless assertion / **manual or user acceptance** for anything the environment couldn't execute — stated as such, never faked).
- **Reconcile the deviation registry back into the spec / contract** as errata.
- **Write the closeout doc:** acceptance check-off, known limitations (recorded, not hidden), items explicitly deferred to a future milestone.
- The residual risk the automated nets can't catch is handled by **naming it explicitly**, not by pretending it's covered.

---

## Anti-patterns: how the loop rots

- **The orchestrator writes code.** No independent judge left — you're reviewing your own work by proxy.
- **Same agent implements and reviews.** Collapses the one property that makes this work.
- **Severus trusts Harry's gate report.** Rerun it yourself, always.
- **Advancing with open P1/P2.** The whole point is not shipping the known bug.
- **Faking verification you couldn't run.** State the boundary; route to manual acceptance.
- **Losing a memo between checkpoints.** If it's not in the plan doc, it's gone.
- **Checkpoints too big to review deeply.** If Severus can't hold it in his head, split finer.

---

## Adapting to your project

The method is language- and framework-agnostic. Bind these knobs to the project at hand:

- **Gate commands** — whatever "green" means here (e.g. `cargo test --workspace` + clippy + fmt; or `pytest` + `ruff` + `mypy`; or a build + e2e suite).
- **Source of truth** — the design doc, RFC, roadmap, or ticket the review cites.
- **Communication language** — match the user (e.g. reply in Chinese if that's the project norm) while keeping code / comments in the repo's convention.
- **Repo tooling** — if the project has a code-intelligence layer (a knowledge graph, a review MCP, etc.), tell Severus to prefer it over blind file scanning; it's faster and gives structural context.
- **"Document-first" projects** — if the convention is design-doc-before-code, the plan doc slots in as the execution-tracking layer between design and closeout.

---

## File structure

`SKILL.md` is the overview; the operational detail is split so each piece can be read — or handed to the relevant agent — on its own.

| File | Contents | When to read it |
|------|----------|-----------------|
| [`SKILL.md`](SKILL.md) | Skill overview: the pattern, the three roles, when to use / not use, navigation to the files | The entry point |
| [`references/planner.md`](references/planner.md) | **Albus's** full responsibilities: how to plan, dispatch, judge, commit, keep the run honest. Albus is *you* — so this is your own playbook | While you orchestrate the whole run |
| [`references/implementer.md`](references/implementer.md) | **Harry's** full responsibilities + the **implementer-brief template** you send him | When briefing the builder / hand it to him |
| [`references/reviewer.md`](references/reviewer.md) | **Severus's** full responsibilities, the P1/P2/P3 scheme, adversarial technique, the review-brief template, and the **reviewer variants** (professor bench, review panels, wiring in built-in tooling) | When briefing the reviewer / hand it to him |
| [`references/loop.md`](references/loop.md) | The process itself: the per-checkpoint state machine and — most importantly — the **data handoffs** between the three roles (what payload crosses each edge) | When you run the loop |
| [`references/example.md`](references/example.md) | A concrete worked run: checkpoint-sizing heuristics, a real milestone decomposition, and one full review round-trip (the bug the loop caught that unit tests missed) | When the abstract rules feel too abstract — the fastest way to see the pattern in motion |
| [`assets/plan-template.md`](assets/plan-template.md) | Skeleton for the living plan doc (checkpoint map, gates, acceptance mapping, deviation registry, carry-forward memos) | When you start planning and need to land the plan doc |

---

## The one thing to remember

> Green tests mean "what the author thought to test passed," not "the code is correct." The bugs that ship are the ones on the paths the author didn't think about — the idle case, the empty case, the reentrant call, the concurrent window. An independent reviewer whose job is to **find** those paths is the mechanism that catches them. Everything else in this skill exists to support that one move.
