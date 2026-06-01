# Bring-your-own-model: how the local proxy "unlock" works

> **Concepts and patterns only.** This document explains *how* a local proxy
> can splice your own models into the Devin CLI model picker, and the design
> decisions that make it reliable. It deliberately does **not** ship the proxy
> source, a turn-key installer, or the wire-format details. The goal is to teach
> the approach so you can decide whether and how to build your own.

> [!WARNING]
> **Read the legal/ToS reality first.** This technique works by putting a
> TLS-intercepting proxy between the Devin CLI and its backend gateway so it can
> rewrite a few responses. Doing that, and especially driving other vendors'
> **subscription** plans (Claude Max, ChatGPT/Codex, Cursor, Kimi, Grok) outside
> their official clients, very likely violates the Terms of Service of Devin
> and/or those providers. It can get accounts suspended. Nothing here is legal
> advice or an endorsement. If you build this, you own the risk. Prefer the
> low-risk paths first: **pay-as-you-go API keys** and **models you host
> yourself** (see [models.md](./models.md)).

---

## 1. Why a proxy at all

Out of the box the Devin CLI talks to a single hosted gateway. Every model you
see in `/model` — Claude, GPT, Gemini, SWE, etc. — is resolved server-side. There
is no documented, supported way to point a session at a model the gateway
doesn't already offer (a paid third-party API, or something running on your own
GPU).

The unlock is conceptually simple:

```
                 ┌───────────────────────────────┐
   devin CLI ───►│  local proxy (you run this)    │───► real Devin gateway
                 │  - passes almost everything     │     (auth, sync, telemetry,
                 │    straight through             │      cloud handoff, etc.)
                 │  - rewrites the handful of      │
                 │    responses that decide which  │───► your own model backends
                 │    models exist + which model   │     (API providers, local
                 │    handles a given turn         │      llama.cpp / vLLM / Ollama)
                 └───────────────────────────────┘
```

The proxy is a **man-in-the-middle on your own machine for your own session.**
It is not a patched CLI binary and it is not a fork. That matters for
maintainability: CLI updates rarely break it because you're rewriting protocol
responses, not hooking internal functions.

---

## 2. The three things the proxy has to do

You only need to influence three moments in the conversation between the CLI and
its gateway. Everything else passes through untouched.

1. **Advertise the model.** When the CLI asks the gateway "what models can I
   use?", inject one or more synthetic entries into that response so your model
   shows up in `/model` and is accepted by `--model <uid>`.

2. **Mark the model as allowed.** The CLI also fetches team/account settings
   that gate which model UIDs are permitted. Add your synthetic UIDs to that
   allow-list so the CLI doesn't reject them as "not enabled."

3. **Answer the turn.** When the CLI sends a chat request whose selected model
   is one of yours, *don't* forward it. Instead, translate the request into a
   normal OpenAI-style `chat/completions` (or Anthropic Messages) call, hit your
   chosen backend, and translate the reply back into the exact response shape
   the CLI expects.

Because only those three are rewritten, **auth, session sync, telemetry, usage
metering for stock models, and cloud handoff keep working normally.** That's the
single most important design property: be a scalpel, not a sledgehammer.

---

## 3. The hard part: response fidelity

Advertising a model is easy. Making the *answer* indistinguishable from a real
gateway answer is where all the engineering goes. The CLI has strict
expectations about the streaming response: how tool calls are signaled, how tool
results come back in, what a "stop" looks like, and the invariant that
**tool-call turns carry no visible text.**

Patterns that made this robust:

- **Reverse-engineer from a known-good capture.** Record one real round-trip
  from a stock model (request + streamed response), then build your synthetic
  responses by *editing that capture in place* rather than constructing the
  wire format from scratch. Real messages carry fields you don't understand;
  preserve the unknown ones instead of dropping them. Rebuilding a settings
  blob from only the fields you recognize is the fastest way to get an opaque
  "invalid argument" rejection.

- **Map the role/stop semantics exactly.** User, assistant, and tool-result
  turns each have a distinct shape, and tool-use vs end-of-turn are distinct
  stop signals. Get these wrong and the agent loop stalls or loops.

- **Translate tools in both directions.** Model → CLI: emit the tool call with
  id + name + JSON arguments and the tool-use stop signal. CLI → model: turn the
  CLI's tool-result turns back into the `role: "tool"` messages your backend
  understands. The full agent loop only works if both directions are faithful.

- **Emit terminal usage/stop frames exactly once.** Gateways can send multiple
  bookkeeping frames; track the final usage and stop reason and emit them a
  single time at stream end, or the TUI's accounting gets confused.

---

## 4. Streaming without breaking the "silent tool turn" invariant

You want tokens to appear live in the TUI, not dumped in one block at the end.
But you also must never show text on a turn that turns out to be a tool call.

The pattern that resolves the tension is **stream-when-safe**:

- Forward tokens live *while the response is provably plain text.*
- The moment any tool-call signal appears (a structured tool-call delta, an
  inline tool-call marker, or a reasoning/thinking channel), **stop streaming
  live** and fall back to a buffered, guarded path for the rest of that turn.

This gives you live tokens for ordinary Q&A while preserving the "tool turns
have no visible content" rule for everything else. Make it a toggle so you can
force the fully-buffered path when debugging.

---

## 5. Optional: output guards for weaker/local models

Frontier hosted models mostly behave. Smaller local models drift: they repeat
the same failing search, emit tool-call markup with no actual call, declare
victory before doing the work, or loop. If you route local models through the
proxy, a small **guard stack** on the model's output pays for itself:

| Guard | What it prevents |
| --- | --- |
| Repeat/loop blocker | Identical or near-identical tool calls in a short window (the most expensive failure mode) |
| Tool-arg validation | Malformed tool arguments reaching the executor |
| Pseudo-tool-intent repair | "I'll run X" narration that never becomes a real tool call |
| Task-drift repair | The model wandering off the actual request |
| Evidence-saturation synthesis | Endless searching when there's already enough to answer |

Two rules of thumb:

1. **Keep the cheap, safe guards always on** (loop-blocking, arg validation).
   They prevent real footguns and rarely misfire.
2. **Make the text-*rewriting* guards opt-out behind one master switch.** Any
   guard that can replace what the model actually said is higher friction; you
   want a single flag to get "raw" output when comparing behavior.

Frontier models routed through the proxy generally don't need the rewriting
guards at all.

---

## 6. Operational hygiene (so it doesn't wreck your machine)

A TLS-intercepting proxy that every CLI session depends on is critical
infrastructure. Treat it like a service, not a script you run in a tmux pane.

- **Own it with your init/service manager**, with auto-restart and a start-rate
  limit so a crash-loop can't hammer your box. A runaway restart loop holding a
  port produced a multi-thousand-restart, multi-MB log file in one of our
  incidents — bound restart attempts.
- **One owner per port.** If an orphan process squats on the proxy's port, the
  managed service can't bind. Detect "is the process holding the port actually
  part of my service?" by **cgroup membership, not PID equality** — the
  supervisor's main process often forks the real worker, so PIDs won't match.
- **A tiny watchdog** that health-checks the proxy every couple of minutes and
  repairs port-mismatch/orphan states recovers most failures without you.
- **Combine the system CA with the interceptor CA into one bundle** and point
  the CLI at it, so TLS to the *real* endpoints still verifies. Don't disable
  verification globally.
- **Rotate and size-cap the capture/log directory.** Full request/response
  captures are gold for debugging but grow without bound; prune oldest-first to
  a fixed budget.

---

## 7. When NOT to do this

- You only need a model that's already pay-as-you-go via API → most agent CLIs
  (and Devin's own config) can point at an OpenAI-compatible base URL directly,
  or you can use a much simpler local OpenAI-compatible shim without intercepting
  any TLS. Reach for the MITM approach only when you specifically need the model
  inside *Devin's* picker.
- You're on a work/enterprise account → org policy and auditing make this a bad
  idea, and admin permissions override local choices anyway.
- You'd be redistributing another vendor's subscription access → don't.

---

## 8. Mental model summary

- The proxy is a **scalpel**: rewrite 3 responses, pass through everything else.
- Build responses by **editing real captures**, preserving unknown fields.
- **Stream-when-safe** keeps live tokens without breaking tool turns.
- **Guards** are for weak/local models; keep rewriting guards behind one switch.
- Run it as a **supervised service** with a watchdog, single-port ownership by
  cgroup, and a combined CA bundle.
- Stay on the **low-risk lane** (API keys + self-hosted) unless you fully accept
  the ToS/account risk of the subscription-OAuth lane.
