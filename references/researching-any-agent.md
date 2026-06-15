# Researching an unfamiliar agent's hook system

**Audience:** you are about to write a hook for some AI coding agent — maybe Claude Code or Codex (see `claude-code-and-codex.md`), maybe Cursor, Gemini CLI, Cline, Goose, Windsurf, or one you've never seen.

**Prime directive: do not trust your memory.** These hook APIs are young and change between minor versions — event names, payload shapes, and exit-code semantics all drift. Treat anything you "remember" as a hypothesis to verify, never a fact to ship. Read the docs for the version on disk, then confirm with a live echo. Code from evidence, not recall. (The catalog at the bottom of this file is a *starting map*, not a substitute for checking current docs.)

---

## 1. Find the hook docs — fast

Hit these in order; stop when you have the contract.

- **Official docs site.** Search the agent's docs for `hooks`, `events`, `lifecycle`, `extensibility`, `automation`, `triggers`, `callbacks`, `plugins`. The hooks page usually lists every event with payload and exit-code semantics in one table — the jackpot.
- **The config/settings schema.** The agent reads a config file; find it and read it. Common locations: `~/.<agent>/`, `~/.config/<agent>/`, or per-project `.<agent>/` (e.g. `settings.json`, `config.toml`, `config.yaml`). A shipped JSON Schema (`*.schema.json`) enumerates valid event keys and field types authoritatively.
- **The GitHub repo.** When docs are thin, source is ground truth: grep `docs/`, `examples/`, `README` for `hook`/`event`/`PreToolUse`/`on_*`/`lifecycle`; read **`CHANGELOG.md`** to learn *when* the hook API landed and *what changed*; find the code that **dispatches** hooks (where it spawns a subprocess, reads `stdin`, or checks an exit code) — that code cannot lie.
- **The installed package itself.** `npm root -g`, `pip show -f <pkg>`, or the binary's install dir often contain bundled docs/types newer than your training data.
- **Community.** Awesome-lists, GitHub Discussions/Issues, and existing hook repos give working examples — but treat them as hints, not spec.

> If you can't find a formal hook system after this, jump to **§5 Fallbacks**.

---

## 2. Extract these 5 things before writing a single line

Do not write the hook until you can answer all five for the **specific event** you're targeting.

**(a) Event/trigger names + when each fires.** The exact event identifiers and the precise moment each runs (before/after a tool call, on session start/stop, on user prompt, on file edit). Note ordering and whether multiple hooks can bind one event. Getting the *moment* wrong means the hook can't see the data it needs.

**(b) How the payload is delivered.** Pin down the channel — they are **not** interchangeable: **stdin JSON** (most common), **CLI args**, **environment variables**, **an HTTP request body** (daemon-style agents), or **in-process function args** (TS-plugin agents like OpenCode/Amp). Capture exact field names and nesting. Reading stdin when the agent passes args hangs forever; reading env when it sends stdin gets you nothing.

**(c) How to register the hook.** The exact config file path **and** the exact format/key. Inline command string, path to a script, or a `matcher` + command object? Absolute path required? Does the script need `chmod +x`? (Many agents, like Git, **silently skip** a non-executable hook.) Write down the literal snippet you'll add.

**(d) The return / exit-code contract — block vs allow vs inject.** The heart of the hook: exit `0` = allow/continue; which nonzero code **blocks** (often `2`, but verify); how to **inject context / modify** (usually structured JSON on stdout — a decision field, additional context, a replacement value). Know the difference between "deny this action," "allow but warn," and "feed text back to the model."

**(e) What stdout/stderr mean + any timeout.** Critically: **is stdout the control/protocol channel?** If so, every stray log line corrupts the contract — send logs to **stderr or a file** only. Confirm what stderr surfaces to the user. Find the **timeout** (agents kill slow hooks); keep the hook fast and bounded.

---

## 3. Verify against the *running* version

Docs lie about which version you have. Confirm against the binary on disk.

1. **`<agent> --version`** (or `-v`, `version`). Match it to the CHANGELOG entry where the hook feature/shape last changed.
2. **Inspect the config schema** the installed binary actually validates against — reject-on-unknown-key behavior reveals the real event names.
3. **Use the built-in tester / dry-run** if one exists (`--dry-run`, a `doctor`/`hooks test` subcommand, or a GUI simulator).
4. **Plant an echo hook — the single most reliable step.** Register a trivial hook that dumps the real input to a file, then trigger it once:
   ```bash
   #!/usr/bin/env bash
   # echo-hook: capture the REAL payload shape. Logs to a file, never stdout.
   { date -u +%FT%TZ; echo "ARGS: $*"; echo "ENV:"; env; echo "STDIN:"; cat; } \
     >> /tmp/agent-hook-capture.log 2>&1
   exit 0   # fail open — never block while probing
   ```
   Now you can **see** the actual delivery channel, field names, and nesting — not what you guessed. Build the real hook from this captured shape.

---

## 4. Cross-agent invariants (true even when specifics differ)

When details are uncertain, fall back to these — they hold across nearly every agent:

- **A JSON event on stdin** is the most common delivery. Read stdin, parse defensively, treat every field as optional.
- **stdout is usually a reserved control/protocol channel.** Send all logs to **stderr or a file**. Never `echo`/`print` diagnostics to stdout.
- **Exit codes drive control flow.** Nonzero typically blocks; structured stdout typically injects. Set the code deliberately — never let it fall through.
- **Fail open on your own bugs.** A hook that errors internally (missing `jq`, bad parse) should **allow**, not freeze the agent. A *gate* denies on a real positive match but still allows on its own internal error — a buggy hook must never brick the agent.
- **Never hardcode secrets.** Read tokens from the environment; keep them out of the script and out of logs.
- **Be fast and idempotent.** Respect the timeout; assume the event can fire repeatedly.

---

## 5. Fallback: the agent has no formal hook system

If §1–§2 turn up nothing (e.g. Continue.dev, or Aider beyond its lint/test pipeline), get the behavior another way (rough order of preference):

- **Git hooks.** If the goal is "do X around commits/pushes/edits to the repo," `pre-commit`, `commit-msg`, `pre-push`, or `post-merge` are agent-agnostic and always available. Best when the trigger is a git event, not an agent event.
- **Wrapper script around the agent binary.** Shadow the agent with a script of the same name earlier on `PATH` (or an alias) that runs setup/guards, `exec`s the real binary, then runs teardown. Crude before/after hooks for any tool.
- **MCP server.** If the agent speaks MCP, expose your logic as an MCP tool/server. You don't intercept lifecycle events, but you gate and observe what the agent does through your tools — often the cleanest integration point.
- **The agent's custom-command / rules / config mechanism.** Many agents without "hooks" still have rules files, custom slash-commands, declarative tool-permission policies, or `on_*` config keys that approximate the behavior. Check those before concluding there's no extension point.

> State explicitly which fallback you chose and what it does and does not capture compared to a real hook — the user needs to know the seams.

---

## Known-agent catalog (a starting map — verify against current docs)

As of mid-2026. Hook systems are moving fast; **confirm event names, fields, and exit semantics against the live docs and the installed version** before you rely on any row.

| Agent | Hooks? | Payload | Block / control | Register | Docs |
|---|---|---|---|---|---|
| **Claude Code** | yes (richest) | stdin JSON (HTTP/MCP/prompt variants too) | exit 2 blocks; stdout JSON `hookSpecificOutput.permissionDecision` allow/deny/ask/defer; `additionalContext` injects | `hooks{}` in `~/.claude/settings.json`, `.claude/settings.json[.local]` | code.claude.com/docs/en/hooks |
| **OpenAI Codex** | yes (flagged) | stdin JSON (hooks); legacy `notify` = one JSON **argv** arg | exit 2 / `permissionDecision` / `decision:block` (hooks). `notify` is fire-and-forget, can't block | `~/.codex/hooks.json` or `[hooks]` in `config.toml`; enable `features.hooks` + trust via `/hooks` | developers.openai.com/codex/hooks |
| **Cursor** | yes (beta) | stdin JSON | exit 2 blocks; stdout JSON `{permission: allow/deny/ask, ...}`; **fail-open by default** unless `failClosed:true` | `hooks.json` (project `.cursor/`, user `~/.cursor/`, enterprise) | cursor.com/docs/hooks |
| **Windsurf** (→ Devin Desktop) | yes | stdin JSON | **exit code only** (2 = block on `pre_*`); **no stdout decision**, no input rewrite, no "ask" | `hooks.json` (`.windsurf/`, `~/.codeium/windsurf/`, system) | docs.windsurf.com → docs.devin.ai/desktop/cascade/hooks |
| **Cline** | yes (v3.36+) | stdin JSON | stdout JSON `{cancel, contextModification, errorMessage}`; `cancel:true` blocks. macOS/Linux only | filesystem-discovered: executable named after the event in `~/Documents/Cline/Rules/Hooks/` or `.clinerules/hooks/` | docs.cline.bot/features/hooks |
| **Gemini CLI** | yes ("Hooks v1") | stdin JSON | exit 2 blocks; stdout JSON `{decision: deny, reason}`; **stdout must be JSON-only ("silence mandatory")** | `hooks{}` in `.gemini/settings.json` / `~/.gemini/settings.json` | geminicli.com/docs/hooks/reference |
| **GitHub Copilot CLI** | yes | stdin JSON (HTTP too) | exit 2 (deny for `permissionRequest`); stdout JSON `{permissionDecision...}`. **`preToolUse` is fail-closed**; camelCase vs PascalCase modes | many JSON sources (policy dir, `.github/hooks/`, `~/.copilot/`) | docs.github.com/en/copilot/reference/hooks-reference |
| **Goose** (Block) | yes (new) | stdin JSON | exit 2 or stdout JSON `{decision:block, reason}` (Claude-Code-style) | Open Plugins: `~/.agents/plugins/<name>/hooks/hooks.json` | goose-docs.ai (hooks blog) |
| **OpenCode** | yes (in-process) | **JS/TS function args** (not stdin) | `throw` to block; mutate the `output` object to modify | TS modules in `~/.config/opencode/plugins/` or `.opencode/plugins/` | opencode.ai/docs/plugins |
| **Amp** (Sourcegraph) | yes (in-process) | **TS function args** (not stdin) | handler returns `{action: allow/reject-and-continue/modify/synthesize}` | TS plugins in `.amp/plugins/` (`amp.on(...)`) | ampcode.com/manual (Plugins) |
| **Aider** | partial | CLI args (lint/test cmd) | exit code only; non-zero → auto-fix attempt (no block-before gate); git `pre-commit` only with `--git-commit-verify` | flags / env / `.aider.conf.yml` (`--lint-cmd`, `--test-cmd`) | aider.chat/docs/usage/lint-test |
| **Continue.dev** | **no** | N/A | declarative Tool Permissions only (allow/ask/exclude) — no executable callback | `config.yaml`, `permissions.yaml` | docs.continue.dev (use MCP / git hooks as fallback) |

**Pattern families:** (1) *Claude-Code-style* — stdin JSON + exit-2-blocks + stdout-JSON-decision (Claude Code, Codex, Cursor, Gemini CLI, Copilot CLI, Goose, ~Cline). (2) *Exit-code-only* — Windsurf (block via exit 2, no stdout decision). (3) *In-process TS plugin* — OpenCode, Amp (throw/return, not stdin/exit). (4) *No event bus* — Continue (declarative), Aider (lint/test + git hooks). Identify which family the target falls into first; it tells you most of the contract.
