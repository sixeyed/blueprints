# Blueprint: forgejo-vm-runner — a fresh VM for every CI job

A specification for running each job from a self-hosted CI service in its own virtual
machine. The VM is cloned from a prepared image when the job is waiting, runs that one
job, and is deleted when the job ends. Nothing a job does survives it, and no two jobs
ever share a machine.

The driving case is AI coding agents run as CI jobs with their permission prompts turned
off, on the same small machines that hold your deploy credentials and your interactive
sessions. A persistent runner gives every job the run of its host. This takes that away
without giving up the things that make self-hosted runners worth having: your own
hardware, both CPU architectures, and jobs that run `docker` the ordinary way.

**It is Forgejo-shaped.** The specification names no product, but it needs a CI service that
lists waiting jobs with a handle, and a runner that takes exactly the job a handle names.
Forgejo is the only one known to provide both. GitHub Actions can run the same idea with
some changes; [01](01-concept.md#which-ci-services-this-fits) says what they are.

These documents are written to be **built from**, not just read. Point a coding agent at
this directory and it should have enough to produce a working implementation in whatever
language you prefer, and enough to tell whether what it built is correct.

## The documents

| | |
|---|---|
| [01-concept.md](01-concept.md) | The problem, the idea, the threat model, and when *not* to use this |
| [02-specification.md](02-specification.md) | Normative requirements: taking jobs, coordinating hosts, the VM lifecycle, the image, the toolchain, egress |
| [03-acceptance-criteria.md](03-acceptance-criteria.md) | Testable criteria. Build against these |
| [04-usage.md](04-usage.md) | What living with it looks like: smoke tests, new hosts, toolchain updates, rollbacks |
| [05-reference-implementation.md](05-reference-implementation.md) | Non-normative: one implementation's choices, and the traps it hit |

`02` and `03` are the contract. `01`, `04` and `05` are context — useful, but you can
disagree with them and still have built the right thing.

## Pointing an agent at this

Something like:

> Read this directory and build the scaler and the image build it specifies. Use
> <your language>. `02-specification.md` is normative — where it says MUST, that is not
> a suggestion. Work test-first against `03-acceptance-criteria.md`; every criterion
> should map to a test that fails before you write the code that satisfies it. `05`
> describes a prior implementation: treat its decisions as informative and its **Traps**
> section as hard-won, but do not copy its stack unless it suits you.

Three things worth saying to the agent explicitly, because they invert normal instincts:

1. **Cleanup must not be able to fail.** Every job ends by deleting its VM and its runner
   registration, and the obvious code puts those calls in a `finally`. If anything in
   that block raises — and a hypervisor's list command can — the run is never marked
   finished. It holds a slot for ever, its job is never retried, and the service cannot
   stop because it is waiting for it. The failure is silent: the scaler looks healthy and
   simply does less. See [§5.6](02-specification.md#5-the-job-vm).
2. **One VM operation at a time per host.** Creating, starting and deleting VMs look
   independent and are not: a hypervisor manager that reads every instance's files while
   starting one will fail on a neighbour that is half-created. Two jobs that arrive
   together start two VMs a few milliseconds apart, so this happens on the first busy
   minute, not in a stress test. See [§5.2](02-specification.md#5-the-job-vm).
3. **The CI service's own runner list is the coordination point.** Several hosts may
   serve the same label, and the instinct is to add a lock service or a database. Every
   host can already see every runner registration in its scope; record the job on the
   registration and let the lowest registration win. See
   [§4](02-specification.md#4-several-hosts-one-job).

None of these looks broken when you get it wrong. That is why they are here rather
than left to be discovered.

## Scope of this blueprint

**In:** a scaler per host that watches the CI service's queue, registers a one-job runner
for each waiting job it can serve, and runs it in a fresh VM cloned from a golden image;
coordination between hosts that serve the same labels; the golden image, built per host
from plain lists of tools, versioned and reversible; a smoke test per label and per host;
a per-job record of the domains each job looked up, to build a network allowlist from;
each host visible in the CI service's own runner list.

**Out:** the CI service itself; blocking network egress (the records here are the input
to that, not the fence); caching between jobs; Windows or macOS guests; GPUs; a warm pool
of pre-booted VMs by default ([02 §12.4](02-specification.md#12-sizing) allows one, and [01](01-concept.md#why-this-shape) says why it is not the default); fleets large enough
to want a scheduler. If you need those, you are building a CI platform and this is the
wrong shape.

## Licence and provenance

Written up from a working implementation serving a self-hosted Forgejo from two hosts —
an amd64 mini PC and an arm64 Mac — where it replaced every persistent runner. Nothing in
`02` is tied to that CI service, hypervisor or language, and `03` names a common tool
only where a test needs a concrete case; where an example is needed, it uses `example.com` placeholders. [05](05-reference-implementation.md) ends with
the places the reference still falls short of this specification.
