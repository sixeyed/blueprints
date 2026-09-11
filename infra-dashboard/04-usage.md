# 04 — Usage

## Reading a row

The page is a list, one row per machine, and the row is meant to be read in one pass:

```
node-a.example.com   small node       Running          ready · 14 workloads    [Stop]
node-b.example.com   small node       POWERED, NO ANSWER   ready (4m ago)      [Stop] [Cut power]
desk-01.example.com  workstation      Off                 —                    [Start]
vm-01.example.com    guest on node-a  Running             —                    [Stop]
bench-01            bench group       SIGNALS DISAGREE    —                    [Stop]
```

Two states in that list are doing the work.

**`POWERED, NO ANSWER`** is the state a network probe alone cannot produce. The outlet is
drawing power, the operating system is not answering. That is a machine to go and look
at, or to cut power to deliberately — and because the last membership reading is still
shown with its age, you also know it was a healthy cluster member four minutes ago, which
is a different problem from one that has been wedged since Tuesday.

**`SIGNALS DISAGREE`** means it answers on the network and its outlet reports off. Almost
always the outlet is mapped to the wrong machine in your configuration. *Fix that before
you press anything, because the next Stop will graceful-shutdown one machine and cut
power to another.*

## The states, and what each one asks of you

| State | What it means | What you do |
|---|---|---|
| Running | Reachable, powered | Nothing. Stop is safe, subject to workloads |
| Powered but unresponsive | Has electricity, not answering | Investigate. If you are sure, cut power with the override |
| Off | Unreachable, outlet off | Nothing. Start if you want it |
| Unreachable, power unknown | It does not answer and nothing can say why | Check the power controller before anything else — you are one signal down |
| Signals disagree | Contradiction | Fix the configuration. Do not act on the row |
| State unknown | The probe itself failed | Look at the reason on the row, not at the machine |

## A typical stop

Stopping a cluster node is the common case and the one worth walking through.

1. **Look at the membership column first.** Fourteen workloads on the node is not a
   reason not to stop it, but it is a reason to know. If the tool offers to prepare the
   node for removal from service, that is a separate action with its own job — do it
   first and watch the count fall.
2. **Press Stop.** A job opens with a log.
3. **Watch the log, not the bar.** The bar is an estimate built from a number you put in
   the configuration. The log says what actually happened:
   ```
   14:02:11  preconditions ok — no guests, no active group job
   14:02:11  issuing graceful shutdown
   14:02:13  connection dropped — treating as shutdown accepted
   14:02:13  waiting for unreachable (bound 180s)
   14:03:47  unreachable confirmed after 94s
   14:03:47  outlet off
   14:03:49  done
   ```
4. **If it ends `indeterminate`, read why.** Usually the machine did not go unreachable
   inside its bound. Power is untouched, deliberately. That is the tool declining to
   guess, and it is the behaviour you want — the alternative is a green tick over a
   machine that was still writing to disk.

## Living with power groups

A group is several machines on one plug. Four small nodes on a bench, say.

**Start the group, not a machine.** There is no per-machine Start on a group member,
because one plug cannot bring up one of four. If you find yourself looking for it, you
are looking for something that does not exist — the control is on the group row.

**Group start with something already running is refused.** You will see the members it
would power-cycle, named. Nine times in ten that refusal is correct and you wanted to
start one member you cannot start. The override exists for the tenth: you know the bench
is wedged and you want the lot cycled. Take it deliberately.

**Group stop is best-effort and that is the point.** One member is usually already off.
The job stops the others and reports per member:

```
bench-01  stop group
  bench-a  unreachable at start — skipped
  bench-b  shutdown accepted, unreachable after 41s
  bench-c  shutdown accepted, unreachable after 38s
  bench-d  still reachable after 180s
  → indeterminate: power left on, bench-d did not stop
```

*Read the per-member lines, not the headline.* A group of four where one never went down
must not end green, and that job log is the reason this specification insists on
per-member outcomes.

## Declaring a machine

The inventory is configuration. Adding a machine means writing it down, which is the
point — a machine that is off cannot be discovered, and a machine that is off is what you
came to look at.

```yaml
machines:
  - id: node-a
    display_name: node-a.example.com
    hardware: small node, 32GB
    address: node-a.example.com
    shell_user: ops
    outlet: outlet.node_a
    orchestrator_node: node-a
    expected_stop_seconds: 120
    expected_start_seconds: 90

  - id: vm-01
    display_name: vm-01.example.com
    hardware: guest VM
    address: vm-01.example.com
    shell_user: ops
    host: node-a                # no outlet: it has no power of its own
    expected_stop_seconds: 30
    expected_start_seconds: 45

power_groups:
  - id: bench-01
    display_name: bench group
    outlet: outlet.bench
    members: [bench-a, bench-b, bench-c, bench-d]
```

Three things to get right when you add one:

- **Measure the durations, do not guess them.** Stop one by hand with a stopwatch. These
  numbers only drive the progress estimate, and an estimate that is wrong by a factor of
  three is worse than no estimate.
- **Group members get no `outlet`.** Their power reading comes from the group. The
  application should refuse a configuration that gives them one.
- **Check the name resolves to the right address**, against the resolver that is
  authoritative — not merely that it resolves. A wildcard record answers for names you
  have never configured, so a typo "works" until the first Stop goes somewhere else.

## Simulated mode, in practice

The application starts simulated. Everything works: states change, jobs run and log,
groups refuse and override, and nothing anywhere is touched. Run it that way to develop,
to demonstrate, and to try an action you are not sure about before doing it for real.

Real mode is a deliberate configuration change, and the page says which mode it is in on
every view that has a button. *If you ever find yourself unsure which mode a deployment
is in, that is a bug in the interface, not in your attention.*

## Anti-patterns

- **Cutting power because the row says unreachable.** Unreachable with power unknown is
  one signal, and it is the one that tells you least. Find out about the outlet first.
- **Acting on a "signals disagree" row.** It means your configuration maps an outlet to
  the wrong machine. Acting on it is how you find out which one.
- **Treating `indeterminate` as success.** It is the tool telling you it does not know.
  Go and look.
- **Raising the expected durations until the bar looks right.** The bar is cosmetic; the
  bounds on the waits are not. Raise the bound if a machine genuinely takes longer,
  having timed it.
- **Using the group-start override as the normal route.** If you take it routinely, one
  of your members is wedged and you have been power-cycling three healthy machines to
  avoid diagnosing it.
- **Leaving a stale machine in the inventory after decommissioning it.** It sits there
  permanently unknown and trains you to ignore unknown rows, which are the rows that
  matter.
