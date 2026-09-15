# 03 — Acceptance criteria

Build against these. Each is a behaviour you can assert on; each should map to at least
one automated test that fails before the code satisfying it exists.

The failure mode of this system is **an action that reports success while the machine is
in an unknown state**. The stop job goes green, the row says off, and the machine is
actually mid-shutdown, or wedged with power still on, or was never reachable to begin
with. Nothing crashes and nothing logs an error — you find out by walking to the rack.
Criteria in [§3](#3-stop), [§4](#4-start) and [§6](#6-jobs-and-concurrency) are what
catch it.

Its close relative is **a signal that looks healthy because an error was swallowed to
keep the page rendering**. Every implementation grows that suppression for a good reason:
one unreachable machine must not blank the page. Criteria in
[§7](#7-error-surfacing) are what stop it spreading into the paths where it is fatal.

Because most of these criteria are about hardware you cannot have in a test, **the
simulated adapters are the test harness**. That makes [§8](#8-simulated-mode-and-adapter-parity)
load-bearing for every other section: if the fake stops resembling the real thing, every
test above it is asserting against fiction.

---

## 1. Inventory and configuration

- [ ] **1.1** A machine with no outlet and no power group loads successfully and reports
      its power signal as not-applicable, not as unknown and not as off.
- [ ] **1.2** A configuration giving a power-group member its own outlet is refused at
      startup.
- [ ] **1.3** A configuration with a guest whose host is not in the inventory is refused
      at startup.
- [ ] **1.4** A cycle in host references is refused at startup.
- [ ] **1.5** Every configuration field the loader accepts changes observable behaviour.
      *Set each one to a distinctive value and assert it reaches the place it governs.
      A field nothing reads should not be accepted.*
- [ ] **1.6** The account used for remote commands comes from configuration on every code
      path that issues one. *Change it to a value that cannot work and assert every
      remote path fails, not one of them.*
- [ ] **1.7** Startup fails on invalid configuration rather than at the point of use.

## 2. Signals and derived state

- [ ] **2.1** Reachable plus outlet on derives Running.
- [ ] **2.2** Unreachable plus outlet on derives **Powered but unresponsive**, as its own
      state, not as off.
- [ ] **2.3** Unreachable plus outlet off derives Off.
- [ ] **2.4** Reachable plus outlet off derives **Signals disagree**, and is not silently
      resolved to Running.
- [ ] **2.5** Unreachable plus an unavailable power controller derives unknown power —
      distinct from off.
- [ ] **2.6** A failure of one signal does not prevent the other two being reported for
      the same machine.
- [ ] **2.7** A machine that is an orchestrator member shows membership even while
      unreachable, with the age of the reading.
- [ ] **2.8** Membership never changes the derived state. *A NotReady node that is
      reachable is still Running; assert both fields independently.*
- [ ] **2.9** Signals for the whole estate are acquired concurrently. *Make several
      machines each take a known delay and assert total elapsed time is close to the
      slowest, not to the sum. Then revert the concurrency and watch this fail —
      see §10.2.*
- [ ] **2.10** No remote shell command is issued to a machine that failed its
      reachability probe. *Assert on the adapter not being called at all, not on the
      result.*
- [ ] **2.11** The connect phase is bounded independently of the command. *Simulate a
      host that accepts nothing and never refuses, and assert the probe returns within
      the configured connect bound — well inside the platform default.*
- [ ] **2.12** A listing renders every machine when one machine's signals time out, with
      that machine's signals unknown.
- [ ] **2.13** The aggregate listing bound is shorter than the configured refresh
      interval, or a second acquisition cannot start while one is in flight. *Assert one
      or the other holds; a configuration where neither does should be refused.*

## 3. Stop

- [ ] **3.1** A stop issues a graceful shutdown before touching any outlet. *Assert on
      call order, not on the end state.*
- [ ] **3.2** A remote shell connection dropping during the shutdown command is recorded
      as success of that command, not as a failure.
- [ ] **3.3** Power is changed only after the machine is confirmed unreachable.
- [ ] **3.4** A machine that does not become unreachable within its bound produces an
      `indeterminate` job, power untouched, naming the machine.
- [ ] **3.5** Power is never cut to a reachable machine without the explicit override.
- [ ] **3.6** The override is off by default, and its use appears in the job log.
- [ ] **3.7** Stopping an already-off machine succeeds without issuing any command, and
      the log says nothing was done.
- [ ] **3.8** Stopping a host with a reachable guest is refused, naming the guest, and
      nothing is issued.
- [ ] **3.9** A machine that is an orchestrator member exposes its workload count before
      the stop is confirmed.

## 4. Start

- [ ] **4.1** A start succeeds only once the machine is observed reachable.
- [ ] **4.2** The power controller acknowledging the outlet change does not, on its own,
      complete the job. *Have the fake confirm the switch and never bring the machine up;
      assert the job does not report success.*
- [ ] **4.3** A machine that does not come up within its bound produces `indeterminate`,
      not `failed` and not `succeeded`, with the power left engaged.
- [ ] **4.4** Starting a guest whose host is unreachable is refused, naming the host.
- [ ] **4.5** A machine with no outlet, no group and no host has no start control in any
      response, and a directly requested start is refused.

## 5. Power groups

- [ ] **5.1** A member with no outlet of its own reports the group outlet's power state.
- [ ] **5.2** A member unreachable with the group outlet on derives **Powered but
      unresponsive**. *This is §2.2 for the case that has no plug of its own, and it is
      the one implementations miss.*
- [ ] **5.3** No response contains an individual start control for any group member.
      *Assert on absence from the payload or markup — not on it being disabled, and not
      on a tooltip.*
- [ ] **5.4** A directly requested individual start for a member is refused.
- [ ] **5.5** A group start with any member reachable is refused by default, and the
      refusal names the reachable members.
- [ ] **5.6** That refusal is overridable only by an explicit parameter, and the override
      is recorded in the job log.
- [ ] **5.7** A group stop where one member is already off still stops the others.
- [ ] **5.8** A group stop where one member times out still attempts every remaining
      member. *Assert each member's adapter was called.*
- [ ] **5.9** A group stop cuts group power only when every member is confirmed
      unreachable.
- [ ] **5.10** A group stop where one member stays reachable ends `indeterminate` with
      power still on, naming that member.
- [ ] **5.11** A group job result carries a per-member outcome, not only an aggregate.
- [ ] **5.12** An individual stop of a member never changes the group outlet.

- [ ] **5.13** Stopping a set where every machine shuts down cleanly ends with
      the shared power control **off**, and the job says so.
- [ ] **5.14** Stopping a set where one machine is still reachable afterwards
      leaves the power control **on**, and the job names that machine. *Verify
      the control was never switched, not merely that the job reported a
      problem.*
- [ ] **5.15** Stopping a set with no power control configured still succeeds,
      touches nothing, and reports that the set remains powered.
- [ ] **5.16** A machine that has halted on a set whose control still reads *on*
      reports **stopped**, not unreachable.
- [ ] **5.17** A machine with its **own** power control, unreachable while that
      control reads *on*, still reports **unreachable**. *This is the wedged
      case; 5.16 must not cost you it where the signal really
      does describe the machine.*
- [ ] **5.18** A set whose control reads *on* while none of its machines respond
      reports that state at the set level.

## 6. Jobs and concurrency

- [ ] **6.1** No job reports `succeeded` without having observed the intended end state.
      *For each action, build a case where the end state is unobservable and assert the
      terminal status is `indeterminate`.*
- [ ] **6.2** `indeterminate` is presented distinctly from both success and failure, in
      every place a status is shown.
- [ ] **6.3** A second job on the same subject is refused while one is active, naming the
      conflicting job.
- [ ] **6.4** A member job is refused while a job on its group is active.
- [ ] **6.5** A group job is refused while a job on any member is active.
- [ ] **6.6** A guest job is refused while a job on its host is active, and vice versa.
- [ ] **6.7** A job survives the client disconnecting; reattaching by id returns the log
      from the beginning.
- [ ] **6.8** Exceeding the expected duration does not, alone, change a job's status.
- [ ] **6.9** No job log contains a credential or key material. *Search the whole log of
      a run through every action.*
- [ ] **6.10** The log records each step as attempted, including the decisions that led
      to a refusal.

## 7. Error surfacing

- [ ] **7.1** An unreachable machine does not blank or truncate the listing for the
      others.
- [ ] **7.2** Every suppressed listing error produces an `unknown` signal with a reason
      attached, reachable by the person without host access.
- [ ] **7.3** An authentication failure, an unrecognised command and an unreachable host
      produce three distinguishable reasons. *All three currently render as an empty
      result in the natural implementation; that is the criterion.*
- [ ] **7.4** No action path suppresses an error. *Make each adapter raise on an action
      and assert the job fails or goes indeterminate — never succeeds, never silently
      returns.*
- [ ] **7.5** An unrecognised adapter failure remains generic and is not mapped to a
      specific meaning.

## 8. Simulated mode and adapter parity

- [ ] **8.1** With no configuration at all, the application starts in simulated mode.
- [ ] **8.2** Real mode requires explicit configuration to enable.
- [ ] **8.3** Every route in the application returns a non-error response in simulated
      mode. *Enumerate the routes from the router and iterate — do not maintain a hand-
      written list, which is how a route escapes the check.*
- [ ] **8.4** Real and simulated adapters expose identical operation names and
      signatures. *Compare them programmatically; assert the check fails when a parameter
      is added to one side only.*
- [ ] **8.5** For each condition either adapter can raise, both raise the same type.
      *Especially: where the real adapter can only report a generic failure, the fake
      does not report something more specific.*
- [ ] **8.6** The simulated remote shell rejects a command without its path prefix with
      the same not-found exit status a real machine gives.
- [ ] **8.7** The simulated adapters refuse an operation against a machine they model as
      off.
- [ ] **8.8** The active mode is present in the health response and in every view
      offering an action.
- [ ] **8.9** Mode cannot be changed by any request.
- [ ] **8.10** Real mode with a missing adapter credential refuses to start, and does not
      fall back to simulated for that adapter.

## 9. Interface

- [ ] **9.1** Derived state is conveyed by a semantic marker in the markup, not only by
      styling. *Assert on a state attribute. A test asserting on a class name will break
      on a redesign for no behavioural reason, and teach you to edit tests until they
      pass.*
- [ ] **9.2** Every style class the markup uses exists in the served stylesheet. *If the
      stylesheet is a committed build output, grep the artefact. No DOM test catches
      this, because DOM tests assert structure and not computed style.*
- [ ] **9.3** Every page that shares a script loads it without an unhandled error.
      *Load each page and assert the console is clean — a top-level access to an element
      only one page has kills every handler defined after it.*
- [ ] **9.4** Actions with physical consequence require a confirmation naming what is
      affected.
- [ ] **9.5** A running job is reachable from its machine's row.

## 10. Operational

- [ ] **10.1** Restarting the application preserves inventory and the record of terminal
      jobs.
- [ ] **10.2** For every test added to cover a fix, the fix has been reverted and the
      test observed to fail. *Two tests in the reference build passed with their subject
      removed — one named for concurrency that pinned a timeout rather than the fan-out,
      and one gate with no coverage at all.*
- [ ] **10.3** A deployment change is verified against the live object, not the
      deployment tool's reported success.
- [ ] **10.4** A newly added machine's name resolves to the expected address, checked
      against the authoritative resolver. *Not merely that it resolves — a wildcard
      answers for names you never configured.*

---

## Coverage note

An implementation passing §2, §3 and §4 works. An implementation passing §6, §7 and §8 is
*trustworthy*, and the second set is where a reimplementation is most likely to fall
short — not through carelessness, but because the natural implementation of each is
obviously correct and quietly wrong. "Report success when the command was accepted" reads
as reasonable. "Return an empty list when the machine cannot be reached" reads as
defensive. Both produce a page that is confidently incorrect about hardware, which is the
one thing this application must never be.
