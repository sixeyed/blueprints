# 02 — Specification

Normative. **MUST**, **MUST NOT**, **SHOULD** and **MAY** carry their usual weight.
Nothing here names a language, framework, cluster, smart-home platform or remote-access
protocol.

Three external systems are referred to by role throughout:

| Role | What it is |
|---|---|
| **the power controller** | Whatever can report and change the state of an outlet |
| **a remote shell** | Whatever can run a command on a machine and return its output and exit status |
| **the orchestrator** | Whatever holds cluster membership for the machines that are members |

Each is an *adapter*. The application MUST NOT depend on the particular identity of any
of them.

---

## 1. Entities

### 1.1 Machine

| Field | Notes |
|---|---|
| `id` | Stable, unique, URL-safe. Never reused |
| `display_name` | Shown to a person |
| `hardware` | Free text. What it physically is |
| `address` | How the reachability probe and remote shell reach it |
| `shell_user` | The account used for every remote command on this machine |
| `outlet` | Reference the power controller understands. Nullable |
| `power_group` | Reference to a group (§4). Nullable |
| `orchestrator_node` | Name this machine has in the orchestrator. Nullable |
| `host` | `id` of the machine this one runs on, for guests. Nullable |
| `expected_stop_seconds` | Used for progress only (§5.6) |
| `expected_start_seconds` | As above |

**1.1.1** The inventory MUST be declared in configuration, not discovered at runtime.
*Discovery cannot see a machine that is off, and a machine that is off is the one the
page exists to show you.*

**1.1.2** Every field MUST be nullable where the corresponding capability may be absent,
and the absence MUST be meaningful rather than defaulted. A machine with no `outlet` and
no `power_group` has no power signal and no start capability; that MUST be a modelled
state, not an error and not a guess.

**1.1.3** A machine MUST NOT carry both an `outlet` and a `power_group`. Group members
draw their power signal from the group (§4.2).

**1.1.4** Where a machine has a `host`, the host MUST also be a machine in the inventory.
Host references MUST NOT form a cycle, and the application MUST refuse to start if they
do.

### 1.2 Power group

| Field | Notes |
|---|---|
| `id` | Stable, unique, URL-safe |
| `display_name` | |
| `outlet` | Reference the power controller understands. Required |
| `members` | Two or more machine `id`s |

**1.2.1** A machine MUST belong to at most one power group.

**1.2.2** A group with one member is permitted but MUST behave identically to a machine
with its own outlet in every respect except where §4 says otherwise.

### 1.3 Job

| Field | Notes |
|---|---|
| `id` | Unique |
| `subject` | What the job acts on (§5.3) |
| `action` | `stop`, `start`, or another declared action |
| `status` | `pending`, `running`, `succeeded`, `failed`, `indeterminate`, `refused` |
| `created`, `started`, `finished` | Timestamps |
| `expected_seconds` | Copied from the subject's configuration at creation |
| `log` | Ordered, append-only entries |

### 1.4 Observation

A single reading of one signal for one machine, carrying a value, an explicit
acquisition outcome, and the time it was taken.

**1.4.1** An observation that could not be acquired MUST be recorded as such, with a
reason, and MUST NOT be represented as a negative value. *"The outlet is off" and "the
power controller did not answer" are different facts, and collapsing them is precisely
what makes a wedged machine look switched off.*

---

## 2. Signals and derived state

### 2.1 The three signals

| Signal | Values |
|---|---|
| **Reachability** | `reachable`, `unreachable`, `unknown` |
| **Power** | `on`, `off`, `unknown`, `not_applicable` |
| **Membership** | `ready`, `not_ready`, `cordoned`, `absent`, `not_applicable` |

**2.1.1** Each signal MUST be acquired independently. The failure of one MUST NOT prevent
the other two being acquired or reported.

**2.1.2** Each signal MUST have an `unknown` value distinct from its negative value, and
`unknown` MUST be visually and structurally distinguishable from the negative wherever
the signal is presented.

**2.1.3** `not_applicable` MUST be used only where the machine genuinely has no such
dimension — no outlet and no group; not a member of any cluster. It MUST NOT be used to
paper over a failed acquisition.

### 2.2 Derivation

**2.2.1** The derived state MUST be a pure function of the three observations. It MUST
NOT consult any other source, and MUST NOT be cached separately from the observations it
came from.

**2.2.2** The derivation MUST produce at least these distinct states:

| Reachability | Power | Derived state |
|---|---|---|
| `reachable` | `on` or `not_applicable` | Running |
| `unreachable` | `on` | **Powered but unresponsive** |
| `unreachable` | `off` | Off |
| `unreachable` | `unknown` or `not_applicable` | Unreachable, power unknown |
| `reachable` | `off` | **Signals disagree** |
| `unknown` | any | State unknown |

**2.2.3** "Powered but unresponsive" MUST be a distinct state with its own presentation.
*This state is the reason the application exists; an implementation that renders it as
"off" has built a worse version of the network probe it replaced.*

**2.2.4** "Signals disagree" MUST be surfaced as its own state. An implementation MUST
NOT resolve the contradiction by preferring one signal. *A reachable machine whose outlet
reports off means a stale reading or an outlet mapped to the wrong machine, and the
second of those will otherwise be discovered by switching off something important.*

**2.2.5** Membership MUST NOT override reachability or power in the derived state. It
MUST be presented alongside, and MUST be shown whenever `orchestrator_node` is set —
including when the machine is unreachable, where the last known membership and its age
MUST be shown rather than nothing.

**2.2.6** Where membership indicates the node is carrying workloads, that count MUST be
available to the person before they act. §3.2.1 depends on it.

### 2.3 Acquisition

**2.3.1** Signals for all machines MUST be acquired concurrently. Sequential acquisition
across an estate MUST NOT be used. *One unresponsive machine then delays the whole page
by its own timeout, and the estate's worst case is the sum of every machine's.*

**2.3.2** Every acquisition MUST be bounded in time, and the connect phase MUST be
bounded **separately and explicitly** from the operation that follows it. *A command
timeout does not cover establishing the connection; a machine that has power and is
dropping packets stalls for the operating system's default connect timeout, which is far
longer than anything the application configured.*

**2.3.3** A remote shell command MUST NOT be attempted against a machine that has not
first passed a short reachability probe.

**2.3.4** The aggregate acquisition for a full listing MUST complete within a bound that
is shorter than the interval at which any client refreshes it. Where that cannot be
guaranteed, the application MUST prevent a second acquisition starting while one is in
flight. *A backstop longer than the poll interval does not fix a stall, it multiplies it
— three overlapping requests instead of one.*

**2.3.5** A listing MUST render with partial data. Signals that did not arrive within the
bound MUST be reported as `unknown` with a reason, and MUST NOT block the response.

---

## 3. Actions

### 3.1 General

**3.1.1** Every action MUST be performed as a job (§5).

**3.1.2** A control for an action that cannot succeed for a given subject MUST NOT be
presented for that subject. It MUST NOT be presented in a disabled form. *A disabled
control asserts that the action is possible and currently inconvenient; removal asserts
what is true, and cannot be re-enabled by someone who did not read this document.*

**3.1.3** Every action MUST be refused, before doing anything, when its preconditions are
not met, and the refusal MUST name the precondition that failed.

### 3.2 Stop

**3.2.1** A stop MUST proceed in this order:

1. Verify preconditions and record them in the job log.
2. Issue a graceful shutdown through a remote shell.
3. Wait for the machine to become unreachable.
4. Only once unreachability is confirmed, and only if the machine has an outlet or a
   group outlet with no other running members, change the power state.

**3.2.2** Loss of the remote shell connection during or immediately after the shutdown
command MUST be treated as success of that command. *A shutdown does not return an exit
status — the connection drops. An implementation that treats a dropped connection as a
failure reports every successful stop as a failure.*

**3.2.3** Step 3 MUST have its own bound. Exceeding it MUST NOT be reported as success,
and MUST NOT silently proceed to step 4. The job MUST end `indeterminate`, stating that
the machine did not become unreachable within the bound.

**3.2.4** Power MUST NOT be cut to a machine that is still reachable, except by an
explicit override supplied with the request. Such an override MUST be a separate,
deliberate input, MUST be recorded in the job log, and MUST NOT be the default.

**3.2.5** Stopping a machine that is already unreachable and whose power is `off` MUST
succeed without action, and the log MUST say that nothing was done.

**3.2.6** Where the machine is an orchestrator member, its membership and workload count
MUST be presented before the stop is confirmed. The application MAY offer to prepare the
node for removal from service; where it does, that MUST be a distinct action from the
stop, with its own job.

**3.2.7** Stopping a machine that hosts guests MUST be refused while any guest is
reachable. The refusal MUST name the guests.

### 3.3 Start

**3.3.1** A start MUST be presented only where a start mechanism exists: an outlet, a
group outlet, or a host that can start a guest.

**3.3.2** A start MUST:

1. Verify preconditions and record them.
2. Engage the start mechanism.
3. Wait for the machine to become reachable, within a bound.

**3.3.3** Completion MUST be determined by observing the machine, never by the start
mechanism's own acknowledgement. *The power controller confirming that it switched an
outlet says nothing about whether anything on the other side of it booted.*

**3.3.4** Where the machine does not become reachable within the bound, the job MUST end
`indeterminate`, not `failed` and not `succeeded`. The power state MUST be left as the
start left it, and the log MUST say so.

**3.3.5** Starting a guest MUST be refused while its host is not reachable. The refusal
MUST name the host.

---

## 4. Power groups

A power group exists because one outlet feeds several machines. Each requirement below
follows from that and has no other justification.

**4.1** A machine's power signal MUST be derived from its own `outlet` where it has one,
and from its group's outlet where it does not. *Without the fallback, "off" and "powered
but unresponsive" are indistinguishable for every group member.*

**4.2** A group member MUST NOT be given an outlet of its own in the inventory, and the
application MUST reject a configuration that gives it one.

**4.3** Start MUST be a group-level action. Individual Start MUST NOT be presented for
any member of a group — it MUST be absent from the interface and MUST be refused if
requested directly. *One outlet cannot bring up one of four machines, and no amount of
retrying will change that.*

**4.4** A group start MUST be refused when any member is currently reachable. The refusal
MUST name the reachable members and MUST state that starting the group will power-cycle
them.

**4.5** The refusal in §4.4 MUST be overridable only by an explicit parameter supplied
with the request, which MUST NOT be the default, MUST be recorded in the job log, and
MUST require a deliberate confirmation that names the affected machines. *A set start
that silently hard-cycles a machine somebody is using is worse than any error message.*

**4.6** A group stop MUST attempt a graceful stop of each member individually, and MUST
be best-effort: a member that is unreachable, already off, or that exceeds its bound MUST
NOT abort the remaining members. *A set stop that gives up because one machine was
already off is useless, and one machine being already off is the ordinary case.*

**4.7** A group stop MUST cut group power only after every member is confirmed
unreachable. If any member is still reachable, the group job MUST end `indeterminate`
with power left on, naming the members that did not stop — unless the override of §3.2.4
was supplied.

**4.8** A group job MUST report a per-member outcome. A single aggregate status MUST NOT
be the only result. *An aggregate "succeeded" over four members, one of which never went
down, is the characteristic failure of this whole design.*

**4.9** Individual Stop MUST remain available for a group member, and MUST NOT touch the
group outlet.

---

## 5. Jobs

**5.1** Every action MUST produce a job with an identifier, a status, an ordered
append-only log, and a copied expected duration.

**5.2** Job state MUST survive a client disconnecting, navigating away and returning. A
client MUST be able to attach to a running job by its identifier and see the log from the
beginning.

**5.3** A job MUST be keyed by **subject**, where a machine and a group are distinct
subjects. *A check that looks only at the machine's own jobs will happily start a second
concurrent stop for a member in the middle of a set stop, or cycle its outlet while it is
shutting down.*

**5.4** Before creating a job, the application MUST refuse if there is an active job on:

- the same subject;
- the group a machine subject belongs to;
- any member of a group subject;
- the host of a guest subject, or any guest of a host subject.

Each of these MUST be checked explicitly. The refusal MUST name the conflicting job.

**5.5** A job MUST NOT report `succeeded` unless the intended end state was observed.
Where the outcome cannot be established, the terminal status MUST be `indeterminate`, and
that status MUST be presented as distinct from both success and failure. *This is the
single most important requirement in this document: an action that reports success while
the machine is in an unknown state is the failure this application is built to prevent,
and it is the one that leaves no trace.*

**5.6** Progress indication MUST be derived from the copied expected duration and MUST be
presented as an estimate. Reaching or exceeding the expected duration MUST NOT, on its
own, change the job's status. *Expected durations exist so the person knows whether to
keep waiting; a bar that hits 100% and claims completion teaches them to stop looking.*

**5.7** The log MUST record each step as it is attempted — the decision taken, the
command issued and its outcome — not only the final result. It MUST NOT contain
credentials, key material or the contents of any secret.

**5.8** Terminal jobs MUST be retained long enough to be read after the fact, and their
retention MUST be bounded.

---

## 6. Errors and reporting

**6.1** A failure to acquire a signal MUST be recorded with its reason and surfaced as
`unknown`, never as a healthy value and never as a silently empty result.

**6.2** Suppressing an error to keep the page rendering is permitted **only** on the
listing path, and only to the extent of turning a signal into `unknown` with a reason.
*One dead machine must not blank the whole page — but a suppression that has no reason
attached makes an authentication failure, a wrong command path and a powered-off machine
look identical, and each of those needs a different response from you.*

**6.3** Error suppression MUST NOT be applied to any action path. An action that cannot
complete MUST fail loudly (§5.5).

**6.4** Where a signal is `unknown`, the reason MUST be available to the person without
reading logs on the host.

**6.5** Adapter failures MUST preserve their distinctions. An unrecognised failure MUST
remain a generic failure and MUST NOT be mapped onto a specific meaning such as "no such
thing". *Turning an unfamiliar error into a familiar one hides the problem rather than
reporting it, and the mapping is always written from one example.*

---

## 7. Simulated mode

**7.1** The application MUST support a simulated mode, in which every adapter is replaced
by an in-memory model of the estate and no external system is contacted.

**7.2** Simulated mode MUST be the default. Real mode MUST require explicit configuration
to enable. *A deployment that lost or never received its configuration must be incapable
of powering anything off.*

**7.3** **Every request path MUST function in simulated mode.** A path that cannot be
exercised without real hardware is a defect in the simulation, not a limitation of it,
and MUST be fixed in the fake rather than excused in the path.

**7.4** The simulated adapters MUST be **strict**. They MUST reject input that the real
adapter would reject, and MUST reproduce the real failure — including a malformed
command, a missing path prefix, an unknown host, and an operation on a machine that is
off. They MUST NOT succeed where the real adapter would fail. *A permissive fake converts
every bug into a bug that only appears in real mode.*

**7.5** The real and simulated implementations of each adapter MUST expose the same
interface — the same operations, with the same parameters and the same return shapes. An
automated check MUST assert this, and MUST fail when they diverge. *Without it the two
drift apart silently, and a green suite against a drifted fake is worse than no suite,
because it is believed.*

**7.6** The real and simulated implementations MUST raise the same error types for the
same conditions. Where the real adapter can only report a generic failure, the simulated
one MUST NOT report something more specific. *Otherwise the request paths get written
against the fake's richer contract, and the tests assert outcomes production never
produces.*

**7.7** The active mode MUST be stated in the interface, on every view that offers an
action, and MUST be reported by the health endpoint.

**7.8** The active mode MUST NOT be changeable through the interface or by any request.
It is deployment configuration. *A runtime toggle puts real hardware one click away from
whoever has the page open.*

**7.9** In real mode, the application MUST validate at startup that each adapter has the
configuration it needs, and MUST refuse to start otherwise. It MUST NOT fall back to
simulated mode for an adapter that is misconfigured. *A silent fallback is how you end up
trusting a fake reading of a real machine.*

---

## 8. Interface

**8.1** There MUST be a single view showing every machine in the inventory with its
derived state.

**8.2** The derived state MUST be distinguishable at a glance across the whole list,
without reading text. *The question the page exists to answer is "is anything wrong right
now", and that must be answerable in one look.*

**8.3** Each machine's row MUST make its three underlying signals available, along with
how recently each was observed.

**8.4** Controls MUST reflect capability per §3.1.2: present only where the action can
succeed for that subject.

**8.5** An action that cuts power, power-cycles a running machine, or overrides a refusal
MUST require a deliberate confirmation that names what will be affected.

**8.6** The view MUST refresh on an interval, and the refresh MUST be resilient to one
machine being slow (§2.3.4, §2.3.5).

**8.7** A running job MUST be visible from the machine's row, and its log reachable from
there.

**8.8** A health endpoint MUST exist. It MUST report the active mode and MUST NOT
disclose credentials or inventory detail beyond what the interface already shows.

---

## 9. Configuration

Minimum surface:

| Setting | Required | Notes |
|---|---|---|
| Inventory (machines, groups) | yes | §1 |
| Mode | no | Defaults to simulated (§7.2) |
| Power controller credentials | conditional | Real mode only |
| Remote shell credentials | conditional | Real mode only |
| Orchestrator credentials | conditional | Real mode only |
| Reachability probe timeout | no | |
| Connect timeout | no | Separate from command timeout (§2.3.2) |
| Command timeout | no | |
| Listing aggregate bound | no | Must relate to the refresh interval (§2.3.4) |
| Refresh interval | no | |
| Job retention | no | |

**9.1** Configuration MUST be validated at startup, not at point of use. Validation MUST
include §1.1.4, §1.2.1 and §4.2.

**9.2** Every configuration field that is accepted MUST be used. A field that is parsed
and then ignored MUST NOT exist. *An accepted setting that does nothing is a lie told at
the worst moment — you change it, redeploy, and reason about the result as though it took
effect.*

**9.3** Where a value is configurable per machine, every code path that needs that value
MUST read it from configuration. It MUST NOT be duplicated as a constant anywhere. *One
path honouring the configured account and three others assuming a different one produces
a system where you create something as one user and cannot see it as the other, with no
error that explains why.*

**9.4** Credentials MUST be supplyable by file reference as well as by value, so that they
can be mounted rather than placed in an environment or a process listing.

**9.5** Machine identifiers MUST NOT be reused for a different physical machine. *Job
history and outlet mappings are keyed on them.*

---

## 10. Deployment

**10.1** The application MUST be reachable only by people authorised to power machines
off. It MUST NOT be exposed without authentication. *Every control on this page has a
physical consequence and none of them is reversible from the page once power is gone.*

**10.2** Where the application holds state for running jobs in memory, exactly one
instance MUST run. *Two instances each see only their own jobs, and §5.4's concurrency
refusals stop working across them.*

**10.3** Deployment changes MUST be verified against the live object, never against the
deployment tool's reported success. *A deployment tool that reports success and bumps a
revision is not evidence that the configuration you edited was applied.*

**10.4** Name resolution for a newly added machine MUST be verified by the address it
resolves to, not by whether it resolves at all. *A network with a wildcard record answers
for every name in the domain, so a machine you have not configured still "resolves", to
the wrong address.*
