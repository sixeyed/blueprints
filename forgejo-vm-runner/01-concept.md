# 01 — Concept

## The problem

A self-hosted CI runner is a long-lived program on a machine you own. It asks the CI
service for work, runs each job's steps, and asks again. Everything a job does happens on
that machine, as that runner's user, and stays there after the job ends: files in its
home, images in its docker, processes it forgot to stop, credentials it wrote to disk.

For a long time that was fine, because the jobs were your own build scripts. Then the
jobs started including AI coding agents — a session told to deliver an issue, run with
its permission prompts off because nobody is there to answer them. Now the job is a
capable, unsupervised program running arbitrary commands, on the machine that also holds
your cluster's deploy credentials, your other repositories' secrets, and maybe the
terminal you were working in this morning.

You can no longer use that machine for anything you trust. That is the whole problem.

| Approach | Why it falls short |
|---|---|
| A persistent runner on the host | The problem as stated. Every job has the host, and every job inherits what the last one left |
| A persistent runner in its own VM | Better: the host is protected. But the VM is shared by every job, for ever. One job's leftovers are the next job's environment, and a secret one job wrote is readable by the next |
| The runner's container mode | Each job runs in a container, which looks like isolation. Then a job runs `docker run -v $PWD:/work`, the path is the container's, not the host's, and the inner container sees an empty directory. Fixing that means mounting the host's docker socket into the job — which hands the job the host |
| A Kubernetes runner controller, such as GitHub's [Actions Runner Controller](https://github.com/actions/actions-runner-controller) | Ephemeral pods, solved properly — if you run a cluster for it, every job fits in a pod, and you have no Mac. Jobs that run `docker` need either a privileged docker-in-docker sidecar, which brings the socket problem back inside the pod, or its Kubernetes mode, which runs each container step as its own pod and so needs workflows written for it |
| Hosted runners | Somebody else's machines. Fine until the job needs your LAN, your registry, your hardware, or an architecture they charge extra for |
| Snapshot and revert one VM between jobs | Close. But one VM is one job at a time, a revert that fails leaves the next job on dirty state, and nothing tells you it failed |

The recurring failure is that each of these either shares something between jobs, or
breaks the jobs that run `docker` the ordinary way.

## The idea

**A VM per job. Cloned when the job is waiting, deleted when it ends.**

A small service on each host — the scaler — watches the CI service's queue. When a job is
waiting for a label the host serves, the scaler registers a runner that may take exactly
one job, clones a VM from a prepared golden image, and starts that runner inside it. The
runner takes the job, runs it, and exits. The scaler deletes the VM and the registration.

Inside the VM the runner works in host mode: steps run directly on the VM, with docker
installed, so `docker run -v $PWD:/work` means exactly what it says. The VM is the
isolation boundary, not a container.

The golden image is built on each host, from plain lists of tools, and stamped with the
commit it was built from. Changing a tool is a line in a list and a rebuild.

## Why this shape

**A clone is cheap; a fresh install is not.** Copy-on-write clones of a stopped image start
in seconds and take only the disk the job writes. Installing the toolchain per job would
take minutes and fetch the world every time. So the toolchain is baked into the image, and
the image is rebuilt deliberately, not per job.

**One scaler per host, not one controller for all.** Each host knows its own capacity,
architecture and hypervisor; a central controller would need an agent on each host anyway.
A scaler that only manages its own VMs is a single small program with no network surface
of its own: it makes outbound calls to the CI service and local calls to its hypervisor.

**The CI service is the coordinator.** Two hosts can serve the same label, and both will
see the same waiting job. They need to agree which one takes it. The CI service already
keeps a list of registered runners that every host in the same scope can read. A host
writes the job's identifier into its registration, re-reads the list, and keeps the job
only if its registration is the lowest. No lock service, no shared disk, no clock.

**Host mode inside a VM, not containers inside a host.** The runner's container mode
cannot bind a job's working directory to the host, so jobs that start containers of their
own break in confusing ways. In a VM that is thrown away, host mode is both correct and
safe: the paths line up, and there is nothing outside the job to protect inside the VM.

**The runner carries the latest supported version of each tool.** A project that needs a
different version installs its own in the job, and should use the runner's when it is new
enough. So the image is never held back for one project, and "what is on the runner" has
one answer, in one generated document.

**Watch the network before fencing it.** A job VM can reach whatever its host can. The
fence that should exist — the CI service, the registry, a known list of package sources —
cannot be written well until you know what real jobs reach. So each VM logs the names it
resolves, the scaler keeps them per job, and the allowlist is read off weeks of real
traffic rather than guessed.

**No warm pool, by default.** A pre-booted spare VM per host would cut the start from
tens of seconds to a couple. It would also keep a VM running on every host whenever there
is no work. On hardware you also use for other things, an idle host that is actually idle
is worth more than the seconds. The spec allows a pool; it does not require one.

## Threat model

Start here, because it changes what "done" means.

**Defends against:**

- one job reading or tampering with another job's files, processes, images or secrets;
- a job leaving anything behind for the next one — the VM is deleted, not cleaned;
- a job reaching the host's own files, credentials, sessions and other workloads, except
  through the network;
- a broken image reaching jobs: the image is checked before it replaces the last one, and
  the last one is kept.

**Requires:**

- the scaler's credential to stay on the host. It can register runners, so it can take
  any job in its scope. It MUST NOT be written into a VM.
- every host serving a label to use the same registration scope, or they cannot see each
  other's claims and two VMs start for one job;
- a hypervisor boundary you trust. This is VM isolation, with that hypervisor's escape
  surface.

**Does not defend against:**

- **a job's own secrets.** The CI service sends a job its secrets so the job can use them. Anything
  that job runs — including scripts from the branch it is building — can read them. A VM
  per job limits the blast to that job; it does not make an untrusted branch safe to give
  a deploy key.
- **the network, until you fence it.** A job VM can reach your LAN. The egress records
  are there to make the fence possible; they are not the fence.
- **a job that is the attacker.** It gets a clean VM, its secrets, and network access for
  as long as the job's timeout. That is the point of a CI runner.

*If the jobs you are protecting against need secrets they should not have, fix the
secrets.* This design stops jobs harming each other and the host. It cannot stop a job
misusing what it was given.

## Which CI services this fits

The design leans on one capability more than any other: **naming a job.** The scaler
sees a waiting job, starts a VM for it, and the runner in that VM takes that job and no
other. The per-job record, the egress record, cooling a job that failed to start, and
the way hosts decide who takes a job all depend on it.

| Needed ([02 §1](02-specification.md#1-what-the-ci-service-must-provide)) | Forgejo | GitHub Actions |
|---|---|---|
| A list of waiting jobs, with labels and a handle | yes | per repository, or through job-queued webhooks; no handle |
| One-job runner registration from an API | yes | yes: just-in-time runners |
| A runner that takes exactly the named job | yes | no: a just-in-time runner takes any queued job its labels match |
| Registrations listed with a free-form description | yes | names and labels only |
| A registration that declares labels without taking work | yes | not known: an online runner is one that is asking for work |

**Forgejo** fits as written; it is what the reference implementation runs against.

**GitHub Actions** fits the idea, not the letter. Without a handle the scaler works in
counts, as GitHub's own [Actions Runner Controller](https://github.com/actions/actions-runner-controller)
does for pods:

- Count the queued jobs per label set, and start that many VMs, each with a just-in-time
  runner for those labels. A VM is for *a* job on those labels, not *the* job.
- Coordinating hosts on a shared label becomes sharing a count: hosts must agree how many
  VMs each starts for the same queue. A shared counter, or one host per label set, replaces
  the lowest-registration rule.
- Cooling applies to a label set that keeps failing to start, not to a job.
- Egress records are per VM, and the job is whatever the runner reports it ran.
- Host visibility (§10) needs another surface; the runner list cannot show a host that
  takes no work.

Everything about the VM itself — the golden image, the lifecycle, the lock, cleanup that
cannot fail, the toolchain, the smoke tests — carries over unchanged.

**Others** are unverified. A CI service where the server hands work to runners, rather than
listing it for a scaler to read, does not fit this shape at all.

## When not to use this

- **Hosted runners serve you.** If your jobs need nothing on your network and nothing
  unusual, someone else's ephemeral VMs are already this, maintained by someone else.
- **You already run a cluster for CI.** A pod per job, with a controller the CI vendor
  maintains (Actions Runner Controller, for GitHub), is the same idea with less of your
  own code. This blueprint is for the
  machines that are not in a cluster — including Macs.
- **More than a handful of hosts.** Each scaler polls the CI service. At dozens of hosts
  you want a scheduler that places jobs, not hosts that race for them.
- **Very short, very frequent jobs.** A VM boot is tens of seconds. If most of your jobs
  run for five, the boot is most of your CI time. This includes jobs that only exist to
  be skipped: CI services that evaluate a job's condition on a runner give each one a VM
  just to decide not to run (see [04](04-usage.md#jobs-that-only-exist-to-be-skipped)).
- **Jobs that need a warm cache to be tolerable.** Nothing survives between jobs, so
  every build starts with cold package and image caches unless you add a cache service.
  That is a separate piece of work, and the reason most persistent runners exist.
- **Windows or macOS guests, or GPUs.** Possible in principle, and outside this
  blueprint.

The sweet spot is two to five machines you own, of mixed architecture, running a CI
service you host, with jobs you would rather not trust with the whole machine.
