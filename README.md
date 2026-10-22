# Jarvis Board (Joe's live work board)

Live: https://tatedo2eo-beep.github.io/jarvis-board-f370f0/  (repo `tatedo2eo-beep/jarvis-board-f370f0`, GitHub Pages, branch `main`)

Scope: **the full book**: 34 chapters, 885 dishes, 227 book flavours, plus extra flavours (open-ended).

- `index.html`: the whole board (HTML, CSS and JS in one file, no external fonts, images, libraries or CDNs).
  Two layouts, switched by the toggle at the top and remembered per device in `localStorage` (`jb_layout`):
  - **Command Room** (A): `JARVIS + CLAUDE CODE` header, 8 KPI cards (DISHES, FLAVOURS, EXTRAS, CHAPTERS, NOW,
    KNOWLEDGE, ANIMATIONS, SOURCES), an isometric SVG room with a bot per workstream desk (Recipes, Knowledge,
    Animations, Site, Board, Claude Code), speech bubbles, a **Full book** rocket (share of dishes written), compact
    bars for all 34 chapters, a written-over-time chart, `jarvis.log` and a small node graph.
  - **Spider** (B): a `book.scope` text block listing all 885 dishes by chapter, threads through a focal point to an
    arc with one tick per dish, chips for the chapters that have started, a big pink dishes counter, stacked cards,
    a DECISION STREAM, 34 per-chapter bars, a dot matrix (one dot per dish) and a wireframe globe.
  - `?layout=A` or `?layout=B` forces a layout (for testing); `?toast` shows a demo toast.
- `data.json`: **the single source of truth.** Everything on the board is rendered from it.
- `update.sh "msg"`: runs `derive.py --stamp` (refresh derived fields, stamp `updatedAt` in Cairo time, validate),
  then commits `data.json` plus anything you staged (for example `index.html`, `README.md`) and pushes.
  `NO_DERIVE=1 ./update.sh` only stamps. If the push fails, run
  `git -c credential.helper= -c credential.helper='!gh auth git-credential' push -q origin main`.
- `derive.py` (local only, not pushed): fills the derived fields below, read-only on all its sources.

## How the page behaves (light on the phone)

- Fetches `data.json` on load, then at most every `max(60, refreshSeconds)` seconds. It skips the fetch while the tab
  is hidden, and fetches once when the tab becomes visible again if the data is stale.
- If the text of `data.json` is unchanged, nothing is re-rendered and nothing animates.
- When something changed: the bars animate to their new values, the changed panels glow once, and **one** toast shows
  the newest log entry for 5 s. Toasts never loop.
- `data.json` keeps each dish on one line (about 85 KB, roughly 15 KB gzipped over the wire).

## What is real data and what is decoration

Every number and every label with a count comes from `data.json`. Motion that loops is decoration only: bot bobbing,
monitor flicker, flowing graph links, node pulses, the radar scan over the dish list, the focal ring spin, the ticker
scroll, the rotating globe, the rotating speech bubble (it cycles through the real `running` tasks) and the blinking
cursor. The globe's green dots are real: one per percent of dishes written. All motion stops with
`prefers-reduced-motion`. The globe only draws (about 15 fps) while layout B is shown, the tab is visible and the globe
is on screen.

## Derived fields (written by derive.py)

Sources, all read and never written:
- Scope: `/workspace/rw/tami_manifest.json` via `/workspace/rw/registry.py` (dish names, public titles, book flavours):
  34 chapters, 885 dishes, 227 book flavours.
- Done: `/workspace/public-site/content/recipes/*.json`, counted the same way as `rw/progress.py`:
  - a **dish** is written when a recipe has that `dish` with `flavour: "Classic"`;
  - a **book flavour** is written when a recipe has that `dish` and the exact book flavour name;
  - an **extra flavour** is a recipe with `extra_flavour: true`.
- Chapter names: `/workspace/public-site/src/lib/book-chapters.json`.
- `/workspace/public-site/content/PROGRESS.txt`: its totals are cross-checked (a mismatch is printed), and its
  `Order:` line ("pasta, cakes, then …") picks `focus`, the first unfinished chapter in that order.
- Knowledge stats: `/workspace/public-site/knowledge/*.md` (topic files, technique IDs, unique source URLs).

Derive.py also sorts `log` newest first, and appends a `history` point whenever the counts change.

## data.json schema (v2, full book)

| key | type | used for |
|---|---|---|
| `schemaVersion` | number | `2` |
| `brand`, `title` | string | footer text / reference |
| `updatedAt` | ISO 8601 with offset | "updated N min ago"; stamped by `update.sh` |
| `timeZone` | string | `Africa/Cairo` (the clock always uses Cairo) |
| `refreshSeconds` | number | poll interval; values under 60 are raised to 60 |
| `site` | `{label, url, status, note?}` | test-site button |
| `headline` | `{label, done, total}` | **derived**: dishes written / 885 (DISHES card, big counters, rocket %) |
| `flavours` | `{label, done, total}` | **derived**: book flavours written / 227 |
| `extraFlavours` | `{label, done}` | **derived**: extra flavours written (no fixed total) |
| `chapters[]` | `{no, key, name, done, total, flavours:{done,total}, extra}` | **derived**, 34 entries in book order. `done/total` are dishes. A chapter counts as complete when all its dishes and book flavours are written |
| `dishes[]` | `{s, n, c, b, f:[done,total], x}` | **derived**, 885 compact rows: slug, public name, chapter no, base written (0/1), book flavours written/total, extra flavours. On the board a dish is green when the base and all book flavours are written, pink when something is written (or it is listed in `dishesInProgress`), grey otherwise |
| `dishesInProgress[]` | dish slugs | optional, hand-set: shows those dishes pink |
| `focus` | chapter name | **derived**: the chapter being written now (NOW card, "chapter in progress" card) |
| `history[]` | `{t, recipes, total, flavours?, extra?, metric?}` | written-over-time chart and wall screen. Old points keep their own scope (`total` 100, then 193 from 15:50, counting recipes); points from 18:20 count dishes against 885 (`metric: "dishes"`). The chart marks each scope change |
| `knowledge` | `{status, detail, items[], stats:{files, techniques, sources}}` | KNOWLEDGE and SOURCES cards, room wall board; `stats` is derived |
| `animations` | `{status, detail, total, done}` | ANIMATIONS card. `done: null` shows "planned" |
| `running[]` | `{name, detail, status, node?}` | Running list. `node` names the room desk whose bubble shows the task (`Recipes`, `Knowledge`, `Animations`, `Site`, `Board`, `Claude Code`) |
| `waiting[]` | `{name, detail, owner}` | Waiting list; `owner` containing "Joe" counts in "waiting on Joe" |
| `graph` | `{root, hub, children[{name, active, task?}]}` | node graph; a desk is also active when its graph child is active. `task` is an optional bubble text |
| `stages[]`, `milestones[]`, `setup[]` | lists | kept for reference (not drawn in the current layouts) |
| `log[]` | `{t, e}` | `jarvis.log` and DECISION STREAM, newest first; `t` like `Oct 22 18:18` (Cairo) |
| `derived` | `{at, from, recipesOnSite, chaptersComplete}` | when derive.py last ran |

## Typical update

1. Edit the hand-kept parts of `data.json` (running, waiting, add a `log` entry; order doesn't matter).
2. `./update.sh "18:30 pasta 25/46"`: counts are re-derived from the site content automatically.
3. Phones pick it up within about 1 to 2 minutes (Pages build plus the 60 s poll).
