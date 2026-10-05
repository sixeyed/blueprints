# Blueprint: a launcher for long-lived remote CLI sessions

A specification for a small self-hosted page that starts an interactive CLI session on
one of several machines you own — over a remote shell, inside a terminal multiplexer,
with a terminal attached in the browser. You start it from a phone and pick it up later
in the vendor's own app.

The driving case is Claude Code's Remote Control mode, which lets you drive a session
from a phone *once something has started that session on a machine*. This is the thing
that starts it.

These documents are written to be **built from**, not just read. Point a coding agent at
this directory and it should have enough to produce a working implementation in whatever
language you prefer, and enough to tell whether what it built is correct.

## The documents

| | |
|---|---|
| [01-concept.md](01-concept.md) | The problem, the idea, the threat model, and when *not* to use this |
| [02-specification.md](02-specification.md) | Normative requirements: authentication, launch, terminal, listing, deployment |
| [03-acceptance-criteria.md](03-acceptance-criteria.md) | Testable criteria. Build against these |
| [04-usage.md](04-usage.md) | What living with it looks like, on a phone |
| [05-reference-implementation.md](05-reference-implementation.md) | Non-normative: one implementation's choices, and the traps it hit |

`02` and `03` are the contract. `01`, `04` and `05` are context — useful, but you can
disagree with them and still have built the right thing.

## Pointing an agent at this

Something like:

> Read this directory and build the service it specifies. Use <your language>.
> `02-specification.md` is normative — where it says MUST, that is not a suggestion.
> Work test-first against `03-acceptance-criteria.md`; every criterion should map to a
> test that fails before you write the code that satisfies it. `05` describes a prior
> implementation: treat its decisions as informative and its **Traps** section as
> hard-won, but do not copy its stack unless it suits you.

Three things worth saying to the agent explicitly, because they invert normal instincts:

1. **Do not report whether the launch worked.** Creating a session in a multiplexer
   returns success whether the process inside it connected or died in 200 milliseconds,
   and there is no server-side signal that separates them. The instinct is to capture the
   session's output and classify it; that is pattern-matching on console text that will
   change without warning. Attach the terminal instead and let the operator read the
   screen. See [01](01-concept.md#why-this-shape) and
   [§3.7–3.9](02-specification.md#3-launching-a-session).
2. **Authentication is a prerequisite, not a follow-up task.** The command box is free
   text by design, so this page is remote code execution by design for whoever reaches
   it. It must guard the page, the API *and* the terminal transport — guarding only the
   page leaves the endpoint that runs commands answering to any HTTP client. And any
   second network path to the application defeats all of it, silently, while the guarded
   path keeps working perfectly. See [§2](02-specification.md#2-authentication).

3. **One filter for every name.** The operator may name a session, and that name reaches
   a shell on the target machine. It goes through the same allow-list as the generated
   name — not a sanitiser of its own — and the test for it must fail when the filter is
   bypassed. See [§6](02-specification.md#6-naming).

None of these looks broken when you get it wrong. That is why they are here rather
than left to be discovered.

## Scope of this blueprint

**In:** a configured set of target machines; launching a session in a chosen directory
with a configured default command or one you type, under a generated name or one you
choose; a browser terminal attached on launch and reattachable later; listing and
terminating sessions across machines; authentication as a hard prerequisite.

**Out:** anything the driving tool does once it is running — transcripts, notifications,
approvals, history. Provisioning machines, installing software on them, per-directory
authorisation, multi-tenancy, or scheduling. If you need those, you are building a
remote development platform and this is the wrong shape.

## Licence and provenance

Written up from a working implementation running against four machines. Nothing here
is tied to that implementation's language, multiplexer, identity provider or hosting;
where an example is needed, it uses `example.com` placeholders.
[05](05-reference-implementation.md) ends with the places the reference still falls
short of this specification.
