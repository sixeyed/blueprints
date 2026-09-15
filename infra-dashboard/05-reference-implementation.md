# 05 — Reference implementation

**Non-normative.** One way this was built, and — more usefully — the things that went
wrong. Treat the **Traps** section as the valuable part. Several of those failures are
invisible in testing, and the ones that are not invisible are worse: they show up as a
green job over a machine in an unknown state.

## Shape

Python 3.13, FastAPI, Jinja2 templates, an asynchronous SSH library for the remote
shell, a smart-home platform's HTTP API for outlets, the cluster's own asynchronous
client for membership. Jobs held in memory with an append-only log per job, streamed to
the page. A Tailwind stylesheet committed as a build output. Deployed by Helm chart —
to a *different* cluster from the one it manages, deliberately: a dashboard that stops
machines must not be hosted on the machines it stops.

Each of the three external systems sits behind an adapter with two implementations, real
and simulated, and a test that asserts the pair have matching signatures.

About ten machines in the inventory: three cluster nodes, two workstations, four guests
on two of the nodes, and a group of small nodes sharing one plug.

## Decisions worth repeating

**Adapters in pairs, with a parity test.** Every external system has a real and a
simulated implementation and an automated check that they expose the same operations with
the same signatures. This is [§7.5](02-specification.md#7-simulated-mode) and it is the
highest-value structural choice here, because everything else about the simulation is
only worth what that check is worth.

**Simulated as the default, real as opt-in.** Not a flag that switches off safety — the
absence of configuration *is* simulated mode. A deployment that never received its
secrets serves a plausible dashboard of an estate that does not exist, which is the
correct worst case.

**One function derives state.** Three observations in, one state out, no I/O, no cache of
its own. Every interesting case in this design is a combination, and combinations
discovered in three different rendering paths diverge.

**Jobs keyed by subject, with explicit relationship checks.** A machine and a group are
different subjects, and the guard that a job may start walks the relationships — group to
members, host to guests — by name rather than relying on one identifier matching. The
version that compared machine ids happily started a second stop for a member halfway
through a set stop.

**Disconnect is success.** One branch, commented, at the exact point it matters. Without
the comment somebody will "fix" it within a year.

**Controls removed, not disabled.** The template does not render an individual Start for a
group member, and the route refuses one. Two places, deliberately, because the template
is the honest one and the route is the one that has to survive somebody's URL.

## Traps

### The two adapters drifting on error contracts

The simulated adapter raised distinct exception types for "already exists" and "not
found". The real one raised a single generic failure for every non-zero exit status,
because that is all it could tell from the exit status. The routes were written against
the richer contract and mapped those two exceptions to `409` and `404`.

So the test suite asserted status codes that production never returned. Every test
passed. Nothing in production could produce `409` at all.

**Fix the adapter, not the route's mapping.** The real adapter is the one that defines
what can be known; the fake must not offer more. And when you do teach the real adapter to
recognise a specific failure by matching on its output, keep the matching narrow — an
unrecognised failure MUST stay generic. Turning an unfamiliar error into "no such thing"
does not report a problem, it hides one.

### Live I/O inside a request path the page polls

The listing endpoint opened a remote shell to every machine, sequentially, with no
explicit connect timeout, on every request. The browser polled that endpoint every few
seconds.

It was fine in the lab because every machine answered. Then one machine had power and
started dropping packets, and the connect stalled for the operating system's default —
per machine, in series. The page stopped loading entirely, and the cause looked like an
application hang rather than one bad host.

Three separate fixes, all needed:

- **Gate every remote command behind a short reachability probe.** Most of the cost is
  attempting a shell to something that will never answer.
- **Fan out concurrently.** The estate's worst case should be the slowest machine, not
  the sum of all of them.
- **Bound the connect phase explicitly.** A command timeout does not cover establishing
  the connection, and the platform default is measured in minutes.

### A backstop timeout longer than the poll interval

The first attempt at the fix was a backstop timeout on the whole listing: 25 seconds,
against a page that polled every 10.

That does not fix a stall, it multiplies it. Three requests are now in flight at once,
each doing the same expensive work against the same unresponsive host, and the third one
arrives while the first is still holding connections open.

**Keep the aggregate bound below the refresh interval, or prevent a second acquisition
starting while one is in flight.** One or the other. Choosing neither is the default, and
it is invisible until something is slow.

### The inventory and the code that reads it deploy separately

The machine list is configuration, shipped as a file the running application
reads. The parser for it is in the image. Those are two artefacts, and they are
deployed independently - which is easy to forget, because in the source tree
they sit next to each other and change in the same commit.

A schema change was applied to the live inventory while the running image still
required the old shape. The application crash-looped on startup with a
configuration error, and the dashboard went down. The change was correct; the
*order* was not.

Two rules, and the second is the one that makes the first survivable:

1. **Ship the image first, then the config.** The code that can read the new
   shape has to be running before the new shape arrives.
2. **Make the new parser tolerate the old shape**, so step 1 is a no-op that
   changes nothing visible. A field that has been removed should be ignored
   rather than rejected. Without this the two artefacts cannot be deployed
   separately at all, and you are back to needing them atomic - which they are
   not.

Verify the tolerance rather than assuming it: parse the *currently deployed*
config with the *new* parser before deploying either.

*The same edit also reached the repository inside an unrelated commit, because
it was staged with a blanket "add everything" while another change was in
flight. Stage paths you intend, particularly when more than one thing is being
worked on at once.*

### Verifying against the deployment tool instead of the live object

A configuration change was edited, deployed, and reported deployed. The revision number
went up. The change was not there.

The package manager had carried the previous release's stored values forward and ignored
the edited values file. Rendering the templates locally produced correct output the whole
time, which is what made it so hard to see — every piece of evidence short of the live
object agreed that it had worked.

**Diff the live object.** A deployment tool reporting success is a statement about the
deployment tool. For this application specifically, the setting most likely to be silently
carried forward is the one that says which mode it is in.

### A name resolving is not proof your entry landed

The reference network has a wildcard record that answers for any unknown name in the
domain. So every hostname "works" — a machine you configured, a machine you did not, and
a typo.

A new machine was added to the inventory, its name resolved, and the first Stop went to
the wrong address.

**Check the address, not the resolution.** And check it against the resolver that is
authoritative for the domain, rather than the one with the wildcard in front of it. Where
there are two resolvers, a new name has to be added to both, and the wildcard means you
will not notice you only did one.

### Config that is parsed and then ignored

Two instances in this build.

A port field was accepted at load time and read by nothing. It sat in the configuration
schema for months, looking like a knob.

Worse: the account used for remote commands was read from configuration on one code path
and hardcoded on three others. That inconsistency is much worse than the limitation would
have been. You create a session as one user and then cannot see it as the other, with no
error anywhere that explains why — the command succeeds, and it is operating on a
different account's world.

**Every accepted field must be used, and every path that needs a per-machine value must
read it from configuration.** A duplicated constant is not a shortcut, it is a second
source of truth that nobody will remember to change.

### Swallowing errors to keep the page rendering

The listing returned an empty result for a machine it could not reach. That is correct:
one dead machine must not blank the whole page.

But the swallow had no reason attached, so an authentication failure, a command issued
without its path prefix, and a machine that was powered off all rendered
identically — as nothing. Three different problems, three different responses from the
operator, one appearance.

**Scope the swallow to the listing path only, and always attach a reason.** Never let it
cover an action: an action that cannot complete must fail loudly, because the alternative
is a job that reports success over a machine in an unknown state, which is this
application's characteristic failure.

### Generated artefacts that nothing regenerates

The stylesheet is a committed build output. A class that is not in it does not exist at
runtime — the markup references it, the browser ignores it, and the element renders
unstyled.

No test catches that. DOM tests assert structure, not computed style, so the markup is
correct and the page is wrong.

It happened here twice, both times after adding a state to the interface: the new state
had its own class, the class was never built, and the new state rendered looking exactly
like the old one. For a page whose entire job is making states distinguishable at a
glance, that is not a cosmetic bug.

**Either build the artefact in CI, or make checking it part of review — and never accept
"no rebuild needed" without grepping the artefact for the class.**

### A stop that leaves the power on

The set stop shut every machine down gracefully and deliberately never touched
the plug, reasoning that cutting it would hard-cut a member that had not shut
down. The reasoning was sound and the conclusion was too broad.

What the operator got, hours later: four machines halted exactly as asked, the
shelf still drawing power, and every card reading *powered but wedged* - the
one state the dashboard exists to surface as a fault. Nothing was wrong. The
display was describing a correct outcome as a failure.

Cut the plug once **every** member is confirmed down, and not before. If one is
still answering, leave the plug alone and name that machine: the restraint that
motivated doing nothing is right, it simply stops applying when there is
nothing left to protect. Where there is no plug at all, say so - "stopped, shelf
still powered" is a fact worth stating rather than silently skipping.

*Found by an agent doing unattended overnight work through the API, not by a
test. An action that ends in a state nobody looks at until morning is exactly
where this class of defect survives.*

### A shared signal is definitive in one direction only

The corollary, and the more generalisable half.

A machine on a shared plug has no plug of its own, so its power reading comes
from the group's. That looks symmetrical and is not:

- the plug reading **off** is definitive - every machine behind it is off
- the plug reading **on** describes the *shelf*, not the machine in front of
  you: it means "something here has power"

Used symmetrically, a halted machine on a powered shelf becomes "powered but
not answering", which is the signature of a wedged box. The fault is not the
reading; it is treating one direction's certainty as if it ran both ways.

Use the shared reading where it is definitive and stop where it is not. The
ambiguity does not disappear - put it where the signal is still true. A shelf
can honestly say *"powered, and nothing on it is answering"*; a single machine
cannot say anything of the sort from the same data.

The cost is real and worth stating at the point of the change: a genuinely
wedged machine on a shared plug now reads the same as a halted one. That was
the original argument for the fallback, and it is why this reversal needs to be
deliberate rather than incidental.

### Tests that pass with their subject removed

Two found in this build, and both had been trusted for months.

One was named for the concurrent fan-out of signal acquisition. It asserted that the
listing returned within a fixed time. With the concurrency reverted to a sequential loop
it still passed, because the fake adapters were fast enough that the sum was under the
threshold. It pinned a timeout, not the fan-out.

The other was the reachability gate in front of the remote shell. It had no coverage at
all — the behaviour was described in a comment and asserted nowhere.

**When you add a test for a fix, revert the fix and watch the test fail before you trust
it.** For timing-based assertions, assert on the shape of the behaviour — that each
adapter was called, that elapsed time is close to the slowest rather than the sum — not
on a threshold that a fast fake will satisfy either way.

### One script serving two pages

The job log view and the listing view shared a script file. The script had an unguarded
top-level access to an element that only the job view has.

On the listing page that threw, and the throw killed every event handler defined after
it in the file. The buttons rendered and did nothing. No error surfaced to the user; the
page stopped responding to clicks, and only the console said why.

The single-page version had never exercised that path, so the bug arrived with the second
page and looked like a regression in the first.

**Guard top-level DOM access, and load every page in a test asserting the console is
clean.** A shared script is two entry points pretending to be one.

## The interface

A single page: one row per machine, grouped so that a power group's members sit under the
group row.

**State is the loudest thing on the row.** Running is quiet. Powered-but-unresponsive and
signals-disagree are unmistakable without reading. That is the one question the page
exists to answer, so it gets the visual budget.

**The three signals sit behind the state**, each with the age of its reading, so you can
see that a machine has been unreachable for four minutes rather than four hours.

**Controls are per capability.** Stop where a stop is possible. Start only where a start
mechanism exists. Cut power behind a disclosure, with a confirmation that names the
machine. Nothing greyed out — a control that cannot work is not rendered.

**A group row carries Start and Stop for the set**, and its members carry Stop only.

**A running job takes over its row** with a live log and an estimate, and stays reachable
after it finishes so you can read what happened.

Two details that turned out to matter: set the elapsed and remaining figures in tabular
numerals so digits do not jitter as they tick, and make state a semantic attribute in the
markup rather than only a class — the tests assert on the attribute and survive a
redesign.

## Configuration used here

| Variable | Default | |
|---|---|---|
| `..._MODE` | `simulated` | the safety default; `real` is the opt-in |
| `..._INVENTORY_PATH` | `/config/inventory.yaml` | machines and power groups |
| `..._SHELL_KEY` | — | required in real mode; accepts `file:///path` |
| `..._POWER_TOKEN` | — | required in real mode; accepts `file:///path` |
| `..._POWER_BASE_URL` | — | required in real mode |
| `..._ORCHESTRATOR_CONFIG` | — | required in real mode |
| `..._PROBE_TIMEOUT` | `2` | seconds; reachability probe |
| `..._CONNECT_TIMEOUT` | `5` | seconds; separate from the command timeout |
| `..._COMMAND_TIMEOUT` | `15` | seconds |
| `..._LISTING_BOUND` | `8` | seconds; must stay under the refresh interval |
| `..._REFRESH_INTERVAL` | `10` | seconds |
| `..._JOB_RETENTION` | `86400` | seconds |

Expected stop and start durations are per machine in the inventory file, not here.

## Deployment notes

- **One instance only** while jobs are held in memory. Two instances each see only their
  own jobs, and the concurrency refusals stop working across them — which is exactly the
  situation where two stops race one machine.
- **Authenticate it.** Every control has a physical consequence and none of them is
  reversible from the page once power is gone.
- **Deploying it to the cluster it manages is convenient and has one obvious
  consequence**: stopping the node it runs on takes the page with it. Either pin it to a
  node you will not stop from the page, or accept that the last job you start on that
  node is the last thing you see.
- **Check the mode after every deployment**, against the running object rather than the
  deployment output. It is the setting whose silent reversion matters most, and reverting
  it in the safe direction is harmless while reverting it in the other is not.
