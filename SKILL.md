---
name: agent-hook-creator
description: >-
  Generate a production-ready hook script for ANY AI coding agent — Claude Code,
  OpenAI Codex, Cursor, Gemini CLI, Copilot CLI, Cline, Goose, Windsurf, and
  others. Agent hook systems differ a lot and change fast, so this skill
  RESEARCHES the specific agent's current hook contract first (events, how the
  payload is delivered, how to register, exit-code/JSON returns) instead of
  guessing from memory, asks what it needs (which agent, which event/behavior, OS,
  runtime, fail-open vs fail-closed), then writes the hook with defensive input
  parsing, clean stdout/protocol handling, correct exit codes, logging, and
  fail-open-on-error safety — plus registration, a test command, and pitfalls. Use
  whenever the user wants to create, write, scaffold, fix, or harden an AI
  coding-agent hook — e.g. a PreToolUse guard blocking dangerous commands, a
  PostToolUse auto-formatter, a Stop notifier, or a UserPromptSubmit context
  injector — even when they only describe the behavior without naming the event or
  agent.
license: MIT
metadata:
  author: ruydt
  version: '2.0'
---

# Hook Creator

Generate a production-ready **hook for an AI coding agent** — a script the agent
runs for you at a lifecycle moment (before a tool runs, after an edit, when the
agent stops, when the user submits a prompt) to block, observe, inject context,
or notify.

The catch that defines this skill: **every agent's hook system is different, and
they change fast.** Claude Code, Codex, Cursor, Gemini CLI, and Copilot CLI pass
a JSON event on stdin and block via exit code 2; Windsurf blocks via exit code
only (no stdout decision); OpenCode and Amp don't spawn a script at all — they
load in-process TypeScript plugins; Continue has no hook system; Aider only has a
lint/test pipeline. Within a single agent the event list grows between minor
versions. **So you cannot write this hook from memory.** The core discipline of
this skill is: *research the specific agent's current contract first, then write.*

## Workflow

Four steps. Step 2 is non-negotiable — skipping it is how you ship a hook that
silently never fires, hangs on stdin the agent never sends, or corrupts the
agent's decision protocol.

### 1. Clarify the target

Figure out what you're building before you research. Infer what you can from
context (the repo, an open config file, what the user is using), then ask **only
the questions whose answers change the hook**. The ones that usually matter:

- **Which agent?** Claude Code, Codex, Cursor, Gemini CLI, Copilot CLI, Cline,
  Goose, Windsurf, OpenCode, Amp, …? This determines the *entire* contract. If
  unknown, ask — don't assume Claude Code.
- **Which event / what should trigger it?** The user often describes behavior
  ("stop the agent from running `rm -rf`", "format files after the agent edits
  them", "ping me when it finishes") — map that to the agent's event in step 2.
- **What should the hook do**, and is it a **gate** (can block) or an
  **observer/notifier** (just reacts)? This drives the fail-open/closed choice.
- **Language / runtime** — bash, node, python? Is it installed on the target?
- **OS** — macOS (bash 3.2!), Linux, Windows? Affects shebang, shell features,
  and whether the agent needs a separate Windows command (e.g. Windsurf's
  `powershell` field, Codex's `command_windows`).

Don't interrogate. If the user said "a Claude Code PreToolUse hook that blocks
dangerous bash," that's enough — research and build. Ask when the agent is
unstated, or a gate's fail posture is genuinely ambiguous.

### 2. Research the agent's hook contract — do not trust memory

This is the heart of the skill. Establish the real, current contract for **this
agent and this event** before writing a line:

- **Claude Code or Codex** → read `references/claude-code-and-codex.md` (the
  authoritative deep-dive). Still sanity-check event names against the installed
  version — Claude Code adds events frequently.
- **Any other agent** → read `references/researching-any-agent.md`. It has the
  research playbook (where to find the docs, the **5 things to extract**, how to
  verify with `--version` and a live "echo hook") and a grounded catalog of known
  agents (Cursor, Gemini CLI, Copilot CLI, Cline, Goose, Windsurf, OpenCode, Amp,
  Aider, Continue) as a starting map — *to confirm against current docs, not to
  copy blind.*

By the end of step 2 you must be able to state, for the target event: **(a)** the
exact event name and when it fires, **(b)** how the payload arrives (stdin JSON /
argv / env / HTTP / in-process args), **(c)** how to register the hook (exact
config path + format), **(d)** the return contract (which exit code blocks, what
stdout JSON allows/denies/injects), and **(e)** what stdout/stderr mean + the
timeout. If you can't answer all five from docs, plant an echo hook (see the
playbook) and read the real payload off disk.

### 3. Generate to the universal quality bar

Write the hook so it matches the researched contract exactly and clears every
point of the quality bar below.

### 4. Deliver in the standard output format

Script + how to register it + how to test it without waiting for the real event +
the pitfalls for this agent and event. Format under "Output format" below.

## Universal quality bar

These hold across agents even though the specifics differ. Each is about *why*,
not ceremony:

1. **Use the payload channel the research established — never assume.** Most
   agents send a JSON event on **stdin**; some use argv (Codex's legacy `notify`),
   env vars, an HTTP body, or in-process function args (OpenCode/Amp). Reading
   stdin when the agent passes argv hangs forever; reading argv when it sends
   stdin gets nothing.
2. **Keep the control channel clean.** For most agents **stdout is the protocol
   channel** — the agent parses it as the decision/JSON. Send every log, debug
   line, and progress message to **stderr or a file**. One stray `echo`/`print`
   to stdout is the single most common way to brick a hook. (Gemini CLI even
   documents "silence on stdout is mandatory.")
3. **Use the agent's exact block/allow/inject mechanism, and don't mix channels.**
   Per the researched contract: the right exit code to block (often 2), and/or
   the right stdout JSON shape (`permissionDecision`, `decision:"block"`,
   `{permission:...}`, `{cancel:true}`, a thrown error for in-process plugins).
   Note channel interactions — e.g. on Claude Code, exit 2 makes the agent ignore
   stdout, so don't emit a JSON decision *and* exit 2 on the same path.
4. **Fail OPEN on your own errors.** A hook that throws internally (missing `jq`,
   a parse failure, an absent dependency) should **allow** and exit cleanly — a
   buggy hook must never freeze the agent. A *gate* fails CLOSED (denies) on a
   real positive match, but still fails OPEN on its own internal error. Avoid
   `set -e` patterns that abort mid-script into an accidental nonzero exit.
5. **Parse defensively.** Field shapes differ per event, per agent, and per
   version. Treat every field as optional; check the tool name before reading a
   tool-specific field; default safely.
6. **Timestamped logging to a file or stderr** — so you can see what the hook
   decided without polluting the protocol channel.
7. **Correct shebang + executable bit + a portable shell.** Script hooks need the
   right shebang and `chmod +x` (many agents silently skip a non-executable
   hook). Target the shell the OS actually has — **macOS ships bash 3.2**, so
   avoid bash-4-only features (`mapfile`/`readarray`, `declare -A`, `${var,,}`)
   unless you control the interpreter, and provide a Windows variant when the
   agent supports one.
8. **No hardcoded secrets.** Read tokens from the environment; never log them.
9. **Idempotency + respect the timeout.** Events can fire repeatedly; agents kill
   slow hooks. Keep it fast and safe to re-run.
10. **Register it correctly.** Use the exact config path and format from research,
    absolute script paths (or the agent's project-dir variable), and the right
    matcher scoping so the hook fires for exactly the events you intend.

### The fail-open vs fail-closed rule

The most common judgment call:

- **Observers / notifiers / formatters / context-injectors** → **fail open**: on
  any internal error, exit cleanly and let the agent proceed. A broken notifier
  must never block real work.
- **Gates** (command guards, write blockers, policy checks) → **fail closed on a
  real positive match** (a genuine violation blocks loudly) — **but still fail
  open on their own internal errors.** Deny on a real hit; allow on a bug. That
  split is what keeps a buggy guard from bricking the agent.

## Output format

Always deliver these four parts, in this order:

1. **The hook script** — complete and copy-pasteable, correct language tag,
   matching the researched contract. Environment-specific values come from env
   vars, noted as such.
2. **Register / install** — the exact steps to put the hook where the agent finds
   it: the config file + the literal snippet to add (or the filesystem location
   for discovery-based agents), the executable bit, and the matcher/timeout. Use
   the agent's real config path from research.
3. **Test** — one way to exercise the hook *without* waiting for the real event:
   pipe a synthetic payload to it (`echo '<json>' | ./hook.sh; echo "exit=$?"`),
   use the agent's built-in hook tester/simulator if it has one, or the echo-hook
   capture. Show what a correct allow vs block looks like.
4. **Pitfalls** — 2–4 traps specific to *this agent and event* (e.g. stdout
   pollution, wrong matcher casing, the exit-2-discards-stdout interaction,
   fail-open omissions, macOS bash 3.2, a Windows variant). Drawn from the
   reference, not generic.

## References

- `references/claude-code-and-codex.md` — authoritative per-event deep-dive for
  the two most common targets (events, stdin payload, registration, exit-code +
  JSON contract, worked PreToolUse guard, testing).
- `references/researching-any-agent.md` — the research playbook for any other (or
  unknown) agent + a grounded catalog of known agents and their hook models, and
  fallbacks (git hooks / wrapper script / MCP / rules) when an agent has none.
