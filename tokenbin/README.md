# Blueprint: an approval-gated credential store

A specification for a small self-hosted service that holds credentials behind a
human decision. Each secret gets a permanent URL that returns `401` by default. Its
owner opens a window — a few minutes, a handful of reads — and the URL answers for
exactly that long, then closes itself.

![The signed-out landing page: "Every token here has a permanent address that answers 401 - until you open a window for it. Scripts fetch the value during that window. Nothing else, at any other time." Below it, three numbered steps: create a token and it is closed; approve it, choosing minutes and reads; the URL answers, then falls silent again](screenshots/00-signed-out.jpg)

These documents are written to be **built from**, not just read. Point a coding agent
at this directory and it should have enough to produce a working implementation in
whatever language you prefer, and enough to tell whether what it built is correct.

## The documents

| | |
|---|---|
| [01-concept.md](01-concept.md) | The problem, the idea, and when *not* to use this |
| [02-specification.md](02-specification.md) | Normative requirements: data model, state machine, routes, security |
| [03-acceptance-criteria.md](03-acceptance-criteria.md) | Testable criteria. Build against these |
| [04-usage.md](04-usage.md) | What using it looks like, and the calling convention that makes it safe |
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

Two things worth saying to the agent explicitly, because they invert normal instincts:

1. **The agent must never read a secret's value.** Fetching secrets is what the
   *calling scripts* do at runtime. An agent that reads one has put a credential into
   a transcript, which is the failure this design exists to prevent. See
   [04-usage.md](04-usage.md#the-split-that-makes-this-work).
2. **Deleting routes beats filtering them.** If you build this on top of something
   existing, resist the urge to hide secrets from listing pages with a filter. A
   missed filter fails silently. A route that does not exist cannot leak.

## Scope of this blueprint

**In:** named secrets with stable URLs, default deny, human approval with a time
window and a read budget, per-owner isolation, encryption at rest, an audit trail.

**Out:** secret rotation policy, dynamic credentials, fine-grained roles, sharing
between users, machine-to-machine issuance, high-throughput access. If you need those,
you need a real secrets manager and this is the wrong shape.

## Licence and provenance

Written up from a working implementation. Nothing here is tied to that implementation's
language, identity provider, or hosting; where an example is needed, it uses
`example.com` placeholders.
