# Jarvis Mission Control (Joe's live work board)

Live: https://tatedo2eo-beep.github.io/jarvis-board-f370f0/  (repo `tatedo2eo-beep/jarvis-board-f370f0`, GitHub Pages, branch `main`)

- `index.html`: the whole board (HTML, CSS and JS in one file, no external fonts, images or libraries).
- `data.json`: **the single source of truth.** Everything on the board is rendered from it.
- `update.sh "msg"`: validates `data.json`, stamps `updatedAt` (Cairo time), commits and pushes it.
  If the push fails, run `git -c credential.helper= -c credential.helper='!gh auth git-credential' push -q origin main`.
  `update.sh` only stages `data.json`; commit `index.html` / `README.md` yourself when you change them.

## How the page behaves (light on the phone)

- Fetches `data.json` on load, then at most every `max(60, refreshSeconds)` seconds. It skips the fetch while the tab is hidden and fetches once when the tab becomes visible again if the data is stale.
- If the text of `data.json` is unchanged, nothing is re-rendered and nothing animates.
- When something changed: the progress rings and bars animate to their new values, the changed panels glow once, and **one** toast shows the newest log entry for 5 s. Toasts never loop.
- On the first load the toast appears only if the data changed since this device's last visit (remembered in `localStorage`).
- The live clock is plain text updated once a second in `Africa/Cairo`. Pulses on active graph nodes and the terminal cursor are CSS-only and switch off with `prefers-reduced-motion`.
- URL flags for testing: `?notoast` suppresses the "since your last visit" toast; `?toast` shows a demo toast.

## data.json schema (v2)

| key | type | used for |
|---|---|---|
| `schemaVersion` | number | currently `2` |
| `brand` | string | footer text |
| `title` | string | reference only (header text is fixed) |
| `updatedAt` | ISO 8601 with offset | "updated N min ago"; set by `update.sh` |
| `timeZone` | string | `Africa/Cairo` (the clock always uses Cairo) |
| `refreshSeconds` | number | poll interval; values under 60 are raised to 60 |
| `site` | `{label, url, status, note?}` | the big "Open the test site" button |
| `headline` | `{label, done, total}` | the big gold ring (launch total). If missing, it is summed from `chapters` |
| `chapters[]` | `{name, done, total}` | chapter bars; a "Launch total" bar is added automatically from `headline` |
| `stages[]` | `{name, pct}` | small rings (pct 0 to 100; 100 turns green) |
| `running[]` | `{name, detail, status, node?}` | Running list. `status` text is shown as a tag (`NEW` = gold, `LIVE` = green). `node` names the graph node it belongs to (reference) |
| `waiting[]` | `{name, detail, owner}` | Waiting list; `owner` is shown as the tag (for example `Joe`, `Paused`, `Pending`) |
| `knowledge` | `{status, detail, items[{name, done}]}` | Knowledge module card |
| `animations` | `{status, detail, total, done}` | Animations module card. `done: null` shows the status instead of a count |
| `milestones[]` | `{name, date, done}` | timeline; the first item not done is highlighted as next |
| `graph` | `{root, hub, children[{name, active}]}` | node graph: root (Joe) → hub (Jarvis) → up to 6 children. `active: true` makes the node and its link pulse |
| `setup[]` | `{k, v}` | key/value list |
| `log[]` | `{t, e}` | terminal log feed, **newest first**. `t` is a short Cairo-time label like `Oct 22 15:55` |

Removed in v2: `toasts`, `toastEveryMs`, `toastShowMs` (the looping pop-ups). The toast now comes from `log[0]` and appears only on a real change.

## Typical update

1. Edit `data.json`: change the numbers, add a new entry at the **top** of `log`.
2. `./update.sh "15:55 chicken 20/43"`
3. Phones pick it up within about 1 to 2 minutes (Pages build plus the 60 s poll).
