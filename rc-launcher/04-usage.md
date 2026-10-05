# 04 — Usage

## The split that makes this work

> **The launcher starts sessions. It does not drive them.**

Everything after "the session exists and you can see it" belongs to the vendor's own
app. The launcher has no chat interface, no transcript, no history, no notion of what
the driving tool is doing. It is a page with a machine picker, three fields and a
terminal.

That boundary is why the tool stays small enough to audit in one sitting, which matters
a great deal for something that runs arbitrary commands on your machines.

## A typical session

You are on a train. Something occurs to you about a project on the machine at home.

```
1. Open the launcher on your phone. Sign in.

2. Machine:    studio-01
   Directory:  type "blue" — the list narrows to ~/dev/blueprints
   Name:       (left empty — one is made from the directory and the time)
   Command:    claude rc --permission-mode auto     (the configured default)

3. Press Launch. A terminal opens.

   $ claude rc --permission-mode auto --name blueprints-1432
   Error: this machine is not signed in.

4. You are at a prompt, in the right directory. Sign in, right there.

   $ claude setup-token
   Open https://example.com/oauth/device?code=WXYZ-1234
   ...

5. Run it again — up-arrow, Enter.

   $ claude rc --permission-mode auto --name blueprints-1432
   Session blueprints-1432 connected. Control it from the app.

6. Close the tab. The session keeps running.

7. Open the vendor's app. "blueprints-1432" is in the list. Drive it from there.
```

Step 3 is the ordinary case, not the unhappy path. Machines get signed out, tokens
expire, directories go untrusted. The design's entire answer to that class of problem is
step 4: *you are already at a usable prompt, on the right machine, in the right
directory.*

## Choosing a machine

The launcher does not know which machine your project is on, and should not try to
guess. In practice the choice is habitual:

| Machine | What it is for |
|---|---|
| The always-on desktop | Default. Anything long-running, anything you want to pick up tomorrow |
| The second desktop | A different architecture, or a second long job so the first is not competing for the machine |
| The small always-on box | Deployment and infrastructure work — the machine that already holds those credentials |
| The laptop | Rarely. It sleeps, and a sleeping machine's session is not reachable |

That last row is worth internalising. A session on a machine that sleeps is not a
session you can come back to. Launch on a machine that stays awake, or expect to relaunch.

## The command box

It is free text. That is deliberate, and it is the reason the whole service sits behind
authentication.

It starts pre-filled with the default from configuration. The flags you want change more
often than the launcher does, so the default is an edit and a redeploy, not a release.
If you edit the box, your version is remembered; put it back to the default and the
launcher forgets it, so a later change to the default reaches you.

```
claude rc --permission-mode auto            # the driving case, and a sensible default
claude rc --model opus                      # arguments pass straight through
claude --continue                           # resume, if the tool supports it
```

`--permission-mode auto` deserves a moment's thought rather than a copy. A session you
pick up from a phone has nobody at the keyboard to approve tool calls one at a time, so
without it the session stalls on the first approval. With it, the session acts without
asking — on your machine, as your account. That is the right trade for this use, and it
is a trade.

Do not put a name argument in the command. The launcher adds one itself, so that the
multiplexer and the vendor's app agree, and it refuses a command that already has one.

## Naming a session

Leave the name empty and you get the directory plus the time: `blueprints-1432`. That is
fine for one session and useless for five, because five sessions in the vendor's app
that all start `blueprints-` are five sessions you open one by one to find the right one.

Type a name when you will want to find it again — `footer-fix`, `review-pr-12`. Anything
outside letters, digits, `-` and `_` is replaced, so `fix: the footer` becomes
`fix-the-footer`. The box is empty again after every launch, on purpose: a name
identifies one session, and reusing it either collides with the running one or gives
you two that look the same.

Launching twice from the same directory inside the same minute without a name collides
too. The launcher refuses the second rather than typing into the first; wait a minute,
or name it.

What not to put in it:

- **Anything non-interactive.** A build, a test run, a script. Those exit, and you get a
  prompt — which works, but you have used a terminal multiplexer, an SSH bridge and a
  websocket to run something that wanted a one-line SSH command.
- **Anything you would not type at a prompt on that machine.** There is no sandbox here.
  The command runs as your account, with your keys, in that directory.
- **Anything with a credential in it.** It is remembered as your last command and it is
  in the request body. Put credentials in the machine's environment, not in the box.

## Handing off, and coming back

Once the driving tool reports that it is connected, the browser terminal has done its
job. You can close it.

Coming back, the launcher's session list shows what is running across every machine.
Two things you can do with a session:

- **Attach** — opens the terminal on it again. Use this when the vendor's app shows the
  session as gone, or the tool has exited and you want to restart it in place, or you
  want to see what is actually on the screen.
- **Kill** — terminates it. Use this deliberately.

Sessions are cheap but not free: each one is a long-lived process holding a project
directory and, usually, an authenticated tool session. **Kill them when you are done.**
A list of forty abandoned sessions is a list you stop reading, which defeats the point
of having the list.

## On a phone

Two details do more for usability than everything else combined.

**Copy-all.** Selecting text in a terminal on a touch screen is impractical, and the
thing most worth grabbing is a sign-in URL. A copy-all control is not a convenience
here, it is the difference between recovering a signed-out machine from your phone and
not. Note that it must rejoin wrapped rows — see
[§9.6](03-acceptance-criteria.md#9-the-client).

**Focus.** The soft keyboard only appears for a genuinely focused input, and only if the
focus happened inside the user's own gesture. If the terminal does not focus the instant
it opens, the phone experience is a terminal you cannot type into, which reads as
"broken" rather than "focus bug".

## Anti-patterns

- **Launching from your desk.** Open a terminal. You have added a browser, a proxy and
  a websocket to a thing you could type in two seconds.
- **Leaving the browser terminal open as your working interface.** It is a recovery
  console, not a workspace. The vendor's app is better at being the vendor's app.
- **Pre-filling a directory across machines.** A path from one machine on another is a
  guaranteed failed launch, offered helpfully. See
  [§5.5](02-specification.md#5-working-directories).
- **Launching in a home directory** because it was offered. If the driving tool refuses
  to run there, the list should never have contained it.
- **Adding a "run this for me" button** for common commands. Each one is a new path into
  the shell that the free-text box already covers, and one of them will eventually be
  built without the quoting.
- **Exposing it "just on the LAN" without authentication.** Every device on that network,
  and everything running on those devices, now has a shell on your machines. See
  [01](01-concept.md#threat-model).
- **Putting the launch route behind an API key** so your automation can use it. A key
  door is for things that are safe behind one shared secret; running a free-text command
  on a machine is not one of them. See
  [§2.9](02-specification.md#2-authentication).
