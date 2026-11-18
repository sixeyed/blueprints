# 05 — Reference implementation

Non-normative. One build of this blueprint, its choices, and what went wrong on the way.
Treat the **Traps** as hard-won and the stack as one option among many.

## Shape

| | |
|---|---|
| CI service | Forgejo 15, self-hosted |
| Runner | `forgejo-runner` 13, in its `one-job` mode |
| Hosts | an 8-thread amd64 mini PC with 64 GB (Linux, QEMU/KVM) and a 10-core arm64 Mac Studio with 64 GB (macOS, the Virtualization framework) |
| VM manager | Lima 2.0, in `plain` mode: no shared folders, no agent forwarding |
| Guest | Ubuntu 24.04 cloud image, 4 CPUs / 16 GiB / 100 GiB sparse |
| Scaler | one Python file, standard library only, about 500 lines, with unit tests against fakes |
| Service | systemd on Linux, a launchd agent on the Mac |
| Golden build | a bash script driving the VM manager; the toolchain is three text files |

It replaced three persistent runners — a runner on one host, a runner in a long-lived VM on
the same host, and a runner container on the Mac — and is the only thing serving those
labels.

## Decisions worth repeating

**Ask the runner for the job by its handle.** Forgejo's waiting-jobs API gives each job a
handle, and `forgejo-runner one-job --handle <h> --wait` takes exactly that job or nothing.
Without it, a one-job runner takes whichever job matches its labels first, and the scaler's
bookkeeping — this VM is for that job — is fiction.

**Write the registration credential to a file in the VM, and point the runner at it.** The
runner reads `--token-url file:///var/lib/ci-runner/token`. The credential never appears in
a process list on the host or in the VM.

**Point the container engine's DNS at the VM's resolver.** The VM logs lookups with the
system resolver's monitor mode, written to a file. Containers use their own DNS by default
and bypass it, so the resolver also listens on the container bridge address and the engine's
`daemon.json` points there. Without that, the egress records only show what the VM itself
looked up — not the builds.

**Flatten the golden disk.** On QEMU, Lima's copy-on-write disk names its base by absolute
path. The build converts the golden disk to a standalone image (`qemu-img convert`) after
cloning it into place.

**Manage the base image outside the VM manager.** The build downloads the cloud image into
its own cache with the published checksums, and points Lima at the local file.

**The visibility runner speaks the runner protocol, but only `Declare`.** The scaler
registers a persistent runner named after the host and calls the runner service's `Declare`
on every poll with the served labels. Any authenticated runner call marks a runner online;
only `FetchTask` hands out work, and the scaler never calls it. The runners page then shows
`host-a idle amd64,linux,agent` with `ci-scaler, golden 8be754d` as its version.

**Host configurations in the repository, installed by a script.** `hosts/<hostname>.toml`
is the configuration; `install-service.sh` copies it into place, dry-runs it, and installs or
restarts the service for the platform it is on. Changing a host's labels is a commit.

**The toolchain document is generated.** A script reads the three lists and the golden VM's
size and runner environment, and writes `TOOLCHAIN.md`. It holds a description for each tool
and fails on a tool without one; `--check` fails on a stale document.

**The toolchain document is in every job VM.** The golden build refuses a stale
`TOOLCHAIN.md`, then installs it as `/etc/ci-runner/TOOLCHAIN.md`. Claude Code's managed
memory (`/etc/claude-code/CLAUDE.md`, read by every user in the VM) says where it is, that a
project needing another version installs its own, and that the unprivileged user has no
`sudo` and no docker.

## Traps

### Two VM starts at once

Two jobs arrived together. The scaler started two clones 18 milliseconds apart; one failed
with `exit status 1` and nothing else, because the error output was captured and dropped.

Reproduced by hand: two concurrent `limactl clone --start` calls, and one dies with
`open .../<the other instance>/lima.yaml: no such file or directory`. Lima reads every
instance while starting one, and trips over the neighbour that is half-written. Two clones
*without* `--start` do not collide, which is why the first reproduction attempt found
nothing.

A per-host lock now covers clone-and-start, delete and list. Starting two VMs takes twice as
long; a boot is about 17 seconds, so the second job waits that much more.

### …and the run that never finished

The failed job's cleanup deleted its VM and then listed instances to confirm it was gone. The
list ran into the same race, raised inside the `finally`, and the line that marked the run
done never ran.

The run sat in the scaler's table for ever. Its job was never retried, because its handle was
"busy". The service could not restart: it waited to drain a run that would never finish, and
had to be killed. Nothing in the log said any of this — the last line for that job was the
original failure.

Cleanup is now wrapped so each step logs its own failure and the run is always marked done.
The unit test makes the fake hypervisor's delete raise and asserts the run finishes.

### A skipped job still needs a runner

A smoke workflow had one job per architecture, each with `if: inputs.arch == '...'`, so one
dispatch could test either. The job that was skipped stayed queued for ever on a host that
did not exist yet: Forgejo evaluates job-level `if:` on the runner, so a skipped job still
waits for a runner with its labels.

Split into separate workflows (and later, a workflow with one unconditional job per label).
The same behaviour makes every broad-trigger workflow cost a VM per event; see
[04](04-usage.md#jobs-that-only-exist-to-be-skipped).

### Container mode cannot bind the workspace

The first design ran jobs in containers on a shared VM. `forgejo-runner` has no option to
bind a container job's working directory to the host, so a step that ran
`docker run -v $PWD:/work` mounted a path that only existed inside the job container. The
inner container saw an empty directory, and the build failed on files that were plainly
there. Host mode inside a VM that is thrown away is the fix, and the reason for the design.

### The VM manager's cache, when the internet is not there

The Ubuntu image site was unreachable for part of an afternoon. Lima had the image cached, and
still failed to create an instance — it checks the URL before using its cache, and the error
was about a missing base disk. The build now fetches and verifies the image itself and gives
Lima a local path. On the Linux host, the first image was copied from another VM manager's
cache, of the same release.

### A clone of a deleted VM

On QEMU the first job VMs worked until the build replaced the golden VM — then every new clone
failed, pointing at the deleted VM's disk by absolute path. Flattening the golden disk after the
clone into place fixed it. The Mac's framework copies the disk, and never had the problem.

### The scaler that ran twice

During testing a second scaler was started on a host where the first was still running, on a
different registration scope. Both took the same job, both started a VM with the same name, and
one clone failed. A lock file at start-up now makes a second scaler exit. The scope mismatch is
why the spec requires one scope everywhere: the two could not see each other's registrations.

### A credential at the wrong scope

The first scaler credential was scoped to one repository. User-level runner registration
returned 403. It was fine for a repository-scoped smoke test, and useless for the labels real
workflows use from every repository. The scaler's credential now has user-level runner
management and read access to repositories (to name them in egress records), and nothing else.

"User-level runner management" is Forgejo's `write:user`, and that also covers the
account's settings, SSH keys included: far more than the scaler uses. The credential lives in
one file per host, mode 0400 and owned by the scaler's user, is fetched from the secret store
straight into that file, and never enters a job VM.

### Node.js ignored the system trust store

Every TLS check in the image passed: `curl` and `git` trusted the private CA installed into the
system store. Then an upload action failed: `unable to verify the first certificate`. Node.js
uses its own CA bundle. One project's workflow set `NODE_EXTRA_CA_CERTS` to the CA file on that
step; another's set it to empty, because its installer had no CA configured.

`NODE_USE_SYSTEM_CA=1` in the runner's environment makes every job's Node trust the system
store, and a step that blanks the other variable cannot undo it. The smoke test now runs Node
against the CI service with `NODE_EXTRA_CA_CERTS=""`. It failed before the fix, which is the
point of writing it first.

### The smoke check that tested the wrong thing

That Node check first fetched the CI service's version API, and failed with a 403 once TLS
worked: the API wanted a login. Any HTTP answer proves the handshake succeeded; the check now
accepts any status.

### Editing the build script while it runs

A fix to the golden build script was saved while that script was running from the same
checkout. Bash reads a script as it executes it; the build failed half way through on a line
that no longer existed. Never edit the checkout a build is running from.

### The installer that refused to run as root

The Claude Code installer refuses to run under `sudo` unless `CLAUDE_INSTALL_ALLOW_SUDO=1` is
set. Inside an image build, root is the only user there is.

### Release assets name architectures four ways

`amd64`/`arm64`, `x86_64`/`arm64`, `x64`/`arm64` and `x86_64`/`aarch64` all appear among a
dozen tools' release assets. The list has a placeholder for each, so one line covers both
architectures, and a URL check runs for both before any build.

### Tests wrote into the real egress records

The first egress tests used the scaler's default record directory, and their fake jobs turned up
in the real report. The tests now point the scaler at a temporary directory.

### The persistent runner that was still taking jobs

The smoke test's first assertion is that the hostname is a job VM's. Without it, a job taken by
a leftover persistent runner with the same labels passes every other check. It caught exactly
that during the cut-over.

### A cancelled job kept its VM for six hours

A newer push cancelled a preview deploy (`cancel-in-progress`). Forgejo deleted the job's
one-job runner and told it nothing: the runner in the VM carried on deploying, and every log
line and state report it sent came back `unauthenticated: unregistered runner`, twice a
second. The scaler only freed a VM when its runner exited or `job timeout` passed, so the
host ran two working VMs out of three, one of them doing work nobody wanted, until someone
stopped it by hand 17 minutes later. The next queued jobs started at once.

The scaler now watches the runner's output. Once it has said `unregistered runner` for 30
seconds (`unregistered_grace_seconds`), the scaler kills the run the way it kills an
overrun: SIGKILL to the runner's process group, then stop the VM, and the usual cleanup
deletes it. A shorter burst is left alone.

### A 503 that stranded a job for good

The CI service briefly returned 503. A job VM's runner hit it while declaring itself,
and exited before it fetched the job. The scaler's cleanup then tried to delete the runner's
registration and got a 503 too, so the registration stayed, carrying the job's handle.

Fifteen minutes later the retry found that registration, took it for another host's claim,
and left the job, then cooled it and did the same again every ten minutes. The job never ran,
and the only sign was an `info` line saying another host had it. Both hosts serve that label,
so the message even looked plausible.

Each attempt now deletes any registration with its own name and the job's handle before it
looks for other hosts' claims, and one with its own name that will not delete is never
counted as another host's. Two tests cover it, one where the leftover deletes and one where
the fake CI service's delete raises. Both failed on the old code.

### Ubuntu and the tools phoned home from every job

The first egress records showed snapd, pollinate, fwupd, Ubuntu Pro and the apt timers in
nearly every job, and the .NET CLI, PowerShell and Claude Code sending telemetry. None of it
is a job's traffic, and all of it would have ended up on the allowlist. The packages are
purged and the timers masked in the golden VM, the runner sets each tool's opt-out variable,
and `/etc/environment` carries them for shells. The smoke test checks both.

### Which golden VM was that?

A golden rebuild finished while jobs were running. Their egress records looked like the new
VM's, though they ran on the old one. The scaler now reads the job VM's own stamp before
deleting it, puts the commit in the record, and the report filters on it.

### An agent process that outlived the kill

One workflow runs a coding agent as an unprivileged user inside the job VM, kills every
process that user owns after each agent command, and takes the workspace back. A crontab
entry, an `at` job or a lingering user manager would have started a fresh agent process
afterwards, with the workspace still in reach. cron is purged with the other background
packages, `at` is listed for removal in case it ever arrives, and a polkit rule refuses that
user every action, which also stops `loginctl enable-linger`. The smoke test checks for
both.

### The restart that did not drain

Installing a scaler change on the Mac lost a running job. The install script restarted the
LaunchAgent with `launchctl kickstart -k`, which sends SIGTERM and then kills the service 60
seconds later, whatever `ExitTimeOut` says; the plist allowed six hours. The scaler began
draining, was killed a minute in, and the new scaler found the running job's VM with no run
owning it and deleted it as an orphan (§5.7.4). The Linux host, under systemd's
`TimeoutStopSec`, drained the same install properly.

Reproduced with a throwaway agent that drains for 150 seconds: `kickstart -k` killed it at 59.
The script now sends SIGTERM itself (`launchctl kill TERM`), waits for that process to exit,
and then starts the service. A drained scaler exits 0, and `KeepAlive` with
`SuccessfulExit: false` keeps launchd from restarting the old one meanwhile. Test a service
manager's stop with a job running before trusting §11.2 to it.

### Idle runners for a run that could not start yet

The runner list showed three job runners `Idle` for ten minutes. One repository's image
workflow has a concurrency group per branch that does not cancel in progress, so a second
push queued its run behind the first. Forgejo listed the queued run's jobs as waiting, with
no task, exactly like runnable ones, and it has no per-job endpoint that would say otherwise.
Both hosts booted VMs for them. Each runner registered, then asked for its job every few
seconds and got nothing (`received no task`).

The first run finished at 20:52. One job VM, started at 20:44, was still inside its claim
timeout and took its job at once. Two had timed out at 20:51 and cooled for fifteen minutes,
so two builds of a run that was free to start sat queued, behind two VM slots that had
idled for ten.

Now a runner that has been given no task for a minute has its VM stopped, and the job is
tried again after one minute, then two, four, eight, up to `retry_after_seconds`; a claim
resets it.

### Chromium keeps its own trust store

After Node.js, the same lesson applied to the browser before it could bite: Chromium on Linux
reads the user's NSS database, not the system bundle, so a `curl` that trusts the private CA
says nothing about it. The build adds the CA to the job user's NSS database, and the tool
checks load the CI service's page in Chromium, directly with Playwright and again through the
Playwright MCP server over its stdio protocol, as a coding agent would. Either fails on a TLS
error.

## Configuration used here

```toml
forge_url = "https://ci.example.com"
token_file = "/etc/ci-runner/forge-token"
scope = "user"
labels = ["amd64", "linux", "agent"]
golden = "ci-golden"
max_vms = 3
cpus = 4
memory_gib = 16
poll_seconds = 15
claim_timeout_seconds = 600
job_timeout_seconds = 22200
retry_after_seconds = 900
drain_seconds = 21600
unregistered_grace_seconds = 30
not_handed_out_grace_seconds = 60
not_handed_out_retry_seconds = 60
```

That is the amd64 host. The Mac runs `max_vms = 2`: it is also a workstation. Three 4-CPU
VMs on an 8-thread host goes past §12.2's SHOULD. It was a deliberate call: golden builds
and the host's other heavy work are rare, so a third concurrent job was worth more than
the headroom.

`job_timeout_seconds` is a little over the runner's own six-hour limit, so the runner's timeout
always fires first and the job's log says why it ended. `poll_seconds` is well inside the minute
after which Forgejo marks a runner offline, so the visibility runner never flickers.

## Measurements

| | |
|---|---|
| Clone and boot to the runner taking the job | 17–30 s |
| Clone alone | 3.2 s on ext4, a full copy of the 6 GB golden disk; instant on APFS, copy-on-write |
| A skipped job, end to end | about 20 s |
| Golden build | about 5 minutes, either host |
| Golden VM on disk | 6–7 GB, plus the previous one |
| A preview deploy and smoke test, end to end in a job VM | 7 minutes |

## Where it falls short of the specification

- **§4.2, a dead host's claim.** The scaler leaves any job that another registration
  carries, however old. A host that dies between registering and starting its VM would
  block that job until someone deletes the stale registration by hand. Not yet seen; found
  by review, not by an incident.
- **§5.4.1, a registration that vanishes.** The scaler counts a run as claimed only when it
  sees the registration active. A job short enough to finish between polls is never seen
  claimed, and its handle is cooled. Harmless in practice — the job is no longer waiting —
  but it is not what the spec says.
- **§5.7.4.** A killed scaler loses its running jobs, as the spec allows; it does not rebuild
  runs from surviving VMs.
- **§9.4 is honoured; the fence is not built.** On Linux it can be a firewall rule on the service
  account's traffic, since user-mode networking sends a VM's traffic out as the process that runs
  it. On the Mac the VMs run as the logged-in user, whose own traffic must not be fenced, so it
  needs a proxy the VMs are pointed at.
- **§5 several hosts on one label** is unit-tested and running — both hosts serve two shared
  labels — but the simultaneous-registration path has not been observed live, only in tests.
- **Old runner records.** The persistent runners' registrations were instance-level and stay
  listed, offline, until an administrator deletes them.
