# 03 — Acceptance criteria

Build against these. Each is a behaviour you can assert on; each should map to at least
one automated test that fails before the code satisfying it exists.

The failure mode of this system is **a scaler that looks healthy and does less.** A run
whose cleanup raised is never marked done: it holds a slot, its job is never retried, and
the service cannot drain on restart — and the log shows nothing after the one error. Two
VM starts that collide fail one job, which then sits in its cooling period, waiting, while
every dashboard says the runners are up. A registration of its own that a failed delete left
behind looks like another host's claim, and its job waits for ever while the log says, once
every claim timeout, that someone else has it. None of these shows in the happy path, where
jobs arrive one at a time and every command succeeds.

[§3](#3-every-run-finishes), [§4](#4-vm-operations-never-overlap) and 5.6–5.7 are the
criteria that catch it. Test them with a fake hypervisor and a fake CI service whose
commands fail and whose calls you can watch for overlap, and then once against the real
ones with two jobs dispatched together.

[§1](#1-one-job-one-vm) is the reason the system exists. Assert on the VM, from inside the
job, not on the scaler's idea of where the job ran: a job that a stray persistent runner
took passes every other test.

---

## 1. One job, one VM

- [ ] **1.1** A job's steps run on a VM whose name has the job-VM prefix. *The smoke test
      checks the hostname and fails otherwise; a persistent runner that takes the job fails
      this, and nothing else.*
- [ ] **1.2** The VM's architecture is the one its labels ask for.
- [ ] **1.3** After the job ends, the VM no longer exists on the host, and the one-job
      registration no longer exists in the CI service.
- [ ] **1.4** Two jobs run at the same time on one host run in two different VMs.
- [ ] **1.5** A file a job writes outside its workspace is absent in the next job on the
      same host.
- [ ] **1.6** A step that runs `docker run -v "$PWD:/work"` sees the workspace's files
      inside the container.
- [ ] **1.7** A step can build an image with the container engine's build plugin and run it.
- [ ] **1.8** The scaler's credential is not readable anywhere in a job VM: search the
      VM's filesystem and the job's environment for its value.
- [ ] **1.9** The one-job credential is not in any process's arguments on the host or in
      the VM, nor in a step's environment.
- [ ] **1.10** Nothing from the host is mounted in a job VM: list the VM's mounts.
- [ ] **1.11** Where jobs run part of their work as a separate user (§6.7 of the spec),
      that user cannot add a scheduled job or keep a per-user service manager running:
      the smoke test tries each and expects a refusal.
- [ ] **1.12** That user cannot use `sudo`, cannot reach the container engine, and cannot
      read the one-job credential file.

## 2. Taking jobs

- [ ] **2.1** A waiting job whose labels are all served is taken.
- [ ] **2.2** A job needing one label the host does not serve is never taken.
- [ ] **2.3** A job that is not waiting, or has no handle, is never taken.
- [ ] **2.4** With more waiting jobs than free slots, the oldest are taken, and never more
      than `max_vms` at once.
- [ ] **2.5** A finished run frees its slot for the next poll.
- [ ] **2.6** Every host serves its own host label; a job on that label runs on that host,
      from that host's golden VM.
- [ ] **2.7** With two jobs waiting on the same labels, each runner takes the job its
      handle names, never the other.

## 3. Every run finishes

- [ ] **3.1** When deleting the VM raises, the run is still marked done, the registration
      is still deleted, and the error is logged.
- [ ] **3.2** When deleting the registration raises, the run is still marked done.
- [ ] **3.3** When starting the VM fails, the run is marked done, its job's handle cools for
      `retry after`, and the failing command's error output is in the log.
- [ ] **3.4** A run not claimed within the claim timeout is abandoned: runner stopped, VM
      and registration deleted, handle cooled.
- [ ] **3.5** A claimed run is never abandoned by the claim timeout.
- [ ] **3.6** A job that overruns `job timeout` has its runner stopped and its VM deleted.
- [ ] **3.7** Stopping the scaler with no runs in progress returns promptly. With a run in
      progress, it waits for that run, up to `drain`.
- [ ] **3.8** A run whose registration disappears after its runner started counts as
      claimed: its handle is not cooled.
- [ ] **3.9** When reading the VM's resolver log fails, the VM and the registration are
      still deleted and the run still finishes.
- [ ] **3.10** A run whose runner reports its registration as unknown for longer than the
      grace period has its runner stopped and its VM deleted, and its slot takes the next
      waiting job. *Cancel a running job in the CI service to see it live.*
- [ ] **3.11** A run that reports its registration as unknown for less than the grace
      period, and then finishes, is left alone.
- [ ] **3.12** A run whose runner has registered but been given no job for longer than
      `not handed out grace` has its VM stopped, and its job is tried again after a short
      wait that doubles on each such attempt up to `retry after`. *Queue a second run of a
      workflow with a concurrency group that does not cancel in progress.*
- [ ] **3.13** A runner given its job after a few empty polls is left alone.
- [ ] **3.14** After a job is claimed, the next time it is not handed out starts the wait
      from the short end again.

## 4. VM operations never overlap

- [ ] **4.1** With three runs starting at once against a fake hypervisor that records
      concurrent calls, no two create/start calls overlap.
- [ ] **4.2** A VM listing never overlaps a create, start or delete.
- [ ] **4.3** On the real hypervisor, two jobs dispatched together on one host both run.
      *This is the test that fails without §4.1: run it.*

## 5. Several hosts, one job

- [ ] **5.1** A job that already has a registration carrying its handle, from another host,
      is left alone.
- [ ] **5.2** When two hosts register for one job at once, the lower registration identifier
      runs it and the other deletes its registration and starts no VM.
- [ ] **5.3** A job left to another host is not taken again by this host until the claim
      timeout has passed.
- [ ] **5.4** With two hosts serving one label and several jobs waiting, every job runs
      exactly once.
- [ ] **5.5** A registration from another host that has carried a handle for longer than
      the claim timeout without becoming active is deleted, and this host takes the job.
      *Kill a host mid-registration to set this up.*
- [ ] **5.6** A registration with this run's own name and the job's handle, left by an
      earlier attempt on this host, is deleted, and the job runs.
- [ ] **5.7** When deleting that leftover registration fails, the job still runs: the
      leftover is not taken for another host's claim. *Make the fake CI service's delete
      raise. Both tests fail on a scaler that only checks for any registration carrying
      the handle.*

## 6. Recovery

- [ ] **6.1** At start-up, a VM with the job-VM prefix that no run owns is deleted; the
      golden VM and its previous version are not.
- [ ] **6.2** At start-up, an inactive registration with this host's job-runner prefix is
      deleted; an active one, and another host's, are not.
- [ ] **6.3** A second scaler started on the same host exits without taking a job, while
      the first is running.
- [ ] **6.4** After the scaler is killed (not stopped) with a job running, the next start
      deletes that job's VM, the CI service reports the job failed, and nothing else on
      the host is touched.
- [ ] **6.5** After the scaler is killed, the next one starts: the single-instance lock is
      not left behind.

## 7. The golden VM

- [ ] **7.1** A toolchain list line with a wrong version stops the build before the golden
      VM is replaced: the previous golden VM is still the one jobs start from.
- [ ] **7.2** After a successful build, the previous golden VM exists, and the rollback
      command makes it the golden VM again.
- [ ] **7.3** Every job VM shows the golden stamp: commit, build time, host, tool versions.
      A build from a source with uncommitted changes is marked as such.
- [ ] **7.4** Deleting a job VM does not break the golden VM or any other clone.
- [ ] **7.5** A build with the base image's download site unreachable succeeds from a
      verified cached copy.
- [ ] **7.6** Rebuilding the golden VM while the scaler runs leaves a running job
      unaffected, and jobs that arrive during the build still start.
- [ ] **7.7** A job VM runs none of the background services §7.9 of the spec lists, and the
      job environment carries the telemetry opt-out for every tool that has one. The smoke
      test checks both, and no egress record holds a name those services look up.

## 8. The toolchain

- [ ] **8.1** Every tool in the list reports its pinned version in a job VM, on every
      architecture, and every downloaded binary is that architecture's.
- [ ] **8.2** The URL check reports every download in the list for every architecture, and
      fails on one that is missing.
- [ ] **8.3** The toolchain document matches the lists; the generator fails on a tool with
      no description, and its check mode fails on a stale document.
- [ ] **8.4** A Node.js-based step reaches a service signed by a private CA with the
      variable that extends Node's CA bundle set to empty. *This is how an action that
      worked on a persistent runner fails in a job VM.*
- [ ] **8.5** A browser in the job VM loads a page from a service signed by the private CA
      without a certificate error, as the job's user.
- [ ] **8.6** The toolchain document is in every job VM at its fixed path and matches the
      lists the golden VM was built from; a build from lists newer than the document stops.

## 9. Egress records

- [ ] **9.1** After a job that fetches from a known domain, the month's record file has a
      line for that job containing the domain.
- [ ] **9.2** A name that failed to resolve is not recorded; the VM's own hostname is not
      recorded; names are lower case with no trailing dot.
- [ ] **9.3** A run that never started a VM writes no record.
- [ ] **9.4** A lookup made from a container in the VM is recorded, not only lookups made
      by the VM itself.
- [ ] **9.5** Tests write records only to a temporary directory, never the host's real one.
- [ ] **9.6** The report lists each name with its job count, its repositories and when it
      was last seen, and can be limited to records since a date.
- [ ] **9.7** Each record carries the commit of the golden VM its job VM was cloned from. A
      job still running when the golden VM is replaced records the old commit, and the
      report can be limited to one commit.

## 10. Visibility

- [ ] **10.1** While the scaler runs, the CI service lists a runner named after the host,
      online, with the host's served labels and the golden VM's commit as its version.
- [ ] **10.2** That runner never takes a job: with a job waiting on its labels and no job
      VM able to start, the job stays waiting.
- [ ] **10.3** Within twice the CI service's offline interval after the scaler stops, the
      host's runner shows offline.
- [ ] **10.4** Restarting the scaler leaves exactly one visibility runner for the host.
- [ ] **10.5** Restarting one host's scaler does not touch another host's visibility runner.
- [ ] **10.6** A failed label declaration registers afresh on the next poll.

## 11. Operations

- [ ] **11.1** The install script puts the host's configuration from version control in
      place, dry-runs it, and restarts the service; the live configuration then equals the
      file in version control.
- [ ] **11.2** The dry run lists each waiting job and whether this host would take it, and
      changes nothing in the CI service or on the host.
- [ ] **11.3** A smoke workflow with one job per production label passes, and every job
      lands in a VM of an expected architecture.
- [ ] **11.4** The service restarts the scaler after its process is killed, and starts it
      after the host (or, on a host that needs a session, the user session) restarts.
- [ ] **11.5** The scaler's credential file on the host is readable only by the scaler's
      user.
- [ ] **11.6** Running the install script while a job runs lets that job finish before the
      new scaler starts, on every platform's service manager, and the job VM is not deleted
      as an orphan. *A service manager's restart may kill long before the drain time.*
