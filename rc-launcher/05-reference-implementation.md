# 05 — Reference implementation

**Non-normative.** One way this was built, and — more usefully — the things that went
wrong. Treat the **Traps** section as the valuable part. Every one of them is first-hand;
each cost a round trip or shipped and had to be fixed. Most of them are invisible in
testing, and several present as something entirely unlike their cause.

## Shape

Python with an async web framework, an async SSH library, a browser terminal emulator,
and `tmux` on the target machines. Deployed to a Kubernetes cluster; authentication is an
auth subrequest at the ingress to an OIDC proxy, so every route is guarded by
construction. The SSH private key is mounted as a file.

Four target machines: two Mac Studios, an Intel NUC, a MacBook. The mix matters —
different home paths, different package-manager locations, and one machine that sleeps.

Small on purpose. A service that runs arbitrary commands on your machines should be
readable end to end in one sitting.

## Decisions worth repeating

**A shell, then keystrokes.** The session runs an interactive shell in the target
directory; the command is delivered with the multiplexer's send-keys equivalent. This is
[§3.4](02-specification.md#3-launching-a-session) and it is the highest-value structural
choice in the whole build. It converts every instant-exit failure from "the session
vanished and the error with it" into "you are at a prompt with the error above you".

**Attach instead of adjudicate.** Nothing on the server decides whether the launch
worked, because nothing on the server can. The terminal opens and the operator reads the
screen.

**Auth in front, not in the app.** The application has no authentication code at all.
Every route — page, API, websocket upgrade — is guarded because the only path to the
application traverses something that checks first. There is no route to forget to
decorate.

**One name for two things.** The same generated string is the multiplexer session name
and the driving tool's `--name`. What `tmux ls` shows and what the vendor's app shows are
identical, which removes a whole class of "is that the same session?" confusion.

**Nothing new on the machines.** The websocket bridge relays over the SSH connection the
launcher already makes. No daemon, no new port, no second thing to keep patched.

## Traps

### The naive launch shape kills the session

`tmux new-session -d -s NAME -c DIR 'COMMAND'` makes the command the session's process.
The session dies the instant the command does — and the driving tool exits 1
*immediately* when the machine is not signed in to an account with the right
entitlement, or the directory is not trusted.

So the session is created, returns 0, and is gone before the browser can attach. The
terminal shows the multiplexer's "no sessions" and the actual error is destroyed.

The first fix attempted was `remain-on-exit`, which keeps the dead pane so the error is
readable. **It is not enough.** A dead pane accepts no input, so you can read that you
need to sign in and you cannot sign in — which defeats the entire reason for using SSH
and a multiplexer in the first place. Start a plain interactive shell and type the
command into it.

### Classifying the launch from captured output

An earlier draft had the server capture the pane after launching and classify the text —
matching a connection banner against a setup-wizard prompt against a trust error — so it
could return a meaningful status.

That is guesswork against console text, and it breaks silently the first time the tool
changes its wording. Worse, it encourages the caller to believe the verdict.

There is no server-side signal that distinguishes "connected" from "died instantly",
because session creation returns 0 for both. Do not manufacture one. Attach the terminal
and let the operator read it. *Removing the question was cheaper and more correct than
any answer to it.*

### Text frames versus binary frames

The terminal emulator's data callback hands you a JavaScript string, so `WebSocket.send`
emits a **TEXT** frame. The server read the socket as binary and raised a `KeyError` on
the missing key.

Three things stacked to make this invisible:

- The bridge's blanket exception suppression swallowed the error, so the input pump
  ended *normally*. No traceback, no log line.
- It therefore presented as **the terminal cleanly disconnecting on the first
  keystroke** — a symptom with no obvious relationship to its cause.
- It only happened in real mode. Simulated mode returned before reaching the bridge, and
  the bridge's own tests fed it raw bytes from a fake rather than a real socket.

**The defect lived exactly in the gap between the two tested halves.** Both halves had
coverage. Neither covered the contract between them.

Split frames **by purpose, not by encoding**: binary is keystrokes, text carrying a
recognised control object is a control message, and any other text is still keystrokes.
That tolerance is not sloppiness — it is the thing that would have prevented this.

### A blanket exception suppression is how that stayed invisible

The bug above took an hour to find instead of a minute for one reason: a
`suppress(Exception)` wrapped around the input pump.

Fixing the frame bug while leaving the concealer in place means the next contract
mismatch is exactly as invisible. Narrow the suppression to disconnect-shaped errors, and
log whatever you do swallow.

*The concealer is the more serious defect of the two. The frame bug was one mistake; the
suppression is a guarantee that you will not find the next one.*

### The two pump directions are not independent

A browser going away is a **network half-close**, not a hangup on the remote terminal.
Signalling end-of-input to the remote does not reliably make it notice.

So a quiet PTY leaves the output read blocked forever, holding the SSH connection, the
channel and the remote attach client open indefinitely — for a browser tab that no longer
exists. A chatty session masks this completely, because the next write fails and unblocks
everything. You only see it on a session that is sitting idle, which is most of them.

Run both directions as tasks, return on the first to finish, cancel the other, and close
the connection explicitly.

### Resizing the browser terminal alone is a lie

The remote PTY is created at 80x24. Making the browser window wider and calling the
emulator's fit routine changes the local renderer and nothing else, so output still wraps
at 80 columns.

The first thing this breaks is a long sign-in URL — the single piece of text the operator
most needs to copy out of the terminal, in the failure case the terminal exists to
handle. The client must report its dimensions and the bridge must resize the remote PTY,
on attach and on every change.

### A modal dialog steals focus

The terminal opened in a modal dialog. `showModal()` focuses the dialog's first focusable
control, so the terminal's hidden textarea never receives focus and nothing typed reaches
the session.

On a desktop this is a puzzling extra click. **On iOS it is fatal**: the soft keyboard
only appears — and only routes input — for a genuinely focused input, focused inside the
user's own gesture. Focus the terminal *synchronously inside the click handler* so it
still counts as that gesture, and refocus on tap.

### Subscribing to terminal input on every dialog open

The input subscription was wired up each time the dialog opened. But the subscription
belongs to the **terminal**, which is created once; the dialog opens many times.

After N opens, every keystroke is sent N times. It looks like a stuttering connection or
a flaky keyboard, not like a listener leak. Subscribe where the object is created, not
where it is shown.

### A poll guard that also swallows user-initiated refreshes

An in-flight guard stops a periodic poll stacking up requests when a machine is slow.
Correct, and it silently absorbed the refresh issued right after a kill — so the killed
session stayed on screen until the next tick, and clicking it returned 404.

Force past the guard for refreshes the operator asked for. And add a sequence number
while you are there: once two requests can be in flight, a late reply from the older one
puts the killed session back on the screen, which looks like the kill having failed.

### Listing sessions inside the request path

The list endpoint did live, sequential, un-timed-out SSH to every machine on every poll.

A machine that is powered but silently dropping packets — asleep, or on a dead
network path — stalls for the operating system's TCP connect timeout. Per machine. In
series. While the browser queues another identical request every few seconds. One
sleeping laptop made the whole page unusable.

Gate each host behind a short reachability probe, fan out concurrently, and **bound the
connect phase explicitly**. A command timeout does not help: no command has started yet.

Note also that testing this needs a *blackholed* host, not one that refuses connections.
A refusal returns instantly and exercises none of it.

### Copying a wrapped URL from a touch screen

Selecting text in a terminal on a touch screen is impractical, and the thing most worth
grabbing is a sign-in URL. So there is a copy-all control.

The first version copied the terminal's buffer line by line, which inserts a newline
wherever the display wrapped — straight through the middle of the URL. It looks like it
worked. It pastes as two broken halves. Rejoin wrapped rows before copying.

### `PATH` on a non-interactive login

On macOS, neither the driving tool nor the multiplexer is on the path of a
non-interactive SSH login; both live under a package manager's prefix or `~/.local/bin`.
Every remote command needs an explicit location, or it dies with 127.

This is worse than a broken command, because it produces *confident wrong conclusions*:
an early probe using a bare command name reported that the multiplexer was not installed
on a machine where it was installed and running. Interactive logins load the profile
scripts that fix the path, so every manual check you do to verify the probe will disagree
with the probe.

### A second door defeats the front one

The deployment published a NodePort for plain-HTTP access during development. The auth
subrequest lives at the ingress, and an ingress cannot guard a node port.

So for as long as that existed, "everything is behind authentication" was false,
and nothing about the system looked wrong — the guarded path kept working perfectly, and
the guarded path is the one you test. The node port was removed rather than restricted.

*Any bypass route defeats authentication entirely, and a bypass route is invisible from
the front door.* Test this from outside, with no credentials, against every address and
port the deployment publishes.

### An unused field that reached a shell

The launch API accepted an optional caller-supplied `name`. The interface never sent it,
so nobody looked at it — and it bypassed the sanitiser that handled the generated name,
reaching the multiplexer's inner shell, where `$( )` is expanded on the target machine.

It was **removed rather than patched**. A field with no caller is not a feature that
needs fixing, it is surface that needs deleting. Every optional input that reaches a
shell has to be quoted for every level of interpretation, forever; an input nobody uses
does not earn that.

Related: the generated name itself must have the multiplexer's structural characters
stripped. `.` and `:` are session and window separators for `tmux`, so a directory named
`my.project` produces a session that is created successfully and can then never be
attached to or killed, because every targeting command parses the name instead.

### The machine is a prerequisite, not a bug

A machine must be signed in to an account with the right entitlement for the driving tool
to run at all. It refuses instantly otherwise.

This is a **per-machine prerequisite**, and the temptation is to detect it, report it, and
tell the operator to go and fix it on the machine. Do not. The attached terminal is how
you satisfy it, from wherever you are — and if it were not, the launcher would have moved
the need to be at your desk rather than removed it.

## The interface

One page.

**Launch**, at the top: machine, directory, command, button. The directory control is a
filterable list — one machine had 93 trusted directories and another had 4, so a plain
select element is unusable at one end and fine at the other. Typing narrows it; a
free-text entry covers anything not listed. The machine's home directory is excluded,
compared against `$HOME` read live from that machine.

**Sessions**, below: one row per running session across all machines, grouped by machine,
each with its name, directory, age, and Attach / Kill. Unreachable machines are shown as
unreachable rather than being quietly dropped, because a missing machine reads as "no
sessions there" and that is a different fact.

**The terminal** opens in a modal over the page — focused synchronously on the click, with
a copy-all control and a close that detaches rather than kills.

Design notes that turned out to matter: show the machine on every session row even
though the rows are grouped, because the grouping header scrolls away on a phone; and put
Kill behind enough friction that a thumb does not hit it while scrolling.

## Configuration used here

| Variable | Default | |
|---|---|---|
| `..._MACHINES` | — | required; label, host, user, tool paths per machine |
| `..._SSH_KEY` | — | required; accepts a file path, mounted not embedded |
| `..._PUBLIC_URL` | — | required; builds absolute URLs |
| `..._PROBE_TIMEOUT` | `1` | seconds; reachability gate before the real connection |
| `..._CONNECT_TIMEOUT` | `5` | seconds; bounds the connect phase specifically |
| `..._COMMAND_TIMEOUT` | `10` | seconds; distinct from the above |
| `..._POLL_INTERVAL` | `5` | seconds; client-side session list refresh |
| `..._SIMULATE` | off | development only; visible in the UI when on, not selectable per request |

## Deployment notes

- **One ingress, one path, no exceptions.** No node ports, no host ports, no second
  service, no "temporary" plain-HTTP route. Verify from outside with no credentials
  after every deployment change, not after every code change — this breaks in the parts
  of the configuration the application's own tests cannot see.
- **The websocket needs the auth check applied to its upgrade**, and not every proxy
  configuration does that by default. Check it explicitly; it is the route most likely to
  be left open, because it is neither a page nor an API call.
- **Mount the SSH key, do not embed it.** And give it only the access the launcher needs
  on each machine.
- **Machines that sleep will dominate your latency budget.** Either wake them, exclude
  them, or make sure the reachability probe is short enough that one asleep machine costs
  a fraction of a second.
- The launcher is stateless. Restarting it disturbs nothing; the sessions belong to the
  multiplexers on the machines and are rediscovered on the next list.
