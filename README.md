# DevinCLI Unlocked

> A no-fluff field guide for turning Devin for Terminal into a local AI
> engineering harness: persistent instructions, model routing, subagents, MCP
> memory, Telegram, and cloud handoff.

DevinCLI is most powerful when you stop treating it like "a chatbot in a
terminal" and start treating it like an **agent runtime**:

- a small always-on operating spec,
- reusable skills,
- specialized subagents,
- connected tools through MCP,
- a memory layer,
- and a clean path from local hacking to cloud execution.

This guide is built around that stack.

---

## 0. Install and start

```bash
curl -fsSL https://cli.devin.ai/install.sh | bash
```

Then open any repo:

```bash
cd your-project
devin
```

Useful launch patterns:

```bash
devin --model opus -- "Plan the safest way to refactor auth"
devin --model swe -- "Fix the failing lint errors"
devin -p "List the risky files in this repo and why"
devin --prompt-file prompt.md
```

Inside a session:

```text
/model opus
/model swe
/plan
/accept-edits
/context
/compact
```

---

## 1. The first unlock: global instructions that behave like a harness

Most people underuse instructions. They paste a long prompt once, forget it,
then wonder why the agent behaves differently tomorrow.

Devin for Terminal automatically reads always-on rules from:

- `AGENTS.md`
- `AGENT.md`
- `CLAUDE.md`

It also imports rules from common agent tools:

- Cursor: `.cursorrules`, `.cursor/rules/*.md`
- Windsurf: `.windsurf/rules/*.md`, `.windsurf/global_rules.md`
- Claude Code: `.claude/`

The best pattern is not a 2,000-line personality file. It is a small repo-level
"soul" file that defines operating principles and points Devin toward skills
when needed.

Create `AGENTS.md` at the repo root:

```markdown
# Agent Operating Contract

## Mission

Ship small, correct, reviewed changes. Prefer evidence over vibes.

## Defaults

- Read the existing code before editing.
- Prefer minimal diffs over rewrites.
- Never invent APIs, commands, benchmarks, or pricing.
- Cite files and commands when reporting.
- Run the narrowest relevant check before declaring done.

## Delegation

- Use read-only subagents for broad codebase research.
- Use implementation subagents only for isolated, parallelizable work.
- Use the review skill before opening a PR.

## Memory

- Record durable facts only: repo setup, recurring commands, architecture
  decisions, test accounts, and user preferences.
- Do not store secrets in memory.
```

That is the harness. It changes the default behavior of every future session
without bloating every prompt.

### Keep rules small; move workflows into skills

Rules are always injected into context. Too many rules dilute attention. For
repeatable workflows, use skills instead:

```text
.devin/skills/
└── review-pr/
    └── SKILL.md
```

Example skill:

```markdown
---
name: review-pr
description: Review staged changes for correctness, security, and style
model: opus
allowed-tools:
  - read
  - grep
  - glob
  - exec
---

Review the current diff.

Return:

1. Blockers
2. Risky assumptions
3. Missing tests
4. Files that need another pass

Only cite concrete evidence.
```

Use rules for identity and constraints. Use skills for procedures.

---

## 2. The second unlock: orchestration beats one giant prompt

Subagents let the main agent spawn independent workers. Each subagent gets
codebase context and tools, but it runs in its own conversation chain instead of
sharing the parent conversation history.

This matters because the root agent can act like an orchestrator:

```text
Use Opus as the lead architect.
Spawn 6 read-only subagents:
1. Auth flow researcher
2. Database schema researcher
3. API route researcher
4. Frontend state researcher
5. Test coverage researcher
6. Security risk researcher

Have each subagent return file paths, line numbers, and a short risk summary.
Then synthesize one implementation plan.
```

Built-in profiles:

- `subagent_explore`: read-only research, architecture mapping, trace gathering.
- `subagent_general`: implementation work, checks, mechanical changes.

Background subagents can run while the parent keeps working. Foreground
subagents pause the parent and let you approve tools interactively.

### Custom subagents

You can define specialized workers:

```text
.devin/agents/
└── fast-researcher/
    └── AGENT.md
```

```markdown
---
name: fast-researcher
description: Fast read-only codebase mapper
model: swe
allowed-tools:
  - read
  - grep
  - glob
---

You are a read-only research subagent. Find relevant files, trace the flow, and
report concise findings with file paths. Do not edit files.
```

Now the root model can route cheap, fast, isolated research to `swe` while
keeping a smarter model for planning and synthesis.

### The 10-worker pattern

If a fast model streams around ~950 tokens/sec, ten independent workers can
produce roughly 9,500 tokens/sec of aggregate output in ideal conditions. That
is not "one smarter brain"; it is a swarm. The trick is to give each worker a
bounded, non-overlapping job.

Good swarm prompt:

```text
Use the root agent only for orchestration.
Spawn 10 SWE subagents in parallel.
Each subagent gets exactly one directory.
Each must report:
- key files
- risky code paths
- tests to run
- unanswered questions

Do not let subagents edit files.
After all return, produce one prioritized plan.
```

Bad swarm prompt:

```text
Spawn 10 agents to make the app better.
```

That creates overlap, merge conflicts, and noise.

### When not to use subagents

Do not spawn workers for:

- one-file edits,
- tiny questions,
- tasks that require a single coherent design pass,
- anything where the cost of merging answers is higher than the research itself.

Use orchestration when breadth matters.

---

## 3. Why Rust matters for an agentic CLI

Cognition describes Devin for Terminal as written in Rust and highlights a
custom Rust terminal rendering library. That is not just a marketing detail.

For an agentic CLI, Rust is a good fit because the interface is not passive
text. It is a live control surface for:

- streaming model output,
- rendering diffs,
- handling keyboard input,
- managing subprocesses,
- coordinating tool calls,
- updating progress panels,
- and staying responsive while the agent works.

Rust helps here because:

- Low latency keeps the terminal UI snappy while output streams.
- Memory safety reduces runtime crash risk in long sessions.
- Strong concurrency primitives help with tools, subprocesses, and streaming.
- Single-binary ergonomics make distribution easier across machines.
- Predictable performance matters when the CLI is open all day.

The real point: DevinCLI is not just a prompt wrapper. It is a local runtime for
supervising autonomous work.

---

## 4. Local first, cloud when the task outgrows your laptop

Devin for Terminal is built around a simple workflow:

1. Start locally where your repo, shell, and editor already are.
2. Explore, plan, and make early edits.
3. Hand the work to Devin in the cloud when it needs an isolated machine,
   browser testing, recordings, PR work, or long-running follow-through.

Cloud handoff is useful when you want to:

- close your laptop but keep the task moving,
- run multiple agents against the same codebase without managing worktrees,
- let the cloud agent test in its own browser,
- have Devin open a PR and respond to review comments,
- avoid risky commands touching your local machine.

If your build exposes a handoff slash command, use `/help` to confirm the exact
spelling. Public docs describe the handoff flow, but command names can move
faster than static docs.

---

## 5. Telegram: turn DevinCLI into a reachable operator

Telegram becomes powerful when connected through MCP. You can give Devin a
controlled tool surface for sending updates, reading a channel, or interacting
with a bot.

There are two common approaches:

### Option A: Bot API MCP

Best for notifications and simple command workflows.

Example local config:

```json
// .devin/config.local.json
{
  "mcpServers": {
    "telegram": {
      "command": "npx",
      "args": ["-y", "telegram-bot-mcp-server"],
      "env": {
        "TELEGRAM_BOT_TOKEN": "put-this-in-local-config-or-env"
      }
    }
  }
}
```

Use BotFather to create the bot token. Keep it out of git.

### Option B: MTProto Telegram MCP

Best when you want account-level access to chats, channels, and message history.
This is more powerful and more sensitive. Use it only if you understand the
security tradeoff.

Possible use cases:

- "Summarize the last 50 messages in my project channel."
- "Send me a Telegram update when tests finish."
- "Watch a release channel and extract breaking changes."
- "Forward the PR link to my team chat."

The pattern is simple:

1. Add a Telegram MCP server.
2. Store credentials in local config or environment variables.
3. Add permissions so Devin can call only the Telegram tools you want.
4. Give explicit prompts for when to message you.

Example instruction:

```markdown
When a long-running task finishes, send a Telegram summary with:

- branch name
- PR link
- test result
- one blocker if any

Never send secrets, logs with tokens, or private customer data.
```

---

## 6. Lightweight MCP memory that actually works

Memory should not be mystical. The best memory is boring:

- small,
- explicit,
- inspectable,
- easy to delete,
- and shared across sessions/models through MCP.

The official `@modelcontextprotocol/server-memory` package gives you a simple
knowledge-graph memory server with entities, relations, and observations.

Example:

```json
// ~/.config/devin/config.json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    }
  }
}
```

If you prefer file-backed memory, use a markdown/JSON-backed MCP server and keep
the memory folder in a private repo.

### What to store

Store durable facts:

- "This repo uses pnpm."
- "Run `cargo nextest` for Rust tests."
- "The staging API requires VPN."
- "Rob prefers terse PR descriptions."
- "Do not touch generated files in `sdk/gen/`."

Do not store:

- secrets,
- one-off chat noise,
- raw logs,
- speculative guesses,
- credentials,
- private customer data.

### Memory prompt

```text
Before starting, query memory for this repo and this user.
At the end, write back only durable facts that will help future sessions.
Do not save secrets or temporary debugging notes.
```

This is how memory becomes useful instead of becoming another pile of context
sludge.

---

## 7. Model routing: use smart models as leads, fast models as workers

Devin for Terminal supports model switching through:

```bash
devin --model opus -- "do the hard planning"
```

```text
/model swe
/model codex
/model sonnet
```

Skills and custom subagents can also specify a model:

```yaml
model: swe
```

The practical strategy:

- Architecture, high-risk refactors, final synthesis: smartest available model.
- Broad read-only repo research: fast/cheap model.
- Lint fixes and mechanical edits: fast coding model.
- PR review and security pass: smarter model.
- Summarization and status updates: cheap model.

If GPT-5.5 or free promo models are discounted in your account, exploit them for
the right tier of work. Promos change, so check the current model picker/pricing
before publishing exact claims.

High-leverage prompt:

```text
Use the current smart model as the lead.
Delegate broad research to fast subagents.
Use cheap models for mechanical checks.
Before editing, produce a short plan with which model does which job.
```

---

## 8. A complete DevinCLI power setup

Recommended repo layout:

```text
your-project/
├── AGENTS.md
├── .devin/
│   ├── config.json
│   ├── skills/
│   │   ├── review-pr/
│   │   │   └── SKILL.md
│   │   ├── run-tests/
│   │   │   └── SKILL.md
│   │   └── research-architecture/
│   │       └── SKILL.md
│   └── agents/
│       ├── fast-researcher/
│       │   └── AGENT.md
│       └── test-runner/
│           └── AGENT.md
└── .devin/config.local.json   # local only; secrets and personal MCP config
```

Project config example:

```json
// .devin/config.json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Exec(git status)",
      "Exec(git diff)",
      "Exec(npm run)",
      "Exec(pnpm run)",
      "Exec(cargo test)",
      "Exec(cargo nextest)"
    ],
    "deny": ["Exec(rm -rf)", "Exec(sudo)"]
  },
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

Local config example:

```json
// .devin/config.local.json
{
  "mcpServers": {
    "telegram": {
      "command": "npx",
      "args": ["-y", "telegram-bot-mcp-server"],
      "env": {
        "TELEGRAM_BOT_TOKEN": "your-local-token"
      }
    }
  }
}
```

Never commit `.devin/config.local.json` if it contains secrets.

---

## 9. Copy/paste prompts

### Repo map

```text
Use subagents for read-only research.
Map this repo into:
1. main entrypoints
2. core domain modules
3. test strategy
4. risky files
5. commands required before PR

Return file paths and line numbers.
```

### Parallel investigation

```text
Spawn one read-only subagent per top-level package.
Each subagent should identify:
- purpose of the package
- key files
- public APIs
- tests
- likely risks

Synthesize into a dependency map and implementation plan.
```

### Smart lead, fast workers

```text
Act as the lead architect.
Use fast subagents for code search and evidence gathering.
Do not let subagents edit.
After they report, decide the minimal safe diff.
```

### Handoff-ready task

```text
Prepare this task for cloud handoff:
- summarize current goal
- list changed files
- list remaining checks
- note known risks
- include exact commands already run
- include what success looks like
```

### Memory discipline

```text
Query memory for this repo before planning.
At the end, save only stable facts that help future sessions.
Do not save secrets, temporary errors, logs, or guesses.
```

### Telegram updates

```text
If this task takes more than 10 minutes, send a Telegram update when:
1. the plan is complete
2. tests finish
3. the PR is opened
4. a blocker appears

Keep each update under 500 characters.
```

---

## 10. The anti-slop checklist

Before publishing or sharing a DevinCLI guide, remove claims that are not
verified.

Keep:

- exact commands,
- file paths,
- configuration examples,
- workflows that readers can copy,
- links to official docs,
- warnings about secrets and permissions.

Cut:

- vague "10x productivity" claims,
- fake benchmark numbers,
- unsupported pricing claims,
- huge instruction files,
- prompts that ask agents to "make it better",
- examples that commit secrets.

The viral version is not the loudest version. It is the guide people can copy in
10 minutes and feel the power immediately.

---

## Sources

- [Devin for Terminal quickstart](https://cli.devin.ai/docs)
- [Commands and slash commands](https://cli.devin.ai/docs/reference/commands)
- [Rules and `AGENTS.md`](https://cli.devin.ai/docs/extensibility/rules)
- [Skills](https://cli.devin.ai/docs/extensibility/skills/overview)
- [Creating skills](https://cli.devin.ai/docs/extensibility/skills/creating-skills)
- [Subagents and custom subagents](https://cli.devin.ai/docs/subagents)
- [MCP overview](https://cli.devin.ai/docs/extensibility/mcp/overview)
- [MCP configuration](https://cli.devin.ai/docs/extensibility/mcp/configuration)
- [Models](https://cli.devin.ai/docs/models)
- [Devin for Terminal launch page](https://devin.ai/terminal)
- [Cognition launch post](https://cognition.ai/blog/devin-for-terminal)
- [MCP memory server](https://www.npmjs.com/package/@modelcontextprotocol/server-memory)
- [Telegram Bot MCP example](https://www.npmjs.com/package/telegram-bot-mcp-server)
