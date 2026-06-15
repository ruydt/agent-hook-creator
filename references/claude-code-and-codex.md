# AI Coding Agent (Claude Code / Codex) Hooks

## Overview
Hooks are user-defined shell commands (or HTTP/MCP handlers) that an AI coding agent runs automatically at fixed points in its lifecycle — before/after a tool runs, when the user submits a prompt, when the agent stops, etc. The agent spawns the hook as a child process, writes a single JSON object describing the event to the hook's **STDIN**, and waits (up to a timeout) for it to finish. The hook influences the agent through two channels that you must not mix in one path: the process **exit code** (`0` = ok, `2` = block, other = non-blocking error) and a **structured JSON object printed to STDOUT** (`{ "decision": ..., "hookSpecificOutput": { "permissionDecision": ... }, "continue": ... }`). Because STDOUT carries the control protocol, anything you accidentally print there can corrupt the agent's decision parsing — the single most common way to brick a hook.

This file covers **Claude Code** (the authoritative reference) and **Codex**, which ships a near-identical hooks schema; both detect at runtime and share the same payload/exit-code contract.

## Hook types

> Registration for all Claude Code events: a `hooks` object in `~/.claude/settings.json` (global), `.claude/settings.json` (project, shareable), or `.claude/settings.local.json` (project, gitignored). Codex uses `~/.codex/hooks.json` or `<repo>/.codex/hooks.json`, or an inline `[hooks]` table in `config.toml`. Structure (both agents):
> ```json
> { "hooks": { "<EventName>": [ { "matcher": "<pattern>", "hooks": [ { "type": "command", "command": "/abs/path/hook.sh", "timeout": 10 } ] } ] } }
> ```
> `matcher` semantics: `"*"`, `""`, or omitted = all; bare names/`|`-lists (e.g. `Edit|Write`) = exact match; anything with regex chars = regex (e.g. `mcp__memory__.*`). The matcher filters the **tool name** for tool events, the **source/trigger** for session/compact events, and is **ignored** for `UserPromptSubmit`/`Stop`. Use `${CLAUDE_PROJECT_DIR}` to reference project-relative scripts portably.

### PreToolUse
- **Fires when:** Before a tool call executes (Bash, Edit, Write, Read, MCP tools, etc.). This is the gate event.
- **Receives:** STDIN JSON with common fields (`session_id`, `transcript_path`, `cwd`, `hook_event_name`, `permission_mode`) plus `tool_name` (string) and `tool_input` (object, tool-specific — e.g. `tool_input.command` for Bash, `tool_input.file_path` for Edit/Write).
- **Use for:** Denying dangerous commands, blocking writes to protected paths, auto-approving safe tools, injecting context about a decision, rewriting tool input.
- **Can it block/abort the action?:** **Yes.** Preferred: exit `0` + STDOUT `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny"|"allow"|"ask","permissionDecisionReason":"..."}}`. `deny` blocks and feeds the reason to the agent; `allow` bypasses the permission prompt; `ask` escalates to the user. Fallback: exit `2` with the reason on **STDERR** also blocks. (Do not emit both JSON and exit 2 in the same path — exit 2 makes the agent ignore STDOUT.)
- **Lives at / registered via:** `hooks.PreToolUse[].matcher` = tool name (e.g. `"Bash"`, `"Edit|Write"`).

### PostToolUse
- **Fires when:** After a tool call succeeds.
- **Receives:** Common fields + `tool_name`, `tool_input`, and `tool_output` (the tool's result).
- **Use for:** Auto-formatting/linting a file the agent just wrote, validating output, appending context, logging diffs.
- **Can it block/abort the action?:** The tool already ran, so it cannot un-run it. Exit `0` + `{"decision":"block","reason":"..."}` stops the agentic loop and feeds `reason` back so the agent reacts (e.g. "lint failed, fix it"). Exit `2` shows STDERR to the agent. You can also return `hookSpecificOutput.additionalContext`.
- **Lives at / registered via:** `hooks.PostToolUse[].matcher` = tool name.

### UserPromptSubmit
- **Fires when:** The user submits a prompt, before the agent processes it.
- **Receives:** Common fields + `prompt` (the user's text). No matcher — always fires.
- **Use for:** Injecting context (current branch, ticket, repo conventions) into the turn; blocking prompts that violate policy; redacting secrets.
- **Can it block/abort the action?:** **Yes.** Exit `0` + `{"decision":"block","reason":"..."}` blocks and erases the prompt; exit `2` + STDERR also blocks. To **inject context** without blocking: exit `0` + `{"hookSpecificOutput":{"hookEventName":"UserPromptSubmit","additionalContext":"..."}}` — the string is added alongside the prompt. (For this event only, plain STDOUT text on exit 0 is also added as context — but prefer the JSON form to be explicit.) Default timeout is shorter here (~30s).
- **Lives at / registered via:** `hooks.UserPromptSubmit`.

### SessionStart
- **Fires when:** A session begins or resumes.
- **Receives:** Common fields + `source` (`"startup"` | `"resume"` | `"clear"` | `"compact"`). Matcher filters on `source`.
- **Use for:** Loading project context, dumping git status / open PRs / TODOs into the session.
- **Can it block/abort the action?:** No meaningful block. Primary output is context injection: exit `0` + `{"hookSpecificOutput":{"hookEventName":"SessionStart","additionalContext":"..."}}`.
- **Lives at / registered via:** `hooks.SessionStart[].matcher` = `"startup"`/`"resume"`/etc.

### SessionEnd
- **Fires when:** The session terminates.
- **Receives:** Common fields + an end-reason (matcher filters on `"clear"`, `"logout"`, `"prompt_input_exit"`, `"other"`, etc.).
- **Use for:** Cleanup, flushing logs, persisting state, notifications. No decision control.
- **Can it block/abort the action?:** No.
- **Lives at / registered via:** `hooks.SessionEnd`.

### Stop
- **Fires when:** The agent finishes responding (end of a turn).
- **Receives:** Common fields (+ a stop reason). No matcher.
- **Use for:** Enforcing "did you actually run the tests?" gates, continuing work, desktop notifications on turn completion.
- **Can it block/abort the action?:** **Yes.** Exit `0` + `{"decision":"block","reason":"..."}` (or exit `2` + STDERR) **prevents the agent from stopping** and feeds `reason` back so it keeps working. Guard against infinite loops.
- **Lives at / registered via:** `hooks.Stop`.

### SubagentStop
- **Fires when:** A spawned subagent finishes.
- **Receives:** Common fields + `agent_type`. Matcher filters on agent type.
- **Use for:** Validating subagent results before they return to the main agent.
- **Can it block/abort the action?:** **Yes** — same mechanism as `Stop` (`decision:"block"` or exit 2 keeps the subagent going).
- **Lives at / registered via:** `hooks.SubagentStop`.

### Notification
- **Fires when:** The agent emits a notification (e.g. it needs permission, or has been idle).
- **Receives:** Common fields + `notification_type` and `message`. Matcher filters on type.
- **Use for:** Desktop/Slack pings, OS notifications. No decision control.
- **Can it block/abort the action?:** No — observability only.
- **Lives at / registered via:** `hooks.Notification`.

### PreCompact
- **Fires when:** Before context compaction. Matcher filters `"manual"` | `"auto"`.
- **Receives:** Common fields + a compaction trigger.
- **Use for:** Snapshotting the transcript before it's summarized.
- **Can it block/abort the action?:** **Yes** in Claude Code (exit 2 / `decision:"block"` blocks compaction); treat as observability if unsure.
- **Lives at / registered via:** `hooks.PreCompact`.

> **Note (Claude Code is expanding fast — verify before relying on a specific event).** The nine events above are the stable, portable core. Recent builds have added many more — `Setup`, `UserPromptExpansion`, `PermissionRequest`, `PermissionDenied`, `PostToolUseFailure`, `PostToolBatch`, `MessageDisplay`, `SubagentStart`, `TaskCreated`, `TaskCompleted`, `StopFailure`, `TeammateIdle`, `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`, `PostCompact`, `Elicitation`/`ElicitationResult` — plus new handler types (`http`, `mcp_tool`, `prompt`, `agent`), an `if` permission-rule filter, `${tool_input.field}` arg substitution, a `defer` value for `permissionDecision`, and new `permission_mode` values (`auto`, `dontAsk`). Several are explicitly experimental and their JSON schemas may change. Which events can be blocked by exit-2 is event-specific (e.g. `PostToolUse`, `Notification`, `SessionStart` cannot block). Do **not** assume an event exists in an older build — re-verify against `https://code.claude.com/docs/en/hooks` (the old `docs.claude.com/.../hooks` URL 301-redirects there) for the installed version. For any agent other than Claude Code / Codex, use `references/researching-any-agent.md`.

> **Codex specifics & cross-agent pattern:** Codex has **two** mechanisms. (1) A real lifecycle-hook system deliberately modeled on Claude Code — same event names, same STDIN-JSON / exit-2 / `hookSpecificOutput.permissionDecision` / `decision:"block"` contract — but it is **gated behind a feature flag (`features.hooks`) and a per-hook trust step** (`/hooks` in the CLI; `--dangerously-bypass-hook-trust` to skip once). It's registered in `~/.codex/hooks.json` or an inline `[[hooks.<Event>]]` table in `~/.codex/config.toml` (and the `<repo>/.codex/` equivalents); only `type="command"` handlers actually run today (`prompt`/`agent` are parsed but skipped); matcher is a regex (`"^Bash$"`). It may not exist in older builds — check `codex --version`. (2) The **legacy `notify`** key in `config.toml` (`notify = ["python3", "/path/notify.py"]`) runs a program with a **single command-line JSON argument** (NOT stdin), only on turn completion, and is **fire-and-forget — its exit code and output are ignored, so it cannot block or alter the turn.** The notify payload's key casing has drifted across versions (`turn-id` vs `turn_id`, presence of `cwd`); verify against `developers.openai.com/codex/config-reference` before parsing it. **To write one hook that works in both agents, detect the agent at runtime:** Claude Code sets `CLAUDECODE=1` (and writes transcripts under `~/.claude/`); branch on `$CLAUDECODE` only for cosmetic differences, since the hook JSON contract is shared.

## Generation checklist (domain-specific)
- **Read the event payload from STDIN as a single JSON object** (Claude Code & Codex hooks). The legacy Codex `notify` program is the exception — it gets one JSON string as `argv[1]`. Never assume args/env for the payload.
- **Keep STDOUT clean.** The only thing allowed on STDOUT is the control JSON (or nothing). All logs, debug prints, and progress go to STDERR or a log file. One stray `print`/`echo`/`console.log` to STDOUT corrupts the protocol.
- **Pick ONE control channel per code path:** either exit `0` + JSON on STDOUT, or a nonzero exit + STDERR. Exit `2` makes the agent ignore STDOUT entirely.
- **Use the right block field for the event:** `PreToolUse` → `hookSpecificOutput.permissionDecision` (`allow`/`deny`/`ask`); `PostToolUse`/`Stop`/`SubagentStop`/`UserPromptSubmit` → `decision:"block"` + `reason`. `additionalContext` (in `hookSpecificOutput`) is how you inject text without blocking.
- **Match the matcher to the event class:** tool name for tool events, source/trigger for session/compact events, and remember `UserPromptSubmit`/`Stop`/`Notification` ignore the matcher.
- **Set a sane `timeout`** in the registration; the agent blocks the turn while the hook runs. Default is ~30s for `UserPromptSubmit`/`prompt` hooks, longer elsewhere.
- **Treat every field as optional.** `tool_input` shape differs per tool; `transcript_path` can be `null`; subagent-only fields are absent in the main session. Use safe lookups with defaults.
- **`chmod +x` the script** and give it a correct shebang; reference it by absolute path or `${CLAUDE_PROJECT_DIR}/...`.
- **Decide fail-open vs fail-closed deliberately:** a logger/notifier must fail OPEN (exit 0, never block real work); a security gate is a judgment call — fail OPEN on *internal* errors (so a buggy hook can't brick the agent) but fail CLOSED (deny) on a *positive* dangerous-pattern match.

## Common pitfalls
- **Writing debug logs to STDOUT** -> the agent parses your "DEBUG: starting" line as control JSON, gets garbage, and may behave unpredictably. Fix: log to STDERR or a file; STDOUT is reserved for the one control JSON.
- **Emitting JSON *and* exiting 2** -> exit 2 discards STDOUT, so your carefully built `permissionDecision` is ignored and only STDERR is used. Fix: choose one channel per path.
- **Blocking when you meant to warn** -> using `decision:"block"` / `permissionDecision:"deny"` for a soft suggestion halts the agent. Fix: use `additionalContext` and exit 0 to surface advice without blocking.
- **Not failing open on internal errors** -> a `JSON.parse` throw or a missing dependency makes the hook exit nonzero, which on a `PreToolUse`/`Stop` path can block every action and effectively freeze the agent. Fix: wrap everything in try/catch and `exit 0` on internal failure.
- **Wrong matcher** -> `matcher:"bash"` (lowercase) never matches the `Bash` tool; a regex like `Edit` accidentally matches `MultiEdit`/`NotebookEdit`. Fix: use exact `|`-lists (`Edit|Write`) and anchor regexes (`^Bash$`).
- **Assuming a field exists** -> reading `tool_input.command` for a `Write` event (it's `file_path` there) yields `undefined` and crashes downstream. Fix: check `tool_name` first, then read the field that tool actually provides.
- **Infinite Stop loops** -> a `Stop` hook that always returns `decision:"block"` prevents the agent from ever ending. Fix: gate the block on a real, resolvable condition and add a counter/sentinel.
- **Hardcoding the agent assumption** -> assuming `notify`-style argv input when the agent actually delivers STDIN JSON. Fix: hooks = STDIN JSON; only Codex legacy `notify` = argv JSON.

## Worked example
**Scenario:** A `PreToolUse` guard for the `Bash` tool that denies clearly destructive commands (`rm -rf /`, `rm -rf ~`, disk-wiping `dd`, fork bombs, recursive chmod on root). It denies via the structured `permissionDecision:"deny"` path, falls back to exit 2 + STDERR if it can't emit JSON, **fails open** (allows) on any internal error so a bug can't freeze the agent, and logs every decision with timestamps to a file (never STDOUT).

```bash
#!/usr/bin/env bash
# pretooluse-bash-guard.sh
# Claude Code / Codex PreToolUse hook for the Bash tool.
# Contract: payload arrives as one JSON object on STDIN.
#   - Block via STDOUT JSON  -> {"hookSpecificOutput":{...,"permissionDecision":"deny",...}}
#   - Fallback block         -> exit 2 with reason on STDERR
#   - STDOUT is reserved for control JSON ONLY. Logs go to a file/STDERR.
# chmod +x this file and register it (see install block below).

set -uo pipefail   # NOT -e: we must control exit codes ourselves and fail open.

LOG="${HOOK_LOG:-${TMPDIR:-/tmp}/bash-guard.log}"
log() { printf '%s [bash-guard] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S%z')" "$*" >>"$LOG" 2>/dev/null; }  # to file, never STDOUT

# --- Fail OPEN: allow the tool, exit clean. Used for any INTERNAL error, so a
# --- buggy guard can never brick the agent. ---
allow_open() {
  log "ALLOW(open): $1"
  exit 0   # exit 0 + no JSON => agent proceeds with the normal permission flow
}

# --- Deny: emit structured JSON (preferred) and exit 0. Build the JSON with jq
# --- so the reason is escaped correctly. If jq is unavailable, fall back to
# --- exit 2 + STDERR, which also blocks. ---
deny() {
  local reason="$1"
  log "DENY: $reason"
  if command -v jq >/dev/null 2>&1; then
    jq -nc --arg r "$reason" \
      '{hookSpecificOutput:{hookEventName:"PreToolUse",permissionDecision:"deny",permissionDecisionReason:$r}}'
    exit 0
  fi
  # Fallback channel: exit 2 makes the agent read STDERR and block.
  echo "BLOCKED: $reason" >&2
  exit 2
}

# 1) Read the entire STDIN payload defensively (empty/missing must not crash).
PAYLOAD="$(cat 2>/dev/null || true)"
[ -z "$PAYLOAD" ] && allow_open "empty payload"

# 2) Extract tool_name and the Bash command. We need jq to parse reliably; if it
#    is absent, do NOT guess — fail open.
if command -v jq >/dev/null 2>&1; then
  TOOL="$(printf '%s' "$PAYLOAD" | jq -r '.tool_name // empty' 2>/dev/null)"
  CMD="$(printf '%s' "$PAYLOAD"  | jq -r '.tool_input.command // empty' 2>/dev/null)"
else
  allow_open "jq not available; cannot parse payload"
fi

# 3) Only police the Bash tool. Anything else: not our job, allow.
[ "$TOOL" = "Bash" ] || allow_open "tool=$TOOL (not Bash)"
[ -z "$CMD" ]        && allow_open "no command field"

log "inspect: $CMD"

# 4) Positive match on a dangerous pattern => fail CLOSED (deny). Patterns are
#    intentionally conservative to avoid false positives.
case "$CMD" in
  *"rm -rf /"*|*"rm -fr /"*)                     deny "refusing recursive force-delete of /" ;;
  *"rm -rf ~"*|*'rm -rf $HOME'*)                 deny "refusing recursive force-delete of home" ;;
  *":(){ :|:& };:"*)                             deny "fork bomb detected" ;;
  *"dd if=/dev/zero of=/dev/"*|*"dd if=/dev/random of=/dev/"*) deny "raw disk overwrite via dd" ;;
  *"mkfs."*"/dev/"*)                             deny "filesystem creation on a device" ;;
  *"chmod -R 777 /"*)                            deny "recursive 777 on root" ;;
  *"> /dev/sda"*|*"of=/dev/sd"*)                 deny "write to raw block device" ;;
esac

# 5) Nothing matched: allow normally.
allow_open "no dangerous pattern matched"
```

```bash
# Install: make executable + register as a PreToolUse hook for Bash.
chmod +x ./pretooluse-bash-guard.sh
mkdir -p .claude && python3 - <<'PY'
import json, os, pathlib
p = pathlib.Path(".claude/settings.json")
s = json.loads(p.read_text()) if p.exists() else {}
s.setdefault("hooks", {}).setdefault("PreToolUse", []).append(
    {"matcher": "Bash", "hooks": [{"type": "command", "command": os.path.abspath("pretooluse-bash-guard.sh"), "timeout": 10}]})
p.write_text(json.dumps(s, indent=2)); print("registered")
PY

# Test (should print a deny JSON and exit 0):
echo '{"hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"rm -rf /"}}' | ./pretooluse-bash-guard.sh; echo "exit=$?"
```

## Test
- **Direct STDIN pipe (fastest):** feed a synthetic payload and inspect STDOUT/exit code, exactly as the agent would:
  ```bash
  echo '{"hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"ls -la"}}' | ./hook.sh; echo "exit=$?"
  echo '{"hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"rm -rf /"}}' | ./hook.sh; echo "exit=$?"
  ```
  Verify: the dangerous case prints valid JSON with `"permissionDecision":"deny"` and exits 0 (or exits 2 in the fallback path); the safe case exits 0 with no JSON. Confirm STDOUT contains **only** JSON — pipe it to `jq .` to prove it parses.
- **Malformed/edge inputs:** pipe `''`, `not json`, `{}`, and a non-Bash event (`{"tool_name":"Read","tool_input":{"file_path":"x"}}`) — all must exit 0 (fail open), never crash.
- **Built-in simulator:** Claude Code's `/hooks` command lists and lets you trust/inspect registered hooks; Codex's `/hooks` does the same and gates trust. Use `--dangerously-bypass-hook-trust` (Codex) only for throwaway local testing.
- **Live trigger:** with the hook registered, run `claude` (or `codex`) and issue an action that matches — e.g. ask it to run a Bash command — then `tail -f $HOOK_LOG` for timestamped decisions. Watch the log, not STDOUT, since STDOUT is consumed by the agent.

Sources: [Claude Code — Hooks](https://code.claude.com/docs/en/hooks), [Claude Code — Hooks reference](https://code.claude.com/docs/en/hooks-reference), [Claude Code — Settings](https://code.claude.com/docs/en/settings).
