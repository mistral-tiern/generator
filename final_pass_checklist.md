# Final-Pass Checklist — Un-trim / De-placeholder

Everything below was intentionally shortened, approximated, or invented to get
a working preview fast. None of it is "wrong" as a demo, but none of it should
ship as-is. Grouped by what needs to happen to fix it.

**Status as of the latest review:** two items are resolved. Everything else
on the original list is still open — the work since then (save/load, print,
import/export, item suggestions, business value pricing, the Player-Owned
Businesses tracker) added real features but didn't touch NPC/business/
inventory fidelity. A new section (I) below covers placeholders introduced
by that work.

---

## A. NPC Engine — Races — ⬜ still open, unchanged

- [ ] **Race list**: expand from 8 → full 35 (`GFTRace()` in `city_gn.php`).
      Each needs its height/weight multiplier pair (already extracted from
      source, just needs entering).
- [ ] **Per-race hair rules**: currently every race gets a flat 30% beard/
      30% moustache chance. The original varies this by race group:
      - Human/dwarf/gnome/hobbit/leprechaun/ogre/centaur/cyclops: full range (beard, moustache, or bald possible)
      - Elf/brownie/fairy/pixie/satyr/sprite: no beard ever, moustache possible
      - Dark-elf/dwurman/greyling/suvart: each has its own fixed hair-color + beard rule (not random)
      - Orc/goblin/hobgoblin/troll: black/white hair only, beard+moustache both common
      - Everything else (exotic races — centaur, minotaur, etc.): no hair system at all (`do_not_config_hair`)
- [ ] **Hair color palette**: trimmed to 7 generic colors; original varies the
      palette by race group (see above — some races have fixed single colors,
      not a random pool).
- [ ] **Child NPCs**: not implemented at all. Original supports a `child` age
      state with halved height/weight, no beard/moustache/interests, and a
      restricted hair-style pool.

> **Now used in two places, not one:** the "Hired a Manager" candidate
> generator in Player-Owned Businesses calls the same `generateNPC()`, so
> every gap here shows up twice — once for shop owners, once for managers.
> That raises this section's priority.

## B. NPC Engine — Names — ⬜ still open, unchanged

- [ ] **Name banks**: currently ~8 hand-picked placeholder names per race.
      Needs the real `names.php` data — thousands of entries, split into
      dedicated generators per race/gender: `humanMaleName()`, `humanFemaleName()`,
      `dwarfName()`, `gnomeName()`, `elfboyName()`, `elfgirlName()`,
      `orcName()`, `goblinName()`, `catName()`, `wolfName()`, `lizardmanName()`,
      `ratmanName()`, `impName()`, `authorName()`, `speciesName()`,
      `mutantName()`, `genericName()`, plus generic `MaleName()`/`FemaleName()`
      fallbacks for races without a dedicated pool.
- [ ] Race → name-generator mapping also has probability gates in the original
      (e.g. "75% chance of a dedicated race name, else generic") — currently
      my version always uses the dedicated pool with no fallback mixing.

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

## D. NPC Engine — Dress & Possessions — ⬜ still open, unchanged

- [ ] **Clothing pools have intentional duplicate weighting in source** (e.g.
      original `PANTS` array is `kilt, long pants, short pants, long pants,
      short pants, long pants, short pants, skirt` — "long/short pants" are
      each listed 3x to make them common, "kilt"/"skirt" rare). I deduped
      these to unique lists, which flattens the weighting. Needs restoring
      per-item repeat counts (or converting to real weights).
- [ ] **Hair/eye/garment color list**: trimmed to 21 of the original 26.
- [ ] **Possessions system is a simplified stand-in for a 6-slot probability
      chain** (`thing1`–`thing6` in source), each gated at a different chance
      (100%, 60%, 70%, 20%, 40%, 20%) and each pulling from different pools:
      - `thing1`/`thing2`: general trinkets (130-item `things` list) — I have ~13
      - `thing3`: "junk" pool (90-item `junky` list) — not implemented separately
      - `thing4`: **weapons** (10-item concealed-weapon list) — not implemented
      - `thing5`: coin purse (implemented, roughly)
      - `thing6`: **gems/jewelry for wealthy NPCs** (32+32 item gem lists,
        100–500gp each) — not implemented at all
      Final pass needs the full `things`/`junky`/weapon/gem lists and the
      original's per-slot probability gates, not my flattened 2-trinket version.

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

Not every open item matters equally for finishing the shop/building
generator on its own (some, like H's settlement scaling, are really about
future generators). Narrowing to just what would make *this* one solid:

1. **A + D (races, hair rules, possessions fidelity)** — highest leverage,
   since it now feeds both shop owners *and* hired managers.
2. **E + F together (business types + their descriptions)** — the generator
   currently only makes 3 kinds of building. Getting to a fuller set (even
   just 6–8 more common ones) is probably the highest-value expansion for
   actual use at the table.
3. **G's stock%/tier fidelity** — lower urgency; the current random range
   works fine for play, just isn't "correct."
4. **B (full names.php)** — mechanical, big, but low-risk; good candidate to
   batch-process whenever you're ready to hand over the file.
5. **I's placeholders** — only urgent if you plan to actually run the
   downtime economy at the table soon; otherwise fine to leave as-is.

C (trait weighting fidelity) and H (settlement scaling) I'd genuinely leave
for last — they're real gaps, but neither blocks the generator from being
useful today.

