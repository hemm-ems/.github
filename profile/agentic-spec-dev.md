# Codex-Orchestrated Spec-Driven Development

**Opus orchestrates and judges; Codex is the implementation army.** A long-running
Claude (Opus) session drives spec-driven development but writes very little code
itself — it briefs Codex agents to implement against specs, then verifies their
work as a skeptical *second* model. This is a writeup of the pattern, generic
enough to drop onto your own repo.

## What it does

The unit of work is a **spec**, not a chat. Use [Spec Kit](https://github.com/github/spec-kit)
for the structure: `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` give you
a `specs/NNN-feature/{spec.md,plan.md,tasks.md}` triplet with numbered
requirements and a dependency-ordered task list.

> **Spec Kit is the scaffold, not the verifier.** How you encode "done" against
> those tasks is **repo-specific** and the part you must bring yourself: a passing
> test tagged to the requirement, a checked box in `tasks.md`, a coverage gate that
> fails CI on untested requirements — pick what fits. Spec Kit gives you the shared
> structure; the *done-signal* and its enforcement are yours to wire. Treat
> requirement status as **derived from that signal, not from prose.**

The loop per task:

1. **Opus reads the spec/tasks** and decides what to delegate. It owns the plan,
   the judgement, and the thread — the scarce, irreplaceable context.
2. **Opus writes a concrete brief** — a small renderer that fills a fixed template
   and embeds `git status` + `git diff --stat` (*diffstat, never full diffs*). A
   good brief = exact file paths, line numbers, the precise change, the verification
   command, and an explicit out-of-scope list.
3. **Codex implements** in a `workspace-write` sandbox and **commits its own work**
   with strict explicit-path `git add <path>` (never `-A`/`.`). One Codex agent
   **per repo** — the sandbox is root-scoped, so a change spanning sibling repos
   needs a second agent rooted in the sibling.
4. **Cross-model review.** Opus reviews the diff / runs a review pass, and owns the
   heavyweight verification (your gate, integration/container tests) once per phase.
   Bulk reading (big diffs, long logs) is pushed to an **Explore subagent** so it
   never lands in the orchestrator's context.
5. **Hard 2-round cap.** If it isn't green after one implement + one fix round,
   Opus stops delegating and finishes inline. No endless ping-pong.

**Trust = moderate, and earned.** Codex commits and self-verifies unit tests; Opus
reviews the diff and confirms the gate, re-verifying deeply only for safety-touching
work. Don't assume this level — *upgrade to* it once a model has performed well
across real work on your repo. In practice the delegation failures tend to be
git-hygiene, not logic (a broad `git add` sweeping a build artifact into a commit —
hence the explicit-path rule). Calibrate trust from a model's actual track record on
your code, not from a vibe.

**When *not* to delegate.** The pattern has real edges:
- **Delegate implementation, not test *design*.** The nastiest churn is usually
  test-design (shared-session/integration state, races, environment coupling), not
  the feature code. Cross-session-stateful and safety-critical work stays inline (or
  gets a purpose-built debug hook first).
- **Keep verification with the orchestrator.** A sandbox often can't run
  network/integration suites, and you want *one* authoritative gate run per phase —
  so Opus owns the full gate, Codex owns unit tests only.
- **Respect the 2-round cap.** If round two isn't green, the task is usually
  under-specified or genuinely fiddly — finish it inline rather than briefing a third
  time.

> **Sidenote — a fresh codebase map.** A current `CODEBASE_MAP.md` makes a great
> shared context to spawn every Codex agent with (`--map`). You can regenerate it
> with the same orchestration trick: have Codex rewrite it in a detached run, let a
> deterministic verifier return one PASS/FAIL line, and never read the map into the
> session. Useful, but a support act — the spec loop above is the main event.

## Why

**1. Long-running Opus with clean context.**
The orchestration session is the bottleneck resource. Every large diff, log, or
file you read *into* it burns context and drags it toward compaction. Keep the bulk
in Codex and subagents and the session stays sharp across a whole multi-phase build.

**2. Token distribution across the smallest plans.**
Expensive Opus tokens buy orchestration and judgement only; the voluminous
implementation and log-reading run on Codex. On the *smallest* Claude and Codex
plans this spreads load over two budgets instead of detonating the Claude one —
heavy lifting billed where it's cheap, Opus spent only where it's irreplaceable.

**3. Cross-model verification beats self-review (the real win).**
LLM self-review is reliably mediocre: a model trusts its own output and waves it
through — a structural bias, not a prompt you can fix. Models do *not* extend that
trust to another model's work; they scrutinise it. So **Opus reviewing Codex** is a
genuinely skeptical check in a way Codex-checks-Codex or Opus-checks-Opus never is.
The output is trustworthy *because* the producer and the reviewer are different
models. (A deterministic gate is the strongest form of this: a checker that trusts
no one.)

## How to set up (recreate from scratch)

Brief on purpose — an LLM can fill in the code; the wiring and the discipline are
what matter.

**1. Codex CLI + plugins in Claude Code.**
- Install the Codex CLI and log in: `npm install -g @openai/codex` → `codex login`
  (`codex exec` must run headless).
- Add the marketplaces and install the plugins (in Claude Code, per marketplace:
  `/plugin marketplace add <repo>` then `/plugin install <name>@<marketplace>`):
  - `openai/codex-plugin-cc` → the **`codex`** plugin (Codex rescue/runtime skills).
  - `kingbootoshi/codex-orchestrator` → the **`codex-orchestrator`** skill *and* the
    `codex-agent` / `codex-bg` CLIs (installed under `~/.codex-orchestrator/bin` —
    add it to `PATH`). This is the "Opus orchestrates, Codex army" pipeline.
  - `kingbootoshi/cartographer` → the **`cartographer`** map skill (for the sidenote).

**2. Spec-driven scaffolding (Spec Kit).**
- Install Spec Kit (`github/spec-kit`) and use `/speckit.specify` → `.plan` →
  `.tasks` to author `specs/NNN-*/`.
- **Wire your own done-signal** on top (see the callout above): tag tests to
  requirement IDs, or enforce `tasks.md` checkboxes, or run a coverage gate in CI.
  This is the repo-specific glue Spec Kit deliberately leaves to you.
- Note: Spec Kit's templates may not match a bespoke requirement format you parse
  with tooling — if you have one, hand-author `spec.md` from an existing example and
  skip the full clarify/plan/tasks ceremony for small specs.

**3. The brief tooling + house rules.**
- A small `codex_brief` renderer: fixed template + embedded `git status` /
  `git diff --stat` (small by construction — no full diffs).
- Bake the non-negotiables into the template: touch only listed files; explicit-path
  `git add`; leave the tree uncommitted if `.git/index.lock` is denied; unit tests
  only (orchestrator owns the gate); hard 2-round cap.
- Compress long logs before they re-enter a brief.

**4. Orchestrator habits (the cheap wins).**
- Delegate implementation by default; don't type code inline.
- Never bulk-`Read` Codex output or large diffs — send an **Explore subagent** to
  evaluate and report conclusions/failures only.
- One Codex agent per repo; commit each repo separately; flip spec checkboxes after
  the work lands.
- Keep the codebase map fresh (sidenote) so every brief starts from truth.
