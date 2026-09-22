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
| "max 6 turns" | hard stop after 6 messages total (default 12) |
| "stop when it says LOOKS GOOD" | stop early on that exact phrase |
| "bridge to %7" | skip the window lookup, use that pane |

A good first test, small enough to watch end to end:

> bridge to window 2, ask it what repo it is in, max 2 turns

## Stopping it

Two buttons, because you may have more than one bridge running.

**Stop this one.** The agent prints this command when the bridge starts, and
again whenever it reports a problem. The path ends in `.abort` and names the
pane:

```bash
touch /var/folders/.../agent-bridge-<uid>/<hash>-<pane>.abort
```

**Stop everything.** One file, checked by every bridge you are running. It lives
inside your own state root, so no other user on the host can create it:

```bash
touch /var/folders/.../agent-bridge-<uid>/global.stop
```

The agent prints that exact path too, as `abort_all_command`. The older path
`/tmp/agent-bridge.stop` is still honoured if it is what your fingers know, but
prefer the printed one.

Both are checked before every send, and while a send is waiting for a busy peer.
Clear them with `reset` (this pane) or `reset --all` (everything). Set
`AGENT_BRIDGE_ABORT` to move the global path.

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

It clears itself after 3 hours (`AGENT_BRIDGE_STALE_TIMEOUT`). To clear it
now, in that pane:

```bash
python3 <skill>/scripts/agent_bridge.py reset
```

`reset` ends the old exchange and removes the abort file too, so the pane is
ready immediately. `clear-abort` only removes the abort file — it will not
unstick this.

`status` tells you which case you are in: `start_blocked` and
`expires_in_seconds`.

## Starting over with a fresh bridge

Tell either agent you want a "fresh bridge" or a "new bridge". The agent that
opens it runs `start --fresh`. That ends the old bridge in its own pane and
marks the new frame as fresh. The other pane then drops its old bridge with
that same peer and takes the new one. A fresh frame from any other pane is
still refused, and an abort file still stops everything.

## Locale warnings on macOS

If bridge commands print `bash: warning: setlocale` about `C.UTF-8`, the agent's
command environment uses a locale macOS does not provide. The skill specifies
zsh and a supported UTF-8 locale for macOS commands, including Python shims.
See [the troubleshooting steps](references/failure-modes.md#bash-warning-setlocale-on-macos)
for a check that sends no messages. An already-running agent may need to reread
the updated skill to use these launch settings.

## What stops a bridge

Neither agent stops on its own. Both keep replying until both have said, in a
message, that the bridge is closed: one asks to close, the other confirms. Only
the reasons below end a bridge without that agreement.

- the turn limit is reached — hard; no frame goes out past it
- the goal phrase appears
- the target pane never goes idle
- the frame could not be submitted into the peer's input box
- a frame arrives corrupted (its integrity checksum does not match)
- no reply within the ack timeout (default 10800s, set `AGENT_BRIDGE_ACK_TIMEOUT`)
- an unfinished bridge goes stale (default 10800s, set `AGENT_BRIDGE_STALE_TIMEOUT`)
- the peer is on another tmux server
- you create the abort file

Only a confirmed close, the goal phrase, and your own abort file end things for
good. For every other
reason — the turn limit included — the agent should tell you how the exchange
ended and offer you two choices: reset and open a fresh bridge that carries the
context forward, with however many turns you grant it, or reset and stop. It does
not decide that on its own, and it never continues without you saying so.

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
| `AGENT_BRIDGE_INPUT_TAIL` | `5` | how many bottom lines count as the input box; raise if the target has a thick status bar |
| `AGENT_BRIDGE_READY_TIMEOUT` | `900` | seconds to wait for a busy peer to go idle before giving up |
| `AGENT_BRIDGE_FOCUS` | `notify` | tells the target pane it has focus, so a TUI that holds unfocused keystrokes accepts them; nothing moves on screen. `off` skips it |

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

`type` is the old path: it sends small paced chunks. Two knobs shape it, and both
do nothing in `paste` mode:

| Variable | Default | Meaning |
| --- | --- | --- |
| `AGENT_BRIDGE_CHUNK` | `8` | characters per chunk |
| `AGENT_BRIDGE_CHUNK_PAUSE` | `0.08` | seconds between chunks; raise if characters get dropped |

The other cause is the copy, not the transport: the receiving agent re-types the
frame out of its prompt instead of saving it byte for byte, and escapes quotes
or drops indentation on the way. So a body over a few hundred characters is
written to a file and the frame carries only the path and a SHA-256 of the
contents — about 350 characters, with no quotes and no escapes to get wrong.
This is automatic; both sides just need the same version of the script.

## Tests

```bash
python3 -m unittest discover -s tests
```

254 tests, no dependencies, no tmux server needed — they stub the transport and
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

Logs and turn state live under `$TMPDIR/agent-bridge-<uid>/`, one set per pane. Run
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

```
                        ONE TMUX SERVER
   +---------------------+                 +---------------------+
   |   Agent A  (%1)     | --- 1 frame --> |   Agent B  (%2)     |
   |                     | <-- 1 reply --- |                     |
   +---------------------+                 +---------------------+
             |                                       |
             v                                       v
     <hash>-1.state.json                     <hash>-2.state.json
     <hash>-1.log                            <hash>-2.log
     <hash>-1.abort   <-- stops this bridge  <hash>-2.abort
             |                                       |
             +------------- both check --------------+
                                |
                                v
                    <state root>/global.stop
                       stops EVERY bridge
```

Per-pane files live under `$TMPDIR/agent-bridge-<uid>/`, named by tmux socket hash
and pane number. That is why panes 1↔2 and 3↔4 can run at the same time
without touching each other. The only shared thing is the global stop file.

## Troubleshooting

Start with `references/failure-modes.md`. It lists each symptom with its cause,
including two readiness rules that were tried and rejected because they failed
against real panes: requiring a prompt-shaped line (breaks on a themed zsh
prompt) and matching spinner glyphs (marks every idle Claude Code pane as busy).
