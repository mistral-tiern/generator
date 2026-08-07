# Moongear — Current (As-Built) Architecture

This documents how the project is *actually* built today, as distinct
from `wizardawn_architecture_plan.md`, which is the original aspirational
plan for the full rebuild. Short version of the relationship between the
two: the architecture plan describes where this is eventually headed;
this document describes the fast, single-file prototype we built instead,
in order to validate content and mechanics before committing to that
larger build. Nothing here is meant to be permanent.

---

## File structure

One file: `shop_generator_v2.html`. No build step, no server, no
dependencies. Everything — markup, CSS, data, and logic — lives in this
one file, split into two `<script>` blocks:

1. A `<script type="application/json">` block holding the embedded item
   catalog (~1,900 rows, exported from the original Wizardawn database as
   `store_items.csv`, converted to JSON, currency-cleaned).
2. A single `<script>` block with all application logic (~1,100 lines).

There is no module system, no imports, no compilation — everything is
plain, immediately-invoked browser JavaScript.

## The content engine, as actually implemented

The plan called for an "extended" pattern/token grammar engine: weighted
tables, dice/range expressions embedded in pattern strings, bound
variables, ruleset-conditional tables, and queried/dynamic tables. What
actually got built is simpler:

- `expand(pattern, tables)` — the core `{token}` substitution engine,
  recursive, same as originally scoped.
- `pick(arr)`, `randInt(min,max)`, `chance(pct)` — plain helper functions
  used *inline* in regular JS logic (e.g. `if (chance(30)) {...}`) rather
  than as a declarative mini-language embedded in pattern strings (e.g.
  there's no `{roll:1d6+2}` syntax — dice rolls are just JS function
  calls).
- Weighting is handled by literal duplicate entries in arrays (matching
  source fidelity), not a `{value, weight}` table format.
- "Bound variables" (pick once, reuse) are handled by ordinary JS
  variables in `generateNPC()`, not a context object the engine manages.
- "Queried tables" (e.g. inventory filtered by store+game) are plain
  `Array.filter()` calls against the embedded JSON, not an abstraction
  the engine provides.

Net effect: the *content* (races, hair rules, possessions, business
names, descriptions) is faithful to what was planned. The *engine* is a
much thinner, more ad-hoc layer than originally scoped — it works, but
none of it is a reusable package yet. Porting to real packages
(`content-engine`, `data/*`) per the original plan would mean extracting
these patterns into an actual declarative format.

## Data organization (all in-file, not yet split into `data/` domains)

- **NPC domain**: `RACES` (35 entries), `HAIR_GROUPS`/`FIXED_HAIR` (race →
  hair rules), `THINGS`/`JUNKY`/`WEAPON_TRINKETS`/`GEM1`/`GEM2`/`ROYALTY`
  (possessions), `NAMES`/`GENERIC_NAMES` (placeholder — see README),
  trait arrays (`TRAIT1`–`TRAIT6`, `BRAVERY`, `ENERGY`, `INTEREST`).
- **Business domain**: `TYPE_WORDS`, `HUT2_CREATURE`/`HUT2_ROLE`/`HUT3`
  (naming), `OWNER_TITLES`, `STORE_LOOKUP` (business type → catalog store
  name), `descTables` (exterior + per-type ambiance word banks).
- **Inventory**: the embedded `STORE_ITEMS` JSON, filtered live by
  `pickInventory()`/`fullCatalogPool()`.

All of this is global `var` declarations in one scope — fine at current
scale, would need real namespacing/modules if this file kept growing.

## Application logic, by feature

- `generateNPC()` — the NPC engine described above, single function,
  ~90 lines.
- `businessName()` — naming, single function.
- `newShop()` — orchestrates NPC + naming + description + inventory into
  one shop object; this is the closest thing to a "generator" in the
  sense the architecture plan meant.
- `recomputeValue()` — the Business Value formula (base + inventory
  worth × multiplier), recalculated on every inventory edit.
- Rendering is **string concatenation into `innerHTML`**, not a
  component framework — `renderCurrent()`, `renderBizDetail()`, etc. each
  build an HTML string from current state and replace a container's
  `innerHTML` wholesale. Editable fields use inline `oninput="..."`
  handlers that call back into global functions. This is intentionally
  primitive — fast to write, fine at this scale, not how the real
  frontend (React, per the original plan) would do it.

## Player-Owned Business subsystem

A second, mostly-independent state machine layered on top of the same
page:
- `makePlayerOwned()` deep-copies a saved shop into a new business
  record (separate identity from the source shop from that point on).
- `runBusinessCycle()` handles both "Run This Month" and "Advance 30
  Days" (shared function, different `includeManualDays` flag).
- Upgrades (`UPGRADE_TYPES`, `addUpgradeRecord()` and friends) are their
  own small sub-system, including reusing `generateNPC()` for the
  "Hired a Manager" candidate flow.

## Storage layer

Two `localStorage` keys, both plain JSON arrays:
- `wizardawn_saved_shops`
- `wizardawn_player_businesses`

(Left un-renamed to Moongear deliberately, to avoid orphaning anything
saved during testing — see prior note when the app was renamed.)

No server, no accounts, no sync across devices/browsers — by design, per
the earlier agreed tradeoff ("browser-local for now"). Export/Import
(JSON file download/upload) is the only way data moves between browsers
today.

## What this diverges from in the original plan, explicitly

| Planned | Actual (current prototype) |
|---|---|
| Node/Express API + React frontend | None — static HTML, all client-side |
| `content-engine` as a shared package | Inline helper functions, not packaged |
| `data/<domain>/*.json` files | Global JS arrays/objects in the same file |
| Database for user accounts/saves | `localStorage`, single browser |
| One page per generator | One page, two tabs, one generator built |
| Weighted/dice/conditional table engine features | Handled ad-hoc in plain JS per-feature |

None of this is a problem for a prototype — it let us validate content,
mechanics, and UX fast without standing up infrastructure first. But it
is real technical debt relative to the original plan, and worth keeping
in view: the more features get added to the single file (14 business
types and a full downtime economy already live in it), the more that
debt costs to pay down later. Worth deciding at some point whether to
keep extending this file or start the real migration.
