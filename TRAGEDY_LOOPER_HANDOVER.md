# Tragedy Looper — Digital Mastermind Companion
## Project Handover Document

> **For use in Claude Code.** This document reflects the full state of the project as of the latest commit and supersedes all prior versions of this file. It exists so development can continue in a fresh session without loss of context.

---

## Overview

A single-file HTML/CSS/JavaScript web app (zero dependencies, ~335KB) implementing a near-complete digital Mastermind companion for **Tragedy Looper: New Tragedies (WizKids 2022)**, with full secondary support for the **Z-Man Games (2014)** edition — including its **Cosmic Evil** expansion (Prime Evil / Cosmic Mythology tragedy sets).

Started as a static player aid, and has grown into a fully interactive digital board: drag-and-drop character movement, facedown card play with a proper reveal/resolution phase (matching real tabletop rules), automatic win-condition tracking, a filterable game log, a Final Guess end-game screen, original character portrait art, and an optional local-scan portrait override for private builds.

**Output file:** `tragedy-looper-advisor.html`
**Redirect file:** `index.html` — GitHub Pages root redirect to the app (see *Hosting*, below)
**Architecture:** All code lives in one self-contained HTML file (5,020 lines, ~343KB). Two `<script>` blocks share global scope — Block 1 holds game data and core logic, Block 2 holds the game log, card-play/reveal system, board rendering, portrait, and Final Guess systems.

**Live URL (GitHub Pages):** `https://calsolum.github.io/tragedy-looper-companion/` — confirmed working by the user. Note the repo itself has since been renamed/moved to `Calsolum/tragedy-looper-companion` (GitHub reports this automatically on push to the old `Calsolum/hello-world` remote; `git push` still succeeds via the redirect).

---

## File Structure

```
tragedy-looper-advisor.html
├── <style>              CSS (custom properties, mobile-first, dark theme)
├── HTML Screens         9 screens (all fixed-position, show/hide via .on class)
├── HTML Modals          Card play, next day, GW ability, forbid/warning, portrait, reveal
└── <script> × 2         Game data + all logic (see function map below)
```

---

## Screens (navigation via `go(id)`)

| ID | Screen | Description |
|----|--------|-------------|
| `sv` | Version Select | First-run screen — NT vs Z-Man edition (default `.on`) |
| `sh` | Home | Navigation hub, active script banner |
| `sl` | Script Library | Load preloaded or saved scripts |
| `ss` | Setup | Plot selection, characters, incidents, save slots |
| `sd` | Day Input | Board state entry (fallback to board view) |
| `sb` | Board View | **Primary play screen** — interactive drag/drop board |
| `sg` | Game Log | Filterable chronological event log |
| `sa` | Advice | Rule-based threat analysis and card recommendations |
| `sfg` | Final Guess | End-game role-guessing screen (new — see below) |
| `sr` | Rules Reference | Cards, incidents, roles reference |

---

## Global State Object (`G`)

```js
const G = {
  version: 'nt',           // 'nt' | 'zm'
  scriptName: '',
  tL: 4, dL: 6,             // total loops, days per loop
  cL: 1, cD: 1,             // current loop, current day
  mainPlot: null,           // plot ID string e.g. 'tsi'
  subPlots: [],             // array of plot ID strings (max 2)
  chars: [],                // array of character objects
  incidents: [],            // array of incident objects
  loc: {City:0, Shrine:0, Hospital:0, School:0},  // intrigue per location
  wcProgress: {},           // manual WC tracking: wcId → {val: n}
  patientUnlocked: false,   // set true by Doctor's ♥♥♥ GW ability this loop
};
```

### Character Object
```js
{
  id: Number, name: String, role: String,
  loc: String,           // 'City' | 'Shrine' | 'Hospital' | 'School'
  unease: Number, maxUnease: Number,
  intr: Number,           // personal intrigue tokens
  gw: Number,             // protagonist goodwill tokens
  dead: Boolean,
  forbiddenLocs: []        // array of location strings this char cannot enter
}
```

### Incident Object
```js
{ id: Number, day: Number, type: String, culprit: String, fired: Number }
```

---

## Key Data Constants

### `PLOTS_NT` / `PLOTS_ZM`
Arrays of plot objects: `id`, `set`, `type` ('main'|'sub'), `name`, `roles[]`, `wcs[]`, `desc`.

- **`PLOTS_NT`** — First Steps + Basic Tragedy, unchanged from original build.
- **`PLOTS_ZM`** — rewritten and verified against the official Z-Man Player's/Mastermind's Handbooks and the Cosmic Evil expansion handbook (many entries in the original data were fabricated/incorrect and were corrected). Now spans **4 tragedy sets** (55 plot entries total):
  - `fs` — First Steps
  - `bt` — Basic Tragedy
  - `prime_evil` — Prime Evil (Cosmic Evil expansion)
  - `cosmic_mythology` — Cosmic Mythology (Cosmic Evil expansion)

Plot-set grouping in the UI is driven by `SET_LABELS` / `SET_ORDER` (`renderPlotGroup()`, `renderPlotLists()`, `makePlotEl()`), which iterate all four sets generically rather than hardcoding First Steps/Basic Tragedy — new sets can be added by extending those two constants plus the plot arrays.

### `INCIDENTS_NT` / `INCIDENTS_ZM`
- `INCIDENTS_NT` — 9 incidents, unchanged.
- `INCIDENTS_ZM` — corrected against the rulebooks, and extended with 12 Cosmic Evil incident names (Sacrilegious Murder, Evangelium of the Dead, Fountain of Filth, Night of Madness, Awakened Curse, The Executioner, Evil Contamination, Insane Murder, Discovery, Uproar, Fire of Demise, Hound Dog Scent).

### Win Condition Types (`wc.type`)
| Type | Tracked By | Description |
|------|-----------|-------------|
| `key_person_death` | Auto (unease bar) | Key Person at/near unease limit |
| `loc_intrigue` | Auto (G.loc) | Location intrigue ≥ threshold |
| `char_intrigue` | Auto (c.intr) | Character personal intrigue ≥ threshold |
| `char_start_loc_intrigue` | Auto (G.loc at c.loc) | Intrigue at char's *starting* location |
| `char_loc_intrigue` | Auto (G.loc at c.loc) | Intrigue at char's *current* location |
| `char_max_goodwill` | Auto (c.gw) | Character GW ≤ threshold at loop end |
| `incident_fired_type` | Manual (G.wcProgress) | Specific incident type fired |
| `kill_count` / `death_count` | Manual (G.wcProgress) | N kills / deaths |
| `role_death` / `role_death_either` | Manual (G.wcProgress) | Specific role(s) died |

### `PRELOADED_SCRIPTS` (NT, 13 scripts)
Unchanged — all verified against the physical New Tragedies Mastermind's Handbook.

### `PRELOADED_SCRIPTS_ZM` (Z-Man, **23 scripts**)
Grew from 0 to 23 across several passes of sourcing/verification:
- 10 scripts adapted/reconstructed from the official Z-Man rulebooks and cross-checked against a scanned physical card archive
- 3 "Card Deck Variant" alternates — kept as **separate** entries rather than overwriting the originals, because the card-scan source and the rulebook-adapted source disagreed and the user chose to keep both rather than pick one
- 10 Cosmic Evil expansion scripts (Prime Evil / Cosmic Mythology), added once the corresponding plot definitions existed
- Some entries are flagged with a ⚠ "PARTIAL DATA" note in their `note` field where a location or incident-day value was inferred rather than confirmed from a source — **do not silently upgrade these to unmarked/confirmed status** without re-verifying against a rulebook or card scan.

### `GW_ABILITIES`
Unchanged in structure — 28 characters, each with `gwNeeded`, `label`, `desc`, `effect`, `target`, optional `oncePer`.

### `KNOWN_FORBIDDEN` / `FORBIDDEN_COND`
Unchanged:
```js
{ 'Shrine Maiden': ['City'], 'Office Worker': ['School'],
  'Patient': ['City','Shrine','School'], 'Henchman': [] }
```

### `CHAR_PORTRAITS`
Object keyed by character name → base64 SVG data URI. **28 characters covered**, including a portrait added for "Teacher" this cycle. All portraits are **original, programmatically-generated SVG artwork** (background gradient, hair colour/style, skin tone, outfit colour/archetype) — never scans or reproductions of the physical card art, official or fan-made (see *Copyright note*, below).

### Card Definitions
`MM_CARDS` (14 cards) and `PROT_CARDS` (9 cards) — unchanged. Each: `{ id, label, icon, tag ('mm'|'prot'), desc, effect, delta?, dir?, usesPerLoop }`.

---

## Facedown Card / Reveal System (major addition)

The original build resolved cards immediately on tap, which doesn't match real Tragedy Looper play (cards are placed face-down, then revealed together and resolved in a fixed order). This is now fully implemented:

- **`PENDING_CARDS`** — array of cards placed face-down but not yet resolved this day.
- **`addPendingCard(targetType, targetId, cardId, cardLabel, cardIcon, tag)`** — queues a card instead of applying it immediately.
- **`pendingCountFor(targetType, targetId)`** — badge count shown on a character/location while cards are pending.
- **`findCardDef(cardId, locKeyHint)`** — resolves a card id back to its full definition (handles both `MM_CARDS`/`PROT_CARDS` and location-only card ids).
- **`pendingCardTier(p)`** with **`TIER_LABELS = ['Forbid Movement','Movement','Other Forbids','Other']`** and **`orderedPendingCards()`** — enforce the official resolution order.
- **`FORBID_CANCELS = {mm_fu:'unease', mm_fg:'goodwill', p_fi:'intrigue', p_fi_loc:'loc_i', p_fm:'move'}`** and **`pendingCancelledIds()`** — implement the official rule that a Forbid card cancels the effect of any Action card of the *same type* played this day.
- **`openRevealModal()`** — the reveal UI; walks through pending cards in tier order, shows which are cancelled, and lets the mastermind/player step through resolution.
- **`_resolveOnePendingCard(p, cancelledIds)`** / **`resolvePendingCards()`** — apply effects in order, skipping cancelled cards.
- **`pendingClearDay()`** — clears the queue on day advance.

This replaced the old "stub" functions noted as incomplete in the previous version of this document — they are now fully wired into the card-play and next-day flow.

---

## Final Guess Screen (major addition)

Implements the official end-game mechanic: after the final loop, protagonists guess every character's secret role one at a time; a single wrong guess is an immediate loss, all correct is a win. Not available in First Steps scripts (that Tragedy Set has no hidden roles to guess).

- **`sfg`** — new screen, added to the screen table above.
- **`FG_ORDER`, `FG_RESULT`, `FG_LOSE_CHAR`** — tracking state for guess order, per-character result, and which character (if any) caused a loss.
- **`fgIsFirstSteps()`** — gates the feature off for First Steps scripts.
- **`fgReset()`**, **`fgGuess(charId, correct)`**, **`renderFinalGuess()`** — screen lifecycle and rendering.
- Character portrait thumbnails on this screen use the same `aspect-ratio:5/7` treatment as the board (see *Portrait System*, below) rather than a fixed small square.

---

## Portrait System

Portraits are base64-encoded inline SVG data URIs in `CHAR_PORTRAITS`, generated programmatically per character (background gradient, hair, skin tone, outfit archetype). No network requests, no external image files.

**Display placement (current):** the portrait renders as a **full-height sidebar image** on the right edge of each character card in Board View, not a small header thumbnail. `.char-card` is a flex row (`display:flex;align-items:stretch`) with two children:
- `.char-card-body` — flex:1 column holding the existing header/stats/history content
- `.char-card-portrait` — `width:74px;flex-shrink:0;object-fit:cover;align-self:stretch` — an `<img>` sibling that stretches to the full height of the card

This was changed from an earlier iteration where the portrait was a small 34px thumbnail inside the card header, which left a large empty area on the right side of every card (user-reported and fixed). The portrait's aspect ratio (`5/7`) matches the measured aspect ratio of the physical cards (500×700 / 480×680 scans, ≈0.7143) — this is a factual/functional detail, not copyrighted expression, and was adopted deliberately as the app's portrait template.

**Portrait modal** (`#portrait-modal-img`, `openPortraitModal()` / `closePortraitModal()`) — full-size view on click, also sized `width:min(230px, 62vw);aspect-ratio:5/7`.

**Copyright note (important, discussed at length with the user):** the app must **never** embed scans or reproductions of the physical card artwork, official or fan-made — fan art is independently copyrighted by its creator, and official art is Z-Man/WizKids' property. Only original, self-authored SVG artwork may be embedded in the repo/app. Bare facts (character names, roles, dimensions, mechanics) are fine to use; illustrations are not. This constraint was reaffirmed when the user asked about using original card images for a **private, local-only, non-hosted** copy of the app — the answer given was: reproducing copyrighted art is technically still infringement even for private/offline personal use (no blanket personal-use exemption in US copyright law), but enforcement risk for a private local copy is effectively nil, and it's the user's call to make for their own private build. **I (Claude) will not source, download, or embed the actual copyrighted card scans myself, even locally** — but the app has been adapted (see *Local Portrait Override*, below) to read portrait images from a local folder path if the user wants to drop in their own scans themselves.

To add a new **original** portrait: add an entry to `CHAR_PORTRAITS` keyed by the character's exact `name` string, value a base64 SVG data URI. **Known past bug:** a manual edit once left a literal placeholder string (`PLACEHOLDER_TEACHER_B64`) instead of real base64 data — always verify the string decodes to a real image before committing.

### Local Portrait Override (implemented)

The app can optionally load portraits from a local `portraits/` folder placed next to `tragedy-looper-advisor.html`, instead of the built-in SVG art — for a private, offline copy where the user drops in their own scans of the physical cards. This is fully implemented (was previously discussed-but-not-built):

- **`LOCAL_PORTRAITS`** — boolean, persisted in `localStorage` (`tl_local_portraits`). Toggled via the **Portrait Source** card on the Home screen (`toggleLocalPortraits()`), which flips the flag, updates the card's label (`updatePortraitToggleLabel()`, also called from `updateVerLabels()` so it's correct on load/navigation), and re-renders the board/Final Guess screen so the change is visible immediately.
- **`portraitSrc(charName)`** — when local mode is on, returns `portraits/{charName}.{ext}` (first of `PORTRAIT_EXTS = ['jpg','jpeg','png','webp']`, filename matches the character's exact name, e.g. `portraits/Shrine Maiden.jpg`); otherwise returns the built-in `CHAR_PORTRAITS[charName]` data URI directly.
- **`portraitOnError(imgEl, charName)`** — wired to every portrait `<img>`'s `onerror`. In local mode, cascades through the remaining extensions in `PORTRAIT_EXTS` one at a time (tracked via `imgEl.dataset.pStage`); once exhausted (or if local mode is off and the built-in URI itself somehow fails), falls back to the built-in `CHAR_PORTRAITS[charName]` art. No local folder is required to exist — a page with local mode on and no `portraits/` folder present falls straight through to the built-in art with no broken images, confirmed via headless browser testing.
- Wired into all four portrait render sites: the board sidebar image, the hover popup (`showCharPortrait`), the fullscreen modal (`openPortraitModal`), and the Final Guess remaining-characters thumbnail.
- No settings UI beyond the single Home-screen toggle — there's no folder picker; the convention is fixed (`portraits/` relative to the HTML file, filename = exact character name).

---

## Runtime Trackers (reset cycles)

| Tracker | Type | Resets On |
|---------|------|-----------|
| `GW_USED` | `{}` charId_abilityIdx → true | Loop change |
| `CARD_USED` | `{}` cardId → count | Loop change |
| `DAY_CARDS` | `{ mm:0, prot:0 }` | Day change |
| `PENDING_CARDS` | Array | Day change (`pendingClearDay()`) |
| `G.patientUnlocked` | Boolean | Loop change |
| `GAME_LOG` | Array | Manual clear only |
| `G.wcProgress` | `{}` wcId → {val} | Loop change |

Clear functions: `gwClearLoop()`, `cardClearLoop()`, `cardClearDay()`, `pendingClearDay()`. Triggered by `bAdjLoop()`, `bAdjDay()`, `advanceDay()`.

Location card plays (`mm_li1a`, `mm_li2`, `p_fi_loc`) are now correctly subject to the same daily/once-per-loop limit tracking as character cards — this was a confirmed gap in the prior version of this document and has been fixed.

---

## Modal System (z-index stacking)

| Modal | Element ID | z-index | Trigger |
|-------|-----------|---------|---------|
| Card Play | `card-play-modal` | 9002 | 🃏 button on char card or location |
| Reveal / Resolve | (via `openRevealModal()`) | 9002+ | Reveal Cards button in board footer |
| Next Day | `next-day-modal` | 9002 | Next Day → button in board footer |
| GW Ability | `gw-modal` | 9010 | ♥ button on char card |
| Forbidden/Warning | `forbid-modal` | 9010 | Forbidden move, once/loop override, day limit |
| Portrait | `portrait-modal` | 9800 | Click portrait sidebar image |
| Pip tooltip | `pip-tip` (dynamic) | 9500 | Hover/tap on history pip |
| Portrait hover | `char-portrait` (dynamic) | 9600 | Mouse hover on char card (desktop) |
| Touch drag ghost | `touch-ghost` (dynamic) | 9999 | Touch drag start |

`forbid-modal` is reused for forbidden-location, once-per-loop, and daily-limit warnings; `resetForbidModalStyle()` restores its default red appearance after amber overrides.

---

## Board View Architecture

### Location Layout
```
┌─────────────┬─────────────┐
│  HOSPITAL   │   SHRINE    │
├─────────────┼─────────────┤
│    CITY     │   SCHOOL    │
└─────────────┴─────────────┘
```
Each zone: `id="lz-{locKey}"`, `data-loc="{locKey}"`.

### Drag and Drop
- Mouse: HTML5 `ondragstart`/`ondragover`/`ondragleave`/`ondrop`.
- Touch: custom `touchstart`/`touchmove`/`touchend` on `document`, ghost clone follows finger, `document.elementFromPoint()` for drop target.
- Both call `bSetLoc(charId, locKey)`, which enforces forbidden locations (red = forbidden, amber = allowed).

### Character Card
`.char-card` is now a flex row (`char-card-body` + `char-card-portrait`, see *Portrait System*). `renderCharHistoryPips(charId)` still filters `GAME_LOG` for the current loop and renders 18×18 pip icons with hover/tap tooltips via `showPipTip()`.

---

## Win Condition Engine

Unchanged in design: `getDerivedWCs()`, `getWCProgress(wc)`, `getWCAutoStatus(wc)`, `analyseThreats()` (feeds the Advice screen).

---

## Game Log (`GAME_LOG`)

Unchanged: `{ loop, day, type, icon, main, meta, charId, charName, ts }`, types `'mm'|'prot'|'gw'|'effect'|'sys'`, `addLog(...)`, `renderLog()`, `setLogTab(t)`.

---

## Next Day Modal / Advance Day Logic

`openNextDayModal()` still builds MANDATORY / OPTIONAL / STATUS INFO sections, but now also surfaces the reveal step for any unresolved `PENDING_CARDS` before the day can advance. `advanceDay()` increments day/loop, clears daily trackers (including pending cards), logs the transition, re-renders the board.

---

## Patient Special Case

Unchanged — `G.patientUnlocked` (set by Doctor's ♥♥♥ GW ability) is checked in `bSetLoc()`, `onDragOverForbid()`, `onTouchMove()`, and dropdown rendering in `renderBoard()`. Resets in `gwClearLoop()`.

---

## Version Differences (`nt` vs `zm`)

| Feature | New Tragedies (nt) | Z-Man (zm) |
|---------|-------------------|-----------|
| Unease token label | "Unease" | "Paranoia" |
| Mastermind card | Forbid Unease | Forbid Paranoia |
| Preloaded scripts | 13 | **23** (10 core + 3 variants + 10 Cosmic Evil) |
| Tragedy Sets | First Steps, Basic Tragedy | First Steps, Basic Tragedy, **Prime Evil, Cosmic Mythology** |
| First Steps main plots | A Place to Protect, Premeditated Murder, Fire of the Avenger | Heinous Plot, Premeditated Murder, Light of the Avenger |
| Basic Tragedy Sealed Item | Brain + Cultist roles | Witch role |
| Incidents | 9 | 9 base + **12 Cosmic Evil** = 21 |

`getPlots()`, `getIncidents()`, `getPreloadedScripts()`, `uneaseLabel()` all switch on `G.version`.

---

## Hosting (GitHub Pages)

- `index.html` was added at the repo root as a meta-refresh redirect to `tragedy-looper-advisor.html`, so the Pages root URL loads the app directly.
- GitHub Pages was enabled manually by the user in the repo settings (no MCP tool exists to toggle Pages programmatically).
- Confirmed working live at `https://calsolum.github.io/tragedy-looper-companion/` (user-confirmed: "Yes it loads").
- Note: this session's own network sandbox cannot reach `github.io` domains to self-verify (`EGRESS_BLOCKED` on both WebFetch and curl) — verification always has to come from the user.

---

## Known Pending / Suggested Next Work

### Confirmed gaps
- **`loadAndEdit`** — loads a preloaded script then navigates to Setup; verify the plot dropdowns visually pre-select the correct plots (flagged as unverified in the prior handover; status since then not re-confirmed).
- Some `PRELOADED_SCRIPTS_ZM` entries carry a ⚠ "PARTIAL DATA" note for inferred (not source-confirmed) values — worth another verification pass if more source material turns up (rulebook scans, card photos).
- The `deck92` card-scan source's groups 03–05 were never identified/used — unknown content, low priority.
- `deck91` (a fan alt-art card deck found in the uploaded archive) was explicitly deprioritized as cosmetic-only and not pursued.

### Feature ideas discussed / logical next steps (not yet started)
- **Time Spiral screen** — inter-loop discussion/notes view.
- **Custom script builder** with the difficulty-estimation formula from the rulebook.
- **Undo stack** — everything mutates `G` directly; would need snapshotting.
- **Export/import full mid-game state** (not just setup/script).
- **Configurable location intrigue maxes** — `LM` is hardcoded `{City:3,Shrine:2,Hospital:2,School:2}`; some scripts may vary.
- **Butterfly Effect auto-flagging** — win-condition check exists via `wcProgress`, but nothing auto-flags when a Butterfly Effect incident actually resolves on board.
- **Scrollable board zones** for small screens with many characters in one zone.

---

## CSS Custom Properties

```css
:root {
  --bg:#0e0c0a;  --bg2:#161310;  --bg3:#1e1a16;  --bg4:#272018;
  --bdr:#3a2f22; --bdr2:#4a3c2a;
  --t:#e8ddd0;   --t2:#a89880;   --t3:#6b5a48;
  --red:#c0392b; --red2:#e74c3c; --rbg:#2a1210;
  --amb:#d4891a; --abg:#201508;
  --grn:#4a8c5c; --gbg:#0d1f12;
  --blu:#3b7ab5; --bbg:#0c1824;
}
```
Typography: `'Cinzel'` (serif, headings/labels), `'Crimson Pro'` (body text), loaded via Google Fonts.

---

## Development Notes

- **No build step.** Edit the HTML file directly. Both `<script>` blocks must pass `node --check`.
- **Testing syntax:** extract the two script blocks to `.js` files first (a `<!doctype>`-aware regex over `<script>...</script>`), then `node --check` each. `new Function(...)` has false positives on Unicode comments — don't rely on it.
- **Testing behavior:** this repo's established workflow uses a headless Playwright test — Chromium at `/opt/pw-browsers/chromium`, launched with `args:['--no-sandbox']` from `/opt/node22/lib/node_modules`, all non-`file://` routes aborted, always `browser.close()` + `process.exit(0)`. Load a preloaded script via `loadPreloaded(idx)` (not a made-up `loadScript()` helper — that name doesn't exist), then `go('sb')` to reach the board, `go('sfg')` for Final Guess, etc.
- **`go(id)` is the router.** New screen: add `id="sX" class="scr"` HTML, a case `if(id==='sX') renderX();` inside `go()`, and a nav entry on Home.
- **Adding a modal:** z-index above 9002 (wide-modal base); above 9010 for warnings that must appear over the card-play modal.
- **`renderBoard()` is idempotent** — rebuilds innerHTML from scratch each call; any state change just needs a fresh `renderBoard()` call to propagate.
- **`jsAttr(v)` helper (important, was a real shipped bug):** `JSON.stringify(v).replace(/"/g,'&quot;')`. Any time a stringified value is embedded inside a double-quoted HTML attribute (`onclick="fn(${...})"`), it **must** go through `jsAttr()`, not raw `JSON.stringify()`. Raw `JSON.stringify()` produces an embedded `"`, which prematurely closes the surrounding attribute and silently truncates everything after it — this broke the portrait button for every character in the app before it was caught and fixed. Grep for `JSON.stringify(` inside any `onclick=`/`onmouseenter=` string before adding new inline handlers.
- **`addLog()` signature:** `addLog(type, icon, main, meta='', charId=null, charName='')` — always pass `charId`/`charName` for character-targeted events so they show in that character's pip history.
- **Git workflow for this repo:** commits pushed directly to `master` (no PR flow established), with descriptive multi-paragraph commit messages (what changed, why, how it was verified).

---

## Data Sourcing Discipline (important for future sessions)

A hard rule was established and followed throughout this project: **never fabricate game data from inference when a better source is available or when uncertain.** Concretely:
- Corrected fabricated/wrong entries in `PLOTS_ZM`/`INCIDENTS_ZM` against the official Z-Man Player's/Mastermind's Handbooks and Cosmic Evil expansion handbook rather than trusting the pre-existing (flawed) data.
- When a scanned physical-card source and a rulebook-adapted source disagreed on 3 scripts, both were kept as separate entries rather than silently picking one — surfaced the conflict to the user instead.
- Partial/inferred data is explicitly flagged (⚠ "PARTIAL DATA") rather than presented as confirmed fact.
- If you (a future session) are asked to add or "verify" more script/plot/incident data, apply the same standard: cite what source justifies each fact, and flag or ask rather than guess when a source is ambiguous or missing.

---

## File Locations

| File | Path |
|------|------|
| Main app | `tragedy-looper-advisor.html` (repo root) |
| GitHub Pages redirect | `index.html` (repo root) |
| This document | `TRAGEDY_LOOPER_HANDOVER.md` (repo root) |
| Project readme | `README.md` (repo root) |

Primary sources used during development (official rulebook PDFs, a scanned physical card archive) were supplied as uploads in earlier chat sessions and are not stored in this repository. If deeper rules verification is needed again, ask the user to re-supply the relevant PDFs/scans rather than assuming their prior contents from memory.
