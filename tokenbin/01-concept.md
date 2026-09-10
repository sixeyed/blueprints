# 01 — Concept

## The problem

A script needs a credential at runtime. A deploy script needs an API token; a backup
job needs a storage key; a publish step needs an app password. The credential has to
reach the script somehow.

Every ordinary answer has the same shape: something holds the credential permanently,
and anything that can reach that something can read it, at any time, without anyone
noticing.

| Approach | Why it falls short |
|---|---|
| Paste it into a chat or agent session | The credential enters a transcript and a second store, both of which outlive the task |
| A pastebin URL | Works, but that URL is readable by anyone who has it, forever. No gate, no revocation short of deletion |
| OS keychain | Machine-local. Useless to a second machine or a cluster |
| A secrets manager (Vault, Infisical, Bitwarden) | Solid, and the right answer at scale — but always readable to whatever holds the client credential. There is no moment where a human decides *yes, now* |

That last row is the point. These tools are not insecure; they are *unattended*. They
answer the question "is this caller authorised?" and never the question "should this be
readable right now?"

## The idea

**A secret is unreadable by anybody until its owner deliberately opens a window.**

Each secret has a permanent, stable address. That address returns `401` — not just to
strangers, but to everyone, including its owner. When a script is about to need it, the
owner opens a window: some minutes, some number of reads. During that window the address
returns the value in plain text to whoever asks. Then it closes, on whichever limit runs
out first, and returns to `401`.

Nothing is scheduled and nothing is automatic. Approval is a person deciding.

## Why this shape

**Pre-approval, not request-and-grant.** There is deliberately no endpoint for a caller
to *request* access and no notification to approve. That would need a callback channel,
a pending-request queue, and a way to authenticate the requester — and it puts the human
in the loop at the worst moment, under time pressure, primed to say yes. Pre-approval
inverts it: you decide before you start, when you are calm and know what you are about
to run.

**N reads, not one.** Single-use is tempting and wrong in practice: scripts retry,
pipelines have several steps, and a failed run should not burn the credential. A small
budget absorbs that without becoming unlimited.

**Both limits, checked together.** Time alone leaves a window open after the work is
done. Reads alone leave a window open indefinitely if the script never runs. Whichever
expires first closes it.

**Lazy evaluation.** Both conditions are checked when someone reads, inside the same
operation that consumes the read. There is no background sweeper, no timer, and no
window that stays open because a scheduler died.

## Threat model

**Defends against:** a credential lying permanently readable at a known URL; a stolen
database file, backup or volume snapshot; another user of the same instance; someone
enumerating names to discover what you hold; a credential leaking into a chat transcript
or an agent's context.

**Does not defend against:** anyone who can read the URL *during* an open window; a
compromise of the host while it is running, where the encryption key is in memory; a
compromised owner account; a malicious script that legitimately fetches a value and then
does something else with it.

The window is still a window. Ten minutes and three reads is enormously better than a
permanent URL, and it is not zero. That trade is the whole design, made deliberately.

## When not to use this

- **Machine-to-machine at any real volume.** A human approving each window does not
  scale past a handful of runs a day. Use a real secrets manager.
- **Unattended or scheduled jobs.** If nobody is present to approve, the gate is just
  an outage waiting to happen.
- **Anything needing rotation policy, leases, dynamic credentials, or audit for
  compliance.** This is a small tool with a narrow trick.
- **Inside a cluster, for workloads.** Kubernetes Secrets and their equivalents already
  inject credentials into pods. This is for callers *outside* — a laptop, a CI runner,
  a home server — that have no such mechanism.

The sweet spot is a small number of high-value credentials, used occasionally, by
scripts a person is actively running.
