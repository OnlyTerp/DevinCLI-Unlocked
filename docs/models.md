# Models: the full lineup, and how to add the safe ones

Devin's stock picker is already strong (Opus, GPT, Gemini, SWE, Kimi, etc.). The
"unlocked" setup adds *your own* models on top — some via pay-as-you-go API keys,
some self-hosted on your GPU, and some via vendor **subscription** plans.

This doc:

- names the **whole lineup** so you know what's possible,
- gives **reproducible setup** only for the low-risk paths (API key + local),
- and is explicit about which paths carry **account-ban risk** so you can opt
  out.

> [!IMPORTANT]
> The proxy/injection mechanism that makes any of these show up inside Devin's
> `/model` picker is described conceptually in [byok-proxy.md](./byok-proxy.md),
> along with the ToS warning. This doc is about the *models*, not the plumbing.

---

## 1. The lineup at a glance

| Model | How it's reached | Risk tier | Setup here? |
| --- | --- | --- | --- |
| **MiniMax-M3** | MiniMax API (paid key) | ✅ Low (you pay per token) | ✅ Yes |
| **MiMo v2.5 Pro** | Provider API (paid token plan) | ✅ Low (you pay per token) | ✅ Yes |
| **Local Qwen3.6 / AEON / Gemma** | Your own llama.cpp / vLLM / Ollama | ✅ Low (your hardware) | ✅ Yes |
| **Claude Opus 4.7 / 4.8** | Anthropic **Max** subscription OAuth | ⚠️ ToS risk | ❌ Named only |
| **GPT-5.5 / Codex** | ChatGPT **Plus/Pro** subscription OAuth | ⚠️ ToS risk | ❌ Named only |
| **Cursor Composer 2.5** | Cursor subscription via cursor-agent bridge | ⚠️ ToS risk | ❌ Named only |
| **Kimi (k2.x)** | Moonshot Kimi plan OAuth | ⚠️ ToS risk | ❌ Named only |
| **Grok (build / 4.x)** | xAI / SuperGrok OAuth | ⚠️ ToS risk | ❌ Named only |

"Named only" means: yes, it's technically possible to route these into Devin, but
because it drives a subscription seat outside its official client it likely
breaks that vendor's ToS and can get your account banned. This guide will not
hand you a copy-paste for those. The pattern is the same as the API-key models;
the difference is purely the auth source and the risk.

---

## 2. The lowest-risk path: a pay-as-you-go API key model

This is the path to start with. You're using a normal, metered API the provider
*wants* you to call programmatically. MiniMax-M3 and MiMo v2.5 Pro both expose
OpenAI-compatible endpoints, so the setup is the same shape for any such model.

### 2a. Get a key and confirm the endpoint

You need three facts from the provider's dashboard/docs:

- the **base URL** (OpenAI-compatible, e.g. ends in `/v1`),
- the **model name** the API expects,
- an **API key**.

Sanity-check it *before* wiring it into Devin, with a plain curl:

```bash
curl -s https://YOUR_PROVIDER_BASE/v1/chat/completions \
  -H "Authorization: Bearer $YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "model": "PROVIDER_MODEL_NAME",
        "messages": [{"role":"user","content":"reply with: pong"}],
        "max_tokens": 16
      }'
```

If you get `pong` back, the credentials and endpoint are good and any problem
later is in the wiring, not the provider.

### 2b. Keep the key out of git

Put secrets in a file that is **not** committed and is loaded separately from
your non-secret config — e.g. a user-level env file your service/launcher
sources, or Devin's local (gitignored) config scope:

```text
~/.config/devin/<provider>.env        # chmod 600; never committed
# contains only:  PROVIDER_API_KEY=sk-...
```

Non-secret settings (UID, label, base URL, model name, context window) can live
in your committed config; the key must not.

### 2c. Choose a UID that sorts well in the picker

The model's UID is what you type after `/model`. A neat trick: name it so it
**autocompletes next to related stock models.** If you pick a UID that starts
with the same prefix the picker groups on, typing `/model mini` (or whatever)
surfaces your model right alongside the official ones. Pick a clear display
label too — you'll be staring at it in the picker.

### 2d. Decide the context + output caps

Set the advertised context window to something sane rather than the model's
theoretical max. Devin auto-compacts long sessions anyway, so a smaller window
saves tokens with little practical downside. Set a reasonable `max_output_tokens`
to avoid runaway generations.

### 2e. Handle "thinking" models cleanly

Reasoning models emit chain-of-thought you usually don't want rendered as the
visible answer. If the provider returns reasoning on a **separate channel**, keep
it out of the visible text. If it inlines `<think>…</think>` into the content,
strip that span before it reaches the TUI. Either way the user should see the
*answer*, not the scratchpad.

That's the whole pattern. MiniMax and MiMo differ only in base URL + model name +
which key file holds the secret.

---

## 3. Local / self-hosted models (also low risk)

If you have the GPU, hosting your own model is the most durable unlock: no per-
token cost, no ToS exposure, full control. Any OpenAI-compatible local server
works — **llama.cpp** (`llama-server`), **vLLM**, **Ollama**, **LM Studio**, etc.

### 3a. Serve it

Start your server with enough context for an agent (Devin sends a large system
prompt + tool schemas every turn — budget well above your prompt size) and
confirm it answers:

```bash
# example shape; flags differ per engine
curl -s http://127.0.0.1:PORT/v1/models
curl -s http://127.0.0.1:PORT/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"YOUR_LOCAL_MODEL","messages":[{"role":"user","content":"pong?"}],"max_tokens":8}'
```

### 3b. Context vs. throughput is a real tradeoff

Lessons from running a 27B–35B-class model on a single 32 GB card:

- **Don't enable long-context rope/YaRN scaling unless you actually need it.**
  Static scaling applies a constant factor regardless of input length and can
  tank throughput on short prompts (we saw ~70 tok/s collapse to single digits).
  Serve the model's native trained context and stop there.
- **KV-cache quantization** (e.g. q8/q4 K and V) is how you fit long context in
  limited VRAM — but aggressive KV quant can hurt quality and speculative-decode
  acceptance. Tune it; don't cargo-cult.
- **Slots vs. per-slot context.** With N concurrent slots and total context C,
  each slot gets ~C/N tokens. Decide whether you want *many short* slots (swarm
  workers) or *few long* slots (big-context single tasks) — you usually can't
  have both on one card.
- **Speculative decoding** (ngram or a draft model) is a large free win for
  coding-shaped output, but watch draft-acceptance: settings that look faster in
  a microbenchmark can regress on real agent prompts.

### 3c. On-demand start

If a local backend hogs the GPU, don't keep it resident. Wire a **lazy-start
hook**: the model registers in the picker immediately, but the heavy GPU process
only spins up the first time you actually select it (with a health check + a
generous startup timeout for cold model loads). This lets several
GPU-competing backends coexist in one picker, each starting on demand.

---

## 4. The subscription-OAuth lane (named, not documented)

These are the models that make the setup look magical — your existing Claude Max,
ChatGPT, Cursor, Kimi, or Grok seat, driven from inside Devin. They're listed in
[the lineup table](#1-the-lineup-at-a-glance). The plumbing is the same shape as
the API-key models; the **only** difference is that the credential is a
subscription session token rather than a metered API key.

That difference is the whole problem:

- It uses a seat **outside the official client** the subscription is sold for,
  which generally violates that provider's ToS.
- Providers fingerprint their official clients (specific headers, system
  preambles, request shapes). Matching them to make calls succeed is exactly the
  behavior that gets flagged.
- A public, copy-pasteable recipe puts **readers'** accounts at risk, not just
  the author's.

So this guide names them as "what's possible" and stops there. If you choose to
explore it, you do so knowing it can cost you the very subscription you're
trying to use, and you should never publish credentials, tokens, or a turn-key
recipe for it.

---

## 5. Routing: stop using one model for everything

Once you have a deep bench, the win isn't any single model — it's **routing the
right model to the right turn.** A practical default split:

| Task | Use |
| --- | --- |
| Architecture, risky refactors, final synthesis | Strongest reasoning model (e.g. Opus-class) |
| Everyday coding & mechanical fixes | Fast coding model (`swe`, or a fast BYOK coder) |
| Broad read-only repo research | Cheap/fast model via explore subagents |
| Security review | A different strong model than wrote the code |
| Bulk parallel scouting in a swarm | Cheap, high-throughput models / local |
| Summaries & status | Cheapest capable model |

Switch per session with `--model <uid>`, or mid-session with `/model`. For
thinking-capable models, cycle effort with `Alt+T` (`Opt+T` on macOS). The point
of a big lineup is **diversity under one harness** — a second strong model is a
great independent reviewer of the first one's diff.

See [swarms.md](./swarms.md) for mapping these models onto subagent profiles so a
single session can fan work out across many of them at once.
