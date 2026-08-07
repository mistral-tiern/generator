# Final-Pass Checklist — Un-trim / De-placeholder

Everything below was intentionally shortened, approximated, or invented to get
a working preview fast. None of it is "wrong" as a demo, but none of it should
ship as-is. Grouped by what needs to happen to fix it.

**Status as of the latest review:** sections A, B, and D are now resolved
(races/hair, names, and dress/possessions all ported from real source data —
see below). C, E, F, G, H, I are still open. The full original Wizardawn PHP
source (146 files) is now available in `wizardawn-source/`, which is what
made A/B/D resolvable — most of what's left in this doc is the same kind of
"port it from source" work, not blocked on anything from you except I's
downtime-economy numbers (those are new mechanics, not in the original app).

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

## E. Business Naming — ⬜ still open, one sub-item fixed

- [x] ~~Tavern/Inn names had trailing mood words duplicating the prefix
      adjective~~ — **fixed** (the "Sleeping Serpent Flagon Growling" bug).
- [ ] **Verify source pathway**: the naming logic I ported (`HUT2_CREATURE`,
      `HUT2_ROLE`, `HUT3`, tavern/inn word banks) came from `wizardBusiness()`
      in `city_fantasy.php` (the OSRIC pathway), not `GFTBusiness()` in
      `city_gn.php` (the pathway the NPC generator is actually using). I
      never fully read `GFTBusiness()`'s body past its signature — need to
      check whether it has its own separate word banks before finalizing,
      since right now NPCs and business names are technically from two
      different rule sets stitched together.
- [ ] **Word bank sizes trimmed**: creature list 24/98, role list 24/52,
      generic adjective list 26/33.
- [ ] **Blacksmith naming was invented**, not ported — I never located a
      confirmed Blacksmith case in either `wizardBusiness()` or
      `GFTBusiness()`'s source. Needs verification against source or an
      explicit decision to keep it as new content.
- [ ] **Only 3 of ~15+ business types built**: Tavern, Inn, Blacksmith done.
      Still needed: Provisioner, Bowyer, Leatherworker, Tailor, Stables,
      Carpenter, Alchemist, Baker, Library, Jeweler, Music, Church, Bank,
      Guardhouse, and guild halls (Fighters/Wizards/Thieves/Rangers/Assassins/Bards).

## F. Description Generator (exterior/ambiance) — ⬜ still open, unchanged

- [ ] **Store-type coverage**: only Tavern/Inn/Blacksmith have ambiance word
      banks. Needs the same treatment for every business type in section E's
      list — a Church shouldn't read like a Tavern.
- [ ] **Word bank depth**: 4–7 entries per category (material, condition,
      roof, feature, sound, smell, etc.) — fine for a demo, will repeat
      noticeably at real usage volume. Worth 2–3x'ing each list.

## G. Inventory — ⬜ still open, unchanged (new features built around it, not into it)

- [ ] **Only `game='DD'` is wired up.** No support yet for TT (Tunnels &
      Trolls), or the sci-fi/post-apoc codes (MF/PA/BU) — needed once we're
      not just targeting the default fantasy ruleset. This now also affects
      the item-suggestion dropdown, which is DD-only for the same reason.
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

## H. Not Yet In Scope — ⬜ still open, unchanged

- [ ] **Menus/navigation isn't wired in at all.** `menus.csv` defines which
      game systems each generator applies to; the generator always assumes
      the default fantasy ruleset regardless.
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

**Update:** A, B, and D (races/hair, names, possessions) are now done — see
above. The 14 business types and the Settlements tab (multi-building
generation) also already shipped, ahead of where this section originally
assumed. Re-prioritizing what's left:

1. **E's remaining sub-items** — the naming/description word banks for the
   11 non-original (Tavern/Inn/Blacksmith) business types are still original
   content, not verified against `GFTBusiness()` in `city_gn.php` (which we
   now know exists independently with its own banks, including a real,
   non-invented Blacksmith case — confirmed this session). Porting those 11
   against source is the next highest-leverage item.
2. **F (description word bank depth + coverage)** — same source now
   available (`descriptions.php`, `atmosphere.php`), same "only 3 types
   have real ambiance banks" gap.
3. **G's stock%/tier fidelity** — lower urgency; the current random range
   works fine for play, just isn't "correct."
4. **I's placeholders** — only urgent if you plan to actually run the
   downtime economy at the table soon; otherwise fine to leave as-is.

C (trait weighting fidelity) and H (settlement scaling — note: the
Settlements tab already exists, so H is really just "no multi-settlement
economy/city-size linkage beyond building counts" now, not "single building
only") I'd still leave for last.

