# Jarvis Board (Joe's live work board)

Live: https://tatedo2eo-beep.github.io/jarvis-board-f370f0/  (repo `tatedo2eo-beep/jarvis-board-f370f0`, GitHub Pages, branch `main`)

- `index.html`: the whole board (HTML, CSS and JS in one file, no external fonts, images, libraries or CDNs).
  Two layouts, switched by the toggle at the top and remembered per device in `localStorage` (`jb_layout`):
  - **Command Room** (A): pink/magenta header `JARVIS + CLAUDE CODE`, 8 KPI cards, an isometric SVG room with a bot per
    workstream desk (Recipes, Knowledge, Animations, Site, Board, Claude Code), speech bubbles, a Launch rocket, a
    recipes-over-time chart, `jarvis.log` and a small node graph.
  - **Spider** (B): a `launch.scope` text block listing every launch dish by chapter, threads through a focal point to an
    arc with one tick per launch recipe, a big pink counter, stacked cards, a DECISION STREAM, per-chapter bars,
    a dot matrix (one dot per launch recipe) and a wireframe globe.
  - `?layout=A` or `?layout=B` forces a layout (for testing); `?toast` shows a demo toast.
- `derive.py` (local only, not pushed): fills the derived fields below, read-only from the site sources.
- `data.json`: **the single source of truth.** Everything on the board is rendered from it.
- `update.sh "msg"`: validates `data.json`, stamps `updatedAt` (Cairo time), commits and pushes it.
  If the push fails, run `git -c credential.helper= -c credential.helper='!gh auth git-credential' push -q origin main`.
  `update.sh` only stages `data.json`; commit `index.html` / `README.md` yourself when you change them.

## How the page behaves (light on the phone)

- Fetches `data.json` on load, then at most every `max(60, refreshSeconds)` seconds. It skips the fetch while the tab is hidden and fetches once when the tab becomes visible again if the data is stale.
- If the text of `data.json` is unchanged, nothing is re-rendered and nothing animates.
- When something changed: the bars animate to their new values, the changed panels glow once, and **one** toast shows the newest log entry for 5 s. Toasts never loop.
- The live clock is plain text updated once a second in `Africa/Cairo`. Pulses on active graph nodes and the terminal cursor are CSS-only and switch off with `prefers-reduced-motion`.

## What is real data and what is decoration

Every number and every label with a count comes from `data.json`. Motion that loops is decoration only: bot bobbing,
monitor flicker, flowing graph links, node pulses, the radar scan over the dish list, the focal ring spin, the ticker
scroll, the rotating globe, the rotating speech bubble (it cycles through the real `running` tasks) and the blinking
cursor. The globe's green dots are the real coverage (one per percent of launch recipes written). All of it stops with
`prefers-reduced-motion`, and the globe only draws (about 15 fps) while layout B is shown, the tab is visible and the
globe is on screen.

## Derived fields (written by derive.py, which update.sh runs)

`derive.py` reads, and never writes, these sources:
- `/workspace/rw/tami_manifest.json` via `/workspace/rw/registry.py` for the launch scope: the dish names and flavour names
  of the 5 launch chapters (burgers, chicken, cookies-bars-brownies, pasta, cakes-cupcakes). Each dish counts as
  1 + its number of flavours, which gives 13 + 43 + 39 + 57 + 41 = 193.
- `/workspace/public-site/content/recipes/*.json` for what is written: recipes are counted by their `dish` field.
- `/workspace/public-site/knowledge/*.md` for knowledge stats (topic files, technique IDs, unique source URLs).

It writes only `data.json` in this folder: `dishes`, `chapters`, `headline.done/total`, `knowledge.stats`,
`derived`, and it appends to `history` when the recipe count changes. Run `NO_DERIVE=1 ./update.sh` to skip it.

## data.json schema (v2)

| key | type | used for |
|---|---|---|
| `schemaVersion` | number | currently `2` |
| `brand` | string | footer text |
| `title` | string | reference only (header text is fixed) |
| `updatedAt` | ISO 8601 with offset | "updated N min ago"; set by `update.sh` |
| `timeZone` | string | `Africa/Cairo` (the clock always uses Cairo) |
| `refreshSeconds` | number | poll interval; values under 60 are raised to 60 |
| `site` | `{label, url, status, note?}` | the test-site button |
| `headline` | `{label, done, total}` | derived launch total (RECIPES card, Spider counter) |
| `chapters[]` | `{name, done, total}` | derived. Chapter KPI cards, bars, arc chips |
| `stages[]` | `{name, pct}` | the `Launch` stage pct is shown on the rocket (the flame lights above 0%) |
| `running[]` | `{name, detail, status, node?}` | Running list. `status` text is shown as a tag (`NEW` = gold, `LIVE` = green). `node` names the graph node it belongs to (reference) |
| `waiting[]` | `{name, detail, owner}` | Waiting list; `owner` is shown as the tag (for example `Joe`, `Paused`, `Pending`) |
| `knowledge` | `{status, detail, items[{name, done}]}` | Knowledge module card |
| `animations` | `{status, detail, total, done}` | Animations module card. `done: null` shows the status instead of a count |
| `milestones[]` | `{name, date, done}` | kept for reference (not drawn in the current layouts) |
| `graph` | `{root, hub, children[{name, active, task?}]}` | node graph: root (Joe) → hub (Jarvis) → up to 6 children. `active: true` makes the node pulse; a desk is active if a `running` item points at it or its graph child is active. `task` is an optional bubble text |
| `setup[]` | `{k, v}` | kept for reference (not drawn in the current layouts) |
| `dishes[]` | `{slug, name, chapter, units, done, status}` | derived. `units` = 1 + flavours; `status` is `done`, `partial` or `waiting`. Spider list, arc ticks and dot matrix |
| `dishesInProgress[]` | dish slugs | optional, hand-set: shows those dishes pink (in progress) in Spider |
| `focus` | chapter name | the chapter being written now (for example `Cookies`); shown pink and in the "chapter in progress" card |
| `history[]` | `{t, recipes, total}` | recipes written over time (Command Room chart and wall screen). Seeded from this repo's git history of `headline`; `total` was 100 before the launch scope became 193 at 15:50 |
| `knowledge.stats` | `{files, techniques, sources}` | derived: KNOWLEDGE and SOURCES KPI cards, knowledge board in the room |
| `derived` | `{at, from, recipesOnSite}` | when derive.py last ran |
| `running[].node` | desk name | which desk in the Command Room shows this task in its speech bubble (`Recipes`, `Knowledge`, `Animations`, `Site`, `Board`, `Claude Code`) |
| `log[]` | `{t, e}` | terminal log feed, **newest first**. `t` is a short Cairo-time label like `Oct 22 15:55` |

Removed in v2: `toasts`, `toastEveryMs`, `toastShowMs` (the looping pop-ups). The toast now comes from `log[0]` and appears only on a real change.

## Typical update

1. Edit `data.json`: change the numbers, add a new entry at the **top** of `log`.
2. `./update.sh "15:55 chicken 20/43"`
3. Phones pick it up within about 1 to 2 minutes (Pages build plus the 60 s poll).
