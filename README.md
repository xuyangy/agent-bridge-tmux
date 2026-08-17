# agent-bridge-tmux

Let the agent in one tmux pane talk to the agent in another, for a fixed number
of turns, with a stop button.

This file is for you, the human. `SKILL.md` is the instruction set the agent
reads — you never need to open it.

## Requirements

- tmux, with **both** agents on the **same** tmux server
- `python3`
- both agents running inside a tmux pane

An agent reached over SSH is on a different tmux server. That case is refused on
purpose, with a clear message, rather than silently dropping your messages.

If you want to reach an agent that is a *session* rather than a tmux pane, this
is the wrong tool — the built-in `SendMessage` / `ListAgents` handle that.

## How to use it

Talk to the agent in your own pane, in plain words:

> bridge to window 2 and have it review my uncommitted changes, max 6 turns

The agent then finds the target pane, confirms it with you, does the task, sends
its result, and reports each reply as it arrives.

Things you can set, all optional:

| You say | Effect |
| --- | --- |
| "max 6 turns" | hard stop after 6 messages total (default 10) |
| "stop when it says LOOKS GOOD" | stop early on that exact phrase |
| "bridge to %7" | skip the window lookup, use that pane |

A good first test, small enough to watch end to end:

> bridge to window 2, ask it what repo it is in, max 2 turns

## Stopping it

Two buttons, because you may have more than one bridge running.

**Stop this one.** The agent prints this command every turn, so it is always in
front of you. The path ends in `.abort` and names the pane:

```bash
touch /var/folders/.../agent-bridge/<hash>-<pane>.abort
```

**Stop everything.** One file, checked by every bridge on the machine:

```bash
touch /tmp/agent-bridge.stop
```

Both are checked before every send. Clear them with `reset` (this pane) or
`reset --all` (everything). Set `AGENT_BRIDGE_ABORT` to move the global path.

## Running several bridges at once

Panes 1↔2 and 3↔4 at the same time is fine. Each pane keeps its own state file
and each bridge mints its own random token, so a frame from the wrong exchange is
rejected rather than answered. `reset` and `clear-abort` touch only the pane they
run in, unless you pass `--all`.

One limit: a pane can hold one bridge at a time. 1↔2 and 2↔3 together will be
refused, because pane 2 is already busy.

## If a pane says "this pane already has an active bridge"

The last bridge in that pane did not finish. It was aborted, you interrupted it,
or the agent crashed while it still owed a reply. The pane remembers that and
will not start a new one.

It clears itself after 15 minutes (`AGENT_BRIDGE_STALE_TIMEOUT`). To clear it
now, in that pane:

```bash
python3 <skill>/scripts/agent_bridge.py reset
```

`reset` ends the old exchange and removes the abort file too, so the pane is
ready immediately. `clear-abort` only removes the abort file — it will not
unstick this.

`status` tells you which case you are in: `start_blocked` and
`expires_in_seconds`.

## What stops a bridge

- the turn limit is reached — hard, no exceptions
- the goal phrase appears
- the target pane never goes idle
- the frame could not be submitted into the peer's input box
- a frame arrives corrupted (its integrity checksum does not match)
- no reply within the ack timeout (default 900s, set `AGENT_BRIDGE_ACK_TIMEOUT`)
- an unfinished bridge goes stale (default 900s, set `AGENT_BRIDGE_STALE_TIMEOUT`)
- the peer is on another tmux server
- you create the abort file

## If a message gets stuck in the other pane

Agent TUIs treat fast input as a paste, and a newline inside a paste is a line
break, not a submit. So a frame can arrive fully typed and never be sent.

The helper waits for the pane to settle, presses Enter, checks that it went, and
backs off before trying again. If it still cannot submit, it says so and stops —
it will not pretend the message was delivered.

When that happens: **press Enter in that pane yourself.** The text is already
there. Do not resend, or the peer gets it twice.

Knobs, if your setup needs them:

| Variable | Default | Meaning |
| --- | --- | --- |
| `AGENT_BRIDGE_SUBMIT_DELAY` | `0.8` | wait before the first Enter; raise for a slow TUI |
| `AGENT_BRIDGE_SUBMIT_ATTEMPTS` | `4` | total Enter presses; `1` disables the check |

Use `1` only when the target is not an agent TUI — a plain `cat` or a dumb REPL
echoes your text back, which looks identical to an unsent frame and trips a
false alarm.

## If a reply comes back garbled

A busy TUI can drop keystrokes while a frame is typed into it — a URL like
`git@github.com:xuyangy/...` has arrived as `@gitcom:xngy/...`. Every frame
carries a checksum, so the receiver rejects a corrupted frame loudly instead of
answering from mangled text.

The default transport (`AGENT_BRIDGE_TYPE=paste`) hands the frame over as one
atomic paste, so there are no keystrokes to drop. If you have set
`AGENT_BRIDGE_TYPE=type`, put it back to `paste`.

## Tests

```bash
python3 -m unittest discover -s tests
```

64 tests, no dependencies, no tmux server needed — they stub the transport and
check framing, the integrity checksum, the state machine, turn bounds, timeouts,
and the submit check. They do not prove delivery; that part is checked against a
real pane by hand.

## Where things are

```
SKILL.md                     instructions for the agent
scripts/agent_bridge.py      all the tmux work: identity, framing, state, sending
references/failure-modes.md  read this when it misbehaves
agents/openai.yaml           display metadata
```

Logs and turn state live under `$TMPDIR/agent-bridge/`, one set per pane. Run
`python3 scripts/agent_bridge.py identity` inside a pane to print its exact
paths.

If the peer's tmux session is unattached, delivery still works but nothing shows
on screen — the log file is then the only view.

## Design in one line

Each agent detects its own pane and stamps that address into every message it
sends; the peer replies to the stamped address. Nobody types a peer's address on
its behalf, so nobody can be wrong about it or be redirected by message content.

Bodies are treated strictly as data. An agent never executes what arrives, and
never takes an address from message text.

## Troubleshooting

Start with `references/failure-modes.md`. It lists each symptom with its cause,
including two readiness rules that were tried and rejected because they failed
against real panes: requiring a prompt-shaped line (breaks on a themed zsh
prompt) and matching spinner glyphs (marks every idle Claude Code pane as busy).
