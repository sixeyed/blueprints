# 01 — Concept

## The problem

Interactive CLI tools have started growing remote-control modes. You run one on a
machine, it registers a session, and you drive that session afterwards from a phone
app or a web UI. Claude Code's Remote Control mode is the driving case here, and the
shape generalises.

The catch is in the first sentence. *You run one on a machine.* Something has to start
the session, on the right machine, in the right project directory, before any of the
remote driving can happen. So the feature that frees you from your desk requires you
to be at your desk to use it.

That is the whole problem. It sounds small. It is the difference between a tool you
use and a tool you keep meaning to use.

| Approach | Why it falls short |
|---|---|
| SSH from a phone client | Works, and is miserable. You are typing an absolute project path on a soft keyboard, and mobile SSH clients drop the connection when the app backgrounds — taking the session with it unless you have already solved the multiplexer problem yourself |
| Remote desktop | Enormous for the job. You are driving a whole GUI over a phone screen to type one command |
| Leave a session running permanently | Now it is in whatever directory you left it in, possibly days stale, and you cannot start a second one for a different project |
| A web-terminal daemon on each machine | Solves the terminal, and opens a shell to anyone who reaches that port. It listens on the machine, so it is reached without passing whatever authenticates your other services |
| A hosted cloud dev environment | A different product. Your machines, your files and your signed-in accounts are the reason you are doing this at all |

The recurring failure is that each of these makes you do the fiddly part — choosing a
machine, choosing a directory, keeping the session alive — by hand, on a phone, at the
moment you least want to.

## The idea

**One page. Pick a machine, pick a directory, press a button, and a terminal opens on
a session that will outlive the page.**

The launcher connects to a machine you own, creates a session inside a terminal
multiplexer, starts the driving tool in the directory you chose, and attaches a
terminal in the browser. You watch it come up, satisfy whatever it asks for, and close
the tab. The session is still there. You pick it up in the vendor's own app.

The launcher's job ends the moment the session exists and you can see it. It is a
starter motor, not an engine.

## Why this shape

**The multiplexer is not an implementation detail, it is the design.** Three properties
come from it and nothing else provides all three: the session outlives the request that
created it, it has a real terminal attached to it, and a client hanging up detaches
rather than kills. The middle one is the subtle one — without a terminal, a tool built
for interactive use drops into non-interactive mode and exits, so the failure you get is
*it started and then vanished*, not *it hung*.

**Start a shell, then type the command into it.** The obvious launch is to make the
command the session's own process. Then the session dies the instant the command does —
and the command dies instantly and routinely, because the machine is signed out, or the
directory is not trusted, or a dozen other ordinary reasons. The session evaporates
before the terminal can attach, and the error goes with it.

Keeping the dead process's output on screen is the natural next thought, and it is not
enough: you can read *sign in to continue* and you cannot sign in, because a retained
corpse takes no input. Starting an ordinary interactive shell and delivering the command
to it as input costs nothing, runs the command identically, and leaves you at a working
prompt in the right directory with the error above it. *Every recoverable failure becomes
recoverable from the phone that hit the problem.*

**The launch is not reported as success or failure.** Creating a multiplexer session
succeeds whether the process inside it connected or died in 200 milliseconds. There is no
server-side signal that separates those. An earlier design captured the session's output
and classified it — matching a connection banner against a sign-in prompt against a trust
error — which is pattern-matching on console text that will change wording without
warning.

Attaching the terminal immediately dissolves the problem rather than solving it. Nobody
adjudicates. The session is created, the terminal opens on it, and what happened is on
the screen where a human can read it.

**Nothing new listens on the machines.** The launcher reaches them through the remote
shell service they already run, and bridges the browser terminal over that same
connection. The alternative — a terminal daemon per host — is both a new shell exposed
to the network and a route that bypasses the front door, and therefore the
authentication.

## Threat model

Start here, because it changes what "done" means.

**The command box is free text by design.** It has to be: the point is to run an
arbitrary CLI tool with arbitrary arguments in a directory of your choosing. That makes
this page remote code execution as a *feature*, on every machine it knows about, for
whoever can reach it.

There is no version of this where the input is sanitised into safety. Filtering free
text produces a boundary people believe in and attackers step over. The only real control
is that nobody unauthenticated ever reaches the page, the API, or the terminal transport.

**Defends against:** nothing, on its own. Every security property this system has comes
from the authenticating layer in front of it, and from there being exactly one way in.

**Requires:** authentication on every route, enforced before the request reaches the
application; no second network path to the launcher's operations by any address, port or
protocol — including an API-key door added for automation; and browser-supplied values
quoted or filtered for every shell that will interpret them, so that a *directory name* or a *session name* cannot become a command.

**Does not defend against:** anyone holding a valid identity. They have a shell on your
machines. That is what you built.

*If you cannot put real authentication in front of this, do not build it.* An
unauthenticated launcher is worse than no launcher, and it will not look broken.

## When not to use this

- **You are at the machine.** Open a terminal. This exists for the case where you are
  not, and using it from your own desk adds a browser, a proxy and a websocket to a
  thing you could type.
- **The driving tool already starts sessions remotely.** If the vendor ships a way to
  create a session from the app, use it. This blueprint is a workaround for the absence
  of that feature, and it should lose to a first-party one.
- **Machines you do not own, or that hold other people's data.** The launcher's gate is
  "is this person authenticated", not "is this person allowed to do this here". There
  is no per-directory authorisation and adding one does not fix the free-text command
  box.
- **You cannot deploy it behind an authenticating proxy.** See above. This is the one
  entry on the list that is not a matter of taste.
- **More than a handful of machines.** The listing path touches every machine you have
  configured; at a few dozen you want an agent on each host reporting in, which is a
  different and much larger system.
- **Unattended or scheduled work.** The whole design assumes a person is watching the
  terminal, because the design's answer to every failure is "you can see it and fix it".
  Nobody watching means nobody fixing.

The sweet spot is four or five machines you personally own, a handful of projects on
each, and an idea that arrives while you are somewhere else.
