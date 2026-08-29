# Tragedy Looper — Digital Mastermind Companion
## Project Handover Document

> **For use in Claude Code.** This document covers everything built in the originating chat session so development can continue without loss of context.

---

## Overview

A single-file HTML/CSS/JavaScript web app (~270KB, zero dependencies) that serves as a near-complete digital implementation of **Tragedy Looper: New Tragedies (WizKids 2022)** with secondary support for the **Z-Man Games (2014)** edition.

Started as a Mastermind player aid, expanded into a digital board with interactive card play, automatic rule enforcement, win condition tracking, a game log, and character portraits.

**Output file:** `tragedy-looper-advisor.html`  
**Architecture:** All code in one self-contained HTML file. Two `<script>` blocks share global scope — Block 1 contains all game data and logic (~191KB), Block 2 contains the game log, card play, next day, and new UI systems (~30KB).

---

## File Structure

```
tragedy-looper-advisor.html
├── <style>              CSS (~430 lines, CSS custom properties, mobile-first)
├── HTML Screens         8 screens (all fixed-position, show/hide via .on class)
├── HTML Modals          5 modal overlays (forbid, gw, card-play, next-day, portrait)
├── <script> Block 1     Game data + all core logic (~191KB)
└── <script> Block 2     Log, card play, next day, portrait functions (~30KB)
```

---

## Screens (navigation via `go(id)`)

| ID | Screen | Description |
|----|--------|-------------|
| `sv` | Version Select | First-run screen — NT vs Z-Man edition |
| `sh` | Home | Navigation hub, active script banner |
| `sl` | Script Library | Load preloaded or saved scripts |
| `ss` | Setup | Plot selection, characters, incidents, save slots |
| `sd` | Day Input | Board state entry (fallback to board view) |
| `sb` | Board View | **Primary play screen** — interactive board |
| `sg` | Game Log | Filterable chronological event log |
| `sa` | Advice | Rule-based threat analysis and card recommendations |
| `sr` | Rules Reference | Cards, incidents, roles reference |

---

## Global State Object (`G`)

```js
const G = {
  version: 'nt',           // 'nt' | 'zm'
  scriptName: '',
  tL: 4, dL: 6,            // total loops, days per loop
  cL: 1, cD: 1,            // current loop, current day
  mainPlot: null,           // plot ID string e.g. 'tsi'
  subPlots: [],             // array of plot ID strings (max 2)
  chars: [],                // array of character objects (see below)
  incidents: [],            // array of incident objects (see below)
  loc: {City:0, Shrine:0, Hospital:0, School:0},  // intrigue per location
  wcProgress: {},           // manual WC tracking: wcId → {val: n}
  patientUnlocked: false,   // set true by Doctor's ♥♥♥ GW ability this loop
};
```

### Character Object
```js
{
  id: Number,           // Date.now() + index, used as key throughout
  name: String,         // must match CHAR_PORTRAITS / GW_ABILITIES / KNOWN_FORBIDDEN keys
  role: String,         // e.g. 'Key Person', 'Brain', 'None', 'Person'
  loc: String,          // 'City' | 'Shrine' | 'Hospital' | 'School'
  unease: Number,       // current unease tokens
  maxUnease: Number,    // unease limit from card
  intr: Number,         // personal intrigue tokens
  gw: Number,           // protagonist goodwill tokens
  dead: Boolean,
  forbiddenLocs: []     // array of location strings this char cannot enter
}
```

### Incident Object
```js
{
  id: Number,
  day: Number,
  type: String,         // incident name e.g. 'Homicide', 'Butterfly Effect'
  culprit: String,      // character name
  fired: Number         // times fired this game (manual counter)
}
```

---

## Key Data Constants

### `PLOTS_NT` / `PLOTS_ZM`
Arrays of plot objects. Each plot has:
- `id`, `set` ('fs'|'bt'), `type` ('main'|'sub'), `name`
- `roles[]` — required secret roles this plot adds
- `wcs[]` — win condition objects (see Win Condition Types below)
- `desc` — human-readable summary

**NT Main Plots (First Steps):** `apt`, `pm_fs`, `fota`  
**NT Subplots (First Steps):** `sotr`, `aur_fs`, `ahs`  
**NT Main Plots (Basic Tragedy):** `pm_bt`, `tsi`, `swm`, `ctf`, `gtb`  
**NT Subplots (Basic Tragedy):** `cof`, `aur_bt`, `pv`, `tof`, `lp`, `uf`, `ala`

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
| `kill_count` | Manual (G.wcProgress) | N characters killed |
| `death_count` | Manual (G.wcProgress) | N deaths total |
| `role_death` | Manual (G.wcProgress) | Specific role has died |
| `role_death_either` | Manual (G.wcProgress) | Either of two roles died |

### `PRELOADED_SCRIPTS` (13 scripts)
Each entry: `{ num, name, creator, set, loops, days, difficulty (1-5), mainPlot, subPlots[], note, chars[], incidents[] }`

All 13 are from the New Tragedies Mastermind's Handbook, verified against the physical rulebook:
1. The First Script (FS, diff 1)
2. In the Godless Temple (FS, diff 1)
3. Magical Girls' Superiority (BT, diff 1)
4. The Cat Box (BT, diff 2)
5. The Assassin from the Future (BT, diff 3)
6. Crushed by the Hospital Building in Doronoki (BT, diff 3)
7. Those with Antibodies (BT, diff 4)
8. Un Rerum (BT, diff 4)
9. Prologue (BT, diff 5)
10. Never Ending Happy & Sad Story (BT, diff 5)
11. The Illusion under the Cherry Tree (BT, diff 4)
12. A Little Friend (BT, diff 3)
13. Fall-Sakura Gathering (BT, diff 3)

### `GW_ABILITIES`
Object keyed by character name. 28 characters have entries. Each ability:
```js
{
  gwNeeded: Number,     // minimum goodwill tokens required
  label: String,        // short name
  desc: String,         // full rules text
  effect: String,       // effect type code
  target: String,       // target scope
  oncePer: 'loop'|'day' // restriction (omit if unrestricted)
}
```
**Effect codes:** `remove_unease`, `add_goodwill`, `add_intrigue`, `remove_loc_intrigue`, `remove_intrigue`, `remove_all_loc_intrigue`, `remove_all_char_intrigue`, `reveal_role`, `reveal_subplot`, `reveal_incident_culprit`, `allow_patient_move`, `kill_char`, `revive_char`, `move_char`, `prevent_incident`, `prevent_death`, `prevent_serial_kill`, `spread_unease_here`, `transfer_unease`, `restore_card`, `prevent_unease`, `passive_note`, `manual`

### `KNOWN_FORBIDDEN`
```js
{
  'Shrine Maiden': ['City'],
  'Office Worker': ['School'],
  'Patient': ['City','Shrine','School'],
  'Henchman': []
}
```
`FORBIDDEN_COND` stores explanatory text for why (e.g. Patient's Doctor ability note).

### `CHAR_PORTRAITS`
Object keyed by character name → base64 data URI of a custom SVG portrait (120×160px). 28 characters covered. SVGs are embedded inline — no network requests.

### Card Definitions
**`MM_CARDS`** (14 cards): Unease +1 ×2, Unease −1, Forbid Unease, Forbid Goodwill, Intrigue +1 ×3, Intrigue +2 (once/loop), Move Vertical ×2, Move Horizontal ×2, Move Diagonal (once/loop)

**`PROT_CARDS`** (9 cards): Unease +1, Unease −1, Goodwill +1, Goodwill +2 (once/loop), Forbid Intrigue, Move Vertical, Move Horizontal, Forbid Movement, (×3 sets but modelled as single entries)

Each card: `{ id, label, icon, tag ('mm'|'prot'), desc, effect, delta?, dir?, usesPerLoop }`

---

## Runtime Trackers (reset cycles)

| Tracker | Type | Resets On |
|---------|------|-----------|
| `GW_USED` | `{}` charId_abilityIdx → true | Loop change |
| `CARD_USED` | `{}` cardId → count | Loop change |
| `DAY_CARDS` | `{ mm:0, prot:0 }` | Day change |
| `G.patientUnlocked` | Boolean | Loop change (in gwClearLoop) |
| `GAME_LOG` | Array | Manual clear only |
| `G.wcProgress` | `{}` wcId → {val} | Loop change |

**Clear functions:**
- `gwClearLoop()` — clears GW_USED + resets patientUnlocked
- `cardClearLoop()` — clears CARD_USED + resets DAY_CARDS
- `cardClearDay()` — resets DAY_CARDS only

**Triggered by:** `bAdjLoop()`, `bAdjDay()`, `advanceDay()`

---

## Modal System (z-index stacking)

| Modal | Element ID | z-index | Trigger |
|-------|-----------|---------|---------|
| Card Play | `card-play-modal` | 9002 (`.wide-modal`) | 🃏 button on char card or location |
| Next Day | `next-day-modal` | 9002 (`.wide-modal`) | Next Day → button in board footer |
| GW Ability | `gw-modal` | 9010 | ♥ button on char card |
| Forbidden/Warning | `forbid-modal` | 9010 | Forbidden move, once/loop override, day limit |
| Portrait | `portrait-modal` | 9800 | 🖼 button on char card |
| Pip tooltip | `pip-tip` (dynamic) | 9500 | Hover/tap on history pip |
| Portrait hover | `char-portrait` (dynamic) | 9600 | Mouse hover on char card (desktop) |
| Touch drag ghost | `touch-ghost` (dynamic) | 9999 | Touch drag start |

The `forbid-modal` is reused for three different warnings (forbidden location, once-per-loop card, daily limit) with `resetForbidModalStyle()` restoring its red appearance after amber overrides.

---

## Board View Architecture

### Location Layout
```
┌─────────────┬─────────────┐
│  HOSPITAL   │   SHRINE    │
│  (top-left) │ (top-right) │
├─────────────┼─────────────┤
│    CITY     │   SCHOOL    │
│(bottom-left)│(bot-right)  │
└─────────────┴─────────────┘
```
Each zone: `id="lz-{locKey}"`, `data-loc="{locKey}"` — used as drop targets.

### Drag and Drop
- **Mouse:** HTML5 `ondragstart`/`ondragover`/`ondragleave`/`ondrop` on character cards and location zones
- **Touch:** Custom implementation using `touchstart`/`touchmove`/`touchend` on `document`. Ghost clone follows finger. `document.elementFromPoint()` used to find drop target.
- **Both paths** call `bSetLoc(charId, locKey)` which enforces forbidden locations
- Forbidden zones highlight **red** (`drop-forbidden`); allowed zones highlight **amber** (`drop-hover`)

### Character Card Per-Char History Pips
`renderCharHistoryPips(charId)` filters `GAME_LOG` for entries with matching `charId` and current loop, renders coloured 18×18 pip icons. Hover/tap shows `showPipTip()` tooltip with full entry detail.

---

## Win Condition Engine

`getDerivedWCs()` — collects `wcs[]` from all active plots + adds Friend/Lover death conditions if those roles are present.

`getWCProgress(wc)` — returns 0–1 float. Auto-calculates for intrigue/goodwill types from live board state. Manual types read from `G.wcProgress`.

`getWCAutoStatus(wc)` — returns human-readable string e.g. "Shrine: 1/2 Intrigue".

`analyseThreats()` — scores all WCs + incident proximity + goodwill watch, returns sorted threat array used by the Advice screen.

---

## Game Log (`GAME_LOG`)

Each entry: `{ loop, day, type, icon, main, meta, charId, charName, ts }`

**Types:** `'mm'`, `'prot'`, `'gw'`, `'effect'`, `'sys'`

`addLog(type, icon, main, meta='', charId=null, charName='')` — call this whenever something game-meaningful happens.

`renderLog()` groups entries by loop → day, renders with coloured tags.
`setLogTab(t)` — filters by type ('all'|'cards'|'effects'|'gw').

---

## Next Day Modal Logic

`openNextDayModal()` dynamically builds lists of:
- **MANDATORY (red):** Serial Killer triggers (with Apply Kill button), firing incidents (with Mark Fired button), met win conditions, Paranoia Virus conversions
- **OPTIONAL (amber):** Murderer kills, Loved One protagonist kill, Time Traveler loss, Conspiracy Theorist ability, Brain ability, An Unsettling Rumour
- **STATUS INFO (white):** Blocked incidents, near-threshold win conditions

`advanceDay()` — increments day (or loop + reset), clears daily card counters, logs the transition, re-renders board.

---

## Patient Special Case

`G.patientUnlocked` (default `false`) is set `true` when Doctor's ♥♥♥ Goodwill ability fires.

Checked in **four places** — all must respect it:
1. `bSetLoc()` — bypasses forbidden check
2. `onDragOverForbid()` — shows amber (not red) hover on forbidden zones
3. `onTouchMove()` — same for touch drag
4. Dropdown rendering in `renderBoard()` — removes ✕ markers from forbidden options

Resets in `gwClearLoop()` at loop end.

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

Typography: `'Cinzel'` (serif, headings/labels), `'Crimson Pro'` (body text) — loaded via Google Fonts.

---

## Known Pending / Suggested Next Work

### Confirmed gaps
- **`loadAndEdit`** loads a preloaded script then navigates to Setup — the plot dropdowns don't visually pre-select the correct plots on load (they render correctly but the `sel` class may not apply). Needs verification.
- **`applyLocCardPlay`** exists for location card plays but the `cardId` values for location plays (`mm_li1a`, `mm_li2`, `p_fi_loc`) are not in `CARD_USED` / daily limit tracking — they bypass the limit system.
- **`pendingClearDay`** and `addPendingCard`/`removePendingCard` functions exist as stubs but the **facedown card pending system** was never fully implemented. The intent was to allow cards to be placed face-down and revealed all at once (matching real gameplay), but currently cards resolve immediately on tap.
- **Z-Man scripts** — no preloaded scripts for the Z-Man edition. Only the NT scripts are preloaded.
- **Save slots** — `forbiddenLocs` is populated from `KNOWN_FORBIDDEN` when loading preloaded scripts, but old saves (created before the forbidden location feature) won't have it. A migration guard exists in `renderChars` (`if(!c.forbiddenLocs) c.forbiddenLocs=[]`) but not in `loadFromSlot`.

### Feature ideas discussed / logical next steps
- **Facedown card reveal phase** — implement the pending card system so MM plays 3 facedown, protagonists play 1 each, then all are revealed and resolved in order (Forbid Movement → Movement → Other Forbids → Other)
- **Automatic Serial Killer position checking** at day end (currently only fires in Next Day modal if exactly 1 other char is present — could auto-highlight on board)
- **Final Guess screen** — a screen where protagonists submit their role guesses for all characters at game end
- **Time Spiral screen** — inter-loop discussion/notes view
- **Custom script builder** with the difficulty estimation formula from the rulebook
- **Undo last action** — since everything modifies G directly, an undo stack would require snapshotting G
- **Export/import G state** — allow saving the full mid-game state (not just setup), not just the script
- **Multiple location intrigue maxes** — LM is hardcoded `{City:3,Shrine:2,Hospital:2,School:2}`; some scripts may vary
- **Butterfly Effect incident tracking** — the win condition auto-detects if the incident type has been fired via `wcProgress`, but there's no automatic flag when a Butterfly Effect incident card is resolved on board
- **Scrollable board** — on very small screens with many characters in one zone, cards can overflow. Could add a scroll within each zone or a collapsible character list

---

## Portrait System

Portraits are base64-encoded SVG data URIs in `CHAR_PORTRAITS`. Generated programmatically with character-specific:
- Background gradient colour
- Hair colour and style (long, short, bun, spiky, twin pigtails, wavy, medium, bald)
- Skin tone
- Outfit colour and archetype (student, shrine, doctor, patient, police, suit, idol, ghost, robot, tree, divine, robe, alien, soldier, casual)

To add a new character portrait, add an entry to the `CHAR_PORTRAITS` object with a base64-encoded SVG. The key must exactly match the character's `name` field.

---

## How Versions Differ

| Feature | New Tragedies (nt) | Z-Man (zm) |
|---------|-------------------|-----------|
| Unease token label | "Unease" | "Paranoia" |
| Mastermind card | Forbid Unease | Forbid Paranoia |
| Preloaded scripts | 13 (all) | None |
| First Steps main plots | A Place to Protect, Premeditated Murder, Fire of the Avenger | Heinous Plot, Premeditated Murder, Light of the Avenger |
| Basic Tragedy Sealed Item | Brain + Cultist roles | Witch role |
| Giant Time Bomb | Witch's starting location | Two variants (A and B) |
| Incidents | 9 (incl. Butterfly Effect, Foul Evil) | 7 (different names/effects) |

Functions `getPlots()`, `getIncidents()`, `uneaseLabel()` all return version-appropriate data.

---

## Development Notes

- **No build step.** Edit the HTML file directly. Both script blocks must pass `node --check`.
- **Testing syntax:** `node --check tragedy-looper-advisor.html` won't work directly — extract the script blocks to `.js` files first. `new Function(scriptContent)` has false positives on Unicode comments.
- **`go(id)` is the router.** Adding a new screen: create the HTML div with `id="sX" class="scr"`, add a case `if(id==='sX') renderX();` in `go()`, and add a nav card on the Home screen.
- **Adding a new modal:** use z-index above 9002 (wide-modal base). Above 9010 for warnings that must appear over the card-play modal.
- **All `renderBoard()` calls are idempotent** — the function rebuilds innerHTML from scratch each call, so any state change just needs to call `renderBoard()` to propagate.
- **`addLog()` signature:** `addLog(type, icon, main, meta='', charId=null, charName='')` — always pass `charId` and `charName` when the event targets a specific character so it appears in that character's pip history.

---

## File Locations

| File | Location |
|------|----------|
| Main app | `/mnt/user-data/outputs/tragedy-looper-advisor.html` |
| Also in project | `/mnt/project/tragedy-looper-advisor.html` |
| NT Rulebook PDF | `/mnt/user-data/uploads/TRAGEDY_LOOPER_RULEBOOK_.pdf` |
| Mastermind Handbook PDF | `/mnt/user-data/uploads/TRAGEDY_LOOPER_MASTERMIND_RULEBOOK.pdf` |

The rulebook PDFs are the authoritative source for all game rules, plot data, character abilities, and script details. When in doubt about a rule, check the PDFs.
