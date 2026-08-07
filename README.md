# Moongear

A shop/building generator and player-owned business tracker, built as a
spiritual successor to **Wizardawn**, a personal RPG toolset the original
author built over ~10 years and made available for free. This project is
a from-scratch rebuild — reusing the original's *ideas* and, where
possible, its actual data — not a port of its code.

Current status: **early prototype.** One generator (shops/buildings) is
functional end-to-end; the rest of the originally-envisioned app (NPCs,
monsters, dungeons, towns, spells, etc. as standalone generators) hasn't
been started. See "What's left before shipping" below.

---

## What this actually is right now

A single self-contained HTML file — `shop_generator_v2-4.html`. No install,
no server, no build step. Double-click it and it opens in your browser.
All the app's data (a real ~1,900-row item catalog exported from the
original Wizardawn database) is embedded directly in the file, so it
works fully offline.

This is deliberately **not** the final architecture. It's a fast way to
validate the content and mechanics before committing to the real stack
(a proper backend + database + multi-page frontend, as laid out in the
architecture plan). Everything built here — the NPC engine, the business
naming, the inventory system — is meant to port over conceptually once
that rebuild starts; it's not throwaway.

## Features

**Shop Generator tab**
- Generates a building across 14 business types (Tavern, Inn, Blacksmith,
  Alchemist, Baker, Bowyer, Carpenter, Church, Leatherworker, Library,
  Music Shop, Provisioner, Stables, Tailor), each backed by real inventory
  data — not placeholder items.
- Full NPC generation for the owner: 35 races (with per-race hair rules,
  fixed hair colors for the "dark" races, and race-appropriate
  dress/possessions), personality, appearance, and a 6-slot possessions
  system including a chance at a concealed weapon or gems/jewelry.
- Procedurally generated exterior and interior/ambiance descriptions,
  tailored per business type.
- A computed **Business Value** (base building value + a multiple of
  actual inventory worth — edit the inventory and it updates live).
- Fully editable inventory: adjust price/quantity on generated items,
  remove them, or add your own — with autocomplete suggestions pulled
  from that business type's full catalog as you type.
- Save/Load, Print (browser print dialog, clean layout), and Export/Import
  (JSON file) for saved shops.

**Settlements tab**
- Enter a name + population; generates how many of each of the 14 business
  types the settlement supports, using the Medieval Demographics Made Easy
  support-value formula (population ÷ SV, with the fractional remainder as
  a % chance of one more).
- Each building is a fully real, independently generated shop (owner NPC,
  name, description, inventory) — not a stub or count-only placeholder.
- Persists in localStorage; clicking a building jumps to the Shop Generator
  tab with it loaded, so Save/Buy/print/export all keep working unchanged.
  Deleting a settlement cascades to its member buildings.

**Player-Owned Businesses tab**
- Convert any saved shop into a player-owned business ("Buy").
- Monthly downtime resolution: pay maintenance upfront, roll d100 + days
  spent tending, resolve against a results table (loss / breakeven /
  profit tiers with scaling dice), with occasional complications.
- A second "Advance 30 Days" action for when nobody's personally tending
  it — relies only on active Upgrades.
- **Upgrades**: hire an NPC manager (re-rollable via the same NPC engine
  used for shop owners) or mark a PC as managing it themselves, plus
  Exotic Stock / Trophy&Decor / Hired Staff / Custom upgrades, each with
  a DM-set bonus to the monthly roll.
- Selling magic items: list any inventory item at a rarity, and it
  resolves automatically after a rolled number of days at a
  rarity-based percentage of its value.
- Full running ledger per business.

**Data & storage**
- Currency is cp/sp/gp/pp only — electrum (ep) was removed from the
  source data and converted to sp at the standard 1ep = 5sp rate.
- Saves are stored in browser `localStorage` — local to one browser on
  one device, no account system yet.
- Mobile-responsive (viewport meta tag, restacking inventory cards,
  16px+ inputs to avoid iOS auto-zoom).

## Running it

Download `shop_generator_v2-4.html`, double-click it. That's it. Works in
any modern browser (Chrome, Edge, Firefox; Safari should work but has
historically had quirks with the native autocomplete dropdown used for
item suggestions).

## Project files

- `shop_generator_v2-4.html` — the actual app.
- `wizardawn_architecture_plan.md` — the intended full-scale architecture
  (content-engine + data domains + API/frontend split) once this moves
  past prototype stage.
- `final_pass_checklist.md` — a running, checked-off list of every
  placeholder/simplification made along the way, with more granular
  detail than the summary below.

---

## What's left before shipping

### Blocking / high-value
- [x] ~~Real NPC name banks~~ — **resolved.** Ported the real `names.php`
  data (~19,500 entries across the 20 race/gender-specific generator
  functions `generateNPC()` actually reaches), including the source's real
  probability gates (dedicated pool at its real hit chance, else a generic
  gendered fallback) instead of always using the dedicated pool. Affects
  both shop owners and hired managers, since both call `generateNPC()`.
- [ ] **Business-naming word banks are unverified against source for 11 of
  14 business types.** Confirmed this session that `GFTBusiness()` in
  `city_gn.php` *does* exist independently with its own word banks
  (not shared with `wizardBusiness()`/`city_fantasy.php`), and that its
  Blacksmith case is real, not invented — but Tavern/Inn/Blacksmith are
  the only 3 types actually ported against it. The other 11 business
  types' naming/description banks are still original content, not
  checked against `GFTBusiness()` at all. Word bank sizes for the ported
  3 are also trimmed from source (creature list 24/98, role list 24/52,
  adjective list 26/33).
- [ ] **Only `game='DD'` inventory is wired up.** No Tunnels & Trolls or
  sci-fi/post-apocalyptic support yet, despite that data existing in the
  source catalog.
- [ ] **`menus.csv` (site navigation / game-system gating) isn't wired
  into anything.** The generator always assumes the default fantasy
  ruleset.

### Real gaps, lower urgency
- [ ] Trait-weighting exactness (several personality trait arrays use
  intentional duplicate entries in source for weighting — some were
  ported as deduped unique lists instead, flattening the distribution).
- [ ] Child NPCs not implemented (source supports a distinct child state;
  not relevant for shop owners specifically, but relevant for any future
  "generate a townsperson" feature).
- [ ] Inventory stock % is an arbitrary flat random range (65–95%), not
  tied to settlement size/wealth like the original likely does.
- [ ] The item catalog's tier/quality column (`Simple`/`Complex`/`Deluxe`
  etc.) and category/stat-block columns (weapon type, vehicle stats) are
  present in the data but completely unused.

### Player-Owned Business placeholders (need your input, not more coding)
- [ ] `MAGIC_SELL_TABLE` (days-to-sell / %-of-value by rarity for selling
  magic items) is an invented placeholder — the monthly Results Table
  and Complications list use your real numbers, this table doesn't.
- [ ] `COMPLICATION_TRIGGER` (currently: natural roll ≤5) is an
  assumption about when a complication check fires, not a confirmed rule.
- [ ] Default `dailyMaintenance` (5 gp/day) is a placeholder per business
  — needs real per-building-type maintenance costs.
- [ ] Upgrade bonuses stack with no cap — open design question (cap it,
  taper it, or leave it uncapped as an earned reward).

### Bigger than "finish this generator" — future scope
- [x] ~~Everything is a single building~~ — a Settlements tab now exists:
  enter a name + population, and it generates the right number of each
  business type via the Medieval Demographics Made Easy support-value
  formula, with each building a fully real generated shop. What's still
  open is deeper city-size/economy linkage beyond building counts.
- [ ] No accounts / server-side storage — by design for now (agreed
  tradeoff to ship something usable immediately), but real deployment
  ("on the web, for others to use") will need this.
- [ ] Every other originally-planned generator (NPCs standalone, monsters,
  items/loot, spells, dungeons/maps) hasn't been started — this project
  currently covers exactly one of ~9 planned content domains.
