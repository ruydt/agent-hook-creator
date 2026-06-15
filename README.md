# agent-hook-creator

A skill that generates **production-ready hooks for any AI coding agent** — Claude
Code, OpenAI Codex, Cursor, Gemini CLI, GitHub Copilot CLI, Cline, Goose,
Windsurf, OpenCode, Amp, and more.

Agent hook systems differ a lot and change fast. Instead of writing a hook from
stale memory, this skill **researches the specific agent's current hook contract
first** (its events, how the payload is delivered, how to register, the
exit-code / JSON return semantics), asks the few questions that matter (which
agent, which event, OS, runtime, fail-open vs fail-closed), then writes the hook
with defensive input parsing, clean stdout/protocol handling, correct exit codes,
logging, and fail-open-on-error safety — plus how to register it, how to test it,
and the pitfalls for that agent and event.

## Install

Via [skills.sh](https://skills.sh):

```bash
npx skills add ruydt/agent-hook-creator
```

Or manually — copy this folder into your agent's skills directory (e.g.
`~/.claude/skills/agent-hook-creator/`).

## What it does

Give it a goal in plain words and it produces the complete hook for that agent
and event:

- "stop Claude Code from running `rm -rf`" → a `PreToolUse` Bash guard
- "run prettier after the agent edits a file" → a `PostToolUse` formatter
- "block Cursor from reading or editing my `.env`" → a Cursor hook
- "Windsurf: block `git push --force`" → a Windsurf `pre_run_command` hook
- "ping me on Slack when the agent finishes" → a `Stop` notifier

## What's inside

- `SKILL.md` — the research-first workflow + universal quality bar + output format.
- `references/claude-code-and-codex.md` — authoritative per-event deep-dive for the
  two most common targets (events, stdin payload, registration, exit-code + JSON
  contract, a worked guard, testing).
- `references/researching-any-agent.md` — the playbook for any other / unknown
  agent (find the docs → extract the 5 things → verify with `--version` + a live
  echo hook → cross-agent invariants), a catalog of 12 agents' hook models, and
  fallbacks (git hooks / wrapper script / MCP / rules) when an agent has none.
- `evals/evals.json` — the test cases used to validate the skill.

## Supported agents

Claude Code and Codex (deep-dive), plus a verified catalog covering Cursor, Gemini
CLI, GitHub Copilot CLI, Cline, Goose, Windsurf (exit-code-only contract),
OpenCode & Amp (in-process TS plugins), Aider (lint/test pipeline), and Continue
(no hook system → use the documented fallbacks). Hook APIs move fast, so the skill
always verifies against the agent's current docs rather than trusting memory.

## License

[MIT](./LICENSE)
