# 02 — Specification

Normative. **MUST**, **MUST NOT**, **SHOULD** and **MAY** carry their usual weight.
Nothing here names a CI service, hypervisor, operating system, language or service
manager.

Terms used throughout:

| Term | Meaning |
|---|---|
| **the CI service** | The self-hosted service that holds repositories, workflows, the job queue and runner registrations |
| **job** | One unit of work in the CI service's queue, with the set of labels it must run on |
| **runner** | The program that asks the CI service for a job and runs its steps |
| **registration** | The CI service's record of a runner: an identifier, a name, a description, the labels it declares, and a credential |
| **one-job registration** | A registration the CI service deletes after the runner has taken one job |
| **host** | A physical machine that runs job VMs |
| **the scaler** | The service this document specifies, one per host |
| **golden VM** | The stopped VM image every job VM on a host is cloned from |
| **job VM** | A VM cloned from the golden VM to run one job |

---

## 1. What the CI service must provide

This design works with any CI service that can do the following. If yours cannot, this is
the wrong blueprint as written. At the time of writing Forgejo is the only CI service known
to provide all of it; [01](01-concept.md#which-ci-services-this-fits) says what changes for
GitHub Actions, where §1.1's handle and §1.4 are missing.

**1.1** An API that lists the jobs waiting for a runner, with each job's identifier, its
repository, its required labels, its status, and a **handle**: a value that names this attempt of this
job, so that a runner can ask for exactly that job. The list MAY include jobs the CI service
will not hand out yet, such as a run queued behind another in its concurrency group, and
nothing in it may tell them apart; §5.4.3 handles them. *Forgejo lists such jobs as
waiting, with no task and nothing naming the run.*

**1.2** An API that creates a registration, given a name and description, and returns its
identifier and its credential, with an option to make it a one-job registration.
Registration identifiers MUST be ordered by creation (an increasing number, or a
creation time with a tie-break), because §4.3 compares them.

**1.3** An API that lists registrations in a scope, with each one's identifier, name,
description and status; and one that deletes a registration.

**1.4** A runner that can take exactly the job named by a handle, and no other, and then
exit. *A runner that takes whichever job matches its labels makes the scaler's record of
which VM serves which job fiction.*

**1.5** Registration scopes (a user, an organisation, a repository or the whole service),
where a registration in a scope can take jobs from that scope's repositories.

**1.6** A way for a registration to declare its labels, and be shown as online, without
asking for work. Only §10 needs it; without it, §10 does not apply.

---

## 2. Entities

### 2.1 Host

| Field | Notes |
|---|---|
| `name` | Its short hostname. Unique across hosts |
| `labels` | The labels its job VMs serve |
| `max_vms` | How many job VMs may exist at once |
| `vm size` | CPUs, memory and disk per job VM |
| `scope` | The registration scope; see §3.3 |
| `timeouts` | Claim, job, retry and drain; see §§4–6 |

**2.1.1** A host's configuration MUST be held in version control beside the scaler, and
installed from there. *A host configured by hand is a host whose labels nobody can see.*

**2.1.2** Each host MUST also serve a label of its own, derived from its name (for
example `vm-runner-<name>`), so that a test can target one host.

### 2.2 Run

The scaler's record of one job it is serving.

| Field | Notes |
|---|---|
| `job` | The waiting job, as listed |
| `handle` | From the job |
| `vm` | The job VM's name; unique per job attempt |
| `registration` | Its identifier, once registered |
| `started` | When the scaler took it |
| `claimed` | Whether the runner has taken the job |
| `done` | Set when the run has finished, whatever happened |

**2.2.1** A run MUST be held only in the scaler's memory. Everything the scaler needs to
recover after a restart — VMs that exist and registrations it owns — MUST be discoverable
from the hypervisor and the CI service (§5.7).

---

## 3. The scaler's credential

**3.1** The scaler MUST hold a credential that can list waiting jobs and create, list and
delete registrations in its scope, and MAY read repository names for the egress records
(§9.2). It MUST NOT hold anything else.

**3.2** The credential MUST NOT be written into a job VM, passed to a job, or logged. A
job VM receives only its one-job registration credential (§6.3). *The scaler's credential
can register runners, so whoever holds it can take any job in its scope — with that job's
secrets.*

**3.3** Every host that serves a label MUST use the same scope. *Hosts coordinate by
reading registrations (§4). A host registering at repository scope and another at user
scope cannot see each other's registrations, and both will start a VM for the same job.*

**3.4** The scope SHOULD be the widest one the labels are served for: the labels real
workflows use come from many repositories.

**3.5** The credential MUST be stored on the host in a file only the scaler's user can read.
Where the CI service grants registration rights at the wider scope only through a broader
permission, the credential carries that permission too, and it MUST be treated as that
broad. *Forgejo's user-level runner registration needs user write access, which also covers
the account's settings and keys.*

---

## 4. Several hosts, one job

Any number of hosts may serve the same label. Each sees the same waiting job.

**4.1** A registration MUST carry the job's handle in its description.

**4.2** Before registering, a scaler MUST list the registrations in its scope. If any
carries the job's handle, it MUST leave the job and not register — unless that
registration has carried the handle for longer than the claim timeout without becoming
active, in which case the scaler MUST delete it and go on. *Otherwise a host that died
holding a job blocks it for every other host, for ever.*

**4.3** After registering, a scaler MUST list the registrations again. If the lowest
registration identifier carrying the handle is not its own, it MUST delete its
registration and leave the job. *Both hosts may pass §4.2 at once. Exactly one of them
holds the lowest identifier, and the CI service, not a clock, decided which.*

**4.4** A job left because another host holds it MUST NOT be taken again by this host
until the claim timeout (§5.5) has passed. *If the other host dies holding it, this host
picks it up after the timeout; until then it does not race.*

**4.5** Two hosts MUST NOT have the same name. *Registration names start with the host's
name, and a scaler cleans up its own stale registrations by that prefix (§5.7). Two hosts
with one name delete each other's.*

**4.6** A registration carrying this run's own name (§5.7.0) is never another host's.
Before §4.2, the scaler MUST delete every registration that carries the job's handle and
this run's name. One that will not delete MUST be left out of §4.2 and §4.3. *An earlier
attempt on this host leaves its registration behind when the CI service fails the delete
(§5.6.2): a transient error at the wrong moment is enough. Counted as another host's claim,
it holds the job against every retry, and the job waits for ever.*

---

## 5. The job VM

### 5.1 Taking a job

**5.1.1** The scaler MUST take a job only if its status is waiting, it has a handle, and
**every** one of its required labels is a label this host serves (§2.1).

**5.1.2** Jobs MUST be taken oldest first, up to `max_vms` runs at once.

**5.1.3** A handle the scaler is already running, or is cooling (§5.5, §4.4), MUST be
skipped.

### 5.2 Starting the VM

**5.2.1** The job VM MUST be a clone of the golden VM, with the configured size.

**5.2.2** The VM's name MUST identify the job and the attempt (for example
`ci-job-<job>-<attempt>`), and MUST carry a prefix that only job VMs use.

**5.2.3** On one host, creating, starting and deleting VMs, and listing them, MUST be
serialised: one such operation at a time, across every run, the sweep (§5.7) and the
golden build (§7). The lock MUST work across processes: the golden build is a separate
program.
*A hypervisor manager may read every instance while starting one. Two starts a few
milliseconds apart — two jobs arriving together — then fail on each other's half-created
instance.*

### 5.3 Running the job

**5.3.1** The scaler MUST register the one-job registration (§4), start the VM, deliver
the registration's credential into the VM (§6.3), and start the runner in the VM, asking
for the job by its handle (§1.4).

**5.3.2** The scaler MUST pass the runner the host's full label set, including the
host's own label.

**5.3.3** The runner's output SHOULD be captured into the scaler's log, prefixed by the
VM's name.

**5.3.4** The scaler MUST reach into the VM only through the VM manager's own execute and
copy channel, which carries no host credentials and mounts nothing (§6.6). It delivers the
credential, starts the runner and reads the resolver log that way.

### 5.4 Claim detection

**5.4.1** A run is claimed when its registration shows the runner working on a job, or
when the registration has gone after the runner started: the CI service deletes a one-job
registration when its job ends, which can be before a poll sees it working. The scaler
MUST poll until the run is claimed or finished.

**5.4.2** A run not claimed within `claim timeout` MUST be abandoned: its runner stopped,
its VM deleted, its registration deleted.

**5.4.3** A run whose runner has registered and asked for its job, and been given nothing,
for `not handed out grace` (about a minute) MUST be abandoned as in §5.4.2. Its handle MUST
then cool for a short time that doubles on each such attempt, up to `retry after`; a claim
MUST reset it. A runner that never registers is still governed by §5.4.2. *A job queued
behind its own previous run looks runnable. Without this, each one holds a VM slot for the
whole claim timeout and then cools for `retry after`, so it starts long after the queue
ahead of it has cleared.*

### 5.5 Cooling

**5.5.1** A run that ends without its job being claimed MUST put the job's handle in a
cooling set for `retry after`. *A job whose runner cannot start — a broken image, a
full disk — would otherwise be retried every poll, starting a VM each time.*

**5.5.2** A handle MUST leave the cooling set when its time has passed.

### 5.6 Finishing

**5.6.1** When the runner exits, or a run is abandoned or overruns, the scaler MUST, in
this order: read the VM's resolver log (§9.1) if the VM was started, delete the VM, and
delete the registration if the CI service has not already.

**5.6.2** A run MUST be marked done whatever happens in §5.6.1. A failure to read the log,
delete the VM or delete the registration MUST be logged and MUST NOT prevent the run from finishing. *If
cleanup can raise past the point where the run is marked done, the run holds a slot for
ever, its job is never retried, and the service cannot drain. Nothing reports it.*

**5.6.3** A job that runs past `job timeout` MUST have its runner stopped, and then
finish as in §5.6.1. `job timeout` SHOULD be configured a little longer than the runner's
own job limit, so the runner's limit fires first and the job's log says why it ended. The
limits MUST nest: a workflow's own job limit below the runner's, the runner's below
`job timeout`, and that below any limit the CI service applies to a running task. *A job
that saves its work before its own deadline loses it to an outer limit that fires first.*

**5.6.4** When a command the scaler runs fails, its error output MUST be logged with the
failure. *"exit status 1" with the message thrown away cost an afternoon.*

**5.6.5** A run whose job the CI service has cancelled MUST have its runner stopped, and
then finish as in §5.6.1, within a grace period of seconds. Where the CI service cancels by
deleting the registration without telling the runner, the scaler MUST detect it from the
runner's side — for example, the runner reporting that its registration is unknown — and
MUST NOT act on a burst of that report shorter than the grace period. *Otherwise a
cancelled job keeps its VM until `job timeout`: hours of a slot running work nobody wants,
while newer jobs queue behind it.*

### 5.7 Orphans

**5.7.0** Names: a job VM is `<vm prefix>-<job>-<attempt>`; its registration is
`<host>-<job VM name>`; the visibility registration (§10) is exactly `<host>`. The
job-runner prefix, `<host>-<vm prefix>-`, never matches the visibility registration.

**5.7.1** At start-up and on every poll, the scaler MUST delete any VM with the job-VM
prefix that no current run owns.

**5.7.2** At start-up, the scaler MUST delete any registration whose name starts with
this host's job-runner prefix and that is not active.

**5.7.3** The sweep MUST NOT touch VMs without the job-VM prefix: the golden VM, its
previous version, and anything else on the host.

**5.7.4** A scaler that stops without draining (§11.2) — killed, or the host lost power —
loses the jobs it was running. At its next start no run owns their VMs, so §5.7.1 deletes
them, and the CI service fails those jobs when their runners stop reporting. Operators
MUST be told this. A scaler MAY instead rebuild runs from surviving VMs and active
registrations.

---

## 6. Inside the job VM

**6.1** The runner MUST run the job's steps directly on the VM (host mode), not in a
container. *Container mode cannot bind the job's working directory to the host, so a step
that starts a container with the workspace bind-mounted gets an empty directory.*

**6.2** The job's user MUST be able to run the container engine without elevation.

**6.3** The one-job credential MUST be delivered into the VM as a file readable only by
the job's user, not as a command-line argument or an environment variable visible to the
job's steps.

**6.4** Certificate authorities the jobs need MUST be in the VM's system trust store, and
every runtime the runner uses to run actions MUST trust that store. *A runtime with its own
bundle — and a step that blanks the variable meant to extend it — fails where other TLS
clients succeed.* A browser in the image MUST trust them too, in whatever store it reads,
and the tool checks MUST prove it by loading a page from a service signed by that authority.
*Chromium on Linux reads the user's NSS database, not the system store.*

**6.5** The VM MUST log every name it resolves, from the VM and from containers on it, to
a file the scaler can read before deleting the VM (§9).

**6.6** The VM MUST NOT mount anything from the host. No shared folders, no forwarded
agent sockets, no host credentials.

**6.7** If a job runs part of its work as a separate, less privileged user, and kills that
user's processes when that part ends, the VM MUST give that user no way to start a process
afterwards: no job scheduler, periodic or one-shot, and no per-user service manager that
outlives its session. *Killing every process the user has is not enough when one of them
left an entry for a scheduler, which starts a fresh one later with the workspace still in
reach.*

**6.8** Such a user MUST NOT be able to elevate, use the container engine (on most engines
that is root), or read the one-job credential (§6.3).

---

## 7. The golden VM

**7.1** Each host MUST build its own golden VM, for its own architecture, from the same
sources in version control.

**7.2** The build MUST install a pinned base operating system image, then the toolchain
(§8), then the runner and its configuration.

**7.3** Before the new golden VM replaces the current one, the build MUST run every tool
check (§8.4) inside it and stop on any failure. *A bad line in a list never reaches a job.*

**7.4** The build MUST keep the current golden VM as the previous version before replacing
it, and there MUST be a command that restores the previous version.

**7.5** The golden VM MUST carry a stamp — readable in every job VM and on the host — with
the source commit (marked if the source had uncommitted changes), the build time, the host
and the installed tool versions. The scaler SHOULD log the stamp at start-up, and the smoke
test MUST print it.

**7.6** Clones MUST NOT depend on a file outside their own directory. *Where the
hypervisor's copy-on-write disk refers to its base by absolute path, a clone of a clone
breaks when the first is deleted. Flatten the golden disk.*

**7.7** The build MUST NOT require the base image's download site to be reachable if a
verified copy is cached on the host.

**7.8** The build MUST be safe to run while the scaler is running: jobs already running
are unaffected, and new jobs start from whichever golden VM exists when they start.

**7.9** The golden VM MUST NOT run background services the jobs do not need — package
update timers, package store refreshes, entropy seeding, firmware and subscription checks —
and the job environment MUST opt every tool that has telemetry out of it. The smoke test
MUST check both. *Otherwise every job's egress record carries their lookups, and they end
up on the allowlist (§9.4) as if a job needed them.*

---

## 8. The toolchain

**8.1** Every tool in the golden VM MUST come from one of three plain lists in version
control: operating system packages, pinned tools (name, version, download URL, and a
command that prints the version), and container images to pre-pull.

**8.2** A tool's download URL MUST be written once, with placeholders for the version and
the architecture names release assets use, so one line covers every architecture.

**8.3** The runner MUST carry the latest supported version of each tool: the newest release
in the major version the workloads use, or, where the workloads pin an exact version (an SDK
pinned by a project file), that version. A major version change MUST be a deliberate
decision, not a routine update.

**8.4** A check MUST exist that runs every tool's version command and confirms the version,
and the architecture of every downloaded binary, plus the container engine and its plugins,
and trust of the services the jobs reach.

**8.5** A check MUST exist that confirms every download in the list exists for every
architecture, runnable before any build.

**8.6** A document listing the toolchain — each tool, its version and what it is for, the
packages, the pre-pulled images and the job environment — MUST be generated from the lists,
and the generator MUST fail on a tool it has no description for.

**8.7** That document MUST be installed in the golden VM at a fixed path, and the build MUST
refuse a stale one (§11.7). Where jobs run a coding agent, the agent's own instruction file
in the image SHOULD point at it, and say what the agent's user cannot do. *That is how an
agent learns what it can use without probing for it.*

---

## 9. Egress records

**9.1** Before deleting a job VM, the scaler MUST read the names it resolved (§6.5), keep
those that resolved successfully, normalised to lower case without a trailing dot, and drop
the VM's own hostname.

**9.2** The scaler MUST append one record per job to a file per calendar month, in a
directory set by configuration: the job, its repository, its labels, the host, the finish
time, the commit of the golden VM the job VM was cloned from (§7.5), and the names. The
scaler MUST read the commit from the job VM's stamp, not the host's. *A job that finished
just after the golden VM was replaced ran on the old one; without its own commit, its record
cannot be told apart from the new VM's.*

**9.3** A report MUST exist that summarises the records per name: how many jobs, which
repositories, last seen. It SHOULD be able to limit the records to one golden commit, to
show what a golden VM change did to egress.

**9.4** Egress SHOULD NOT be blocked until the records cover enough real jobs to write an
allowlist from them.

---

## 10. Visibility

**10.1** Each scaler MUST register one persistent registration named after its host, that
declares the host's served labels and never asks for a job. *The CI service then shows each
host, its labels, and whether its scaler is running, in its own runner list, with no extra
service.*

**10.2** It MUST declare its labels at least as often as the CI service marks a runner
offline, while the scaler runs, and stop when the scaler stops.

**10.3** Its version field SHOULD carry the golden VM's commit.

**10.4** A failed declaration MUST NOT stop the scaler; the next poll MUST register
afresh.

**10.5** A scaler start MUST replace the host's previous visibility registration, matched by
name and a description only the scaler uses, and MUST NOT touch any other.

---

## 11. Running the scaler

**11.1** At most one scaler MUST run per host. A second MUST exit at start-up. The lock
MUST be one the operating system releases when its holder dies, so a crash never leaves a
stale lock.

**11.2** On stop, the scaler MUST stop taking jobs, let running jobs finish for up to
`drain` time, then abandon what remains.

**11.3** The scaler MUST run as a service that starts at boot (or at login, where the
hypervisor needs a user session) and restarts on failure.

**11.4** A script MUST install a host's configuration from version control and install or
restart the service.

**11.5** A dry-run mode MUST list the waiting jobs and whether this host would take each,
changing nothing.

**11.6** A smoke test MUST exist as workflows in the CI service: one job per production
label, and one that targets a named host by its own label (§2.1.2). Each job MUST fail
unless it runs in a job VM of the expected architecture, and MUST print the golden stamp,
run the tool checks (§8.4), bind-mount the workspace into a container, and build and run
an image. It MUST NOT use job-level conditions, which some CI services evaluate on a
runner.

**11.7** The toolchain document generator (§8.6) MUST have a mode that fails when the
document is out of date.

---

## 12. Sizing

**12.1** `max_vms` × VM memory, plus everything else the host runs, MUST fit in the host's
memory.

**12.2** `max_vms` × VM CPUs SHOULD NOT exceed the host's hardware threads.

**12.3** Each concurrent VM SHOULD have 30–50 GB of free disk for jobs that build images,
on a data disk rather than the root filesystem.

**12.4** A pre-booted spare VM MAY be kept per host to cut start time. If it is, it MUST be
replaced after each use and whenever the golden VM changes, and MUST count towards
`max_vms`.
