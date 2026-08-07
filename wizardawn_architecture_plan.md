# Wizardawn Rebuild — Architecture Plan

## Core idea

Every "generator" in the original app — NPCs, shops, monsters, loot, dungeon rooms,
towns, spells — does the same fundamental thing: pick from word/phrase banks and
combine them into text (and sometimes numbers). The original code does this with
hand-written PHP string concatenation, a different ad-hoc style in every file.

We replace all of it with one shared **content engine** (the token/pattern grammar
technique, same idea as donjon's `generator.js`) plus **data files** per domain.
One engine, ~15 data domains, ~90 generator pages that mostly become thin
wrappers calling into that data.

---

## 1. The content engine (core library)

The prototype engine (static tables, `{token}` substitution) needs a few
extensions to cover everything the original app does:

| Original PHP pattern | Engine feature needed |
|---|---|
| `$trait2[mt_rand(0,23)]` | already have — flat token tables |
| `if (mt_rand(1,100) > 70){$beard=1;}` | **Weighted/conditional tables** — entries with probability weights, or "include this token X% of the time" |
| `$hp = $hd * mt_rand($might1,$might2*$might1)` | **Dice/range expressions** embedded in patterns, e.g. `{roll:1d6+2}` |
| Race picked once, then reused to filter hair/name pools later | **Bound variables** — pick a token once, store it in context, reference it again later in the same generation (the prototype engine is stateless; this needs a context object) |
| `if ($game == "Labyrinth Lord"){ $armor = ... } else { ... }` | **Ruleset-conditional tables** — same token name resolves differently depending on a context flag (game system) |
| `SELECT * FROM store_items WHERE store='Tavern' AND game='DD'` | **Queried/dynamic tables** — a table backed by your CSV/JSON data, filtered at generation time, not hand-written prose |
| Original always returns one flat HTML string | **Structured output** — generator functions return an object (`{name, stats, description, inventory}`), not a single blob, so the frontend can lay it out properly instead of dumping raw HTML |

This becomes one small TypeScript/JS package (`content-engine`) that every
generator imports. It's the single highest-leverage piece of the whole rebuild —
get it right once, every domain benefits.

## 2. Data domains

Grouping the ~90 original files by what they actually generate:

1. **NPCs / Characters** — `city_fantasy.php`, `city_gn.php`, `city_tt.php`,
   `city_post_apocalyptic.php`, `characters_osric.php`, `stat_adventurer.php`,
   `tool_bxadvg.php`, `tool_oadvg.php`, `tool_ladvg.php`, `tool_bmbs.php`
   — appearance, dress, stats, personality. This is the biggest shared engine
   since almost every other domain (shops, guards, guilds) calls into it.
2. **Businesses / Settlements** — `fantasy_stores_*.php`, `tool_ftown/mtown/ttown/wtown/suburb/villg/acity.php`,
   `furnish.php`, `rooms_fantasy.php` — shop naming, building layout, our new
   exterior/ambiance layer.
3. **Items / Equipment** — `item_fantasy*.php`, `magic_item_*.php`,
   `weapon_equipped.php`, `tt_arms_and_armor.php`, `containers.php`,
   `tool_packs/loot/alchemist/potions/coins.php`, `money.php`, `potion_appearances.php`
   — also where `store_items.csv` plugs in as a queried table.
4. **Monsters / Encounters** — `monster.php`, `monsters.php`, `monsters_tt.php`,
   `monsters_add.php`, `tool_encounter.php`, `encounters.php`
5. **Dungeons / Maps** — `tool_delve.php`, `tool_dmap/smap/tmap.php`,
   `map_draw*.php`, `geomorphs_draw.php`, `hex_map.php`, `tool_hexcrawl.php`,
   `tool_locale.php`, `tool_door.php`, `tool_scavenge.php`, `tool_wander.php`,
   `tool_uruins/uexfl.php` — mostly procedural layout + room content, less prose-heavy.
6. **Spells / Magic** — `spells.php`, `tool_llb_spells.php`, `tool_osric_spells.php`,
   `magic_item_properties.php`, `curses.php`
7. **Flavor / Shared vocabulary** — `names.php`, `descriptions.php`,
   `atmosphere.php`, `colors.php`, `retro_info.php` — cross-cutting token tables
   used by everything above (this is where our new storefront/ambiance tables live too).
8. **Pure utility tools** — `tool_dice.php`, `tool_coins.php` (roll math),
   `tool_area.php`, `tool_data.php`, `tool_books.php` — not content generators
   at all, just calculators/formatters. No content engine needed; straightforward
   ports.
9. **Static reference pages** — all ~25 `rpg_*.php` files — confirmed these are
   just blurb + external-link pages per game system, no generation logic.
   Become simple static routes/content, not part of the engine at all.

## 3. Project structure

```
wizardawn/
  packages/
    content-engine/        # the core token/pattern engine (shared library)
    data/
      npc/                 # token tables: appearance, dress, personality, names
      settlements/         # shop naming, exteriors, ambiance, building layout
      items/               # equipment, magic items, loot tables
      monsters/
      dungeons/
      spells/
      flavor/              # shared vocab: colors, materials, descriptions
  apps/
    api/                    # Node/Express — one endpoint per generator
    web/                    # React frontend — one page per generator
  content-imports/
    store_items.csv         # your real exported data, loaded at build/runtime
    menus.csv
```

## 4. Migration order (proposed)

1. **Content engine core** — build the extended engine (weights, dice, bound
   variables, conditional/queried tables) as its own tested package.
2. **Flavor/vocabulary domain** — port `names.php`, `colors.php`, `descriptions.php`,
   `atmosphere.php` first since everything else references them.
3. **NPC engine** — port `dressMeup`/`wizardCitizen`/`GFTCitizen`/`GFTStats` into
   the new format. This unblocks the most other domains (shops, guards, guilds
   all just call the NPC engine).
4. **Settlements/shops** — finish what we started: business naming + real
   inventory (`store_items.csv`) + the exterior/ambiance layer, now composed
   with the ported NPC engine. This becomes the first fully working end-to-end
   generator page.
5. **Items, monsters, spells, dungeons** — same treatment, roughly in order of
   how often they're referenced by other domains.
6. **Utility tools + static rule pages** — mechanical, low-risk, can be done in
   parallel by whoever/whenever since they don't depend on the content engine.

## 5. What I still need from you, as we get to each domain

Same pattern as `store_items.csv`: anywhere the original app pulled from a
live MySQL table instead of a PHP array, I'll need an export when we reach
that domain. So far that's `store_items` (done) and `menus` (still open).
I'll flag any others as I hit them file-by-file.

---

**Next step:** build the extended content engine core, then port the NPC
domain — since that unblocks the most other work and we already have the
source logic fully mapped out from `city_fantasy.php`/`city_gn.php`.
