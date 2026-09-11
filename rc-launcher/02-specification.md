# 02 — Specification

Normative. **MUST**, **MUST NOT**, **SHOULD** and **MAY** carry their usual weight.
Nothing here names a language, framework, multiplexer, transport protocol, proxy or
identity provider.

Terms used throughout:

| Term | Meaning |
|---|---|
| **the launcher** | The service this document specifies |
| **target machine** | A machine the launcher can start sessions on |
| **the driving tool** | The long-lived interactive CLI program the operator wants running |
| **terminal multiplexer** | Software on a target machine that owns terminal sessions independently of any client attached to them |
| **terminal client** | The operator's terminal emulator, running in the browser |
| **terminal transport** | The bidirectional connection carrying that terminal's traffic |

---

## 1. Entities

### 1.1 Target machine

| Field | Notes |
|---|---|
| `id` | Stable, used in URLs and stored preferences |
| `label` | Display name |
| `address` | Where the remote-execution service answers |
| `account` | The identity the launcher uses on that machine |
| `credential reference` | From configuration; see §2.7 |
| `tool locations` | Absolute locations for the driving tool and the multiplexer; see §10.2 |

**1.1.1** Target machines MUST be configured. The set of machines MUST NOT be
discoverable or extendable by a client request. *A caller who can add a machine can
point the launcher at a host they control and harvest its credentials.*

### 1.2 Session

| Field | Notes |
|---|---|
| `name` | The single identifier; see §6 |
| `machine` | Target machine `id` |
| `directory` | Working directory on that machine |
| `created` | Timestamp |
| `attached` | Whether a terminal client is currently connected |

**1.2.1** Sessions MUST be owned by the multiplexer on the target machine, not by the
launcher. The launcher MUST be able to restart without disturbing any running session,
and MUST rediscover sessions by querying the machines. *State kept only in the launcher
is state lost on a redeploy, while the sessions it described keep running unlisted.*

---

## 2. Authentication

This section is a prerequisite, not a feature. The launcher executes operator-supplied
commands on configured machines by design (§8.1); authentication is the only control
that stands between that capability and the network.

**2.1** Every route the launcher serves — the page, every API operation, and the
terminal transport — MUST require an established identity. There MUST NOT be any route
that performs work, returns machine state, or accepts a connection without one.

**2.2** A health check MAY be unauthenticated. It MUST disclose nothing: no machine
names, no addresses, no session names, no counts.

**2.3** Identity MUST fail closed. An absent, empty, malformed or unverifiable identity
MUST be refused, and MUST NOT resolve to a default, anonymous or first user.

**2.4** Guarding only the page MUST NOT be treated as satisfying **2.1**. *A launch
operation reachable by a plain command-line HTTP client is the part that matters; the
page is only how a human gets to it.*

**2.5** Authentication SHOULD be enforced in front of the application, by a single
component that guards routes by construction — for example a reverse proxy performing
an auth subrequest before forwarding — rather than by decorating each route in the
application. *A route protected by remembering to protect it is a route that will one
day be added without the decoration, and nothing will fail.*

**2.6** Where **2.5** is used, the deployment MUST NOT expose any network path that
reaches the application without traversing that component. This includes a second
listener, a directly published port, a node-level or host-level port, an internal
service address, and any debugging or metrics endpoint. *A single such path makes every
other requirement in this section decorative, and it does so silently — the guarded path
keeps working perfectly.*

**2.7** Credentials for reaching target machines MUST come from configuration. They
MUST NOT be supplied, selected or influenced by a client request, and MUST NOT be
returned by any operation.

**2.8** The terminal transport MUST be authenticated at the same boundary as everything
else. Where that boundary cannot apply its check to a protocol upgrade, the application
MUST verify identity on the upgrade request itself, and MUST refuse before opening any
connection to a target machine. *An unauthenticated terminal transport is a shell; it is
also the route most often forgotten, because it is not a page and it is not an API call.*

---

## 3. Launching a session

**3.1** A launch takes a target machine, a working directory, and a command. It MUST
produce a session in the multiplexer on that machine.

**3.2** The session MUST survive the connection that created it, and MUST survive the
request that created it. Querying the machine over a new, independent connection after
the launch MUST show the session running.

**3.3** The session MUST be attached to a pseudo-terminal. *An interactive tool given no
terminal will usually detect it and exit rather than block, so the observed symptom of
this defect is a session that vanishes, not one that hangs.*

**3.4** The requested command MUST NOT be the session's root process. The session MUST
start an interactive shell in the requested working directory, and the command MUST be
delivered to that shell as input.

**3.5** Consequently, when the command exits — for any reason, at any speed — the
session MUST remain, at an interactive prompt, in the requested directory, with the
command's output above it.

**3.6** That prompt MUST accept input. *Retaining a terminated process's output in a
non-interactive state does not satisfy **3.5**: the operator can read that they must sign
in and cannot sign in, which is the failure this design exists to prevent.*

**3.7** The launch response MUST identify the created session. It MUST NOT report
success or failure of the driving tool, and MUST NOT contain a verdict, status or
classification of what happened inside the session.

**3.8** The launcher MUST NOT decide what happened by capturing the session's output and
matching it against known text. *Creating a session succeeds identically whether the
process inside connected or died immediately; there is no server-side signal that
separates them, and console wording is not a contract.*

**3.9** A terminal client MUST be attached to the new session as part of the same
operator action that launched it, with no second interaction required. This is how
**3.7** and **3.8** are made acceptable: nothing adjudicates because the operator can
see.

**3.10** When the attached terminal client disconnects, for any reason including the
browser being closed, the session MUST detach and continue running. It MUST NOT
terminate.

**3.11** A launch against an unreachable machine MUST fail with a message naming the
machine, and MUST leave no partially created session.

---

## 4. The attached terminal

**4.1** The terminal transport MUST carry keystrokes from client to session and output
from session to client, continuously, in both directions at once.

**4.2** Payloads MUST be distinguished **by purpose, not by encoding**. A payload that
is a recognised control message (§4.3) MUST be handled as one; **any other payload, in
any encoding, MUST be treated as keystrokes**. *Terminal client libraries differ in
whether they emit text or binary for the same keypress, and a server that accepts only
one of them fails in a way that looks like a clean disconnect on the first key pressed.*

**4.3** The client MUST report its terminal dimensions, on attach and whenever they
change. The launcher MUST resize the remote pseudo-terminal to match.

**4.4** Resizing only the client's renderer MUST NOT be considered as satisfying **4.3**.
*The remote terminal is created at a default width and wraps there regardless of the
browser window; the first thing this breaks is a long sign-in URL the operator must copy,
which is the exact case the terminal exists to serve.*

**4.5** Both directions MUST run concurrently. When either direction finishes, the other
MUST be cancelled, and every remote resource — the connection, the channel and the
attaching client on the target machine — MUST be released.

**4.6** Signalling end-of-input to the remote MUST NOT be relied on to end the output
direction. *A browser going away is a network half-close, not a hangup on the remote
terminal; a silent session leaves the output read blocked indefinitely, holding a remote
connection and an attached client open for a browser tab that no longer exists.*

**4.7** The transport MUST NOT suppress errors indiscriminately. Only
disconnect-shaped conditions MAY be swallowed, and those MUST be logged. *Blanket
suppression around a relay converts every contract mismatch into a clean, silent,
untraceable disconnect.*

**4.8** It MUST be possible to attach a terminal to an existing session, not only to a
newly created one, and the attached terminal MUST show that session's prior output.

**4.9** Nothing MUST be installed or left listening on target machines beyond the
remote-execution service they already run. *Any listener that serves a terminal is a
shell to whoever reaches it, and it is reached without passing §2.*

---

## 5. Working directories

**5.1** The launcher SHOULD offer the directories the driving tool already records as
trusted or known, read live from the target machine at the time of asking. It MUST NOT
serve a cached or configured list as if it were current.

**5.2** The machine's home directory MUST be excluded from that list, compared against
the value read live from that machine. *Where the driving tool refuses to run in a home
directory, offering it offers a guaranteed failure; and a home path hard-coded in the
launcher is wrong for at least one machine.*

**5.3** The list MUST be filterable by typing. *A machine with ninety entries and a
machine with four are both normal, and an unfiltered list is unusable at the first.*

**5.4** The operator MUST be able to launch in a directory that is not in the list.

**5.5** The most recently used directory MAY be remembered and pre-selected. If it is,
it MUST be remembered **per target machine**. *A path on one machine is meaningless on
another, and pre-filling it pre-fills a failed launch.*

**5.6** The most recently used command MAY be remembered globally.

---

## 6. Naming

**6.1** A single generated identifier MUST serve both the multiplexer session and the
driving tool's own session label, so that the name shown by the vendor's interface and
the name shown by the multiplexer are the same string. *Two identifiers for one thing
is a reconciliation problem nobody asked for.*

**6.2** The name MUST be derived from the working directory, so it is recognisable, plus
a disambiguator, so two launches from the same directory do not collide.

**6.3** The name MUST NOT contain any character the multiplexer treats as structural —
separators for sessions, windows or panes. *Such a session is created successfully and
can then never be attached to or killed, because every command that targets it parses
the name instead.*

**6.4** The name MUST be generated by the launcher. The API MUST NOT accept a
caller-supplied name. *An optional field that reaches a shell is attack surface with no
user; delete it rather than sanitise it.*

---

## 7. Listing and lifecycle

**7.1** The launcher MUST be able to list running sessions across all configured
machines, and MUST be able to terminate one.

**7.2** The listing operation MUST NOT perform unbounded, serialised remote calls inside
a request path. It MUST:

- gate each machine behind a short reachability probe before attempting the full query;
- query machines concurrently, not in sequence;
- bound the connection-establishment phase explicitly, separately from any timeout on
  the command being run.

*A machine that is powered on but silently discarding packets stalls for the operating
system's connect timeout, per machine, in series — while the client, polling, queues
another identical request behind it. A command timeout does not cover the connect phase,
because no command has started.*

**7.3** An unreachable machine MUST be reported as unreachable. It MUST NOT be omitted
from the result, and MUST NOT turn the whole listing into an error.

**7.4** Where the client polls, a poll already in flight MAY suppress the next scheduled
poll. It MUST NOT suppress a refresh the operator asked for. *After terminating a
session, the operator's immediate refresh is the one request that must not be dropped —
otherwise the dead session stays on screen and clicking it fails.*

**7.5** Where two requests for the same data can be in flight, responses MUST be applied
in request order, and a response older than one already applied MUST be discarded. *A
late reply from the request that preceded a kill will otherwise put the killed session
back on the screen.*

---

## 8. Browser-supplied values and the shell

**8.1** The command field is free text by design and MUST be passed through unaltered.
The launcher MUST NOT attempt to validate, filter or restrict it. *Filtering free text
does not produce safety; it produces a boundary people trust. §2 is the boundary.*

**8.2** Every other value originating from the client that becomes part of a command on
a target machine — the session name on any attach or terminate path, the name and
directory on the launch path — MUST be quoted or escaped for the shell that will
interpret it.

**8.3** Escaping MUST account for **every** level of interpretation. A command delivered
as input to a shell running inside a session is interpreted a second time, on the target
machine. *A value escaped once, for the outer transport, is still expanded by the inner
shell — so a directory name can become a command substitution that runs on the machine.*

**8.4** The API MUST NOT accept fields the interface does not send. Unused optional
input that reaches a shell MUST be removed rather than sanitised.

---

## 9. Configuration

Minimum surface:

| Setting | Required | Notes |
|---|---|---|
| Target machines | yes | Label, address, account, per-machine tool locations (§10.2) |
| Remote credential | yes | MUST accept a file reference so it can be mounted rather than embedded |
| Public URL | yes | Used to build absolute URLs; see §10.4 |
| Reachability probe timeout | no | §7.2 |
| Connection timeout | no | §7.2 |
| Command timeout | no | Distinct from the above |
| Poll interval | no | |

**9.1** Configuration MUST be validated at startup, not at point of use. A machine entry
missing an address or a tool location MUST prevent the service from starting.

**9.2** Where a simulated or offline mode exists for development, it MUST be disabled by
default, MUST be evident in the interface whenever it is active, and MUST NOT be
selectable per request by a client. *A mode the client can ask for is a mode an attacker
can ask for; a mode that is not visible is a mode you will demonstrate by accident.*

---

## 10. Deployment

**10.1** There MUST be exactly one network path to the launcher, and it MUST pass
through the authenticating component (§2.5, §2.6).

**10.2** The launcher MUST NOT assume that a non-interactive remote login has the driving
tool or the multiplexer on its command search path. Every remote invocation MUST use a
configured absolute location, or an explicitly set search path. *Interactive logins load
profile scripts that add these locations and non-interactive ones frequently do not, so a
probe using a bare command name reports software as absent when it is installed and
working.*

**10.3** Per-machine prerequisites of the driving tool — a signed-in account, an
entitlement, a trusted directory — MUST be satisfiable through the attached terminal.
*If satisfying them requires physical or separate access to the machine, the launcher
has not removed the need to be at your desk; it has moved it.*

**10.4** Absolute URLs, including any identity-provider callback, MUST be built from
configured public URL rather than from the incoming request. *Behind a proxy that
terminates encryption, the request arrives looking unencrypted, and a URL derived from
it will be rejected as unregistered.*

**10.5** The launcher MUST run with the least privilege that lets it hold the remote
credential, and that credential SHOULD grant only the access the launcher needs on each
target machine.
