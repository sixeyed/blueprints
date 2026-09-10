# Screenshots

From a running instance. They are here to show what the model *feels* like — the
resting state is inert, and live state is unmistakable.

| | |
|---|---|
| [`00-signed-out.jpg`](00-signed-out.jpg) | The landing page. States the whole model in two sentences and three steps |
| [`01-empty.jpg`](01-empty.jpg) | Signed in, holding nothing. The create form and the empty state |
| [`02-closed.jpg`](02-closed.jpg) | One token, closed. The resting state, and the approval control with its inline window and budget |
| [`03-armed.jpg`](03-armed.jpg) | Armed: countdown running, budget remaining, revoke replacing approve |
| [`04-terminal.jpg`](04-terminal.jpg) | The whole product in one paste — denied, armed, served, denied again |

## What they are meant to demonstrate

**`02` and `03` next to each other** are the argument. A closed token is entirely ink
and grey; an armed one gets a rule in the margin, its label in accent, and a countdown.
That difference is the one question the dashboard exists to answer — *which of my
credentials are readable right now* — and it should be answerable from across the room
rather than by reading each row.

**`03`** shows the two limits stated together: `00:52` and `1 read left`. Whichever runs
out first closes the window.

**`04`** is worth more than the other three combined for anyone deciding whether they
want this. Three commands, and the middle one is the only one that returns anything.

## Notes on capture

- Set in a dark theme; the interface defines both, and light reproduces better in some
  contexts if you reshoot.
- Cropped to the content column — full-window shots spend most of their pixels on
  browser chrome.
- The values shown are fake. The interface never renders a stored value back to its
  owner, so a real one cannot appear in a dashboard shot; the terminal shot is the only
  place a value is visible, and that one is invented.
