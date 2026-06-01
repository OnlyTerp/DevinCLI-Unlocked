# Orchestration: Devin commanding a fleet of subagents

The single biggest day-to-day multiplier in Devin isn't a model or a plugin —
it's that **Devin orchestrates subagents really, really well.** It will spawn a
fleet of focused workers, keep track of all of them, read each result as it
lands, refill the freed slot, and *keep the project moving on its own* — without
stopping every five seconds to ask you a question. That autonomy is a big part of
why this CLI is worth building a whole setup around.

> [!IMPORTANT]
> **This is the stock subagent system. There's nothing to install.** When you
> tell Devin "spin up a bunch of subagents," it's using the built-in subagent
> tool — the same one available in any session. The "swarm" idea below is just a
> *way of working* with that built-in system, not a separate engine or a skill
> you need. You can optionally capture the workflow as a skill (see §5) so you
> invoke it consistently, but the power is in the base tool.

---

## 1. Why this is the killer feature

Most agent CLIs make *you* the orchestrator: run one thing, wait, read it,
decide, run the next, babysit. Devin flips that. You hand it a goal, and it will:

- **fan the work out** across many subagents at once,
- **hold all of them in flight**, reading completions the moment they finish,
- **immediately queue more work** into freed slots instead of idling,
- **keep going autonomously** — synthesizing, deciding, and continuing rather
  than halting to ask permission on every step.

There's **no fixed cap** in the tool on how many subagents the parent can summon
and manage. The real limit is your machine's resources — which is exactly why the
[ops runbook](./ops-runbook.md) exists: per-session memory caps and lazy-started
model backends are what let you fan out aggressively without freezing the box.

Subagents run in their own conversation chain. They share tools and codebase
context but **do not** inherit the parent's history — which is precisely what
makes them good for breadth: mapping a repo, inspecting separate packages,
running isolated checks, or independently reviewing a diff, all in parallel.

---

## 2. Foreground vs background

Two ways a subagent runs, and knowing the difference is most of using them well:

- **Foreground:** the parent waits for it, and you approve its tools
  interactively. Best when you need the answer before continuing, or the lane
  needs a tool you haven't pre-approved.
- **Background:** the parent keeps working while it runs and is notified on
  completion; unapproved tools auto-deny. This is the mode that makes "a fleet in
  flight" possible — launch many, keep orchestrating, collect results as they
  land.

If a background lane dies because a tool was denied, just resume it in the
foreground and approve the tool. The built-in profiles:

| Profile | Use |
| --- | --- |
| `subagent_explore` | read-only research, search, tracing |
| `subagent_general` | implementation/checks when tools are approved |

That's enough to do real fleet work today, with zero extra setup.

---

## 3. Custom subagent profiles: a model and role per lane

This part *is* a real configurable feature (and pairs beautifully with a
[deep model bench](./models.md)). You can define named subagents that pin a
specific model and tool policy to a specific kind of work. Devin reads them from:

```text
.devin/agents/<name>/AGENT.md          # project scope
~/.config/devin/agents/<name>/AGENT.md # user scope (available everywhere)
```

A read-only scout pinned to a fast model:

```markdown
---
name: fast-scout
description: Fast read-only codebase mapper
model: <your-fast-model-uid>
allowed-tools:
  - read
  - grep
  - glob
permissions:
  deny:
    - edit
    - exec
---

You are a read-only research subagent. Find relevant files, trace the flow, and
report concise findings with exact paths and line numbers. Never edit files.
```

The high-leverage pattern is a **matched set of roles**, each assigned the model
that's best (or cheapest) for it:

| Role | Job | Tool policy |
| --- | --- | --- |
| `*-scout` | read-only mapping / discovery | read/grep/glob only |
| `*-impl` | isolated implementation on explicit files | edit/exec on a tight scope |
| `*-test` | run targeted tests, summarize failures | exec for tests |
| `*-review` | read-only diff/risk review | read-only |

Pin a fast/cheap model to `scout`/`test` and a stronger one to `impl`/`review` —
or stand up parallel sets across different model families and pick per task. A
second strong model makes a great independent reviewer of the first one's diff.

> Keep custom-subagent prompts short and tool policies tight. A scout that can
> `edit` is no longer a scout.

---

## 4. A disciplined way to run a big fan-out

When you point Devin at a large task, this is the orchestration discipline that
keeps a fleet *productive* instead of chaotic. It's just how to use the built-in
subagents well — no special tooling required.

### The shape

- **Goal is time-to-correctness with useful parallelism — not forced token
  burn.** Saturate lanes only while there's genuinely independent work.
- **The parent is the scheduler/integrator**, not a worker: keep a backlog, read
  each completion immediately, refill the freed lane right away.
- **Scout before you write.** The first wave in an unfamiliar repo is all
  read-only discovery — e.g. architecture/entrypoints, data/config, API/call
  graphs, tests & fixtures, conventions, risks/security, dependencies, cheap
  validation commands, repro/logs, and one alternate-strategy scout.
- **Writers are special.** Use implementation lanes only for an **explicit
  disjoint file set** — a mutex group. Never run two writers on the same or
  coupled files. "Coupled" means shared config, generated files, common imports,
  shared call chains, fixtures, migrations, lockfiles, or public API. When in
  doubt, it's coupled — one writer.
- **Refill without barriers.** Don't wait for the whole wave to finish; the
  moment early lanes unlock safe disjoint work, launch it while others continue.
- **Spare capacity does speculation that can't conflict** — extra scouting, risk
  analysis, or an alternate *plan*. Speculate on plans, never on conflicting
  edits.

### Quality gates (non-negotiable)

1. Clarify ambiguity **before** launching lanes.
2. Require **evidence before edits** — scouts justify what writers do.
3. Require an **explicit mutex group** for every writer lane.
4. Run **cheap targeted tests before** broad/expensive ones.
5. **Review non-trivial diffs** before the final answer.
6. The **parent performs final verification** — never trust a lane's "done"
   blindly.

### Make lanes report in a fixed shape

Free-form lane reports are painful to integrate. Ask every lane to return one
compact, structured block (e.g. YAML) with: what it did, files touched (with
line ranges), evidence, commands actually run (with exit codes), risks, and
suggested follow-up lanes. Structured output is what makes "read completion →
refill lane" fast.

### An optional pattern: parallel candidates, then verify

For a hard, self-contained problem, you can fan out several subagents on the
*same* task with different stances (direct, minimalist, edge-case-focused) plus
adversarial reviewers, then **pick the winner by an actual check** — a test
suite, a verifier command, an exit code — not by vibes. Verifier quality is
everything here: a weak check just launders a wrong answer through many workers.

---

## 5. Optional: capture the workflow as a skill

If you find yourself orchestrating the same way often, save it as a
[skill](../README.md#3-skills-the-underrated-power-feature) so you invoke it
consistently:

```text
.devin/skills/swarm/SKILL.md          # a short playbook the parent triggers
```

This is purely a convenience — the skill is just a prompt that reminds the parent
of the discipline in §4. It doesn't add any capability the built-in subagent
system doesn't already have. Don't oversell it to anyone as a magic plugin; it's
a checklist you taught Devin to follow.

---

## 6. When *not* to fan out

- **Tiny or single-file tasks** — just do them; orchestration overhead dominates.
- **Tightly coupled changes** — they can't be parallelized safely; one writer.
- **Ambiguous requirements** — resolve them first; parallel lanes on a wrong
  premise produce ten wrong answers faster.
- **Anything destructive or auth-gated** — stop and get a human decision.

The reflex worth building: **default to a couple of read-only scouts for anything
non-trivial, escalate to full saturation only when the work is genuinely
independent, and keep all writing behind explicit mutex groups.** That's what
turns "summon a fleet" into a real speedup instead of a mess — and Devin's
willingness to keep driving that fleet autonomously is what makes it feel
unreasonably good.
