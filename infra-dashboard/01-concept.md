# 01 — Concept

## The problem

You own ten machines. Three are nodes in a cluster. Two are workstations that are not in
anything. Four are virtual machines hosted on two of the others. One is a box under a
desk that exists to run one service. Some sit behind a smart plug; some are plugged
straight into the wall.

The question you ask about any one of them is always the same: **what state is this in,
and is it safe to stop it?**

The information needed to answer that lives in three systems that do not know about each
other:

- **Reachability** — does it answer on the network? Does a remote shell connect?
- **Power** — what does its outlet report?
- **Cluster membership** — is the node Ready, is it cordoned, how many workloads is it
  carrying?

Each of those has a tool, and each tool answers confidently about its own dimension and
knows nothing about the other two. Answering the real question means opening three of
them and doing the join in your head — while also remembering which machine has a plug,
which is a guest on which host, and which is a cluster node at all.

| Approach | Why it falls short |
|---|---|
| The cluster's own node list | Knows only about members, and only while they are up. A powered-off node looks like a crashed one. It has never heard of the workstation, the plug, or the host underneath the guests |
| The smart-home app | Tells you the outlet is switched on. Says nothing about whether the operating system on the other side of it is alive |
| A terminal and `ping` | Reachable or not. Never *why* not, which is the only part that decides what you do next |
| A monitoring system | Excellent at machines that are up and reporting. A machine that is off does not appear as off — it appears as absent data, and there is no button |
| Out-of-band management (IPMI, BMC, DCIM) | The right answer if you have it. Consumer and small-office hardware mostly does not, and that is who this is for |

## The idea

**One state per machine, derived from three independent signals.**

The signals are gathered separately, none can veto another, and each can come back
*unknown* as a first-class value distinct from its negative. Then they are combined —
and the combination is the whole point, because the states worth seeing exist only there:

| Reachable? | Outlet says | Derived state |
|---|---|---|
| yes | on, or nothing to ask | **Running** |
| no | on | **Powered but unresponsive** — it has electricity and it is not answering |
| no | off | **Off**. Unremarkable, and nothing to worry about |
| no | unknown | **Unreachable, power unknown** — you have one fact and it is the useless one |
| yes | off | **The signals disagree** — a stale reading, or the outlet is mapped to the wrong machine |

Row two is the reason the tool exists. A wedged machine and a machine somebody switched
off at the wall are the same observation from the network, and completely different
problems. No single signal can tell them apart; the pair does it trivially.

Row five matters almost as much, and the instinct to resolve it with precedence — trust
reachability, ignore the plug — is wrong. *A contradiction between two signals is a
configuration error announcing itself, and suppressing it means you will discover the
mis-mapped outlet by switching off the wrong machine.*

Cluster membership is the third signal and it sits on top rather than beside: a Running
node that is `NotReady`, or cordoned with workloads still on it, is a machine you can
reach and should not stop yet.

## Why this shape

**The inventory is declared, not discovered.** Every machine is described in
configuration: its identifier, display name, hardware, how to reach it, how it stops, how
it starts, how long each is expected to take. Discovery is tempting and cannot work here.
It would find cluster nodes, but not the machine hosting them, not the outlet, and not a
machine that is currently off — which is precisely the machine you opened the page to
look at.

**Stopping is graceful first, then power.** A stop issues a clean shutdown over a remote
shell, waits for the machine to go away, and only then touches the outlet. Cutting power
to a running machine is not stopping it, it is pulling the plug, and a tool that makes
that the easy path will eventually eat a filesystem.

**Expected durations are per machine, in configuration.** A small node and a workstation
with a lot of memory differ by minutes. *A progress indicator that lies is worse than no
progress indicator, because it teaches you to stop watching.*

**A disconnect is the success signal.** A remote shutdown command does not return an exit
status — the connection drops mid-sentence. That has to be modelled as success. Model it
as an error, in the obvious way, and every successful stop in the system reports as a
failure.

**Actions are jobs, not fire-and-forget.** Each has an identifier, a status and a live
log. A physical operation takes minutes, so it must be observable while it runs,
survivable across a page reload, and impossible to start twice.

**A job is keyed by subject, not by machine.** A group action and a member action are
different subjects, and every check that one is safe to start has to look at the other
explicitly. Without that, a member can be handed a second concurrent stop in the middle
of a set stop, or have its outlet cycled underneath it while it is shutting down.

## Power groups

Several machines can share one smart plug. That single fact breaks the per-machine model
in three specific ways, and it is worth being precise about which.

**Start is a set operation.** One outlet cannot bring up one of four machines. The Start
control belongs to the group. Individual Start must be *removed* from members — not
disabled, not greyed with a tooltip. Removed, because it can never work, and a disabled
control is a standing invitation for someone to find out why and delete the guard.

**Stop is per machine and best-effort.** Shut each member down gracefully, skip the ones
that are already unreachable or that time out, and carry on. A set stop that aborts
because one member was already off is useless, and that is the ordinary case.

**A member has no outlet of its own.** Its outlet reference is null by design and always
will be, and its power probe falls back to the group's. Skip that and "off" and "powered
but unresponsive" become indistinguishable for exactly the machines whose power you have
least direct control over.

There is a fourth consequence, and it is the one that needs a deliberate decision.
Starting a group while some members are already running means power-cycling the running
ones. That must be refused by default and overridable only by a deliberate act that names
what it is about to hard-cycle. The alternative — a set start that silently yanks power
from a machine someone is working on — is far worse than an error message.

## Simulated mode is a safety property

The whole application runs against an in-memory copy of the estate, and that is the
**default**. Touching real hardware is an explicit opt-in. A container that loses its
configuration, or a deployment that never received it, cannot power anything off; it
serves a convincing dashboard of a lab that does not exist. Both the interface and the
health endpoint say which mode is live.

It is also what makes the thing developable — every button works end to end with no
remote shell access, no smart-home token and no cluster. That is a genuine benefit and it
is not the reason for the default. The reason is that the worst outcome of a
misconfigured deployment should be a fake dashboard, not a dark rack.

Two rules make the property hold, and both are easy to erode:

- **Every request path must work in simulated mode.** When one stops working that is a
  defect, not a gap. In the reference build one endpoint became *structurally
  unreachable* in simulated mode, because the fake remote shell did not recognise the
  command it issued — so the core flow of a page could not be exercised or demonstrated
  without real hardware, and the test that would have caught it could not be written.
- **The fake must be strict, not permissive.** A command sent without its path prefix
  comes back as exit 127, "command not found", exactly as the real machine would answer.
  A fake that shrugs and succeeds hides the bug until real mode, which is the one place
  you did not want to find it.

An automated **adapter parity test** — that the real and simulated implementations expose
matching method names and signatures — is what stops simulated confidence quietly ceasing
to mean anything. Without it the two drift, and the drift is invisible: the test suite
goes green against a fake that no longer behaves like the thing it stands in for.

## When not to use this

- **Your machines are all cloud instances.** Your provider's console already has one
  state per instance, a stop that is graceful, and a start that works. There is no
  physical power dimension to reconcile, so the central idea has nothing to do.
- **Your machines are all cluster nodes, and nothing else.** If every machine is a member
  and none of them has an outlet you control, two of the three signals collapse into one
  and the cluster's own tooling is better at it than this will be.
- **You have real out-of-band management.** IPMI, a BMC, a DCIM system, managed PDUs with
  per-outlet control and an API — that is this tool's job done properly, by hardware, with
  power state as a first-class fact rather than an inference. Use it.
- **You need this to work unattended.** There is no scheduling, no policy engine and no
  remediation. It is a page you look at and press things on.
- **You have one machine.** The join is not hard for one machine. Open two terminals.

The sweet spot is a heterogeneous estate of roughly five to thirty machines, mixed
between cluster members and standalone boxes, with consumer power control and no
out-of-band management, operated by a person who is present.
