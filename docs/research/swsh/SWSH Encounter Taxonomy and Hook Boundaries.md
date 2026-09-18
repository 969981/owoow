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

There is no single global RNG stream for Field, Egg, Zone, Raid, Dynamax Adventure, scripted fixed encounters and gifts.

| Type | Sword 1.3.2 chain | RNG / policy owner | Preferred hook |
|---|---|---|---|
| ordinary Field / hidden | `D31BB0 -> D328A0/D329D0 -> D311C0 -> D31910 -> 7662A0 -> 777F40` | shared Field context (`+0x98` in owning object) | slot at `D328A0`; spec at `D31910/7662A0` |
| Fishing | `DA4720/DA5550` controller/resource; runtime `DAF350 -> D05080 -> D31BB0 -> ...` | Field outer RNG after fishing-source transition | source guard `D05080` + common spec hook |
| Symbol | resources at `EA24B0`, `E51380`, `F14210`; runtime managers `1484780`, `14B2F50`; common generation sink confirmed but exact indirect bridge open | Field/symbol context; exact bridge still OPEN | symbol source telemetry + guarded `7662A0` |
| Script fixed / event | `event_encount_data.bin -> E4A470 -> E4AA30 -> E4A570 -> 7662A0 -> 777F40` | event record + keyed/script context | `E4AA30` record-to-spec mapping; guarded `7662A0` |
| AddPoke / scripted Gift | `add_poke.bin -> E48D70 -> E48A98 -> 7662A0 -> 777F40` | gift/script record; observed path leaves `useFixedSeed=0` | `E48D70` spec builder / `E48A98` callsite |
| NPC Trade | `field_trade.bin -> E4D5D0 -> 7662A0 -> 777F40` | trade record | trade-record parser + `E4D5D0` |
| Egg | `14018C0 -> 7893F0 -> 789500 -> 789770 -> 7896F0 -> 777F40` | breeding state `25A83A8`; PML Crypto later for unresolved PID/size | breeding state or post-`789770` spec at `789668` |
| Zone/Around | `1629E30 -> 162AD70 -> 162B07C -> 777F40` | local Zone xoroshiro state | state after `F28FC0`; spec before `162B07C` |
| Max Raid / G-Nest | fresh seed `14B8ED0 -> F28FE0 -> 14260F0`; advance `F0A5E0 -> 14260F0`; consume `10298A0 -> EC7080 -> 7662A0 -> 777F40` | save-backed `NestEntry+0x68`; fresh value from Crypto64, deterministic frames advance by `+0x82A2B175229D6A5B` | seed write `14260F0`; advance `F0A5E0`; builder `EC7080` |
| Dynamax Adventures | `15DD330 -> 15DD42C..15DD458 -> 15DD6E0 -> 7662A0 -> 777F40` | persistent Underground Exploration xoroshiro state at object `+0x2AB8/+0x2AC0`; one Next output becomes `fixedSeed` | state at `15DD42C`; spec before `15DD6E0` |
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

The generic `7662A0 -> 7662B4 -> 777F40` path is shared by Field, Script/Event, Raid, Gift, Trade, Dynamax Adventure and several battle/NPC systems. Therefore `7662A0` must not be named or hooked as if it were wild-only.

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

## PID policy sentinels currently identified

```text
0x3FFFFFFFF          generic unresolved/random PID policy
0x1FFFFFFFF          force / guarantee non-shiny (high confidence)
0xFFFF0002FFFFFFFF   force / guarantee shiny (high confidence)
0x2FFFFFFFF          distinct policy; exact semantic label still OPEN
```

`main+0x777370` supports the non-shiny interpretation: it changes PID bit 28 only when the candidate is shiny. `main+0x7773A0` supports the force-shiny interpretation: it preserves an already-shiny candidate and otherwise rewrites the PID so the trainer/PID XOR becomes shiny.

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

`E4AA30` now has substantial direct mapping for species/form/level, trainer context, PID policy, gender, nature, ability and IV/perfect-IV policy encoding.

### Raid / G-Nest

Persistent save path:

```text
main+0x2610798
    -> saveRoot
    -> saveRoot+0x248 = Nest save block

NestEntry = nestBlock + areaIndex*0xA68 + denIndex*0x18
NestEntry+0x68 = 64-bit persistent Raid seed
```

The exact setter is:

```text
14260F0  SetNestSeed64 (research name)
```

It has only three direct writers:

```text
14B9094   fresh Crypto64 seed commit
14B9200   fallback fresh Crypto64 seed commit
F0A6EC    deterministic existing-seed advance commit
```

Fresh seed path:

```text
14B8ED0
    -> F28FE0
    -> nn::crypto::GenerateCryptographicallyRandomBytes(...,8)
    -> 14260F0
    -> NestEntry+0x68
```

Existing seed advance path:

```text
F0A5E0
    oldSeed = NestEntry+0x68
    init xoroshiro with (oldSeed, 0x82A2B175229D6A5B)
    first output = oldSeed + 0x82A2B175229D6A5B
    -> 14260F0
    -> NestEntry+0x68
```

Therefore the persistent Raid frame equation is:

```text
seed[n] = seed[0] + n * 0x82A2B175229D6A5B  (mod 2^64)
```

Backward step is subtraction by the same constant. Since the constant is odd, this additive sequence has full `2^64` period.

Raid generation consumes the saved seed through:

```text
10298A0  load NestEntry+0x68 into X2
EC7080   preserve X2 as raidSeed
EC7770   representative GenerationSpec.fixedSeed write
EC7CA8 / EC877C / EC950C -> generic generation branches
7662A0 -> 777F40
```

The same raid seed is also used to initialize separate Raid-builder local RNG contexts, so model explicit reinitialization points rather than one continuously advanced stream.

Bulk area refresh is centered at:

```text
F0A950
```

which iterates den records and invokes `F0A5E0` for qualifying entries. Multi-area callers around `F06F60`, `F07280`, and `1426440` orchestrate areas 0/1/2. Exact user-visible trigger names still need dynamic classification.

### Dynamax Adventures

```text
15DD42C  read state0 from object+0x2AB8
15DD430  read state1 from object+0x2AC0
15DD434  state0+state1 -> next output
15DD448/450 write advanced state back
15DD454  next output -> GenerationSpec.fixedSeed
15DD458  useFixedSeed = 1
15DD6E0  callsite to 7662A0
```

### Gift/AddPoke

```text
E48D70   gift-specific spec population
E48A98   callsite to 7662A0
```

The observed builder forces level 50 and its caller leaves `fixedSeed=0/useFixedSeed=0`; unresolved random PID/EC/size values therefore do not inherit the raid-style deterministic seed path.

### Zone/Around

```text
1629E30  local RNG setup
162AD70  Zone object/spec build
162B07C  direct call to 777F40
```

## Seed Searcher rule

Do not collapse all searching into one fixed seed.

`FixedSpecSearcher` searches the local inner generation after the spec policy is fixed. `EncounterStateSearcher` searches the outer subsystem RNG and is required for natural slot/level/policy/mark behavior. Egg, Zone, Raid and Dynamax Adventure each have distinct RNG ownership and need source-specific state searchers.

For Raid specifically, the natural persistent frame is now closed:

```text
read current NestEntry+0x68
    -> jump frame N with seed + N*0x82A2B175229D6A5B
    -> reproduce EC7080 Raid table/policy construction
    -> feed same seed into GenerationSpec.fixedSeed
    -> replay shared fixed Pokémon resolver
```

A not-yet-created fresh Raid seed cannot be predicted read-only before `F28FE0`, because that first value comes from Nintendo Crypto. Once it is stored, later deterministic advances are predictable until the next fresh reseed.

For Dynamax Adventure, the natural input remains the 128-bit persistent object RNG state whose next output becomes `fixedSeed`.