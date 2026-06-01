# Ops runbook: keeping a heavy multi-agent setup from eating your machine

Running BYOK models, a TLS proxy, local GPU servers, and a fleet of subagents on
one workstation is great until something runs away and freezes the box. This is
the hard-won operational layer: the patterns that keep a heavy Devin setup stable
under load.

These are **patterns, not a copy-paste config** — your OS, memory budget, and
hardware differ. The lessons generalize; the exact numbers are yours to set.

---

## 1. The core principle: cap every agent so one can't take the host down

The failure that hurts most is a single agent (or model server) ballooning until
the OS OOM-killer fires and takes everything with it. The fix is **per-process
resource ceilings** so a runaway gets killed *inside its own box* long before it
threatens the machine.

On Linux this is what cgroups are for. Wrap each long-lived agent session in its
own transient scope with memory + swap + task limits:

```bash
# conceptual: launch each session in its own capped cgroup scope
systemd-run --user --scope \
  -p MemoryMax=<hard-limit> \
  -p MemoryHigh=<soft-limit> \
  -p MemorySwapMax=<swap-limit> \
  -p TasksMax=<n> \
  your-agent-cli "$@"
```

Wrap your CLI launchers so this happens automatically, and keep an env escape
hatch to bypass the cap when you genuinely need to debug something big. The
point: **a single runaway session OOMs itself, not the OS.**

### Pick limits that leave headroom

- Set a **hard** cap (process is killed past it) and a lower **soft** cap (kernel
  starts reclaiming first) so you get backpressure before death.
- Sum your caps with headroom under total RAM — if you run several sessions plus
  model servers, their combined hard caps must fit with room to spare.
- Give heavier roles bigger budgets than light ones; not every agent needs the
  same ceiling.

---

## 2. Run the proxy and model servers as supervised services

Anything every session depends on — the BYOK proxy, a local model server, a
shim — should be owned by your init/service manager, not a terminal you might
close.

For each service:

- **Auto-restart on failure**, but with a **start-rate limit** (e.g. cap restarts
  in a window) so a crash-loop can't hammer the machine. A restart loop with no
  limit once produced thousands of restarts and a multi-MB identical-stacktrace
  log in this setup — bound it.
- **Memory/task ceilings** on the service too (same reasoning as §1).
- **A lightweight watchdog timer** that health-checks the critical proxy every
  couple of minutes and repairs common failure states (orphan holding the port,
  port mismatch) without you noticing.
- **Log rotation with a size cap.** Capture/debug logs are invaluable and grow
  without bound; rotate and prune oldest-first to a fixed budget.

### Single-port ownership, detected by cgroup

When a managed service can't bind because something already holds its port, you
need to know whether that something *is* your service. **Check cgroup membership,
not PID equality** — a supervisor's main PID often forks the real worker, so the
listener's PID won't match the recorded main PID. Comparing the listener's cgroup
to the unit's cgroup is the reliable ownership signal; evict only true orphans.

---

## 3. Reap idle agents before you fan out

A fleet workflow leaves stragglers: terminal panes with orphaned sessions, idle
background workers holding gigabytes. Before a big multi-agent run or a large
filesystem scan, **reap idle agents first** to reclaim headroom.

Useful habits:

- A `status` one-liner that shows memory, swap, GPU, listening ports, top RAM
  consumers, agent counts, and recent OOM events — so "what's wrong?" is one
  command.
- A `reap-idle` helper with a **dry-run by default** and an explicit `--kill`
  flag, so you can see what *would* die before it does.
- Check the status before fanning out, not after the freeze.

---

## 4. The GPU is a single, contended resource — fail closed

One consumer GPU can host exactly one large model comfortably. Treat it as a
mutex:

- **One large model backend active at a time.** Make GPU-competing backends
  **lazy-start on demand** (register in the picker immediately; spin up the heavy
  process only when selected) and stop/swap the previous one as part of that hook.
- **Fail closed on the risky path.** If a particular backend has crashed your
  display/host before (e.g. serving on the same GPU that drives your monitors),
  gate it behind an explicit, opt-in acknowledgement file/flag — a plain restart
  should *not* be able to re-arm a known-dangerous configuration. Make the safe
  backend the default and the dangerous one require a deliberate "I accept the
  risk" step.
- **Watch VRAM separately from RAM.** They have different ceilings and different
  failure modes; your `status` view should show both.

---

## 5. If you're on WSL: treat it as a sidecar

A specific but common case: Devin + agents in **WSL**, heavy GPU/model serving on
the **Windows host**. Hard-earned WSL lessons:

- **Cap the WSL VM's RAM and swap** in `.wslconfig`. An uncapped WSL VM competing
  with Windows-side model servers is how you get whole-VM freezes.
- **A big model weight file can't mmap inside a small WSL VM.** If the VM's commit
  limit is below the model size, the server fails with ENOMEM — which is a strong
  reason to serve large models **Windows-native** and reach them from WSL over
  loopback, rather than inside the VM.
- **Mirrored networking changes loopback addressing.** Under WSL mirrored mode,
  Windows-side servers are reachable from WSL at `127.0.0.1:<port>` and the old
  Hyper-V vmswitch gateway IP stops routing. If you flip networking modes, every
  consumer's host address has to change with it.
- **Don't hand-tune Linux TCP/network sysctls under mirrored networking.**
  Aggressive `net.ipv4.tcp_*` / `net.core.*` tuning can make sockets show LISTEN
  while handshakes to `127.0.0.1` hang — silently breaking your proxy/shim ports.
  Keep a tiny loopback smoke-test you run after any network change.
- **Avoid broad scans of the Windows drive bridge** (`/mnt/c`) from inside WSL on
  hot paths — it's slow (9P) and memory-pressuring. Keep active model caches on
  the fast native filesystem; archive cold assets elsewhere.
- **Don't run two things on the same port across the boundary.** If the Windows
  host owns a gateway port, make sure the WSL-side equivalent is disabled, or it
  fail-loops on "address already in use."

---

## 6. Backups and disk hygiene

- **Snapshot critical state, not the whole VM, routinely.** Full VM exports are
  slow and, if interrupted, can leave the VM in a conversion-locked state needing
  manual recovery. Prefer targeted snapshots of config/state for the regular
  cadence; gate full exports behind an explicit, deliberate flag.
- **Deleting files inside a VM doesn't shrink its disk image.** Enable automatic
  sparse reclaim if your platform supports it, and keep a documented manual
  compact step for when the image bloats.

---

## 7. A "things that have hurt us" mindset

The most valuable artifact in this whole setup is a short, blunt list of past
incidents and their fixes, kept right next to the config. Examples of the
*kinds* of entries worth recording:

- "Agent grew to N GB, OOM-killed the host → per-session cgroup caps."
- "Service crash-looped thousands of times holding a port → start-rate limit +
  watchdog."
- "Crash dumps filled the disk → per-session caps + reaping kept headroom."
- "Sysctl tuning silently broke loopback under mirrored networking → reverted,
  added a smoke test."

Each entry is one sentence of symptom and one of fix. That file pays for itself
the first time the same class of failure starts to recur — and it's the cheapest
"AGENTS.md memory" you'll ever write, because future agent sessions (and future
you) read it before touching infrastructure.

---

## 8. Minimum viable version

If this all sounds like a lot — the 20% that prevents 80% of the pain:

1. **Cap each agent session** in its own memory-limited cgroup scope.
2. **Run the proxy as a restart-limited service** with a 2-minute watchdog.
3. **Keep one `status` command** that shows RAM/swap/GPU/ports/OOM events.
4. **Lazy-start GPU backends**, one large model at a time.
5. **Write down each incident** as one line of symptom + fix.

Everything else is refinement on top of those five.
