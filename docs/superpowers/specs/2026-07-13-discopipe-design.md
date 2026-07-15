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
   message content to stdin, close stdin, capture stdout and stderr
   **separately** — verified: `claude` emits warnings on stderr (e.g.
   `⚠ claude.ai connectors are disabled…`) even on success, which must not
   leak into every reply.
3. **Reply**: send **stdout only** back as a plain Discord message — *not*
   code-fenced, since agent output is markdown and Discord renders it.
   Empty output → `(no output)`. Nonzero exit → send stdout anyway, then
   append `(exit N)` and the captured stderr (fenced), which is where the
   failure reason lives. No retry: the operator reads the error and decides.

### Agent command

`DISCOPIPE_CMD` defaults to:

```
claude -p --continue --dangerously-skip-permissions
```

- `--continue` resumes the latest conversation in `DISCOPIPE_CWD`, which is
  what makes a Discord channel behave like one ongoing session. Verified
  2026-07-13: with no conversation to continue, `claude -p --continue`
  silently starts a fresh one and exits 0, and a second `--continue` run
  does resume it — so no first-run special case exists. There is **no
  retry-without-`--continue` rule**: a nonzero exit can mean a transient
  API/network error, and re-running the request in a fresh conversation
  would replay side effects (e.g. create a duplicate PR) while silently
  dropping the session context. Failures are reported, never retried.
- Because `--continue` means "the latest conversation *in this directory*",
  `DISCOPIPE_CWD` must be **dedicated to the bot**. Running interactive
  `claude` in the same directory (e.g. over ssh) creates a newer
  conversation that the bot's next `--continue` silently hijacks — no
  error, just a derailed thread. For this reason `DISCOPIPE_CWD` is
  **required with no default**: a `$HOME` default would silently collide
  with the operator's everyday interactive sessions. Document this in the
  README.
- `--dangerously-skip-permissions` is required because headless runs cannot
  answer permission prompts. Accepted risk: the box is single-purpose and
  owned by the operator, and Discord-side authorization already limits input
  to one user. Documented, not hidden.
- Because the command is config, adding another agent later (e.g.
  `codex exec`) is a config change plus a second bot instance/channel —
  no code.

### Concurrency

One `asyncio.Lock` serializes agent runs (there is only one authorized
channel): two concurrent `--continue` invocations against the same session
directory would race.
Messages queue in arrival order. While the agent runs, the bot shows the
Discord typing indicator.

### Timeout

`DISCOPIPE_TIMEOUT` (seconds, default `600`). On expiry: `killpg` the process
group, recover whatever output was produced (shielded `communicate()`), append
`… timed out`.

### Output limits

Discord caps messages at 2000 chars. Two layers:

- **Primary (operational, no code)**: a `CLAUDE.md` in `DISCOPIPE_CWD`
  instructs the agent to keep replies under ~1800 chars, terse, writing
  details to files and citing paths. This is a soft constraint — the model
  cannot count characters exactly and breaks length instructions when
  quoting logs/diffs — so it cannot be the only layer.
- **Safety net (code)**: `reply_text` tail-truncates to fit 2000 (the end
  of an answer matters most), prefix `… (truncated)`. With the CLAUDE.md
  layer in place this should fire rarely; multi-message chunking stays
  deferred.

Accepted risk: `run_agent` buffers the agent's full stdout/stderr in memory
before truncation (`communicate()`). A `claude -p` final answer is small in
practice, and input is limited to one authorized user on an owned box; a
bounded-read runner is not worth the complexity here.

## Config (environment only)

| var | required | default | meaning |
|---|---|---|---|
| `DISCORD_TOKEN` | yes | — | bot token |
| `DISCOPIPE_USER_ID` | yes | — | the one authorized Discord user (int) |
| `DISCOPIPE_CHANNEL_ID` | yes | — | the one authorized channel (int) |
| `DISCOPIPE_CWD` | yes | — | working directory for the agent — **must be dedicated to the bot** |
| `DISCOPIPE_CMD` | no | see above | agent command line, shlex-split |
| `DISCOPIPE_TIMEOUT` | no | `600` | wall-clock seconds per run |

Missing required var or non-integer id → `sys.exit` naming the variable.

## Components

nbdev repo, single notebook `nbs/00_bot.ipynb`, exported module
`discopipe/bot.py`, console entry point `discopipe`.

- `run_agent(text, cmd, cwd, timeout) -> tuple[str, str, int]` —
  subprocess runner returning (stdout, stderr, returncode), with the
  timeout/killpg/partial-output behavior above
- `reply_text(text, limit=2000) -> str` — tail-keeping truncation (no fences)
- `load_config() -> dict` — env → config dict
- `Discopipe(discord.Client)` — authorize → run → reply
- `main()` — entry point; `message_content` intent, `allowed_mentions.none()`

Estimated exported code: ~50 lines.

## Testing

- **Unit (in-notebook, nbdev-test)**: `run_agent` against a stub CLI (a bash
  one-liner reading stdin), including timeout-kill, partial-output
  recovery, stderr kept separate from stdout, and nonzero exit reported
  (not retried); `reply_text` truncation edges; `load_config`
  missing/malformed vars; `Discopipe.on_message` driven with fake
  message/channel objects — silent drops (bot author, wrong user, wrong
  channel, empty content), stdout-only reply on success, `(exit N)` +
  stderr on failure.
- **E2E (manual, like discomux Level 3)**: real `claude -p` on the VM, real
  Discord channel; verify continuity across two messages, a `gh`-using
  request, timeout behavior, and that output never pings.

## Non-goals (deferred)

- multiple agents / channel→command routing (config-only path exists)
- multi-message chunking of long replies
- attachments, threads, reactions, streaming/progress updates
- any tmux integration

## E2E result

2026-07-14 — real `claude -p` on the VM, real Discord channel, operator-verified. All 7 pass:

1. PASS — cwd reported as the dedicated agent directory
2. PASS — `--continue` continuity across messages
3. PASS — `gh` works through the agent's Bash tool
4. PASS — long output stays within Discord's limit (CLAUDE.md layer / truncation)
5. PASS — `echo @everyone` output pings no one
6. PASS — `DISCOPIPE_TIMEOUT=5` → `… timed out` + `(exit …)` within ~6 s
7. PASS — ``` on stderr renders as one fenced block (ZWSP neutralization holds in Discord's renderer)

Notes: bot reuses the former discomux Discord application; bot username renamed
discomux → discopipe via `PATCH /users/@me` on 2026-07-14. Env lives in
`~/.config/discopipe/env` (chmod 600), agent dir `~/discopipe-agent`.
