# 04 — Usage

## The split that makes this work

> **The script fetching the credential is fine. The agent fetching it is not.**

A deploy script that fetches a token and puts it in a request header is doing ordinary
runtime injection — the same thing `vault exec` or `infisical run --` does. The value
lives in that process, for that run, and goes away.

An agent — or a person in a chat window — fetching the same value has copied a
credential into a transcript, a context window, and a log. That store outlives the task
and is not one you chose.

Everything in this design preserves that split. Which leads to the one convention that
makes it usable in practice.

## The pre-flight convention

Because approval is deliberate and happens *before* a run, the person approving needs to
know what to approve. Requiring them to remember, or to discover it by running the
script and watching it fail, defeats the point.

So each script declares what it needs, in a form anything can read without reading
values:

```javascript
// publish.mjs
export const SECRETS = ["photo-prism-token", "r2-bucket-key"];
```

```python
# publish.py
SECRETS = ["photo-prism-token", "r2-bucket-key"]
```

And a small helper prints the union across whatever you are about to run:

```console
$ ./secrets-needed publish.mjs backup.mjs
photo-prism-token
r2-bucket-key
```

**This convention is load-bearing, not decoration.** It is what lets an agent help you
run a script that needs credentials: the agent reads the *manifest*, tells you which
windows to open, and never touches a value. Without it the agent has no way to help
except by reading secrets, which is the thing that must not happen.

## Typical session

```console
# 1. See what the run needs.
$ ./secrets-needed deploy.sh
photo-prism-token

# 2. Open a window in the dashboard: 10 minutes, 3 reads.

# 3. Run the thing. The script fetches the value itself.
$ ./deploy.sh
   fetching photo-prism-token... ok
   deploying... done

# 4. Nothing to clean up. The window closed itself.
```

## Fetching, in a script

```bash
#!/usr/bin/env bash
set -euo pipefail

fetch_secret() {
    local name="$1"
    local url="https://secrets.example.com/secret/${name}"
    local body status

    body=$(curl -sS -w '\n%{http_code}' "$url") || {
        echo "cannot reach the secret store" >&2; return 1
    }
    status="${body##*$'\n'}"
    body="${body%$'\n'*}"

    case "$status" in
        200) printf '%s' "$body" ;;
        429) echo "rate limited - too many failed reads from this address" >&2; return 1 ;;
        *)   echo "'${name}' is not armed. Approve it at https://secrets.example.com" >&2
             return 1 ;;
    esac
}

TOKEN=$(fetch_secret photo-prism-token)
curl -H "Authorization: Bearer ${TOKEN}" https://photos.example.com/api/v1/albums
```

Two things this gets right, and most first attempts do not:

- **The failure message is actionable.** A bare `401` tells the person nothing. Naming
  the secret and linking to where they approve it turns a dead end into a two-click fix.
- **The value is never echoed, written to a file, or passed as a command-line argument.**
  Arguments are visible in the process table to every user on the machine. Use a
  variable or standard input.

## Fetching, in other languages

```javascript
export async function getSecret(name) {
  const url = `${process.env.SECRET_STORE}/secret/${name}`;
  const res = await fetch(url);
  if (res.status === 200) return (await res.text());
  if (res.status === 429) throw new Error("Rate limited by the secret store.");
  throw new Error(`'${name}' is not armed — approve it at ${process.env.SECRET_STORE}`);
}
```

```python
import os, httpx

def get_secret(name: str) -> str:
    store = os.environ["SECRET_STORE"]
    res = httpx.get(f"{store}/secret/{name}")
    if res.status_code == 200:
        return res.text
    if res.status_code == 429:
        raise RuntimeError("Rate limited by the secret store.")
    raise RuntimeError(f"{name!r} is not armed — approve it at {store}")
```

Note that every one of these returns the response body verbatim. That is why
[§3.3](02-specification.md#3-interface) forbids any envelope: a JSON wrapper would mean
every caller in every language needs a parser, and `$(curl -s ...)` would stop working.

## Choosing a window

| Situation | Suggested |
|---|---|
| A single scripted run you are watching | 5–10 minutes, 2–3 reads |
| A pipeline with several steps | 15–30 minutes, 5–10 reads |
| Debugging, repeated retries | 30–60 minutes, a generous budget — then revoke by hand when done |
| Anything unattended | Do not. See [01](01-concept.md#when-not-to-use-this) |

Prefer a budget that absorbs one or two retries. If you constantly need a large one, the
calling script is probably fetching inside a loop rather than once at the start.

**Revoke deliberately when you finish early.** It costs one click and closes the window
that would otherwise stay open for its full duration.

## Anti-patterns

- **Fetching inside a retry loop or a per-item iteration.** Fetch once, hold it in a
  variable for the run.
- **Caching the value to disk** to avoid re-approving. You have just rebuilt the
  permanently-readable file this exists to replace.
- **Approving everything at the start of the day.** The window is the control; a
  day-long one is a permanent URL with extra steps.
- **Putting the value in a command-line argument**, where it is visible in the process
  table.
- **Asking an agent to fetch a value "just this once".** That is the failure mode. Ask
  it which secrets the script declares, and open those windows yourself.
