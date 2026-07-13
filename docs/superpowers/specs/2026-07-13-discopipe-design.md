# discopipe — design

Date: 2026-07-13
Status: draft for review

## Purpose

Talk to a headless coding agent (Claude Code) running on a VM, from Discord —
typically from a phone. Pure pass-through, no command syntax:

```
Discord message  →  agent CLI stdin
agent CLI stdout →  Discord reply
```

The bot invents nothing: no prefixes, no escape sequences, no subcommands.

## Lineage

Supersedes **discomux** (Discord ↔ tmux bridge). Pivot rationale: the goal was
never the terminal screen but the agent behind it. Bridging tmux forces paste
mechanics on input and screen-scraping (`capture-pane`) on output, because a
tmux pane is a *screen*, not a message stream. Headless CLI (`claude -p`)
gives real message-in/message-out semantics while keeping the agent's full
capabilities — its Bash tool covers `gh`, `git`, and shell, which replaces
discomux's `!sh` and the Shell/Tig windows entirely.

Carried over from discomux (reviewed and hardened there):

- single-user + single-channel authorization with silent drop
- `allowed_mentions.none()` so output can never ping
- tail-keeping truncation with a `… (truncated)` marker
- subprocess skeleton: `start_new_session` + `killpg` + `asyncio.shield` to
  recover partial output on timeout
- env-only config with journal-friendly `sys.exit` messages

## Behavior

1. **Authorize**: drop silently unless the author is `DISCOPIPE_USER_ID` and
   the channel is `DISCOPIPE_CHANNEL_ID` (bots always dropped). Empty
   messages (attachment-only) are ignored.
2. **Run**: spawn `DISCOPIPE_CMD` (shlex-split) in `DISCOPIPE_CWD`, write the
   message content to stdin, close stdin, collect stdout+stderr combined.
3. **Reply**: send the output back as a plain Discord message — *not*
   code-fenced, since agent output is markdown and Discord renders it.
   Empty output → `(no output)`. Nonzero exit → send the output anyway,
   with a short `(exit N)` note appended.

### Agent command

`DISCOPIPE_CMD` defaults to:

```
claude -p --continue --dangerously-skip-permissions
```

- `--continue` resumes the latest conversation in `DISCOPIPE_CWD`, which is
  what makes a Discord channel behave like one ongoing session. On the very
  first message there is nothing to continue and the command exits nonzero.
  Rule: when the command line contains `--continue` and exits nonzero, retry
  exactly once with `--continue` removed; if the retry also fails, report
  that failure. (Exact first-run failure mode to be verified during
  implementation.)
- `--dangerously-skip-permissions` is required because headless runs cannot
  answer permission prompts. Accepted risk: the box is single-purpose and
  owned by the operator, and Discord-side authorization already limits input
  to one user. Documented, not hidden.
- Because the command is config, adding another agent later (e.g.
  `codex exec`) is a config change plus a second bot instance/channel —
  no code.

### Concurrency

One `asyncio.Lock` per channel serializes agent runs: two concurrent
`--continue` invocations against the same session directory would race.
Messages queue in arrival order. While the agent runs, the bot shows the
Discord typing indicator.

### Timeout

`DISCOPIPE_TIMEOUT` (seconds, default `600`). On expiry: `killpg` the process
group, recover whatever output was produced (shielded `communicate()`), append
`… timed out`.

### Output limits

Discord caps messages at 2000 chars. Keep the tail (the end of an answer
matters most), prefix `… (truncated)`. Multi-message chunking is deferred.

## Config (environment only)

| var | required | default | meaning |
|---|---|---|---|
| `DISCORD_TOKEN` | yes | — | bot token |
| `DISCOPIPE_USER_ID` | yes | — | the one authorized Discord user (int) |
| `DISCOPIPE_CHANNEL_ID` | yes | — | the one authorized channel (int) |
| `DISCOPIPE_CWD` | no | `$HOME` | working directory for the agent |
| `DISCOPIPE_CMD` | no | see above | agent command line, shlex-split |
| `DISCOPIPE_TIMEOUT` | no | `600` | wall-clock seconds per run |

Missing required var or non-integer id → `sys.exit` naming the variable.

## Components

nbdev repo, single notebook `nbs/00_bot.ipynb`, exported module
`discopipe/bot.py`, console entry point `discopipe`.

- `run_agent(text, cmd, cwd, timeout) -> str` — subprocess runner with the
  timeout/killpg/partial-output behavior above
- `reply_text(text, limit=2000) -> str` — tail-keeping truncation (no fences)
- `load_config() -> dict` — env → config dict
- `Discopipe(discord.Client)` — authorize → run → reply
- `main()` — entry point; `message_content` intent, `allowed_mentions.none()`

Estimated exported code: ~50 lines.

## Testing

- **Unit (in-notebook, nbdev-test)**: `run_agent` against a stub CLI (a bash
  one-liner reading stdin), including timeout-kill and partial-output
  recovery; `reply_text` truncation edges; `load_config` missing/malformed
  vars; the `--continue` retry path with a stub that fails first.
- **E2E (manual, like discomux Level 3)**: real `claude -p` on the VM, real
  Discord channel; verify continuity across two messages, a `gh`-using
  request, timeout behavior, and that output never pings.

## Non-goals (deferred)

- multiple agents / channel→command routing (config-only path exists)
- multi-message chunking of long replies
- attachments, threads, reactions, streaming/progress updates
- any tmux integration
