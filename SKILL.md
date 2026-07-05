---
name: hogwarts-loop
description: >-
  Run a full plan→build→review→commit loop for a large engineering effort. Albus
  (the Planner) reads the spec/roadmap and decomposes it into a checkpoint plan;
  each checkpoint is then BUILT by an implementer agent (Harry) and independently
  reviewed by a separate reviewer agent (Severus + security/perf/concurrency
  specialists), looping fix→re-review until it passes before Albus commits it and
  advances. Use it to plan and EXECUTE a big build where correctness matters and
  the blast radius is high: milestone/phase work, large multi-file refactors,
  framework/engine internals. Trigger from the planning stage onward — when the
  user wants to plan out a milestone or large change and then build it under
  review, wants separate implementer and reviewer agents, says "plan this milestone
  and build it in stages with review" or "set up Harry to implement and Severus to
  review", or hands you a spec/roadmap/design doc to plan and build end-to-end. NOT
  for a one-off PR/diff review or authoring the design spec itself.
---

## Hogwarts Loop — plan → build → review → commit

A discipline for taking a large engineering effort all the way from a plan to
committed, reviewed code — as a sequence of small, independently-verified
checkpoints. It starts with **planning**: **Albus** (the Planner) reads the
source of truth and decomposes it into a checkpoint plan. Then the engine is a
**builder/reviewer separation**: an *implementer* — **Harry** — writes each
checkpoint's code, an *independent reviewer* — **Severus** (plus a bench of
specialists) — tries to break it, and they loop until it's clean; then Albus
commits it and moves to the next. You (the main agent) *are* Albus — you plan,
dispatch, judge, and commit, but you never write the feature code yourself. (The
cast is a mnemonic: Albus/Dumbledore plans and orchestrates from above, Harry does
the work, Severus scrutinizes it — use the names when dispatching so the agents
stay distinct and accumulate their own context.)

### Why this pattern earns its cost

Running two agents plus a loop is more expensive than just implementing. It pays
off precisely when a mistake is expensive, because it defends against the single
biggest failure mode in real work: **an author cannot see the bug in code they
just convinced themselves is correct.** The implementer's own tests pass because
they test what the implementer *thought about*. An independent reviewer, holding
an adversarial mindset and re-deriving the spec, walks the paths the author
skipped. That gap is where the worst bugs live — the ones that ship green.

So the value is not "more review." It is *structural independence* (a reviewer
who is not the author) + *bounded blast radius* (small checkpoints) + *a
full-workspace safety net rerun on every change* (gates). Keep all three or the
pattern degrades into theater.

### When to use it, and when not to

Use it for: planning *and* building milestone/phase work against a design or spec,
large refactors, engine/framework internals, anything spanning many files or
multiple sessions, anything where a silent regression would be costly. Reach for
it at the *planning* stage — when the user wants to lay out a big piece of work
and then actually build it — not just once code is being written.

Don't use it for: small changes you can do and verify in one pass, exploratory
spikes where the goal is learning not shipping, tasks with no objective gate
(you'll spin without a way to say "this checkpoint passed"), a one-off review of
an existing PR, or authoring the design/requirements spec itself (that's the input
to this loop, not its job). The overhead only pays back when correctness is
load-bearing and there is real code to ship.

### The three roles at a glance

- **Planner / orchestrator — Albus (you).** *First* read the source of truth and
  decompose it into a checkpoint plan (the living plan doc). *Then* drive the loop:
  dispatch each stage, judge pass/fail, commit on pass, carry memos and deviations
  forward. You are the only one who commits, and you never write the feature code —
  that separation keeps your judgment uncontaminated by authorship.
- **Implementer — Harry.** Builds one checkpoint to spec, runs the gates, reports
  what he did and every deviation, does not commit, stays in scope.
- **Reviewer — Severus** (+ the specialist bench: Alastor/security,
  Filius/performance, Minerva/concurrency). Independently verifies against the
  spec, reruns the gates himself, probes adversarially, returns APPROVE or
  REQUEST-CHANGES with categorized findings, rules but never fixes.

Keep Harry and Severus as **distinct agents with persistent context** so each
accumulates project memory across checkpoints (Severus remembers what he flagged;
Harry remembers his own decisions); resume the same two each checkpoint rather
than spawning fresh ones. Only spawn these agents when the user has asked for
this workflow — it's opt-in.

### Where the detail lives

This file is the overview. The operational detail is split so each piece can be
read — or handed to the relevant agent — on its own:

- **`references/planner.md`** — Albus's full responsibilities: how to plan
  (decompose into checkpoints, define gates, write the plan doc), then dispatch,
  judge, commit, and keep the run honest. Albus is *you*, the main agent — not a
  spawned agent — so this is your own playbook, start to finish.
- **`references/implementer.md`** — Harry's full responsibilities and the dispatch
  brief you send him. Read/hand this when briefing the builder.
- **`references/reviewer.md`** — Severus's full responsibilities, the P1/P2/P3
  severity scheme, adversarial technique, his dispatch brief, and **reviewer
  variants**: the professor bench (Severus=correctness lead, Alastor=security,
  Filius=performance, Minerva=concurrency), review panels, and wiring in built-in
  tooling (`/code-review` / `/security-review` / ultrareview). Read/hand this when
  briefing the reviewer.
- **`references/loop.md`** — the process: the per-checkpoint state machine and,
  most importantly, the **data handoff** between Albus (orchestrator), Harry, and
  Severus (what payload crosses each edge). Read this to run the loop itself.
- **`references/example.md`** — a concrete worked run: checkpoint-sizing
  heuristics, a real milestone decomposition, and one full review round-trip
  (the bug the loop caught that unit tests missed). Read this if the abstract
  rules feel too abstract — it's the fastest way to see the pattern in motion.
- **`assets/plan-template.md`** — skeleton for the living plan doc (checkpoint
  map, gates, acceptance mapping, deviation registry, carry-forward memos).

### Adapting to the project

Language- and framework-agnostic. Bind these knobs to the project at hand:
- **Gate commands** — whatever "green" means here (e.g. `cargo test --workspace`
  + clippy + fmt; or `pytest` + `ruff` + `mypy`; or a build + e2e suite).
- **Source of truth** — the design doc, RFC, roadmap, or ticket the review cites.
- **Communication language** — match the user (e.g. reply in Chinese if that's
  the project norm) while keeping code/comments in the repo's convention.
- **Repo tooling** — if the project has a code-intelligence layer (a knowledge
  graph, a review MCP, etc.), tell Severus to prefer it over blind file scanning;
  it's faster and gives structural context.
- **"Document-first" projects** — if the convention is design-doc-before-code,
  the plan doc slots in as the execution-tracking layer between design and
  closeout.
