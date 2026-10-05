# Blueprint: a state and power dashboard for physical machines

A specification for a small self-hosted page that shows every machine you own — cluster
nodes, bare workstations, virtual machines hosted on other machines — and lets you stop
and start them safely, including the ones that share a single smart plug.

Each machine gets one state, derived from three independent signals: whether it answers
on the network, what its outlet reports, and what the cluster thinks of it. The states
worth seeing exist only in the combination.

These documents are written to be **built from**, not just read. Point a coding agent at
this directory and it should have enough to produce a working implementation in whatever
language you prefer, and enough to tell whether what it built is correct.

## The documents

| | |
|---|---|
| [01-concept.md](01-concept.md) | The problem, the idea, and when *not* to use this |
| [02-specification.md](02-specification.md) | Normative requirements: signals, derived state, actions, power groups, jobs, simulated mode, programmatic access |
| [03-acceptance-criteria.md](03-acceptance-criteria.md) | Testable criteria. Build against these |
| [04-usage.md](04-usage.md) | What living with it looks like, and how to read a row |
| [05-reference-implementation.md](05-reference-implementation.md) | Non-normative: one implementation's choices, and the traps it hit |

`02` and `03` are the contract. `01`, `04` and `05` are context — useful, but you can
disagree with them and still have built the right thing.

## Pointing an agent at this

Something like:

> Read this directory and build the application it specifies. Use <your language>.
> `02-specification.md` is normative — where it says MUST, that is not a suggestion.
> Work test-first against `03-acceptance-criteria.md`; every criterion should map to a
> test that fails before you write the code that satisfies it. `05` describes a prior
> implementation: treat its decisions as informative and its **Traps** section as
> hard-won, but do not copy its stack unless it suits you.

Three things worth saying to the agent explicitly, because they invert normal instincts:

1. **Simulated mode is the default, and it is a safety property.** Not a developer
   convenience to be bolted on later, and not something real mode falls back to. A
   deployment that loses its configuration must be incapable of powering anything off.
   See [02 §7](02-specification.md#7-simulated-mode).
2. **Removing a control beats disabling it.** A machine that shares an outlet with three
   others can never be started on its own. Individual Start must not exist for it. A
   greyed-out button with a tooltip is an invitation to work out why and then remove the
   guard.
3. **An action reports success only when it observed the end state.** "The shutdown
   command was sent" is not "the machine is off". Where the end state cannot be
   observed, the correct outcome is *indeterminate*, said out loud.

## Scope of this blueprint

**In:** a declared inventory of machines, three-signal state derivation, graceful stop
followed by power cut, start via power, power groups sharing one outlet (including a hard
power cycle for a wedged set), virtual machines whose start depends on a host,
long-running actions as observable jobs, a simulated estate as the default mode, and an
optional, deliberately narrow programmatic entry point for agents.

**Out:** provisioning, configuration management, scheduling, capacity planning, metrics
and alerting, wake-on-LAN discovery, out-of-band management protocols, anything that
needs to work when you are not looking at it. If you need those, you need a real
infrastructure management system and this is the wrong shape.

**Next door:** the reference implementation has a second page that starts remote CLI
sessions on the machines. That is a separate blueprint, [rc-launcher](../rc-launcher/),
because it stands on its own — and because it runs free-text commands, which puts it
under a threat model this one does not need. If you build both into one service, keep
the session routes off the agent entry point; §2.9 of that specification says why.

## Licence and provenance

Written up from a working implementation running a lab of about ten machines. Nothing
here is tied to that implementation's language, cluster, smart-home platform or hosting;
where an example is needed, it uses `example.com` placeholders.
