# SWSH Encounter Taxonomy and Hook Boundaries

Updated: 2026-09-18. Sword 1.3.2 addresses below were verified against the real `main(3).elf` (GNU Build ID `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`).

## Common architecture

```text
Encounter/source-specific selection + outer RNG
        -> species/form/level + policies
        -> GenerationSpec
        -> shared Pokémon generation
        -> final Pokémon
```

There is no single global RNG stream for Field, Egg, Zone, Raid, scripted fixed encounters and gifts.

| Type | Sword 1.3.2 chain | RNG / policy owner | Preferred hook |
|---|---|---|---|
| ordinary Field / hidden | `D31BB0 -> D328A0/D329D0 -> D311C0 -> D31910 -> 7662A0 -> 777F40` | shared Field context (`+0x98` in owning object) | slot at `D328A0`; spec at `D31910/7662A0` |
| Fishing | `DA4720/DA5550` controller/resource; runtime `DAF350 -> D05080 -> D31BB0 -> ...` | Field outer RNG after fishing-source transition | source guard `D05080` + common spec hook |
| Symbol | resources at `EA24B0`, `E51380`, `F14210`; runtime managers `1484780`, `14B2F50`; common generation sink confirmed but exact indirect bridge open | Field/symbol context; exact bridge still OPEN | symbol source telemetry + guarded `7662A0` |
| Script fixed / event | `event_encount_data.bin -> E4A470 -> E4AA30 -> E4A570 -> 7662A0 -> 777F40` | event record + keyed/script context | `E4AA30` record-to-spec mapping; guarded `7662A0` |
| AddPoke / scripted Gift | `add_poke.bin -> E48D70 -> E48A98 -> 7662A0 -> 777F40` | gift/script record | `E48D70` spec builder / `E48A98` callsite |
| NPC Trade | `field_trade.bin -> E4D5D0 -> 7662A0 -> 777F40` | trade record | trade-record parser + `E4D5D0` |
| Egg | `14018C0 -> 7893F0 -> 789500 -> 789770 -> 7896F0 -> 777F40` | breeding state `25A83A8`; PML Crypto later for unresolved PID/size | breeding state or post-`789770` spec at `789668` |
| Zone/Around | `1629E30 -> 162AD70 -> 162B07C -> 777F40` | local Zone xoroshiro state | state after `F28FC0`; spec before `162B07C` |
| Max Raid / G-Nest | high-confidence caller cluster `EC7CA8 / EC877C / EC950C -> 7662A0 -> 777F40`; secondary path near `10444B0` | raid/nest record + raid-specific seed/policies; exact seed mapping still OPEN | split `EC7xxx..EC9xxx` builders before common spec hook |
| Dynamax Adventures | `15DDxxx -> 15DD6E0 -> 7662A0 -> 777F40` | Underground Exploration route/node/boss context | builder around `15DDxxx`; callsite `15DD6E0` |
| Battle Tower / Rental | caller region near `E4670C -> 7662A0` | rental/trainer records | source-specific builder |
| Trainer/Battle NPC | caller region near `1521EF4 -> 7662A0` | trainer data | source-specific builder |
| Camp NPC Pokémon | caller region near `1235034 -> 7662A0` | camp NPC data | source-specific builder |
| Mystery Gift | subsystem near `FE3BDC`; direct `7662A0` edge not yet proven | likely Wonder Card/downloaded payload | OPEN: trace deserialization vs GenerationSpec path |

## Shared sink callers

```text
7662B4 -> 777F40 : generic GenerationSpec path
7896F0 -> 777F40 : Egg
162B07C -> 777F40: Zone/Around
```

The generic `7662A0 -> 7662B4 -> 777F40` path is shared by Field, Script/Event, Raid, Gift, Trade and several battle/NPC systems. Therefore `7662A0` must not be named or hooked as if it were wild-only.

This LR split is the recommended safety classifier at the final shared boundary.

## GenerationSpec confirmed/high-confidence fields

```text
+0x00 fixedSeed
+0x08 useFixedSeed
+0x10 EC policy (probable/high)
+0x18 PID/shiny policy (high)
+0x20 trainer/shiny context (high)
+0x28 species
+0x2C form
+0x2E level
+0x30 gender policy (high confidence)
+0x32 nature policy (high confidence)
+0x34 ability policy (high confidence)
+0x35 shiny rolls/policy count (high confidence)
+0x36..+0x40 six IV sentinels/values
+0x44 item policy (probable/high)
+0x48 perfect-IV count
+0x4A size policy A / WeightParam in Egg path (high)
+0x4C size policy B / HeightParam in Egg path (high)
```

The local fixed stream uses `(fixedSeed, 0x82A2B175229D6A5B)`.

## Source-specific research boundaries

### Ordinary Field

```text
D328A0   weighted slot RNG
D31910   Field GenerationSpec builder
D310D8   callsite before 7662A0
```

### Scripted fixed/event

```text
E4AA30   event-record -> GenerationSpec
E4A570   callsite to 7662A0
```

This is currently the best target for mapping explicit shiny-lock, gender, nature, ability and IV policies from event data into the shared spec.

### Raid / G-Nest

```text
EC7CA8
EC877C
EC950C
```

form a high-confidence raid/nest generic-generation caller cluster. Keep exact per-function names provisional until runtime or full record-to-spec mapping separates their roles.

### Dynamax Adventures

```text
15DDxxx  Underground Exploration spec construction
15DD6E0  callsite to 7662A0
```

### Gift/AddPoke

```text
E48D70   gift-specific spec population
E48A98   callsite to 7662A0
```

### Zone/Around

```text
1629E30  local RNG setup
162AD70  Zone object/spec build
162B07C  direct call to 777F40
```

### Egg

```text
789770   egg-specific deterministic generation
789668   post-egg-spec boundary
7896F0   direct call to 777F40
```

## Seed Searcher rule

Do not collapse all searching into one fixed seed.

- `FixedSpecSearcher` searches the local inner generation after the `GenerationSpec` policy is fixed.
- `EncounterStateSearcher` searches the outer Field subsystem RNG and is required when natural slot/level/policy/mark behavior must be preserved.
- Egg needs its own breeding-state searcher because it owns `25A83A8`, not the Field stream.
- Zone needs its own local-state searcher because its RNG is initialized under `1629E30`.
- Raid/Nest and Dynamax Adventures require their own upstream seed/policy analysis before they can be faithfully searched; merely forcing `GenerationSpec.fixedSeed` reproduces only the inner generator, not necessarily the natural source selection.

## Current OPEN items

1. Exact indirect Symbol manager -> common Field generation callback.
2. Complete `E4AA30` event-record field map, especially shiny lock/force semantics.
3. Raid/Nest seed ownership and transformation into `GenerationSpec.fixedSeed`.
4. Exact individual roles of `EC7CA8`, `EC877C`, `EC950C` and secondary `10444B0` G-Nest path.
5. Full Dynamax Adventure route/node/boss record map around `15DDxxx`.
6. Runtime validation of which story gifts use `add_poke.bin -> E48D70 -> E48A98`.
7. Mystery Gift: determine whether the subsystem constructs via GenerationSpec or deserializes supplied Pokémon/Wonder Card data directly.

For the detailed cross-system map, see:

```text
docs/research/swsh/SWSH Sword 1.3.2 Wild Fixed Raid Gift Generation Chains.md
```
