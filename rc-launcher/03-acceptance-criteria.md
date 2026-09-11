# 03 — Acceptance criteria

Build against these. Each is a behaviour you can assert on; each should map to at least
one automated test that fails before the code satisfying it exists.

The failure mode of this system is **a launch that reports success while the session is
already gone.** Creating a session in a multiplexer succeeds whether the process inside
connected or died in 200 milliseconds — the creation call returns the same thing either
way, and there is no server-side signal that separates them. So the natural
implementation returns `{"ok": true}`, the operator's terminal opens on nothing, and the
error that would have explained it was destroyed along with the session.

[§1](#1-the-session-outlives-everything) and [§2](#2-the-launcher-claims-nothing-it-cannot-know)
are the criteria that catch it. They are the ones an implementation passes by accident in
the happy path — launch something that works and everything looks right — and fails the
first time a machine is signed out.

[§3](#3-authentication-guards-every-route) is the other half. This service runs
operator-supplied commands on your machines as a feature; §3 is not hardening, it is the
only thing standing between that feature and the network. Test it as an attacker with no
credentials, from outside, against every address the deployment publishes.

Where a criterion says *assert on the machine*, assert on the machine — not on the
launcher's own idea of what it did. Most of these defects are the launcher being
confidently wrong.

---

## 1. The session outlives everything

- [ ] **1.1** After a launch, querying the target machine over a **new, independent
      connection** shows the session running.
- [ ] **1.2** The session survives the terminal client disconnecting. Close the browser,
      reattach, and the same session shows its prior output.
- [ ] **1.3** The session has a pseudo-terminal. *Run a check for terminal-ness inside
      the session and assert on its answer, rather than assuming.*
- [ ] **1.4** A command that exits immediately with a non-zero status leaves a **live**
      session. *Launch a command that does not exist on the machine. This is the test the
      whole design turns on.*
- [ ] **1.5** After that failure, the session is at an interactive prompt and **accepts
      input**. Type a command into it and assert it ran. *Readable is not enough. A
      retained dead process passes "the error is visible" and fails this.*
- [ ] **1.6** That prompt's working directory is the directory that was requested.
- [ ] **1.7** The session's root process is a shell, not the requested command. *Assert
      on the process, not on the output.*
- [ ] **1.8** The requested command appears in the session's scrollback as having been
      entered — evidence it was delivered as input.
- [ ] **1.9** Restarting the launcher does not disturb any running session, and the
      sessions are listed again afterwards and can be reattached.

## 2. The launcher claims nothing it cannot know

- [ ] **2.1** The launch response for a command that dies instantly is **indistinguishable**
      from the response for one that connects and stays up. Both carry the session
      identity; neither carries an outcome.
- [ ] **2.2** No code path decides what happened by matching text captured from the
      session. *Change every string the simulated driving tool prints — the banner, the
      prompts, the errors — and the full suite still passes. If anything fails, something
      is reading console text as a contract.*
- [ ] **2.3** One operator action produces a live attached terminal. No second click, no
      "now open the terminal" step.
- [ ] **2.4** An operator with only a phone can recover a machine that is signed out:
      launch, read the failure in the terminal, satisfy the prerequisite in that same
      terminal, launch again, and get a working session. *This is the end-to-end
      acceptance test for the entire blueprint. Run it by hand at least once.*
- [ ] **2.5** A launch against an unreachable machine reports the machine by name and
      leaves nothing half-created. *Assert on the machine afterwards.*

## 3. Authentication guards every route

- [ ] **3.1** An unauthenticated request for the page is refused or redirected to sign
      in.
- [ ] **3.2** An unauthenticated launch request from a plain command-line HTTP client is
      refused **and nothing runs on any machine**. *Assert the second part on the
      machine. A 302 with a session created behind it is a pass on the status code and a
      total failure.*
- [ ] **3.3** Every other API operation is refused unauthenticated. *Enumerate the routes
      from the running application and assert on each one. A hand-written list of routes
      goes stale the first time someone adds one — which is exactly the case this
      criterion exists to catch.*
- [ ] **3.4** An unauthenticated terminal transport connection is refused **before any
      connection to a target machine is opened**.
- [ ] **3.5** An absent, empty or unverifiable identity is refused, and never resolves to
      a default or first user.
- [ ] **3.6** There is exactly one network path to the application. *From an
      uncredentialed host on the same network, attempt every address and port the
      deployment publishes — service addresses, node-level ports, host ports, metrics,
      debug. Every one refuses.*
- [ ] **3.7** Stopping the authenticating component makes the application **unreachable**,
      not unguarded.
- [ ] **3.8** No response from any operation contains the credential used to reach target
      machines, in any field or log line.
- [ ] **3.9** The health check, if unauthenticated, reveals no machine name, address,
      session name or count.

## 4. Values from the browser never become commands

- [ ] **4.1** A working directory whose name contains shell metacharacters, spaces and
      quotes launches literally. *Create a directory whose name is a command substitution
      that writes a file, launch in it, and assert the file does not exist.*
- [ ] **4.2** The same holds for the value after it has passed through **both** levels of
      interpretation — the outer transport and the shell inside the session. *Assert on
      the machine, not on the string the launcher built.*
- [ ] **4.3** The same holds for the session name on the attach and terminate paths.
- [ ] **4.4** Posting a session name to the launch API does not set the session's name.
      *The field should not exist; assert that supplying it changes nothing.*
- [ ] **4.5** A command containing quotes, pipes and redirection runs exactly as typed.
      *This is the deliberate capability; a test asserting it stops someone "fixing" §4.1
      by escaping the command too.*

## 5. The terminal relay

- [ ] **5.1** A keypress sent as a **text** payload types into the session. So does one
      sent as a **binary** payload. *Test with a real client library against a real
      transport. A fake that feeds the server raw bytes tests the half that was never
      broken.*
- [ ] **5.2** An unrecognised text payload is treated as keystrokes rather than
      discarded or raising.
- [ ] **5.3** A resize control message changes the **remote** terminal's dimensions.
      *Assert on the machine's reported size, or emit a line longer than the default
      width and assert it does not wrap.*
- [ ] **5.4** Disconnecting the client while the session is **silent** releases the
      remote connection, the channel and the attaching client. *Assert on the machine
      that no attached client remains. A chatty session hides this defect completely.*
- [ ] **5.5** A contract mismatch inside the relay produces a log line. *Inject one and
      assert. A relay that swallows it presents as a clean disconnect and is
      undebuggable.*
- [ ] **5.6** Attaching to a pre-existing session works and shows its prior output.
- [ ] **5.7** Nothing listens on target machines beyond the pre-existing remote-execution
      service. *Scan the machine.*

## 6. Directories

- [ ] **6.1** The offered list is read from the machine at the time of asking, not from
      cache or configuration. *Add a directory on the machine and assert it appears
      without restarting the launcher.*
- [ ] **6.2** The machine's home directory is absent from the list, on **two machines
      with different home paths**. *A single-machine test passes with the path hard-coded,
      which is the bug.*
- [ ] **6.3** The list is filterable by typing, and usable with around a hundred entries.
- [ ] **6.4** A directory not in the list can be launched.
- [ ] **6.5** The remembered directory is per machine. *Select one on machine A, switch to
      machine B, and assert A's path is not pre-filled.*
- [ ] **6.6** The remembered command is global and survives a machine change.

## 7. Naming

- [ ] **7.1** The multiplexer's session name and the driving tool's session label are the
      same string. *Assert on both, from both sides.*
- [ ] **7.2** Launching from a directory whose basename contains characters structural to
      the multiplexer produces a session that can be attached to **and terminated** by
      name. *The failure here is a session that exists forever and cannot be addressed.*
- [ ] **7.3** Two launches from the same directory produce two distinct, targetable
      sessions.

## 8. Listing under failure

- [ ] **8.1** With one machine unreachable, the listing returns within a bounded time and
      reports the other machines. *Blackhole the machine — drop packets. A machine that
      actively refuses connections fails fast and does not exercise this at all.*
- [ ] **8.2** Machines are queried concurrently: four unreachable machines take about as
      long as one, not four times as long.
- [ ] **8.3** The connect phase is bounded independently of the command timeout. *Assert
      with a blackholed host and no command that could time out.*
- [ ] **8.4** An unreachable machine is shown as unreachable — not omitted, and not an
      error page.

## 9. The client

- [ ] **9.1** Opening the terminal focuses it, and the **first** keystroke reaches the
      session. *The regression this catches is a dialog that focuses its own button.*
- [ ] **9.2** On a touch device, the terminal raises the soft keyboard and routes its
      input, and tapping the terminal restores focus. *Verify on a real device.*
- [ ] **9.3** Opening and closing the terminal ten times sends each subsequent keystroke
      exactly once. *Assert on the count received, not on what the screen looks like.*
- [ ] **9.4** A refresh the operator asked for is never absorbed by the in-flight poll
      guard. *Terminate a session and refresh immediately; the list updates now, not on
      the next tick.*
- [ ] **9.5** A late response from an earlier request cannot overwrite a newer one.
      *Delay the first request's reply until after the second has been applied, then
      assert the newer state stands.*
- [ ] **9.6** A copy-all control produces text in which a line longer than the terminal
      width is one unbroken line. *Assert on a long URL — that is the thing people
      actually copy, and a newline through the middle of it is silently useless.*

## 10. Operational

- [ ] **10.1** Remote invocations work on a machine where neither the driving tool nor
      the multiplexer is on the default search path. *Test through a non-interactive
      remote command. An interactive login loads profile scripts and hides this
      completely.*
- [ ] **10.2** The service refuses to start on invalid configuration — a machine entry
      missing an address or a tool location — rather than failing at the point of use.
- [ ] **10.3** Any simulated mode is off by default, is not selectable per request, and
      is visible in the interface when active.
- [ ] **10.4** Absolute URLs are built from configuration. *Send a request with a
      misleading host header and confirm generated URLs ignore it.*

---

## Coverage note

An implementation passing §1, §5 and §6 works. An implementation passing §2 is
*trustworthy* — it never tells you a session started when it did not. An implementation
passing §3 and §4 is *safe to expose*.

The second and third sets are where reimplementations fall short, and not through
carelessness. Returning `{"ok": true}` after a successful create call is the most
natural code in the world, and it is a lie roughly half the time you actually need the
tool. Guarding the page with a login and calling the job done is equally natural, and it
leaves the one endpoint that runs commands on your machines answering to anyone who can
reach the port.
