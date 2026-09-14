# 02 — Specification

Normative. **MUST**, **MUST NOT**, **SHOULD** and **MAY** carry their usual weight.
Nothing here names a language, framework, database or identity provider.

---

## 1. Entities

### 1.1 Secret

| Field | Type | Notes |
|---|---|---|
| `name` | string | Unique. Forms the read URL |
| `value` | string | The credential. May be multi-line. MUST be encrypted at rest |
| `owner` | string | Stable identifier of the creator |
| `created` | timestamp | |
| `approved_until` | timestamp | `0`/null means closed |
| `approved_reads` | integer | Reads remaining in the current window. `0` means closed |
| `read_count` | integer | Lifetime successful reads. Display only |
| `last_read` | timestamp | Display only |
| `group` | string | Optional. Empty means ungrouped. See §9 |

**1.1.1** `name` MUST match `[A-Za-z0-9_](?:[A-Za-z0-9_-]{0,62}[A-Za-z0-9_])?` —
letters, digits, underscores and hyphens, 1–64 characters, no leading or trailing
hyphen. The name is the URL, so it MUST NOT require escaping or normalisation.

*Underscores and uppercase are permitted deliberately: the set is wide enough to hold a
valid environment variable name, because a secret is most often named for the variable
it will be assigned to — `HOLMES_OIDC_CLIENT_SECRET`. Restricting to lowercase forces a
mapping between the name in the store and the name in the script, which is a small,
permanent source of confusion.*

**1.1.2** Names MUST be case-sensitive, matching environment variable semantics.
Implementers should note the consequence: a mistyped case produces the uniform denial of
§4, which is indistinguishable from a closed window. This is the correct behaviour and
it will occasionally confuse someone.

**1.1.3** A value MAY span multiple lines — a private key, a certificate, a
service-account document. Line endings MUST be normalised to LF on the way in, and
nothing else about the value may be altered: leading and trailing whitespace, blank
lines and a final newline MUST all survive.

*Browsers submit multi-line form fields with CRLF endings whatever was pasted, so a key
pasted into a form arrives different from the same key read off disk. An SSH private key
with CRLF endings is rejected with a format error that mentions nothing about line
endings. Normalise once, where no path into storage can skip it — leaving it to each
caller means debugging it from the far end of a deploy script.*

*The trailing newline is the reason not to simply strip whitespace: OpenSSH rejects a
private key that lacks one.*

**1.1.4** Names MUST be unique across the whole instance. A caller attempting to create
an existing name MUST be refused. Implementers SHOULD note that this leaks the existence
of another user's name; per-owner namespaces avoid it at the cost of putting an owner
identifier in every URL. Global uniqueness is the recommended trade.

**1.1.5** There MUST NOT be an expiry date, a time-to-live, or any garbage collection of
secrets. A secret lives until deleted. The approval window is the only lifecycle. *A
store that expires or reaps rarely-read entries will silently destroy the credential you
depend on least often.*

**1.1.6** `owner` MUST be a stable identifier that cannot be reassigned to a different
person. Where the identity system offers both, use the immutable subject identifier
rather than an email address, and treat the address as display-only.

### 1.2 Audit record

| Field | Notes |
|---|---|
| `timestamp` | |
| `name` | The name requested, even where no such secret exists |
| `actor` | Owner identifier, or an anonymous marker for reads |
| `peer` | The other end of the connection. Never forgeable |
| `forwarded` | Any claimed forwarded-for value, whole. Trustworthy only per §1.2.2 |
| `event` | `create`, `approve`, `revoke`, `rotate`, `delete`, `read_ok`, `read_denied` |
| `reads_remaining` | Where applicable |

**1.2.1** An audit record MUST NOT contain a secret's value, in any field, ever.

**1.2.2** The connection peer and any claimed forwarded address MUST be recorded as
separate fields, and a claim MUST NOT be stored as though it were the peer.

*A forwarded-for header is set by the caller. It is trustworthy only when the peer is a
proxy you run — one that discards any inbound copy — and worthless otherwise. Folding
the two into a single "client address" produces rows that look like evidence and are
not, with nothing in the row to say which kind it is. Keeping them apart makes the trail
honest with no configuration: the un-forgeable fact is always present, and a claim is
visibly a claim.*

*This is preferable to configuring a list of trusted proxy addresses. Such a list is
correct when written and silently degrades when the proxy moves; recording both facts
cannot rot, and stores strictly more.*

**1.2.3** The forwarded value SHOULD be recorded whole rather than reduced to one entry.
*The chain is the useful part, and truncating it discards the evidence of who added
what.*

---

## 2. State machine

A secret is **readable** if and only if:

```
now < approved_until  AND  approved_reads > 0
```

```
CLOSED  --approve-->  ARMED(until = now + window, reads = budget)
ARMED   --read ok-->  ARMED(reads = n - 1)      while n - 1 > 0 and now < until
ARMED   --read ok-->  CLOSED                    when n - 1 == 0
ARMED   --now >= until-->  CLOSED               evaluated lazily, on read
ARMED   --revoke--->  CLOSED
ARMED   --rotate--->  CLOSED
CLOSED  --read----->  401, no state change
```

**2.1** Both conditions MUST be evaluated on every read attempt. Implementations MUST
NOT rely on a background job, scheduler or timer to close a window.

**2.2** A successful read MUST atomically: verify both conditions, decrement
`approved_reads`, increment `read_count`, set `last_read`, and write an audit record.
Concurrent reads against a budget of one MUST result in exactly one success.

**2.3** Approving an already-armed secret MUST replace the window and budget, not extend
or add to them.

**2.4** Rotating a value MUST close the window. *A window approved against the old value
is not consent to serve a new one.*

**2.5** Where the stored value cannot be decrypted, the read MUST fail as an ordinary
denial (§4) and MUST NOT consume budget.

---

## 3. Interface

| Operation | Auth | Notes |
|---|---|---|
| Read a secret | **none** | The only unauthenticated operation |
| List own secrets | owner | |
| Create | owner | |
| Approve | owner of that secret | Takes window and budget (§5.3) |
| Revoke | owner of that secret | |
| Rotate value | owner of that secret | |
| Delete | owner of that secret | |
| Set a secret's group | owner of that secret | §9 |
| Approve a group | owner | Applies to every member they own |
| Revoke a group | owner | |
| Rename a group | owner | Moves every member they own |
| Health check | none | MUST disclose nothing about content |

**3.1** There MUST be exactly one operation that returns a secret's value. *Every
additional one is another place to get the denial logic wrong.*

**3.2** That operation MUST NOT require authentication. Its callers are scripts running
plain HTTP clients, which cannot complete an interactive sign-in. The window is its
gate. Implementations MAY add a static bearer credential as defence in depth, but MUST
NOT make it the only gate.

**3.3** A successful read MUST return the value as the entire response body, as plain
text, with no envelope, wrapper, whitespace or trailing newline beyond the value itself.
*Callers substitute the response directly into a variable or header; any decoration
breaks every consumer.*

**3.4** A successful read MUST carry headers preventing storage and referrer leakage —
at minimum a no-store cache directive, a no-referrer referrer policy, and a nosniff
content-type option.

**3.5** There MUST NOT be any endpoint that lists, searches, exports, archives, or
renders secrets other than to their owner. If built on an existing application, such
routes MUST be removed rather than filtered. *A missing filter fails silently; a missing
route cannot.*

**3.6** No interface MUST ever render or return a secret's value to its owner, either.
The owner sets the value and can replace it; there is no need to read it back, and
providing one creates a second disclosure path.

---

## 4. Denial

**4.1** Every unsuccessful read MUST produce an identical response: the same status
code, the same empty body, and the same headers. This applies to all of:

- no such name
- window closed
- budget exhausted
- window expired
- value undecryptable
- name owned by a different user

**4.2** `401` is the recommended status. Whatever is chosen MUST be uniform.

**4.3** Denials MUST NOT vary in timing in a way that reveals existence, MUST NOT
include a body, and MUST NOT include an authentication-challenge header.

**4.4** Denials MUST be rate-limited. A single global threshold — N denials per
interval across all callers, then a distinct rate-limit status — is sufficient.

The limiter MUST NOT be keyed on the client address. *Doing so means keying on a
forwarded-address header, which anything able to reach the service without passing the
proxy can set to a fresh value per request — making a per-client limit free to evade
while appearing to work. The threat is enumeration, which is a property of the total
rate of denials rather than of any one source, so a single counter measures the right
thing and has no forgeable input.*

**4.5** A tripped limit MUST NOT affect successful reads. *Anyone can trip a global
counter by asking for names that do not exist; if that also blocked genuine reads, an
unauthenticated caller could halt every script depending on the service.*

**4.6** The limiter MAY be in-memory and MAY reset on restart — it exists to make
guessing slow, not to be exact.

*Without 4.1 and 4.4 the read endpoint is an oracle: an attacker learns which names
exist by probing, and a name is often a strong hint about what the credential unlocks.*

---

## 5. Ownership and identity

**5.1** Every secret MUST have exactly one owner. Owners MUST see and act on only their
own secrets. There MUST NOT be an administrative role that can read, approve or list
another user's secrets. *An admin who can approve on your behalf removes the only
property the design has.*

**5.2** Ownership MUST be enforced as part of the lookup — a single query constrained by
both name and owner — rather than as a check performed after retrieving a record. *This
makes it structurally impossible for an operation to fetch a record and forget to verify
who owns it.* A secret belonging to another user MUST be indistinguishable from one that
does not exist, on every operation.

**5.3** Approval MUST let the owner choose the window duration and the read budget at
the moment of approval. Configured values are defaults, pre-filled for convenience.
Both MUST be positive whole numbers; there is no requirement for an upper bound.

**5.4** Every authenticated operation MUST fail closed when identity cannot be
established. An absent, empty or unverifiable identity MUST NOT resolve to a default,
anonymous or first user.

### 5.5 Satisfying the identity requirement

Any of these is acceptable; the specification cares only that §5.1–5.4 hold.

| Approach | Notes |
|---|---|
| **OIDC** (recommended) | The service is its own relying party. Multi-user, no shared credential, and the identity is verified rather than asserted. See [05](05-reference-implementation.md) |
| **Authenticating proxy** | A proxy authenticates and passes identity in a header. Simpler to build, but the header is forgeable by anything that can reach the service directly — so the service MUST be unreachable except through the proxy, and the proxy MUST strip inbound copies of that header |
| **Single user with a password** | Legitimate for a personal instance. `owner` becomes a constant. Everything else in this specification still applies |

**5.5.1** Where identity is asserted by a proxy rather than verified by the service, the
deployment MUST make it impossible to reach the service by any other path. *Otherwise
identity is a header anyone on the network can set, and §5.1 becomes decorative.*

---

## 6. Storage

**6.1** Values MUST be encrypted at rest.

**6.2** The encryption MUST be authenticated, such that a modified ciphertext fails
loudly rather than decrypting to plausible-looking garbage. *An unauthenticated cipher
turns a corrupted credential into a confusing downstream failure instead of an error.*

**6.3** The key MUST come from configuration, MUST NOT be derived from anything the
reader supplies, and MUST NOT be stored alongside the data. Configuration MUST support
supplying it by file path as well as by value, so that it can be mounted rather than
placed in an environment or process listing.

**6.4** The service MUST refuse to start without a usable key.

**6.5** Losing the key destroys every secret irrecoverably. This MUST be documented
where an operator will see it.

---

## 7. Configuration

Minimum surface:

| Setting | Required | Notes |
|---|---|---|
| Encryption key | yes | MUST accept a file reference |
| Session/signing secret | yes | Where sessions are used |
| Storage location | yes | |
| Public URL | yes | Used to build absolute URLs and callbacks (§8.2) |
| Identity provider settings | conditional | Per §5.5 |
| Default approval window | no | Default pre-filled on the approval control |
| Default read budget | no | As above |
| Rate limit threshold and interval | no | |

**7.1** Configuration MUST be validated at startup, not at point of use. *A
misconfigured deployment should fail immediately, not at the moment someone needs a
credential.*

**7.2** Any development bypass of authentication MUST refuse to start when the service
is configured for a non-local public URL, and MUST log a warning on every request.

---

## 8. Deployment

**8.1** Only the read operation may be exposed without authentication. Where an
authenticating proxy fronts the service, its configuration MUST exempt exactly that one
path and MUST protect everything else. *Inverting this either locks out every script or
exposes the dashboard, and the configuration usually lives outside the service's own
repository where its tests cannot see it.*

**8.2** Absolute URLs — the identity-provider callback in particular — MUST be built
from configured public URL, not from the incoming request. *Behind a TLS-terminating
proxy the request appears as plain HTTP, and a callback derived from it will be rejected
as an unregistered redirect.*

**8.3** The service SHOULD NOT need to trust forwarded-client-address headers at all.
Nothing in this specification depends on the client's address: §4.4 forbids keying the
limiter on it, and §1.2.2 records it as a claim rather than a fact.

*Check for framework settings that enable such trust by default, and remove them if
nothing reads what they produce. Configuration that appears to be a security control,
and is not consulted by anything, is worse than none — it answers the question "is this
handled?" with a yes.*

**8.4** Where storage is a single-writer embedded database, the deployment MUST run
exactly one instance. *Two instances race on the store, and each maintains its own
in-memory rate limiter, silently multiplying the threshold.*


---

## 9. Grouping (optional)

Secrets that are used together — the four a deploy needs — are tedious to approve one
at a time, and the tedium is itself a risk: it pushes people toward long windows so they
do not have to come back.

**9.1** A secret MAY belong to a group. Group membership is optional, and a secret with
no group MUST behave exactly as it did before grouping existed.

**9.2** A group MUST NOT appear in the read URL. Names remain globally unique and the
read operation is unchanged. *A group is an organisational convenience, not part of a
secret's identity; putting it in the URL would make regrouping a breaking change for
every caller.*

**9.3** Approving a group MUST give each member its own window and its own budget, with
the same settings applied to each. Budgets MUST NOT be pooled across a group. *Pooling
means one member's reads silently consume another's, and a retry on one secret can
exhaust the allowance for a secret that has not been read at all.*

**9.4** Group operations MUST be owner-scoped exactly as single-secret operations are.
Two owners using the same group name have two separate groups. A group containing none
of the caller's secrets MUST be indistinguishable from one that does not exist.

**9.5** The empty group MUST NOT be treated as a group. *Ungrouped secrets share the
empty value; a group operation that matched on it would arm everything the owner has —
the exact opposite of a deliberate, scoped approval.*

**9.6** Membership SHOULD be changeable after creation, and renaming a group SHOULD move
its members. *Grouping is most useful applied to secrets that already exist.*
