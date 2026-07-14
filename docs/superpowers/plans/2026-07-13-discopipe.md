# discopipe Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Discord bot that pipes messages from one authorized user/channel to `claude -p --continue` stdin and replies with its stdout — pure pass-through, no command syntax.

**Architecture:** Single nbdev notebook `nbs/00_bot.ipynb` exporting `discopipe/bot.py` (~60 lines): an async subprocess runner (`run_agent`), two pure helpers (`compose`, `reply_text`), env config (`load_config`), and a thin `discord.Client` subclass. Spec: `docs/superpowers/specs/2026-07-13-discopipe-design.md`.

**Tech Stack:** Python ≥3.10, nbdev (pyproject-style, mirroring `~/discomux`), discord.py, asyncio subprocess.

## Global Constraints

- All work happens in `/home/doyu/discopipe` (already a git repo, branch `main`).
- nbdev CLI commands use **hyphens**: `nbdev-new`, `nbdev-export`, `nbdev-test`, `nbdev-prepare` (underscore aliases don't exist on this box).
- Env var names exactly as spec: `DISCORD_TOKEN`, `DISCOPIPE_USER_ID`, `DISCOPIPE_CHANNEL_ID`, `DISCOPIPE_CWD`, `DISCOPIPE_CMD`, `DISCOPIPE_TIMEOUT`.
- Default agent command exactly: `claude -p --continue --dangerously-skip-permissions`.
- **No retry logic anywhere** — nonzero exit is reported, never re-run (spec: replaying side effects is worse than failing).
- stdout and stderr are captured **separately**; replies carry stdout only unless exit ≠ 0.
- Tests live in the notebook as assert cells (nbdev style, as in discomux); `nbdev-test` runs them. Test cells must come **after** the cell defining what they test.
- In notebooks, top-level `await` works (IPython autoawait) — write async tests as plain `await …` cells, no `asyncio.run()`.
- Commit after every task with the message given in the task.

---

### Task 1: nbdev scaffold (reused from ~/discomux)

**Files:**
- Create (copied from `/home/doyu/discomux` and renamed): `pyproject.toml`, `.gitignore`, `.github/workflows/test.yaml`, `MANIFEST.in`, `LICENSE`, `nbs/nbdev.yml`, `nbs/_quarto.yml`, `nbs/styles.css`, `nbs/index.ipynb`, `discopipe/__init__.py`
- Create: `nbs/00_bot.ipynb`

**Interfaces:**
- Consumes: `/home/doyu/discomux` — a proven nbdev repo of the exact same shape (pyproject-style nbdev, console script, CI that runs `nbdev-test` and checks export sync)
- Produces: a working nbdev repo where `nbdev-export` writes `discopipe/bot.py` from `nbs/00_bot.ipynb` and `pip install -e .` provides the `discopipe` console script. Later tasks only add notebook cells.

Rationale: `nbdev-new` in a non-empty directory is fragile; discomux's scaffold is already reviewed and CI-tested. Copy + rename beats regenerate.

- [ ] **Step 1: Copy the scaffold from discomux**

```bash
cd /home/doyu/discopipe && \
cp /home/doyu/discomux/pyproject.toml /home/doyu/discomux/.gitignore \
   /home/doyu/discomux/MANIFEST.in /home/doyu/discomux/LICENSE . && \
mkdir -p .github/workflows nbs discopipe && \
cp /home/doyu/discomux/.github/workflows/test.yaml .github/workflows/ && \
cp /home/doyu/discomux/nbs/nbdev.yml /home/doyu/discomux/nbs/_quarto.yml \
   /home/doyu/discomux/nbs/styles.css /home/doyu/discomux/nbs/index.ipynb nbs/ && \
echo '__version__ = "0.0.1"' > discopipe/__init__.py
```

- [ ] **Step 2: Rename discomux → discopipe in the copied config**

```bash
cd /home/doyu/discopipe && sed -i 's/discomux/discopipe/g' pyproject.toml nbs/nbdev.yml && \
sed -i 's/Discord transport adapter for tmux\/byobu agent sessions on doyu-box/Discord passthrough to a headless coding agent CLI/' pyproject.toml nbs/nbdev.yml && \
grep -n "discopipe" pyproject.toml | head
```

Expected: `name = "discopipe"`, `discopipe = "discopipe.bot:main"` under `[project.scripts]`, `dependencies = ["discord.py"]` (inherited from discomux), and nbdev.yml pointing at `doyu.github.io/discopipe`. Verify description strings were replaced; fix by hand if the sed pattern missed (discomux may phrase it slightly differently — check with `grep -n description pyproject.toml nbs/nbdev.yml`).

- [ ] **Step 3: Create the bot notebook**

NotebookEdit cannot create files — Write `nbs/00_bot.ipynb` as raw JSON first:

```json
{
 "cells": [
  {"cell_type": "markdown", "metadata": {}, "source": [
    "# bot\n", "\n",
    "> Discord passthrough to a headless coding agent CLI: subprocess runner,\n",
    "> reply shaping, env config, and the thin discord.py glue."]},
  {"cell_type": "code", "execution_count": null, "metadata": {}, "outputs": [],
   "source": ["#| default_exp bot"]}
 ],
 "metadata": {"kernelspec": {"display_name": "python3", "language": "python", "name": "python3"}},
 "nbformat": 4, "nbformat_minor": 5
}
```

Later tasks append cells to this existing notebook with NotebookEdit. Also update `nbs/index.ipynb`'s discomux-specific prose minimally (Task 6 rewrites it properly).

- [ ] **Step 4: Install dependencies and the package**

```bash
pip install discord.py && cd /home/doyu/discopipe && pip install -e .
```

Expected: both succeed; `python -c "import discord"` works.

- [ ] **Step 5: Verify the toolchain round-trips**

```bash
cd /home/doyu/discopipe && nbdev-export && nbdev-test && ls discopipe/bot.py
```

Expected: export succeeds, tests pass (nothing to test yet), `discopipe/bot.py` exists.

- [ ] **Step 6: Commit**

```bash
cd /home/doyu/discopipe && git add -A && git commit -m "chore: nbdev scaffold with discord.py dep and discopipe entry point"
```

---

### Task 2: `run_agent` — subprocess runner

**Files:**
- Modify: `nbs/00_bot.ipynb` (append cells; regenerates `discopipe/bot.py`)

**Interfaces:**
- Consumes: nothing
- Produces: `async def run_agent(text: str, cmd: list, cwd: str, timeout: float = 600.0) -> tuple` returning `(stdout: str, stderr: str, returncode: int)`. Never raises on nonzero exit or timeout. Task 5's `on_message` calls it.

- [ ] **Step 1: Add the failing test cell**

Append a markdown cell then this code cell to `nbs/00_bot.ipynb`:

```markdown
## `run_agent`

Run one agent turn as a subprocess: stdin ← message, capture stdout and
stderr **separately** (claude emits warnings on stderr even on success).
Timeout: SIGKILL the whole process group, recover partial output via a
shielded `communicate()` (an unshielded `wait_for` drops the buffer).
Never raises — nonzero exit is data, not an exception, and is **never
retried** (a retry would replay side effects like PR creation).
```

```python
out, err, rc = await run_agent("hello", ["bash", "-c", "cat; echo warn >&2"], ".", 5)
assert out == "hello", out                    # stdin delivered, stdout captured
assert "warn" in err and "warn" not in out    # stderr kept separate
assert rc == 0

out, err, rc = await run_agent("", ["bash", "-c", "echo partial; echo boom >&2; exit 3"], ".", 5)
assert rc == 3                                # nonzero exit reported, not raised
assert "partial" in out and "boom" in err

out, err, rc = await run_agent("", ["bash", "-c", "echo start; sleep 5"], ".", 0.5)
assert "start" in out, out                    # partial output recovered
assert "timed out" in out and rc != 0

out, err, rc = await run_agent("", ["bash", "-c", "echo hi; sleep 60 &"], ".", 1)
assert "hi" in out and "timed out" in out     # killpg reaps the background child
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/doyu/discopipe && nbdev-test --path nbs/00_bot.ipynb`
Expected: FAIL with `NameError: name 'run_agent' is not defined`

- [ ] **Step 3: Insert the implementation cell ABOVE the test cell**

```python
#| export
import asyncio, os, signal

async def run_agent(text:str,            # message for the agent, fed to stdin
                    cmd:list,            # agent command line, already shlex-split
                    cwd:str,             # working directory for the agent
                    timeout:float=600.0, # wall-clock seconds before SIGKILL
                    )->tuple:            # (stdout, stderr, returncode)
    "Run one agent turn; never raises on nonzero exit or timeout."
    proc = await asyncio.create_subprocess_exec(
        *cmd, cwd=cwd, start_new_session=True,
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
    comm = asyncio.ensure_future(proc.communicate(text.encode()))
    try:
        out, err = await asyncio.wait_for(asyncio.shield(comm), timeout)
    except asyncio.TimeoutError:
        try: os.killpg(proc.pid, signal.SIGKILL)   # whole session: agent + children
        except ProcessLookupError: pass
        try:
            out, err = await asyncio.wait_for(asyncio.shield(comm), 1)
        except asyncio.TimeoutError:
            comm.cancel()
            return "… timed out (output withheld by a background child)\n", "", -9   # -9: killed by SIGKILL
        rc = proc.returncode if proc.returncode is not None else -9
        return out.decode(errors="replace") + "\n… timed out\n", err.decode(errors="replace"), rc
    return out.decode(errors="replace"), err.decode(errors="replace"), proc.returncode
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/doyu/discopipe && nbdev-test --path nbs/00_bot.ipynb`
Expected: PASS

- [ ] **Step 5: Export and commit**

```bash
cd /home/doyu/discopipe && nbdev-export && git add -A && git commit -m "feat: run_agent subprocess runner — separate stderr, timeout killpg, no retry"
```

---

### Task 3: `compose` and `reply_text` — reply shaping

**Files:**
- Modify: `nbs/00_bot.ipynb` (append cells)

**Interfaces:**
- Consumes: nothing
- Produces: `def compose(out: str, err: str, rc: int) -> str` (merges stderr/exit-code into the body only on failure) and `def reply_text(text: str, limit: int = 2000) -> str` (tail-keeping truncation, `"(no output)"` for empty). Task 5 sends `reply_text(compose(out, err, rc))`.

- [ ] **Step 1: Add the failing test cell**

Append a markdown cell then this code cell:

```markdown
## `compose` and `reply_text`

Success → stdout verbatim (it's markdown; Discord renders it, no fences).
Failure → stdout, then `(exit N)` and the fenced stderr — that's where the
reason lives. Inner ``` in stderr is neutralized with a zero-width space so
it can't close the fence. `reply_text` tail-truncates to Discord's 2000-char
cap: the primary length control is a CLAUDE.md instruction in the agent's
cwd (operational, see spec), so this is the safety net.
```

```python
assert compose("ok", "warn on stderr", 0) == "ok"     # success: stderr never leaks
c = compose("partial", "boom", 3)
assert c.startswith("partial") and "(exit 3)" in c and "boom" in c
c = compose("x", "evil ``` fence", 1)
assert c.count("```") == 2                            # only compose's own fence pair

assert reply_text("hi") == "hi"
assert reply_text("") == "(no output)"
assert reply_text("  \n") == "(no output)"
long = "\n".join(f"line {i}" for i in range(500))
out = reply_text(long)
assert len(out) <= 2000
assert out.startswith("… (truncated)\n") and out.endswith("line 499")  # tail survives
assert len(reply_text(long, limit=100)) <= 100
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/doyu/discopipe && nbdev-test --path nbs/00_bot.ipynb`
Expected: FAIL with `NameError: name 'compose' is not defined`

- [ ] **Step 3: Insert the implementation cell ABOVE the test cell**

```python
#| export
_ZWSP = "​"   # breaks a literal ``` so stderr can't close the fence

def compose(out:str,    # agent stdout (markdown)
            err:str,    # agent stderr, captured separately
            rc:int,     # agent exit code
            )->str:     # reply body for Discord
    "stdout alone on success; append `(exit N)` + fenced stderr on failure."
    if not rc: return out
    body = err.strip().replace("```", f"`{_ZWSP}`{_ZWSP}`")
    return f"{out}\n(exit {rc})\n```\n{body}\n```"

def reply_text(text:str,        # reply body (markdown, not fenced)
               limit:int=2000,  # Discord message length cap
               )->str:          # len() <= limit, never empty
    "Tail-keeping truncation: the end of an answer matters most."
    body = text.strip()
    if not body: return "(no output)"
    if len(body) <= limit: return body
    marker = "… (truncated)\n"
    return marker + body[-(limit - len(marker)):]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/doyu/discopipe && nbdev-test --path nbs/00_bot.ipynb`
Expected: PASS

- [ ] **Step 5: Export and commit**

```bash
cd /home/doyu/discopipe && nbdev-export && git add -A && git commit -m "feat: compose + reply_text — stderr only on failure, tail truncation"
```

---

### Task 4: `load_config` — env config

**Files:**
- Modify: `nbs/00_bot.ipynb` (append cells)

**Interfaces:**
- Consumes: nothing
- Produces: `def load_config() -> dict` with keys `token: str`, `user_id: int`, `channel_id: int`, `cwd: str`, `cmd: str`, `timeout: float`, and module constant `DEFAULT_CMD = "claude -p --continue --dangerously-skip-permissions"`. Task 5's `main` calls it.

- [ ] **Step 1: Add the failing test cell**

Append a markdown cell then this code cell:

```markdown
## `load_config`

Environment variables only. Missing/malformed required var → clean
`sys.exit` naming the variable (lands in the systemd journal).
`DISCOPIPE_CWD` is **required**: `--continue` resumes "the latest
conversation in this directory", so a `$HOME` default would silently
collide with the operator's interactive claude sessions.
```

```python
import os
_env = dict(DISCORD_TOKEN="tok", DISCOPIPE_USER_ID="1", DISCOPIPE_CHANNEL_ID="2",
            DISCOPIPE_CWD="/srv/agent")
for k in ("DISCOPIPE_CMD","DISCOPIPE_TIMEOUT"): os.environ.pop(k, None)
os.environ.update(_env)
cfg = load_config()
assert cfg == dict(token="tok", user_id=1, channel_id=2,
                   cwd="/srv/agent", cmd=DEFAULT_CMD, timeout=600.0)

os.environ.update(DISCOPIPE_CMD="codex exec", DISCOPIPE_TIMEOUT="30")
cfg = load_config()
assert (cfg["cmd"], cfg["timeout"]) == ("codex exec", 30.0)
for k in ("DISCOPIPE_CMD","DISCOPIPE_TIMEOUT"): os.environ.pop(k)

del os.environ["DISCORD_TOKEN"]
try: load_config(); assert False, "expected SystemExit"
except SystemExit as e: assert "DISCORD_TOKEN" in str(e)

os.environ.update(_env); del os.environ["DISCOPIPE_CWD"]
try: load_config(); assert False, "expected SystemExit"
except SystemExit as e: assert "DISCOPIPE_CWD" in str(e)   # required: no $HOME default

os.environ.update(_env); os.environ["DISCOPIPE_USER_ID"] = "not-a-number"
try: load_config(); assert False, "expected SystemExit"
except SystemExit as e: assert "DISCOPIPE_USER_ID must be an integer" in str(e)

os.environ.update(_env); os.environ["DISCOPIPE_CHANNEL_ID"] = "xyz"
try: load_config(); assert False, "expected SystemExit"
except SystemExit as e: assert "DISCOPIPE_CHANNEL_ID must be an integer" in str(e)

os.environ.update(_env); os.environ["DISCOPIPE_TIMEOUT"] = "soon"
try: load_config(); assert False, "expected SystemExit"
except SystemExit as e: assert "DISCOPIPE_TIMEOUT must be a number" in str(e)

os.environ.pop("DISCOPIPE_TIMEOUT")
for k in _env: os.environ.pop(k, None)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/doyu/discopipe && nbdev-test --path nbs/00_bot.ipynb`
Expected: FAIL with `NameError: name 'load_config' is not defined`

- [ ] **Step 3: Insert the implementation cell ABOVE the test cell**

```python
#| export
import os, sys

DEFAULT_CMD = "claude -p --continue --dangerously-skip-permissions"

def load_config()->dict:  # token, user_id, channel_id, cwd, cmd, timeout
    "Read discopipe config from the environment; exit with a message if incomplete."
    e = os.environ
    try:
        token, cwd = e["DISCORD_TOKEN"], e["DISCOPIPE_CWD"]
        user_raw, chan_raw = e["DISCOPIPE_USER_ID"], e["DISCOPIPE_CHANNEL_ID"]
    except KeyError as ex:
        sys.exit(f"discopipe: missing environment variable {ex.args[0]}")
    try: user_id = int(user_raw)
    except ValueError: sys.exit("discopipe: DISCOPIPE_USER_ID must be an integer")
    try: channel_id = int(chan_raw)
    except ValueError: sys.exit("discopipe: DISCOPIPE_CHANNEL_ID must be an integer")
    try: timeout = float(e.get("DISCOPIPE_TIMEOUT", "600"))
    except ValueError: sys.exit("discopipe: DISCOPIPE_TIMEOUT must be a number")
    return dict(token=token, user_id=user_id, channel_id=channel_id, cwd=cwd,
                cmd=e.get("DISCOPIPE_CMD", DEFAULT_CMD), timeout=timeout)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/doyu/discopipe && nbdev-test --path nbs/00_bot.ipynb`
Expected: PASS

- [ ] **Step 5: Export and commit**

```bash
cd /home/doyu/discopipe && nbdev-export && git add -A && git commit -m "feat: load_config — env-only config with journal-friendly exits"
```

---

### Task 5: `Discopipe` client and `main`

**Files:**
- Modify: `nbs/00_bot.ipynb` (append cells)

**Interfaces:**
- Consumes: `run_agent` (Task 2), `compose`/`reply_text` (Task 3), `load_config`/`DEFAULT_CMD` (Task 4)
- Produces: `class Discopipe(discord.Client)` and `def main()` — the `discopipe` console entry point wired in Task 1's pyproject.

- [ ] **Step 1: Add the failing test cell**

Append a markdown cell then this code cell:

```markdown
## `Discopipe` client and `main`

Authorization first: bot author, wrong user, or wrong channel → silent
drop; empty content (attachment-only) ignored. One `asyncio.Lock`
serializes agent runs — two concurrent `--continue` against the same
directory would race. Typing indicator while the agent works.
`allowed_mentions.none()` so output can never ping.
```

```python
import discord

class _FakeTyping:
    async def __aenter__(self): return None
    async def __aexit__(self, *a): return False

class _FakeChannel:
    def __init__(self, cid): self.id, self.sent = cid, []
    def typing(self): return _FakeTyping()
    async def send(self, content): self.sent.append(content)

class _FakeAuthor:
    def __init__(self, uid, bot=False): self.id, self.bot = uid, bot

class _FakeMsg:
    def __init__(self, content, uid=1, cid=2, bot=False):
        self.content, self.author, self.channel = content, _FakeAuthor(uid, bot), _FakeChannel(cid)

intents = discord.Intents.default()
intents.message_content = True
c = Discopipe(dict(token="x", user_id=1, channel_id=2, cwd=".",
                   cmd="bash -c 'cat; echo warn >&2'", timeout=5.0),
              intents=intents, allowed_mentions=discord.AllowedMentions.none())
assert c.allowed_mentions.everyone is False        # output can never ping @everyone/@here
assert c.allowed_mentions.users is False and c.allowed_mentions.roles is False
assert callable(main)

m = _FakeMsg("hello")
await c.on_message(m)
assert m.channel.sent == ["hello"]                 # stdout only; stderr warn never leaks

for bad in (_FakeMsg("x", bot=True), _FakeMsg("x", uid=999),
            _FakeMsg("x", cid=999), _FakeMsg("")):
    await c.on_message(bad)
    assert bad.channel.sent == []                  # silent drop, all four reasons

c.cfg["cmd"] = "bash -c 'echo partial; echo boom >&2; exit 3'"
m = _FakeMsg("ignored")
await c.on_message(m)
assert len(m.channel.sent) == 1
r = m.channel.sent[0]
assert "partial" in r and "(exit 3)" in r and "boom" in r   # failure carries stderr
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/doyu/discopipe && nbdev-test --path nbs/00_bot.ipynb`
Expected: FAIL with `NameError: name 'Discopipe' is not defined`

- [ ] **Step 3: Insert the implementation cell ABOVE the test cell**

```python
#| export
import shlex
import discord

class Discopipe(discord.Client):
    "Transport adapter: one authorized user, one channel, one agent."
    def __init__(self, cfg, **kw):
        super().__init__(**kw)
        self.cfg = cfg
        self._lock = asyncio.Lock()   # concurrent --continue runs would race

    async def on_ready(self):
        print(f"discopipe: connected as {self.user} (id={self.user.id})")

    async def on_message(self, m):
        cfg = self.cfg
        if m.author.bot or m.author.id != cfg["user_id"] or m.channel.id != cfg["channel_id"]:
            return
        if not m.content: return   # attachment-only/empty: nothing to send
        async with self._lock:
            async with m.channel.typing():
                out, err, rc = await run_agent(m.content, shlex.split(cfg["cmd"]),
                                               cfg["cwd"], cfg["timeout"])
        await m.channel.send(reply_text(compose(out, err, rc)))

def main():
    "Console entry point: `discopipe`."
    cfg = load_config()
    intents = discord.Intents.default()
    intents.message_content = True   # privileged; also enable in the Developer Portal
    Discopipe(cfg, intents=intents,
              allowed_mentions=discord.AllowedMentions.none()).run(cfg["token"])
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/doyu/discopipe && nbdev-test --path nbs/00_bot.ipynb`
Expected: PASS

- [ ] **Step 5: Verify the console script resolves**

```bash
cd /home/doyu/discopipe && nbdev-export && pip install -e . -q && \
python -c "from discopipe.bot import main; print('entry point ok')"
```

Expected: `entry point ok`

- [ ] **Step 6: Commit**

```bash
cd /home/doyu/discopipe && git add -A && git commit -m "feat: Discopipe client + main — authorize, serialize, pass through"
```

---

### Task 6: README, index notebook, and full prepare

**Files:**
- Modify: `nbs/index.ipynb` (nbdev generates `README.md` from it — do not edit `README.md` directly)

**Interfaces:**
- Consumes: everything above (documents it)
- Produces: rendered `README.md`; a green `nbdev-prepare`.

- [ ] **Step 1: Write the index notebook**

Replace the generated content of `nbs/index.ipynb` (keep its final `#| hide` import cell if present) so the markdown body reads:

````markdown
# discopipe

> Discord passthrough to a headless coding agent CLI.

One authorized Discord user in one channel talks to `claude -p --continue`
running on this box. Messages go to the agent's stdin; its stdout comes
back as the reply. No prefixes, no commands — pure pass-through.

## Install

```sh
pip install -e .
```

## Configure (environment only)

| var | required | default | meaning |
|---|---|---|---|
| `DISCORD_TOKEN` | yes | — | bot token |
| `DISCOPIPE_USER_ID` | yes | — | the one authorized Discord user (int) |
| `DISCOPIPE_CHANNEL_ID` | yes | — | the one authorized channel (int) |
| `DISCOPIPE_CWD` | yes | — | working directory for the agent — dedicate it to the bot |
| `DISCOPIPE_CMD` | no | `claude -p --continue --dangerously-skip-permissions` | agent command line |
| `DISCOPIPE_TIMEOUT` | no | `600` | wall-clock seconds per run |

**`DISCOPIPE_CWD` must be a directory dedicated to the bot.** `--continue`
resumes "the latest conversation in this directory", so running interactive
`claude` there (e.g. over ssh) silently hijacks the bot's thread.

**Put a `CLAUDE.md` in `DISCOPIPE_CWD`** telling the agent to keep replies
under ~1800 characters (Discord caps messages at 2000; the bot truncates as
a safety net, keeping the tail). Example:

    Replies are read on a phone via Discord. Keep every reply under 1800
    characters. Be terse. For long output, write it to a file and reply
    with the path and a 3-line summary.

**Security note:** the default command includes
`--dangerously-skip-permissions` because headless runs cannot answer
permission prompts. Run this only on a single-purpose box you own; the
Discord side already restricts input to one user and one channel.

## Run

Put the variables in a `chmod 600` env file (e.g.
`~/.config/discopipe/env`) — never on the command line, where the token
would land in shell history. A systemd unit consumes the same file via
`EnvironmentFile=`.

```sh
set -a; source ~/.config/discopipe/env; set +a; discopipe
```

Nonzero agent exits are reported in the reply as `(exit N)` plus the
agent's stderr — never retried, so a transient failure can't replay side
effects (like creating a duplicate PR).
````

- [ ] **Step 2: Full round-trip**

Run: `cd /home/doyu/discopipe && nbdev-prepare`
Expected: export + test + clean + README generation all succeed; `git status` shows `README.md` updated.

- [ ] **Step 3: Commit**

```bash
cd /home/doyu/discopipe && git add -A && git commit -m "docs: README — config table, dedicated-CWD warning, CLAUDE.md length layer"
```

---

### Task 7: Manual E2E (operator-assisted)

**Files:** none (verification only). This task needs the human operator: a real bot token and a phone/Discord client.

**Interfaces:**
- Consumes: the `discopipe` entry point and a configured Discord application (reuse the discomux app with a fresh channel, or create a new one; enable the *Message Content* privileged intent in the Developer Portal).

- [ ] **Step 1: Prepare a dedicated agent directory**

```bash
mkdir -p ~/discopipe-agent && cat > ~/discopipe-agent/CLAUDE.md <<'EOF'
Replies are read on a phone via Discord. Keep every reply under 1800
characters. Be terse. For long output, write it to a file and reply with
the path and a 3-line summary.
EOF
```

- [ ] **Step 2: Create the env file (never put the token on a command line)**

A token typed inline lands in shell history. Use a `chmod 600` env file —
the same file a systemd unit consumes later via `EnvironmentFile=`:

```bash
mkdir -p ~/.config/discopipe && touch ~/.config/discopipe/env && chmod 600 ~/.config/discopipe/env
cat > ~/.config/discopipe/env <<'EOF'
DISCORD_TOKEN=REPLACE_ME
DISCOPIPE_USER_ID=REPLACE_ME
DISCOPIPE_CHANNEL_ID=REPLACE_ME
DISCOPIPE_CWD=/home/doyu/discopipe-agent
EOF
```

Then the operator edits `REPLACE_ME` values with an editor (not `echo`).

- [ ] **Step 3: Start the bot**

```bash
set -a; source ~/.config/discopipe/env; set +a; discopipe
```

Expected: `discopipe: connected as <botname> (id=…)` on stdout.

- [ ] **Step 4: Verify the checklist from the spec, one message at a time**

1. Send `hello, which directory are you in?` → reply mentions `discopipe-agent`.
2. Send `what did I just ask you?` → reply proves `--continue` continuity.
3. Send `use gh to show my three most recent GitHub notifications` → reply shows `gh` working through the agent's Bash tool.
4. Send a request that produces long output (`explain this repo in detail`) → reply arrives under 2000 chars (CLAUDE.md layer) or shows `… (truncated)` (safety net); either is a pass.
5. Send `run: echo @everyone` → the reply must **not** ping anyone.
6. Restart the bot with `DISCOPIPE_TIMEOUT=5` in the env file, send `sleep for 60 seconds using bash` → reply contains `… timed out` and `(exit …)` within ~6 s. Restore the normal timeout afterward.
7. Make the agent fail with backticks on stderr — e.g. set `DISCOPIPE_CMD=bash -c 'echo "\`\`\`" >&2; exit 1'` temporarily — → the failure reply must render as **one** fenced block on the phone: the zero-width-space neutralization of ``` in stderr is only proven by Discord's actual renderer. Restore `DISCOPIPE_CMD` afterward.

- [ ] **Step 5: Record the result**

Append a `## E2E result` section (date + pass/fail per item) to `docs/superpowers/specs/2026-07-13-discopipe-design.md`, then:

```bash
cd /home/doyu/discopipe && git add -A && git commit -m "docs: E2E checklist results"
```
