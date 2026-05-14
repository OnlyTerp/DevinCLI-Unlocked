# DevinCLI Unlocked

> A current, no-fluff field guide for using
> [Devin for Terminal](https://cli.devin.ai/docs) as a real engineering harness:
> rules, skills, subagents, MCP, hooks, shell integration, cloud handoff, and
> model routing.

DevinCLI is not just "chat in a terminal." The current version is closer to a
local agent runtime with:

- persistent project rules,
- reusable skills,
- foreground/background subagents,
- model overrides,
- MCP tools,
- lifecycle hooks,
- web search,
- shell integration,
- permission modes,
- and a bridge to cloud Devin.

This README is written for people who just found the tool and want the newest
practical patterns without hype.

---

## 0. Install, update, and start

macOS / Linux / WSL:

```bash
curl -fsSL https://cli.devin.ai/install.sh | bash
```

Windows PowerShell:

```powershell
irm https://static.devin.ai/cli/setup.ps1 | iex
```

Then open a repo:

```bash
cd your-project
devin
```

Useful launch patterns:

```bash
devin -- "map this repo and suggest the safest first improvement"
devin --model opus -- "plan a risky auth refactor before editing"
devin --model swe -- "fix the failing lint errors"
devin -p "summarize this repo's test strategy"
devin --prompt-file prompt.md
devin -c
devin -r <session-id>
```

Run `/help` inside the CLI for the exact commands available in your build. The
tool is moving fast.

---

## 1. The high-leverage mental model

The best DevinCLI setup has layers:

- **Rules**: always-on project constraints in `AGENTS.md`, `AGENT.md`, or
  `CLAUDE.md`.
- **Skills**: reusable workflows and slash commands in
  `.devin/skills/*/SKILL.md`.
- **Subagents**: isolated workers for research or implementation, either
  built-in or defined in `.devin/agents/*/AGENT.md`.
- **MCP**: external tools such as GitHub, Linear, Figma, memory, databases, and
  internal APIs.
- **Hooks**: guardrails and automation around tool use in
  `.devin/hooks.v1.json`.
- **Permissions**: what Devin can do without asking through `permissions.allow`,
  `permissions.deny`, and `permissions.ask`.

Do not put everything in one giant prompt. Put stable rules in rules, repeatable
procedures in skills, tool connections in MCP, and enforcement in hooks.

---

## 2. Rules: keep the "soul" small

Devin reads always-on rules from:

- `AGENTS.md` (recommended),
- `AGENT.md`,
- `CLAUDE.md`.

It can also import rules/config from Cursor, Windsurf, Claude Code, OpenCode, VS
Code, and Zed. That is convenient, but it means stale config from another tool
can silently affect Devin. If behavior feels weird, inspect or disable imports
with `read_config_from`.

Good `AGENTS.md`:

```markdown
# Agent Operating Contract

## Defaults

- Read existing code before editing.
- Prefer small, reviewable diffs.
- Do not invent APIs, commands, benchmarks, pricing, or file paths.
- Cite exact files and commands when reporting.
- Run the narrowest relevant check before declaring done.

## Delegation

- Use read-only subagents for broad repo research.
- Use implementation subagents only for isolated work.
- Use the review skill before opening a PR.

## Memory

- Save durable facts only: setup, recurring commands, architecture decisions,
  test accounts, and user preferences.
- Never store secrets.
```

Bad `AGENTS.md`:

- 2,000 lines of personality.
- conflicting rules copied from five tools.
- pricing or model claims that expire.
- secrets, tokens, or private URLs.

Rule of thumb: if the instruction is always true for the repo, put it in rules.
If it is a procedure, make it a skill.

---

## 3. Skills: the underrated power feature

Skills are reusable prompts that can be invoked as slash commands, used by the
agent autonomously, run with their own model, restrict tools, and even run as
subagents.

Project skill:

```text
.devin/skills/review-pr/
└── SKILL.md
```

Example:

```markdown
---
name: review-pr
description: Review the current diff for correctness, security, and tests
argument-hint: "[optional focus area]"
model: opus
allowed-tools:
  - read
  - grep
  - glob
  - exec
permissions:
  allow:
    - Exec(git diff)
    - Exec(git status)
---

Review the current diff.

Focus on:

1. logic bugs
2. security issues
3. missing tests
4. accidental large rewrites
5. risky assumptions

Return only concrete findings with file paths and commands.
```

Use it inside Devin:

```text
/review-pr
```

### Run a skill as a subagent

For focused research that should not pollute the main context:

```markdown
---
name: deep-research
description: Research a code path and report evidence
subagent: true
model: sonnet
allowed-tools:
  - read
  - grep
  - glob
---

Research this topic: $ARGUMENTS

Return exact files, line numbers, and unanswered questions.
```

This is one of the cleanest ways to get specialized workers without bloating the
root conversation.

---

## 4. Subagents: orchestration beats one giant context

Subagents share tools and codebase context, but they run in their own
conversation chain. They do **not** inherit the parent conversation history.
That makes them useful for breadth:

- map a large repo,
- inspect separate packages,
- run isolated tests,
- review a diff independently,
- investigate competing solutions.

Built-in profiles:

| Profile            | Best use                                         |
| ------------------ | ------------------------------------------------ |
| `subagent_explore` | read-only codebase research                      |
| `subagent_general` | implementation or checks when tools are approved |

Subagents can run:

- **foreground**: parent waits; you approve tools interactively.
- **background**: parent keeps working; unapproved tools are denied.

If a background subagent fails because a tool was denied, resume it in the
foreground and approve the needed tool.

Good swarm prompt:

```text
Use the root agent only for orchestration.

Spawn one read-only subagent per top-level package.
Each subagent must report:
- purpose of the package
- key files
- risky code paths
- relevant tests
- unanswered questions

Do not let subagents edit files.
After all return, synthesize one prioritized implementation plan.
```

Bad swarm prompt:

```text
Spawn 10 agents to make the app better.
```

The trick is not "more agents." The trick is bounded, non-overlapping jobs.

### Custom subagents

Custom subagents are experimental, but useful:

```text
.devin/agents/fast-researcher/
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
permissions:
  deny:
    - edit
    - exec
---

You are a read-only research subagent. Find relevant files, trace the flow, and
report concise findings. Never edit files.
```

Now you can ask:

```text
Use the fast-researcher subagent to map the auth flow.
```

---

## 5. Modes and permissions: speed without losing control

DevinCLI is much better when safe actions are pre-approved and dangerous actions
are blocked.

Important modes:

- **Normal**: approval for writes and shell commands.
- **Accept Edits**: workspace file edits are okay, shell commands still prompt.
- **Plan**: read-only planning before implementation.
- **Ask**: answer a question without code changes.
- **Bypass**: broad local machine access.
- **Autonomous / sandbox**: unattended execution with OS-level limits.

Slash commands:

```text
/normal
/accept-edits
/plan
/ask explain the routing layer
/bypass
/mode
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
      "Exec(git log)",
      "Exec(npm run)",
      "Exec(pnpm run)",
      "Exec(cargo test)"
    ],
    "deny": ["Exec(rm)", "Exec(sudo)", "Write(.env*)", "Write(**/*secret*)"],
    "ask": ["mcp__github__create_issue", "mcp__linear__*"]
  }
}
```

Bypass is fast, but unrestricted. Prefer `--sandbox` when you want unattended
work with filesystem/network boundaries.

---

## 6. Shell integration: summon Devin from your real shell

Shell integration is now one of the best everyday upgrades. It wraps your shell
so Devin can see recent commands/output and be invoked instantly.

Setup:

```bash
devin shell setup
```

Then restart or source your shell config.

What it unlocks:

- press `Ctrl+G` in your shell to invoke Devin with the current command line and
  recent shell context,
- in zsh, type `# explain this error` and press Enter to send that comment to
  Devin,
- ask for help after a failed command without manually pasting the whole log.

Notes:

- Works on macOS, Linux, and WSL with Bash, Zsh, and Fish.
- Zsh has the best support.
- PowerShell/CMD shell integration is not supported yet.
- This is separate from `devin setup`; run `devin shell setup` explicitly.

---

## 7. MCP: tools are where the leverage is

MCP gives Devin external tools: GitHub, Linear, Notion, Figma, databases,
internal APIs, memory, and custom scripts.

Add a remote MCP server:

```bash
devin mcp add linear --url https://mcp.linear.app/mcp
devin mcp login linear
```

Add Figma:

```bash
devin mcp add figma --url https://mcp.figma.com/v1
```

Add GitHub Copilot MCP with device flow:

```bash
devin mcp add github --url https://api.githubcopilot.com/mcp/
```

Manage servers:

```bash
devin mcp list
devin mcp get <name>
devin mcp login <name>
devin mcp logout <name>
devin mcp remove <name>
```

Scope matters:

```bash
devin mcp add -s project <name> <url> # shared in .devin/config.json
devin mcp add -s user <name> <url>    # global user config
```

By default, new servers are saved to local scope (`.devin/config.local.json`) so
secrets do not land in git.

### MCP gotchas

- OAuth is per client. If you authenticated in Claude Code or Windsurf, still
  run `devin mcp login <server>` for DevinCLI.
- CLI docs currently say remote MCP servers should use Streamable HTTP, not
  legacy SSE.
- Keep secrets in `.devin/config.local.json`, not committed project config.
- Use `permissions.ask/allow/deny` for sensitive MCP tools.
- Disable noisy imported MCP config with `read_config_from` if another editor is
  polluting your tool list.

---

## 8. Lightweight memory that is actually useful

Memory should be boring:

- explicit,
- inspectable,
- easy to delete,
- shared across sessions,
- never a dumping ground for logs.

The simple option is the official memory MCP server:

```json
// ~/.config/devin/config.json or .devin/config.local.json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "env": {
        "MEMORY_FILE_PATH": "/absolute/path/to/devin-memory.jsonl"
      }
    }
  }
}
```

Store:

- repo setup commands,
- recurring test/lint commands,
- architecture decisions,
- API conventions,
- user preferences,
- known flaky tests.

Do not store:

- secrets,
- temporary bugs,
- guesses,
- raw logs,
- private customer data,
- anything you would not want another model/session to read.

Good memory prompt:

```text
Before planning, query memory for stable facts about this repo.
At the end, save only durable facts that will help future sessions.
Do not save secrets, logs, guesses, or one-off errors.
```

---

## 9. Hooks: turn preferences into enforcement

Hooks let you run commands or prompts at lifecycle events. They are compatible
with Claude Code hook format, so existing Claude hooks can often carry over.

Recommended standalone file:

```text
.devin/hooks.v1.json
```

Useful events:

| Event               | Use it for                               |
| ------------------- | ---------------------------------------- |
| `PreToolUse`        | block dangerous commands before they run |
| `PostToolUse`       | log commands or validate outputs         |
| `PermissionRequest` | auto-approve safe commands               |
| `UserPromptSubmit`  | inject context for certain requests      |
| `Stop`              | prevent stopping before required checks  |
| `PostCompaction`    | log/reinject context after compaction    |
| `SessionStart`      | run setup/context scripts                |
| `SessionEnd`        | cleanup or final logging                 |

Example: block destructive shell commands.

```json
{
  "PreToolUse": [
    {
      "matcher": "exec",
      "hooks": [
        {
          "type": "command",
          "command": "python3 scripts/block-dangerous-command.py"
        }
      ]
    }
  ]
}
```

Hooks receive JSON on stdin and can return JSON like:

```json
{
  "decision": "block",
  "reason": "Destructive command blocked by project policy"
}
```

Use `/hooks` to verify what loaded.

Do not create hook loops. A `Stop` hook that always blocks will trap the agent
forever.

---

## 10. Cloud handoff: local when interactive, cloud when durable

Use local DevinCLI for fast iteration where your repo and shell already live.
Move to cloud Devin when the task needs a separate machine, long-running work,
browser testing, recordings, PR follow-through, or you want to close your
laptop.

Current handoff paths include:

- `/handoff` when available in your build,
- typing `&` on an empty prompt to enter handoff mode,
- handing off a plan when exiting plan mode,
- `devin cloud drs` commands for environment blueprints, sandbox sessions, and
  builds.

Handoff-ready prompt:

```text
Prepare this for cloud handoff:
- current goal
- branch and changed files
- exact commands already run
- remaining checks
- known blockers
- success criteria
- anything the cloud agent must not do
```

Local is best for tight feedback. Cloud is best for persistence.

---

## 11. Model routing: stop using one model for everything

DevinCLI supports frequent model releases from Anthropic, OpenAI, Google,
Cognition, and open-source providers. Short names like `opus`, `sonnet`, `swe`,
`codex`, and `gemini` resolve to the latest model in that family.

Switch models:

```bash
devin --model opus -- "plan this migration"
```

```text
/model
/model swe
/model opus
```

Some models support thinking levels. Cycle them with `Alt+T` (`Opt+T` on macOS).

Practical routing:

- architecture, risky refactors, final synthesis: strongest model.
- broad read-only repo research: fast/cheap model or explore subagents.
- lint fixes and mechanical edits: `swe` / fast coding model.
- security review: stronger model.
- summarization and status updates: cheaper model.

Do not hard-code viral pricing claims in a public README. Promos change. Check
the current model picker and billing UI before publishing exact discount/free
model claims.

---

## 12. New commands and tiny quality-of-life wins

Recent high-signal features worth actually using:

```text
/btw <question>
```

Ask a side question using current context without adding it to the main
conversation.

```text
/loop <prompt>
```

Run a prompt and auto-review the diff in a loop. Start from a clean git state.

```text
/copy
```

Copy the last agent response to the clipboard.

```text
/steps
/revert
/fork
```

Inspect steps, rewind changes, or fork a session.

```text
/usage
/context
/compact
```

Watch quota/context and force compaction when needed.

```text
/org
```

Switch Devin organizations from the terminal.

`Ctrl+R` opens fuzzy search over previous prompts. Number keys can select
numbered options in prompts. These are small, but they make the CLI feel much
faster.

The agent can also use `web_search` during sessions, which is a major upgrade
for debugging fresh libraries, API changes, and release notes.

---

## 13. A complete power-user repo layout

```text
your-project/
├── AGENTS.md
├── .devin/
│   ├── config.json
│   ├── config.local.json      # gitignored; secrets and personal MCP
│   ├── hooks.v1.json
│   ├── skills/
│   │   ├── review-pr/
│   │   │   └── SKILL.md
│   │   ├── deep-research/
│   │   │   └── SKILL.md
│   │   └── run-checks/
│   │       └── SKILL.md
│   └── agents/
│       ├── fast-researcher/
│       │   └── AGENT.md
│       └── reviewer/
│           └── AGENT.md
└── scripts/
    └── block-dangerous-command.py
```

Minimal shared config:

```json
// .devin/config.json
{
  "permissions": {
    "allow": [
      "Read(**)",
      "Exec(git status)",
      "Exec(git diff)",
      "Exec(npm run)",
      "Exec(pnpm run)"
    ],
    "deny": ["Exec(sudo)", "Exec(rm)", "Write(.env*)"]
  },
  "read_config_from": {
    "cursor": true,
    "windsurf": true,
    "claude": true,
    "opencode": true,
    "vscode": true,
    "zed": true
  }
}
```

Personal local config:

```json
// .devin/config.local.json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "env": {
        "MEMORY_FILE_PATH": "/absolute/path/to/devin-memory.jsonl"
      }
    }
  }
}
```

Never commit local config if it contains secrets.

---

## 14. Copy/paste prompts that work

### Repo map

```text
Use read-only subagents for research.
Map this repo into:
1. main entrypoints
2. core domain modules
3. test strategy
4. risky files
5. commands required before PR

Return exact file paths and line numbers.
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
Do not edit files yet.
```

### Smart lead, fast workers

```text
Act as the lead architect.
Use fast read-only subagents for code search and evidence gathering.
Keep implementation decisions in the root conversation.
After subagents report, decide the minimal safe diff.
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

### Review before PR

```text
Before opening a PR:
1. inspect the full diff
2. run the narrowest relevant checks
3. identify risky assumptions
4. remove accidental files
5. write a concise PR summary with test evidence
```

---

## 15. Anti-slop checklist

Keep:

- exact commands,
- exact config paths,
- workflows readers can copy,
- warnings about permissions and secrets,
- links to official docs,
- claims that survive changelogs.

Cut:

- fake token-per-second math,
- unsupported pricing/promo claims,
- model names that do not exist in docs,
- "10x" language,
- giant personality prompts,
- MCP configs that commit secrets,
- examples that ask agents to "make it better."

The viral version is not the loudest version. It is the guide people can copy in
10 minutes and feel the power immediately.

---

## Sources

- [Devin for Terminal quickstart](https://cli.devin.ai/docs)
- [Stable changelog](https://cli.devin.ai/docs/changelog/stable)
- [Commands and slash commands](https://cli.devin.ai/docs/reference/commands)
- [Keyboard shortcuts](https://cli.devin.ai/docs/reference/keyboard-shortcuts)
- [Shell integration](https://cli.devin.ai/docs/shell-integration)
- [Configuration](https://cli.devin.ai/docs/extensibility/configuration)
- [Configuration import](https://cli.devin.ai/docs/reference/configuration/read-config-from)
- [Permissions](https://cli.devin.ai/docs/reference/permissions)
- [Rules and `AGENTS.md`](https://cli.devin.ai/docs/extensibility/rules)
- [Skills overview](https://cli.devin.ai/docs/extensibility/skills/overview)
- [Creating skills](https://cli.devin.ai/docs/extensibility/skills/creating-skills)
- [Subagents and custom subagents](https://cli.devin.ai/docs/subagents)
- [MCP overview](https://cli.devin.ai/docs/extensibility/mcp/overview)
- [MCP configuration](https://cli.devin.ai/docs/extensibility/mcp/configuration)
- [Hooks](https://cli.devin.ai/docs/extensibility/hooks/overview)
- [Lifecycle hooks](https://cli.devin.ai/docs/extensibility/hooks/lifecycle-hooks)
- [Models](https://cli.devin.ai/docs/models)
- [MCP memory server](https://www.npmjs.com/package/@modelcontextprotocol/server-memory)
