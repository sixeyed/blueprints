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
readable end to end in one sitting. It is a few hundred lines of server code and about
three hundred of client.

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

**One name for two things.** The same string is the multiplexer session name and the
driving tool's `--name`, which the launcher appends itself. What `tmux ls` shows and what
the vendor's app shows are identical, which removes a whole class of "is that the same
session?" confusion. The name is `<directory basename>-<HHMM>` unless the operator types
one; either way it goes through one filter — runs of anything outside `[A-Za-z0-9_-]`
become `-`, edges trimmed, 60 characters at most.

**The default command is configuration.** It lives in the same inventory file as the
machines, and the page receives it as a data attribute on the command box rather than
as a constant in the script. One source of truth, and changing the flags is an edit and
a chart deploy. The current default is
`claude rc --spawn=same-dir --permission-mode auto`.

**Simulated is the default mode.** An environment variable picks the adapters: simulated
(an in-memory set of machines; nothing leaves the process) or live. A container that
forgets the variable cannot run anything on anything, and the page carries a banner
saying which mode is live — in both modes, because "Live mode" is worth saying on the
page that runs commands on your machines.

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

### …and then it came back

A day of use later, a list of sessions all called `blueprints-1412`,
`blueprints-1438`, `blueprints-1503` was a list you had to open one by one.
Naming a session turned out to be genuinely useful, so the field returned — with a
caller this time, and behind the **same** filter the generated name already went
through, not a second one written for it.

The test that had asserted "the name field is ignored" was now asserting the wrong
contract. It was rewritten to assert the property that made removal necessary — whatever
comes back contains no character outside the allow-list — and proven by bypassing the
filter and watching it fail. Rewriting a security test because the feature changed is
fine. Deleting it because it went red is how the field reached a shell the first time.

Two follow-ons. The name is **not** remembered between launches, because it identifies
one session. And a command that carries its own `--name` is refused with a 400, because
the launcher appends one and two `--name` flags give tmux one identity and the vendor's
app another.

### A remembered command shadows the configured one

The command box remembers what you last typed, in browser storage. The first version
stored whatever was in the box at launch — usually the default, untouched.

When the default moved into configuration and then changed, the new default deployed
correctly, was served to the page correctly, and never appeared: browser storage still
held a copy of the old default, and a remembered value wins over a default. It looks like
the deploy failed.

Store a command only when it differs from the configured default, and clear the stored
value when the operator puts the box back. The client test pins the box against the
configured value, not a literal, so changing the default does not mean editing the test.

### The simulation that could not reach the core flow

The directory list comes from running a short script over SSH. The simulated SSH adapter
did not recognise that script, so in simulated mode the endpoint could not succeed at all
— the page loaded, you selected a machine, and the one thing the page is for could not be
exercised or demonstrated without a real machine on the network.

Nothing failed, because nothing tested the page end to end in simulated mode. Treat an
operation the simulation cannot reach as a defect, and keep a test that compares the live
and simulated adapters' method names and signatures, or the simulation quietly stops
meaning anything.

### The machine is a prerequisite, not a bug

A machine must be signed in to an account with the right entitlement for the driving tool
to run at all. It refuses instantly otherwise.

This is a **per-machine prerequisite**, and the temptation is to detect it, report it, and
tell the operator to go and fix it on the machine. Do not. The attached terminal is how
you satisfy it, from wherever you are — and if it were not, the launcher would have moved
the need to be at your desk rather than removed it.

### Credentials the launching session cannot read

The sharpest version of the prerequisite above, and the one that cost the most time,
because it does not look like a credential problem at all.

The driving tool stored its credentials in the operating system's user keyring. A
desktop session unlocks that keyring at login; **a non-interactive remote shell does
not**, and cannot. So the tool, run over the launcher's connection, found no
credentials and reported that the operator was not signed in — on machines where they
demonstrably were, and where the same command worked perfectly when typed at the
machine itself.

Three things made it expensive to diagnose:

- **The message named the wrong cause.** "Not signed in" sends you to sign in again,
  which writes to the keyring again, which the remote shell still cannot read. The
  loop is stable and gives no new information.
- **One machine worked.** Its platform had no keyring and used a file, so the same
  build behaved differently on different machines for reasons nothing in the tool
  surfaced.
- **The fix had already happened by accident.** Signing in *through the attached
  terminal* — a remote shell, where the keyring is unavailable — made the tool fall
  back to a file, and that file is readable by every later session. The machine that
  worked had been fixed this way without anyone noticing.

So the rule is: **the credential must live in a store the launching session can read**,
and satisfying that is the operator signing in through the attached terminal, not at
the machine. Keyring and file are independent stores that do not feed each other;
signing in at the desktop is invisible to the launcher, and the reverse is also true.

Two things worth verifying rather than assuming, because we assumed both and were
wrong once each:

- **That refreshes persist to the same store.** A short-lived access token is renewed
  from a longer-lived one. If the renewal is written back to the keyring while the
  session reads the file, everything works until the first expiry and then fails
  identically to the original bug. Force an expiry and prove a real call still
  succeeds afterwards.
- **That a headless token carries the scope you need.** The obvious workaround is a
  long-lived token in the environment, but a token minted for headless use may carry
  only an inference scope, while registering a remote session is a *different* scope.
  A token that can run a prompt and cannot create a session looks like a working
  credential and is not one.

*When testing any of this, prove the instrument works before trusting its verdict:
verify the probe command succeeds on a valid credential first. Three consecutive
diagnoses here were wrong because the probe never ran — a status subcommand that
never refreshes, a timeout utility absent on the platform, and a command that read
the same standard input the test script was being fed through — and each time the
script reported a confident conclusion about authentication instead of noticing it
had measured nothing.*

## The interface

One page, in this order.

**Machines**, at the top: one card per configured machine — its id, its address, a
display name, the directory currently chosen for it, **Select** and **Launch**, and a row
for each session running on it with **Open** and **Kill**. Machines come first because
selecting one is what loads the directory list. With the fields first, the card you had
just selected was off-screen on a tablet and a tap looked like it had done nothing.

**Session name**: optional, empty on every load, with a note saying which characters
survive.

**Command**: pre-filled with the remembered command, or the configured default.

**Project directory**: a filter box over the trusted directories of the selected machine,
read live over SSH from the driving tool's own record of trusted projects, with the
machine's `$HOME` (also read live) removed. One machine had 93 and another had 4, so a
plain select is unusable at one end and fine at the other. Typing narrows the list, at
most 100 rows are drawn, and the chosen row is ticked where the tap happened. Choosing
a row remembers it for that machine and enables that machine's Launch.

**The terminal** opens in a modal that takes 90% of the viewport — focused synchronously
on the click, sized to the dialog, with Copy all and a Close that detaches rather than
kills. Launch opens it on the new session with no second step.

The sessions list polls every ten seconds. Kill and Launch force a refresh past the
in-flight guard, and a sequence number discards stale replies.

## Configuration used here

The machines and the default command are in a YAML inventory, mounted from a ConfigMap:

```yaml
sessionCommand: "claude rc --spawn=same-dir --permission-mode auto"

sessionHosts:
  - id: studio-01
    name: Studio
    ssh: { host: studio-01.example.com, user: you }   # port defaults to 22
    # Neither claude nor tmux is on a non-interactive PATH on macOS.
    pathPrefix: "PATH=/opt/homebrew/bin:/usr/local/bin:$HOME/.local/bin:$PATH"
```

`pathPrefix` is put in front of every remote command — tmux, the directory script, the
driving tool. A missing id, host or user stops the service at startup; so does a
duplicate id.

| Setting | Default | |
|---|---|---|
| `..._MODE` | `simulated` | `simulated` or `real`; never per request |
| `..._SSH_KEY` | a mounted path | a file path; the key is mounted, not embedded |
| reachability probe | 3 s | plain TCP connect to the SSH port, gating every SSH call |
| SSH connect | 10 s | bounds the connect phase specifically |
| list command | 20 s | `tmux ls` on one machine |
| list, per machine | 8 s | overall cap on one machine's share of the fan-out — below the poll interval, so polls cannot stack |
| launch command | 30 s | create and send-keys, in one SSH call |
| directories | 20 s | the trusted-projects script |
| poll interval | 10 s | client-side |

The timeouts are constants in code, not configuration. That has been fine for four
machines; it is the first thing to change for a fifth on a slower link.

## Deployment notes

- **One ingress, and every route behind it.** The Ingress guards every path with the
  auth subrequest, including the websocket upgrade. No route that launches, kills,
  lists directories or attaches a terminal is reachable any other way.
- **No node ports, no host ports, no "temporary" plain-HTTP route.** There used to be a
  NodePort, and it is not coming back — see the trap above. Verify from outside with no
  credentials after every deployment change, not after every code change.
- **The websocket needs the auth check applied to its upgrade**, and not every proxy
  configuration does that by default. Check it explicitly; it is the route most likely to
  be left open, because it is neither a page nor an API call.
- **Mount the SSH key, do not embed it.** And give it only the access the launcher needs
  on each machine.
- **Pin host keys.** Give the SSH client each machine's host key and have it refuse any
  other ([§10.6](02-specification.md#10-deployment)). Whoever answers for a machine's
  name receives the launcher's key and a command to run. Rebuilding a machine then means
  updating its pinned key, which is a small price for that.
- **Machines that sleep will dominate your latency budget.** Either wake them, exclude
  them, or make sure the reachability probe is short enough that one asleep machine costs
  a fraction of a second.
- The launcher is stateless. Restarting it disturbs nothing; the sessions belong to the
  multiplexers on the machines and are rediscovered on the next list.

## Where it falls short of the specification

Written down rather than smoothed over, because a blueprint whose reference quietly
disagrees with it teaches the disagreement.

- **An unreachable machine looks empty.** The listing returns no sessions for a machine
  that failed its probe or timed out, and the card renders exactly as it would for a
  machine with nothing running. [§7.3](02-specification.md#7-listing-and-lifecycle)
  requires it to be reported as unreachable, and that is the right requirement: "no
  sessions there" and "could not ask" are different facts, and the second is the one
  that should stop you launching.
- **No way to type a directory.** The API accepts any directory, but the page only offers
  the trusted list. [§5.4](02-specification.md#5-working-directories) asks for an entry
  box; in practice its absence has been tolerable because trusting a new directory is
  something you do at a prompt anyway.
- **Kill has no friction.** It is a link next to Open, with no confirmation. A thumb has
  not hit it yet.
- **The disambiguator is the minute.** Two unnamed launches from one directory inside
  the same minute collide; the second is refused with a 409 rather than typed into the
  first ([§3.12](02-specification.md#3-launching-a-session)), which is safe but
  surprising.
- **`pathPrefix` is optional and unchecked.** A machine entry without one starts fine and
  fails at the first remote command with 127, which is the point-of-use failure
  [§9.1](02-specification.md#9-configuration) exists to prevent.
- **The health check is not empty.** It returns the mode and a count. It is reachable
  only from inside the cluster, for the kubelet, but
  [§2.2](02-specification.md#2-authentication) says nothing at all.
