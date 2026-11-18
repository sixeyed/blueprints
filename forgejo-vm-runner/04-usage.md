# 04 — Usage

## What changes for the people writing workflows

Almost nothing, which is the point. A workflow asks for a label, as before. Its job runs
on a machine that has the toolchain, docker, and the CA certificates, as before. Three
things are different, and each surprises someone once:

- **Nothing survives the job.** No docker layer cache, no package cache, no images pulled
  by the last job. The first step of every build pulls its base images; a dependency
  restore downloads everything. Pre-pull the images most builds start from into the
  golden VM, and accept the rest — or add a cache service, which is its own project.
- **Every job starts with a boot.** Tens of seconds before the first step. For a 40-minute
  build it is noise. For a 10-second job it is the job.
- **The runner's tools are the latest supported versions.** A project that needs something
  else installs its own in the job. One that needs something *newer* asks for the toolchain
  to be updated, which is a line in a list.

## A typical day

A push starts a build that fans out across two architectures. On the amd64 host the scaler
sees eight waiting jobs and starts two VMs; the arm64 host does the same with its eight.
Each VM runs its job and is gone; the next two start. The build's matrix runs two at a
time per architecture, finishes, and the hosts go quiet.

An agent job is labelled on an issue. One host takes it, boots a VM, pulls the agent's
image, and runs for forty minutes. The other host is untouched. When the job ends, the VM
and everything the agent did on it are deleted; the branch it pushed is all that is left.

At the end of the month, the egress report says what every job looked up.

## Smoke tests

Keep two workflows in the scaler's own repository, run by hand:

| Workflow | Runs on | Proves |
|---|---|---|
| per label | one job per production label | every label has a host, and each job lands in a job VM of the right architecture |
| per host | the host's own label, given as an input | that host's scaler and golden VM, end to end |

Both run the same script: fail unless the hostname is a job VM's, print the golden stamp,
run the tool checks, bind-mount the workspace into a container, build and run an image, and
make an HTTPS call from Node.js with its CA variable blanked.

Run them after adding a host, after rebuilding a golden VM, and after changing the scaler.
Do not give the per-label workflow a job-level condition to pick an architecture: see
*Jobs that only exist to be skipped* below.

## Adding a host

1. Install the hypervisor and its manager. On Linux that is a KVM-backed emulator and the
   manager's release; on a Mac, the manager over the platform's virtualisation framework.
2. A service account whose home is on a data disk, not the root filesystem. On a Mac, where
   the virtualisation framework needs a logged-in session, the logged-in user instead.
3. Clone the scaler's repository to a fixed path, as a user that can pull it.
4. Build the golden VM. Five minutes or so.
5. Put the scaler's credential in place, from wherever you keep secrets, straight into a
   file readable only by the scaler. Never echo it.
6. Write the host's configuration file in the repository — labels, size, scope — commit,
   pull, and run the install script.
7. Check the CI service's runner list shows the host, online, with its labels.
8. Run the per-host smoke test. Only then move production labels to the host.

Start a new host with no production labels at all: it serves only its own label, so the
smoke test can prove it without taking real work.

## Updating the toolchain

1. List what is behind: a script compares each pinned version with the latest release.
2. Take the newest release in each tool's current major. Ask before a major version: a
   helm or Node major breaks things that pinned the old one. An SDK pinned by the projects
   built here moves when they move.
3. Edit the list. Check every download exists for every architecture. Regenerate the
   toolchain document.
4. Commit, then rebuild the golden VM on every host. A host left behind gives jobs different
   tools depending on where they land, and the stamps say so.
5. Run the smoke tests.

Do not edit the build scripts on a host while a build is running from them. The shell reads
a script as it runs it, and the build fails half way through.

## Changing the scaler

Merge the change, pull it on each host, and run the install script there. The install
restarts the service, and a restart drains: the old scaler takes no new jobs and waits for
every running job to finish, up to `drain`, which is six hours if it matches the runner's
job limit. So an install on a busy host can sit for hours before the new scaler starts. Run it
when the host is quiet, or let it wait. Another host serving the same labels keeps taking
jobs meanwhile. Do not kill the service to hurry it along: that loses the running jobs
(§5.7.4 of the spec).

## Reading the runner list

The CI service's runner list is the dashboard:

- **A runner named after a host, idle**: that host's scaler is running and serves the
  labels shown (§10 of the spec). Offline means the scaler has stopped, or is draining for a
  restart and taking no new jobs; it comes back when the running jobs finish.
- **`<host>-<vm prefix>-<job>-<attempt>`, active**: a job VM running its job.
- **The same, idle**: a job VM whose runner has registered and is asking for its job. That
  lasts seconds. Minutes means the CI service is not handing the job out yet, usually
  because its run is queued behind another in a concurrency group; the scaler stops that VM
  after a minute and tries again later (§5.4.3).
- **The same, offline, and staying**: a registration the scaler could not delete. The next
  attempt for that job deletes it (§4.6), and a scaler start deletes the rest.

## A host that is also a workstation

Lower its `max_vms` while you do heavy work on it yourself, a local cluster say, and raise it
afterwards. It is a commit and an install like any other configuration change, so the
history says when the host had less capacity. Jobs on labels another host serves go there
meanwhile.

## Timeouts nest

A long job meets four limits: its own workflow `timeout-minutes`, the runner's job limit,
the scaler's `job timeout`, and the CI service's limit on a running task. Keep them in that
order, each a little above the last. A job that saves its work before its own deadline, as
an agent job checkpointing before it stops, only gets the chance if nothing outside it fires
first.

## When a new golden VM breaks jobs

Restore the previous one on that host with the rollback command. Jobs that start after it
use the previous golden VM; jobs already running finish on the one they started on. Then fix
the list and rebuild.

## Reading the egress report

```
$ egress-report --since 2026-11-01
214 job(s) from host-a, host-b, 23 domain(s)

domain                                    jobs  last seen             repos
ci.example.com                             214  2026-11-30T22:10:04Z  app, infra, site
registry.example.com                       171  2026-11-30T22:10:04Z  app, infra
api.nuget.org                               96  2026-11-30T21:55:12Z  app
...
```

The names every job looks up — the CI service, the registry — are the allowlist's core.
Names one repository looks up are that repository's dependencies. Names that appear in every
job but that no job asked for are the operating system's background services, or a tool's
telemetry; turn them off in the golden VM rather than allowing them. After changing the
golden VM, limit the report to its commit (`--golden <commit>`) to see what the change did:
jobs that were already running when it was replaced carry the old commit.

## Jobs that only exist to be skipped

Some CI services decide a job's condition (`if:`) on the runner, not on the server. A job
whose condition is false still needs a runner to discover that it is skipped. With a
persistent runner that cost nothing. Here each one is a VM boot.

They come from workflows that trigger on broad events and filter in the job: every issue
comment, every label, every pull request event, each starting a VM to decide not to run.
An agent that comments and labels as it works generates several per run, all on its own
label.

Three things help, in order of payoff:

- **Split by trigger, not by condition.** A job that only runs when a pull request closes
  goes in its own workflow triggered only on close.
- **Make the agent's own comments and labels with the job's own token** where the CI
  service does not trigger workflows from the job token's events. Its activity then stops
  triggering its own dispatcher.
- **Schedule less often.** A dispatcher on a 15-minute timer is a VM every 15 minutes.

A warm spare VM per host would make each skip cost a couple of seconds instead of tens. It
also keeps a VM running whenever there is no work, which is why it is not the default.

## Anti-patterns

- **A persistent runner "just for the quick jobs".** It brings back everything this
  removes, for the jobs most likely to be triggered by anyone who can comment.
- **Hosts on different registration scopes.** They cannot see each other's claims; every
  shared-label job starts two VMs.
- **Two hosts with the same short name.** Each cleans up the other's registrations.
- **Configuring a host by hand on the host.** Six months later nobody knows what it serves.
- **Holding a tool back for one project.** That project installs its own version; everyone
  else gets the current one.
- **Blocking egress before reading the records.** The first build that needs a package
  mirror you did not know about fails at three in the morning.
