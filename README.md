# Blueprints

Specifications for small, useful things — written to be **built from** rather than read.

Each blueprint describes a working tool in enough detail that you can point a coding
agent at the directory and get your own version, in whatever language you prefer. They
are deliberately not source code: an implementation carries a hundred decisions that
were right for one person's setup, and copying it means inheriting all of them. A
blueprint carries the decisions that actually matter, and says which are yours to make.

Each one contains:

- **The reasoning** — what problem it solves, and when you should use something else
- **A normative specification** — behaviour and constraints, no framework named
- **Acceptance criteria** — testable statements to build against
- **Usage** — what living with it looks like
- **Implementation notes** — one build's choices, and the traps it hit

## The blueprints

| | |
|---|---|
| [tokenbin](tokenbin/) | An approval-gated credential store. Every secret has a permanent URL that returns `401` until its owner opens a short window |
| [infra-dashboard](infra-dashboard/) | A state and power dashboard for physical machines. One state per machine from three signals — reachability, power, cluster membership — and safe stop and start, including machines sharing one plug |
| [rc-launcher](rc-launcher/) | A launcher for long-lived remote CLI sessions. Start an interactive session on one of your machines from a phone, and pick it up later in the vendor's app |

## Using one

Point an agent at the directory:

> Read `tokenbin/` and build the service it specifies, in <your language>.
> `02-specification.md` is normative — where it says MUST, that is not a suggestion.
> Work test-first against `03-acceptance-criteria.md`. `05` describes a prior
> implementation: treat its **Traps** section as hard-won, and its stack as one option
> among many.

Then read the acceptance criteria yourself before you trust what comes back. That is the
part that tells you whether it built the right thing.
