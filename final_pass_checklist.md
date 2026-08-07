# Final-Pass Checklist — Un-trim / De-placeholder

Everything below was intentionally shortened, approximated, or invented to get
a working preview fast. None of it is "wrong" as a demo, but none of it should
ship as-is. Grouped by what needs to happen to fix it.

**Status as of the latest review:** sections A, B, D, and E are now resolved
(races/hair, names, dress/possessions, and business naming all ported from
real source data — see below). C, F, G, H, I are still open. The full
original Wizardawn PHP source (146 files) is now available in
`wizardawn-source/`, which is what made A/B/D/E resolvable — most of what's
left in this doc is the same kind of "port it from source" work, not
blocked on anything from you except I's downtime-economy numbers (those are
new mechanics, not in the original app).

---

## A. NPC Engine — Races — ✅ resolved (except child NPCs)

- [x] **Race list**: full 35 races, each with its real height/weight
      multiplier pair from `GFTRace()`/`GFTStats()` in `city_gn.php`.
- [x] **Per-race hair rules**: correctly grouped per source (`HAIR_GROUPS`/
      `FIXED_HAIR` in the JS) — full-range races, no-beard elfish races,
      the four dark races with individually fixed colors/rules, orcish
      races, and the no-hair-system exotic races are all distinct groups
      matching `GFTCitizen()`'s branching in `city_gn.php:157-223`.
- [x] **Hair color palette**: full per-group palettes (not one flat 7-color
      list), matching source.
- [ ] **Child NPCs**: still not implemented. Original supports a `child` age
      state with halved height/weight, no beard/moustache/interests, and a
      restricted hair-style pool. Lower priority — not relevant for shop
      owners/managers specifically, only for a future "generate a
      townsperson" feature.

> **Now used in two places, not one:** the "Hired a Manager" candidate
> generator in Player-Owned Businesses calls the same `generateNPC()`, so
> every gap here shows up twice — once for shop owners, once for managers.
> That raises this section's priority.

## B. NPC Engine — Names — ✅ resolved

- [x] **Name banks**: ported from the real `names.php` (~19,500 entries
      across the 20 functions actually reachable from `generateNPC()`) —
      `humanMaleName()`, `humanFemaleName()`, `dwarfName()`, `gnomeName()`,
      `elfboyName()`, `elfgirlName()`, `elfName()`, `orcName()`,
      `goblinName()`, `catName()`, `wolfName()`, `lizardmanName()`,
      `ratmanName()`, `impName()`, `authorName()`, `speciesName()`,
      `mutantName()`, `genericName()`, `MaleName()`, `FemaleName()`. Data is
      embedded as a JSON script block (same pattern as `STORE_ITEMS`), not
      one giant `var` literal.
- [x] Race → name-generator mapping now replicates the real probability
      gates from `GFTCitizen()` (`city_gn.php:260-292`) — each race has its
      dedicated generator at the source's actual hit chance (75% for most,
      50% for daklafar/greyling), falling through to a gendered 50/50
      generic pool on a miss, instead of always using the dedicated pool.
      `MaleName()`/`FemaleName()`'s own internal 1-in-3 prefix+suffix-combine
      vs. flat-list split, and `wolfName()`'s compound-name logic, are both
      replicated too, not simplified to a flat pick.

## C. NPC Engine — Personality & Traits — ⬜ still open, unchanged

- [ ] **Trait6 (morality)**: I deduped to 12 unique entries; original is a
      20-entry array with intentional duplicates for weighting (e.g. "normal"
      appears 4x, "lusty" 4x) — needs the full weighted 20-entry list, not a
      flat unique list.
- [ ] **Interest/collector logic**: original uses one 20-entry array where
      `none` appears 3x and `collector` appears 4x (built-in weighting via
      duplication) among 12 real interests. I approximated this with flat
      `chance(85)` / `chance(20)` calls — close, but not the same distribution
      and not built the reusable way. Needs a real weighted-table mechanism
      (this is the "weighted/conditional tables" engine feature from the
      architecture doc — good first real use case for it).
- [ ] Similar duplicate-for-weighting pattern likely exists in other trait
      arrays I ported (trait1–trait5) — I ported those directly and they
      looked already-deduped in source, but worth a second pass to confirm
      none were silently flattened.

## D. NPC Engine — Dress & Possessions — ✅ resolved

- [x] **Clothing pools now preserve source's intentional duplicate
      weighting** (`PANTS_FULL`/`HCOLOR_FULL` match source's repeat counts —
      e.g. "long pants"/"short pants" 3x each, "light-gray" 2x).
- [x] **Hair/eye/garment color list**: full 26 entries, not trimmed.
- [x] **Possessions system**: full 6-slot probability chain ported —
      `THINGS` (135 entries), `JUNKY` (90), `WEAPON_TRINKETS` (10), `GEM1`/
      `GEM2` (33+33), each gated at its real per-slot chance
      (100/60/70/20/40/20%), matching `GFTCitizen()`'s `thing1`–`thing6`
      chain in `city_gn.php`.

## E. Business Naming — ✅ resolved for the 14 existing types

- [x] ~~Tavern/Inn names had trailing mood words duplicating the prefix
      adjective~~ — **fixed** (the "Sleeping Serpent Flagon Growling" bug).
- [x] **Verify source pathway**: confirmed `GFTBusiness()` in `city_gn.php`
      (lines ~424-770) exists independently of `wizardBusiness()`/
      `city_fantasy.php`, with its own word banks and per-type control
      flow. All 14 business types now ported against it directly, not
      `wizardBusiness()`.
- [x] **Word bank sizes untrimmed**: creature list 101/101, role list
      52/52, adjective list 33/33 (all shared `HUT2_CREATURE`/
      `HUT2_ROLE`/`HUT3`).
- [x] **Blacksmith naming confirmed real**, not invented — `GFTBusiness()`
      has its own genuine Blacksmith case.
- [x] **All 14 business types ported**, preserving GFTBusiness's real
      per-type control flow rather than one uniform template: most types
      override the shared adjective bank 70% of the time with a
      type-specific one (`HUT3_OVERRIDES`); Bowyer/Stables instead
      override the shared creature/role bank (Bowyer 70%, Stables
      unconditionally); Church is structurally different — hut1/hut2/hut3
      all unconditionally overridden, hut1 rendered as "of X" rather than
      a trailing noun; Inn gets a literal `(Nsp per night)` price suffix.
      Leatherworker has no override at all — pure shared banks, as source.
      Verified with a Node harness across all 14 types, no errors.
- [ ] **Not yet in scope, deliberately**: Jeweler, Bank, Guardhouse, and
      guild halls exist in `GFTBusiness()` source (`type==13`, `type==16`,
      `type>16`) but have no UI option or `STORE_LOOKUP`/`TYPE_WORDS`
      entry in the app at all. New business types, not a fidelity gap in
      the 14 that exist.

## F. Description Generator (exterior/ambiance) — ⬜ still open, unchanged

- [ ] **Store-type coverage**: only Tavern/Inn/Blacksmith have ambiance word
      banks. Needs the same treatment for every business type in section E's
      list — a Church shouldn't read like a Tavern.
- [ ] **Word bank depth**: 4–7 entries per category (material, condition,
      roof, feature, sound, smell, etc.) — fine for a demo, will repeat
      noticeably at real usage volume. Worth 2–3x'ing each list.

## G. Inventory

- [x] ~~Only `game='DD'` is wired up~~ — **resolved.** A Setting/Ruleset
      selector now threads a real game code (`DD`/`TT`/`BU`/`MF`/`PA`)
      through `pickInventory()`/`fullCatalogPool()` (both switched from
      exact `game===code` matching to substring `gameMatches()`, mirroring
      source's `game LIKE '%CODE%'`, since post-apoc rows carry combo tags
      like `MFxPA`/`MFxBUxPA`). The item-suggestion dropdown picks this up
      for free since it calls `fullCatalogPool()` with the same `shop.game`.
      Type-list gating is per-ruleset, not just per-genre: Tunnels & Trolls
      only offers the 7 store types with real TT rows; post-apoc's Mechanic
      and Robot types only appear under Broken Urthe, since the catalog has
      zero MF/PA rows for either (checked directly against store-items.csv).
      Still open: `nuclearCitizen()` (post-apoc's separate citizen
      generator with mutations/tech level) isn't ported, so post-apoc shop
      owners use the fantasy NPC generator as a placeholder; post-apoc
      exterior/ambiance is one generic invented template for all 9 types
      rather than fantasy's per-type banks; the era/tier column
      (`Simple`/`Complex`/`Deluxe` for TT, `hi`/`low`/`both` for post-apoc)
      still isn't used to filter stock, so TT/post-apoc shops can generate
      tech-tier-inappropriate items — see the tier/quality bullet below,
      which now also applies to the new rulesets, not just fantasy.
- [ ] **Stock % is a made-up flat random range (65–95%)**, not derived from
      anything. Original likely ties `$stock` to settlement size/wealth —
      need to find where `$stock` actually gets set in the calling code
      (probably in `tool_ftown.php`/`tool_wtown.php`) and replicate that.
- [ ] **Tier/quality column (`Simple`/`Complex`/`Deluxe`/`hi`/`low`/`both`) is
      completely unused** — need to determine what it's supposed to gate
      (settlement wealth tier? shop quality?) and wire it in.
- [ ] **Category and stat-block columns are ignored** — Blacksmith weapon/armor
      items have a `Weapon-Sword`/`Armor-Torso`-style category tag, and
      vehicle/robot items have `Hits:X/Str:Y` stats, neither of which surface
      in the current inventory display.

## H. Not Yet In Scope

- [x] ~~Menus/navigation isn't wired in at all~~ — **resolved**, using the
      real `menus.csv` export rather than guesswork. It confirmed source
      has three separate Fantasy Settlements generators per ruleset
      (`tool_wtown.php`: OSRIC/AD&D 1979; `tool_ftown.php`: Labyrinth
      Lord/D&D 1981/Basic Fantasy/Swords & Wizardry/generic Fantasy/Swords
      & Six-Siders; `tool_ttown.php`: Tunnels & Trolls 5e/7e/Deluxe), but
      `GFTBusiness()`/`wizardBusiness()` both hardcode `game='DD'`
      regardless of which reached them — so the app's "D&D-Style" ruleset
      correctly covers all of `tool_wtown.php` + `tool_ftown.php`, and only
      real Tunnels & Trolls needed its own option. Post-apoc mirrors
      `tool_mtown.php`/`PABusiness()`: Mutant Future and Broken Urthe each
      get a dedicated branch (word banks, creature lists, and — for
      Stables — an unconditional hut2 override, all ported from
      `city_post_apocalyptic.php`); everything else in menus.csv's "Other"
      Game Choice category (Gamma World, Metamorphosis Alpha, Millenniums &
      Mutations, Necropalyx, Space Ryft) falls into the generic PA branch,
      matching source — none of those rulesets has its own settlement
      generator either. The site-navigation/link parts of menus.csv
      (Links, Supplements, Data Files, Free Game rows) have no equivalent
      in this single-page app and weren't ported — only the Game Choice
      gating structure was relevant here.
- [ ] **Single building only** — the real goal (per your stated scope) is
      whole settlements with multiple buildings tied together by
      `city_size`/`economy`, which this generator doesn't attempt yet.

## I. Player-Owned Business System — new since the original checklist

Not part of the original app at all, so nothing to "un-trim" against source —
but it introduced its own placeholders and open assumptions worth tracking
the same way:

- [ ] **`MAGIC_SELL_TABLE`** (days-to-sell and %-of-value by rarity for
      selling magic items) is still an invented placeholder — you gave me
      the real Results Table and Complications list, but not this one.
- [ ] **`COMPLICATION_TRIGGER`** (currently: natural roll ≤5 causes a
      complication check) is my assumption, not a confirmed rule.
- [ ] **`dailyMaintenance` default (5 gp/day)** is a placeholder per
      business — needs real per-building-type maintenance costs.
- [ ] **Upgrade bonuses stack with no cap.** Flagged earlier, still an open
      design question: cap them, taper them, or leave uncapped?
- [ ] Complication *effects* aren't mechanized (by design, for now) — each
      one just logs a narrative flag for you to adjudicate. Worth revisiting
      once/if you want them to have automatic gp or inventory effects.

---

## What I'd actually prioritize for *this* generator specifically

**Update:** A, B, D, and E (races/hair, names, possessions, business naming)
are now done — see above. The 14 business types and the Settlements tab
(multi-building generation) also already shipped, ahead of where this
section originally assumed. Re-prioritizing what's left:

1. **F (description word bank depth + coverage)** — the source
   (`descriptions.php`, `atmosphere.php`) is available; same "only 3 types
   have real ambiance banks" gap E just got fixed for naming.
2. **G's stock%/tier fidelity** — lower urgency; the current random range
   works fine for play, just isn't "correct."
3. **I's placeholders** — only urgent if you plan to actually run the
   downtime economy at the table soon; otherwise fine to leave as-is.

C (trait weighting fidelity) and H (settlement scaling — note: the
Settlements tab already exists, so H is really just "no multi-settlement
economy/city-size linkage beyond building counts" now, not "single building
only") I'd still leave for last.

