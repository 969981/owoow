# Pokémon Sword 1.3.2 — Wild / Fixed / Raid / Gift Pokémon Generation Chains

Updated: 2026-09-18

Target:
- Pokémon Sword / Shield 1.3.2
- Sample: `main(3).elf`
- GNU Build ID: `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`
- Address notation: `main + offset`

This document extends the existing Egg/PML analysis to the other major Pokémon-generation domains: ordinary wild, fishing, symbol/visible overworld, scripted fixed encounters, Zone/Around objects, Max Raid/Nest, Dynamax Adventures, scripted gifts/AddPoke, NPC trades, and other callers of the shared Pokémon constructor.

The central conclusion is that Sword/Shield does **not** have one universal encounter RNG. Multiple upstream systems build a common `GenerationSpec`, then converge on the shared Pokémon-generation core.

---

## 1. Confidence labels

- **CONFIRMED**: direct static call/data-flow evidence in Sword 1.3.2 ELF.
- **HIGH**: strong local structural/xref evidence; exact semantic name is reconstructed.
- **OPEN**: both ends are known, but an indirect callback/virtual edge or full field interpretation is not yet closed.

Names such as `BuildFieldGenerationSpec` or `Pml_CreatePokemonFromGenerationSpec` are research names, not claimed original Game Freak symbols.

---

## 2. Common architecture

```text
Encounter / Script / Raid source
        │
        ├─ select species / form / level
        ├─ resolve encounter-specific policies
        ├─ choose RNG domain / seed policy
        │
        ▼
GenerationSpec
        │
        ├──────────── generic path ────────────┐
        │                                      │
        ▼                                      │
main+0x7662A0                                  │
        │                                      │
        ▼                                      │
main+0x777F40  Shared Pokémon Generation <─────┼── Egg: main+0x7896F0
        │                                      └── Zone: main+0x162B07C
        ▼
main+0x766660
        ▼
main+0x7667A0  GenerationSpec resolver
        ▼
main+0x766F60  write/apply resolved fields
        ▼
Final Pokémon
```

Direct callers of the shared `main+0x777F40` boundary are currently:

```text
main+0x7662B4  -> main+0x777F40   generic Field/Script/Raid/Gift path
main+0x7896F0  -> main+0x777F40   Egg path
main+0x162B07C -> main+0x777F40   Zone/Around path
```

This LR split is useful for runtime telemetry and safe source classification.

---

## 3. GenerationSpec layout used across systems

Current reconstructed layout:

```cpp
struct GenerationSpec {
    uint64_t fixedSeed;       // +0x00 CONFIRMED
    uint8_t  useFixedSeed;    // +0x08 CONFIRMED
    // ...
    uint64_t ecPolicy;        // +0x10 PROBABLE/HIGH
    uint64_t pidPolicy;       // +0x18 HIGH
    uint64_t trainerValue;    // +0x20 HIGH
    uint32_t species;         // +0x28 CONFIRMED
    uint16_t form;            // +0x2C CONFIRMED
    uint16_t level;           // +0x2E CONFIRMED
    uint16_t genderPolicy;    // +0x30 HIGH
    int16_t  naturePolicy;    // +0x32 HIGH
    uint8_t  abilityPolicy;   // +0x34 HIGH
    uint8_t  shinyRolls;      // +0x35 HIGH
    int16_t  iv[6];           // +0x36..+0x40 CONFIRMED
    uint16_t itemPolicy;      // +0x44 PROBABLE/HIGH
    uint8_t  perfectIVCount;  // +0x48 CONFIRMED
    uint16_t sizePolicyA;     // +0x4A HIGH: WeightParam in Egg path
    uint16_t sizePolicyB;     // +0x4C HIGH: HeightParam in Egg path
};
```

When `useFixedSeed != 0`, `main+0x7667A0` initializes a local deterministic stream through `main+0x767700`:

```text
s0 = fixedSeed
s1 = 0x82A2B175229D6A5B
```

This inner fixed-seed stream is distinct from the outer Field, Egg and Zone RNG domains.

---

# 4. Ordinary wild / hidden Field encounters

## 4.1 Verified core chain

```text
main+0xD31BB0    Common encounter-selection dispatcher
    ↓
main+0xD328A0    weighted ten-slot selector
    ↓
main+0xD329D0    selected-entry conversion
    ↓
main+0xD311C0    overworld encounter policy preparation
    ↓
main+0xD30FE0    bridge toward GenerationSpec
    ↓
main+0xD31910    BuildFieldGenerationSpec
    ↓
main+0xD310D8    call site
    ↓
main+0x7662A0    generic Pokémon generation entry
    ↓
main+0x777F40
```

`main+0xD31910` directly proves the common species/form/level positions:

```text
GenerationSpec+0x28 <- species
GenerationSpec+0x2C <- form
GenerationSpec+0x2E <- level
```

It also writes IV policy fields, perfect-IV count, and fixed-seed fields when the upstream policy requests a fixed inner stream.

## 4.2 Field slot RNG

`main+0xD328A0` owns a ten-slot weighted selection path. The 0..99 roll uses rejection sampling rather than modulo:

```cpp
do {
    r = Next32() & 0x7F;
} while (r > 99);
```

The selector then walks ten probability entries with six-byte stride.

Therefore an `EncounterStateSearcher` must reproduce rejection exactly. `Next() % 100` is not equivalent and can shift every later RNG consumer.

## 4.3 RNG ownership

The ordinary Field encounter has an outer Field RNG context before the GenerationSpec is finalized. This outer stream decides or influences:

- slot selection;
- level and encounter-specific policies;
- some lead/field effects;
- mark-related behavior;
- other encounter metadata.

The inner `fixedSeed` is not sufficient to reproduce the entire natural encounter.

## 4.4 Recommended hooks

Natural encounter replay:

```text
slot selection telemetry: main+0xD328A0
post-selection / before conversion: main+0xD31BB0 region
spec inspection: main+0xD31910 / before main+0xD310D8
```

Controlled final generation:

```text
main+0x7662A0
```

with strict caller/source guards.

---

# 5. Fishing

Fishing has its own controller/resource side, but the actual Pokémon selection converges into the common Field chain.

## 5.1 Resource/controller side

```text
main+0xDA4720   fishing controller/state machine
main+0xDA5550   fishing resource loader
```

The latter directly references:

```text
script/fishing.dat
FISHING_OK
FISHING_NG
```

## 5.2 Runtime generation chain

```text
main+0xDAF350
    ↓
main+0xD05080
    ↓
main+0xD31BB0
    ↓
Field selection / GenerationSpec chain
    ↓
main+0x7662A0
    ↓
main+0x777F40
```

So `main+0xDA5550` should not be treated as a species generator. It is a resource/state boundary.

## 5.3 Hook strategy

Use:

```text
source guard: main+0xD05080 / fishing runtime context
slot selection: main+0xD31BB0 / main+0xD328A0
spec override if required: main+0x7662A0 with Fishing-active guard
```

---

# 6. Symbol / visible overworld encounters

Confirmed resources include:

```text
bin/field/param/symbol_encount_mons_param/symbol_encount_mons_param.bin
bin/field/param/symbol_encount/ai.blua
bin/field/param/encount/encount_symbol_k.bin
SymbolEncount
```

Relevant runtime managers include:

```text
main+0x1484780
main+0x14B2F50
```

Current static state:

```text
symbol resources
    ↓
symbol runtime manager(s)
    ↓
[OPEN: exact indirect/virtual bridge]
    ↓
common Field generation
    ↓
main+0x7662A0
    ↓
main+0x777F40
```

The common sink is confirmed, but the exact symbol-manager callback joining the two sides remains OPEN.

Safe implementation:

1. instrument the symbol manager to set a thread/source flag;
2. observe or patch `main+0x7662A0` only while that flag is active.

Do not invent a direct symbol-generation function until the indirect call edge is proven.

---

# 7. Scripted fixed encounters / event encounters

This is now one of the strongest non-Field chains.

Relevant resource:

```text
bin/script_event_data/event_encount_data.bin
```

The subsystem lies around the `main+0xE4Axxx` region.

## 7.1 High-confidence chain

```text
event_encount_data.bin
    ↓
script/event lookup
    ↓
main+0xE4A470      keyed/event generator
    ↓
main+0xE4AA30      BuildScriptGenerationSpec
    ↓
main+0xE4A570      BL main+0x7662A0
    ↓
main+0x7662A0
    ↓
main+0x777F40
```

## 7.2 Spec field writes

`main+0xE4AA30` is valuable because the record-to-spec conversion is visible directly.

Observed examples:

```asm
; species
STR W8,[X19,#0x28]

; form
STRH W8,[X19,#0x2C]

; level
STRH W8,[X19,#0x2E]
```

The same builder continues populating PID/shiny, trainer, gender, nature and other policy fields.

A branch in the PID-policy region constructs the familiar sentinel family around:

```text
0x3FFFFFFFF
```

which is consistent with random/fixed/shiny-policy encoding used by the shared resolver.

## 7.3 Why this is the best fixed-encounter research target

Unlike Field encounters, event records already carry explicit Pokémon-generation policy data. Therefore the reverse-engineering task is largely:

```text
FlatBuffer/event record field
        ↓
E4AA30 reader
        ↓
GenerationSpec field
        ↓
7667A0 interpretation
```

This makes `main+0xE4AA30` the preferred function for fully mapping:

- species/form/level;
- shiny lock / shiny force policy;
- fixed/random nature;
- fixed/random gender;
- fixed/random ability;
- IV policy;
- size policy;
- trainer/shiny context.

---

# 8. Zone / Around visible-object generation

Zone/Around is a separate RNG ownership domain and does not use the generic `0x7662A0` entry.

Verified chain:

```text
main+0x1629E30
    ↓
seed acquisition
    ↓
main+0xF28FC0
local xoroshiro initialization
    ↓
main+0x162AD70
    ↓
main+0x162B07C
    ↓
main+0x777F40
```

Confirmed Zone data transfers include:

```text
ZoneData+0x170 -> species
ZoneData+0x174 -> form
```

RNG ownership:

```text
Zone RNG != Field RNG != Egg RNG
```

Recommended hook:

```text
state immediately after main+0xF28FC0
spec immediately before main+0x162B07C
```

---

# 9. Max Raid / G-Nest generation

A strong cluster of `main+0x7662A0` callers occurs around:

```text
main+0xEC7CA8
main+0xEC877C
main+0xEC950C
```

The surrounding subsystem contains G-Nest-specific strings/resources such as:

```text
Stop_Prop_Gimmick_G_nest_Rare
Stop_Prop_Gimmik_G_nest
Play_Prop_Gimmick_G_nest_Rare
Play_Prop_Gimmik_G_nest
```

This makes the `EC7xxx..EC9xxx` cluster a high-confidence Max Raid / Nest Pokémon-generation subsystem.

## 9.1 Reconstructed architecture

```text
Nest/Raid encounter record
    │
    ├─ species / form
    ├─ star / level-related policy
    ├─ gender policy
    ├─ nature policy
    ├─ ability policy
    ├─ shiny / PID policy
    ├─ perfect-IV count / IV policy
    └─ other raid flags
    ↓
Raid GenerationSpec builder
    ↓
main+0xEC7CA8 / 0xEC877C / 0xEC950C
    ↓
main+0x7662A0
    ↓
main+0x777F40
```

## 9.2 Related resource lead

A Nest-related resource reference exists for:

```text
bin/test/nest_hole/net_nest_hole_dai_encount.bin
```

with loading/reference code near `main+0xEA6B7C`.

This address is a resource/network side lead, **not** the final Pokémon generator. The generation work of interest is the `EC7xxx..EC9xxx` caller cluster.

## 9.3 Secondary G-Nest path

Another direct generic-generation call exists around:

```text
main+0x10444B0 -> main+0x7662A0
```

with nearby G-Nest UI/state references such as:

```text
Play_UI_Gnest_ready
```

Current interpretation:

- `EC7xxx..EC9xxx`: raid/nest data -> Pokémon spec construction cluster;
- `0x10444B0`: a separate raid/battle/UI-side Pokémon creation path.

These should remain separate until exact cross-calls are proven.

---

# 10. Dynamax Adventures / Underground Exploration

A distinct DLC subsystem calls the generic generator at:

```text
main+0x15DD6E0 -> main+0x7662A0
```

The surrounding data/model includes clear Underground Exploration terminology and fields such as:

```text
boss_star
big_boss_star
boss_g_hp
big_boss_g_hp
result_legend_num
result_big_boss_num
result_safety_num
encountTable
pokeType
```

Therefore this path should be separated from ordinary Max Raid/Nest generation.

Reconstructed chain:

```text
Underground Exploration encounter table
    ↓
route/node/boss encounter selection
    ↓
main+0x15DDxxx spec construction
    ↓
main+0x15DD6E0
    ↓
main+0x7662A0
    ↓
main+0x777F40
```

The builder around this call reads policy-like values individually and is a strong future target for a dedicated Dynamax Adventure predictor.

---

# 11. Scripted AddPoke / Gift Pokémon

Relevant resources include:

```text
bin/script_event_data/add_poke.bin
script/field_trade.dat
```

A high-confidence generic generation call is:

```text
main+0xE48A98 -> main+0x7662A0
```

The surrounding initialization resembles the generic spec defaults:

```text
PID sentinel = 0x3FFFFFFFF
IV sentinel/default policy
Height/Weight policy = 0x0100
...
```

with additional spec construction around:

```text
main+0xE48D70
```

Reconstructed path:

```text
add_poke.bin / scripted gift data
    ↓
main+0xE48D70   gift-specific spec population
    ↓
main+0xE48A98
    ↓
main+0x7662A0
    ↓
main+0x777F40
```

Likely consumers include scripted “give Pokémon” events such as starters or NPC gifts, but individual story events still require runtime/source validation before assigning every gift to this path.

---

# 12. NPC Trade

Relevant resource:

```text
bin/script_event_data/field_trade.bin
```

Reference area:

```text
main+0xE4CE2C
```

Generation cluster includes:

```text
main+0xE4D5D0 -> main+0x7662A0
```

High-confidence architecture:

```text
field_trade.bin
    ↓
NPC trade record
    ↓
GenerationSpec
    ↓
main+0xE4D5D0
    ↓
main+0x7662A0
    ↓
Pokémon generation
    ↓
trade-specific OT / nickname / metadata handling
```

This is not a natural encounter RNG path even though it shares the same final PML constructor.

---

# 13. Mystery Gift

A separate Mystery Gift subsystem is visible through references such as:

```text
sd9150_mysterygift
```

around:

```text
main+0xFE3BDC
```

Current status: **OPEN**.

No direct `main+0x7662A0` edge has yet been proven for this subsystem. It may deserialize supplied Wonder Card/Pokémon payloads rather than construct a Pokémon from a normal GenerationSpec.

Do not merge Mystery Gift with `add_poke.bin` until that distinction is closed.

---

# 14. Other generic GenerationSpec consumers

`main+0x7662A0` is not a “wild generator”. It is a common PML generation entry used by several systems.

Known contextual callers include:

### Battle Tower / rental Pokémon

Area around:

```text
main+0xE4670C
```

with resources such as:

```text
rental.bin
tower_trainer.dat
```

### Trainer/battle Pokémon

Area around:

```text
main+0x1521EF4
```

with trainer data resources.

### Pokémon Camp NPC data

Area around:

```text
main+0x1235034
```

with `pokecamp` / NPC-camp resources.

These callers reinforce the semantic naming:

```text
main+0x7662A0 = Pml_CreatePokemonFromGenerationSpec
```

rather than `GenerateWildPokemon`.

---

# 15. Unified generation taxonomy

```text
                               Pokémon Generation
                                       │
        ┌──────────────────────────────┼───────────────────────────────┐
        │                              │                               │
      Field                          Script                          Special
        │                              │                               │
 ┌──────┼──────┐              ┌────────┼────────┐              ┌───────┼────────┐
 │      │      │              │        │        │              │       │        │
Wild  Fish   Symbol         Fixed    AddPoke   Trade           Egg    Zone     Raid
 │      │      │              │        │        │              │       │        │
D31BB0 D05080 managers      E4AA30   E48D70   E4D5D0         789770  162AD70  EC7xxx
 │      │      │              │        │        │              │       │        │
 └──────┴──────┴──────────────┴────────┴────────┴──────┐       │       │        │
                                                       ▼       │       │        │
                                                    7662A0     │       │        │
                                                       │       │       │        │
                                                       └──┐    │       │        │
                                                          ▼    ▼       ▼        ▼
                                                        777F40 shared generation core
```

Dynamax Adventures form another independent upstream branch:

```text
15DDxxx -> 15DD6E0 -> 7662A0 -> 777F40
```

---

# 16. RNG ownership matrix

| Source | Outer RNG / policy owner | Generic fixedSeed usable? | Final sink |
|---|---|---:|---|
| Ordinary Field | Field context RNG | yes, after outer policies | `7662A0 -> 777F40` |
| Fishing | Field context after fishing source transition | yes | `7662A0 -> 777F40` |
| Symbol | Field/symbol context, exact bridge open | yes downstream | `7662A0 -> 777F40` |
| Script fixed | event/script record + keyed context | yes | `7662A0 -> 777F40` |
| Egg | independent breeding RNG | controlled override only after egg-specific decisions | `7896F0 -> 777F40` |
| Zone/Around | local Zone xoroshiro | local spec direct | `162B07C -> 777F40` |
| Max Raid/Nest | raid/nest record + raid-specific seed/policy | likely/yes at common spec level; exact raid seed mapping still open | `7662A0 -> 777F40` |
| Dynamax Adventure | underground exploration context | yes at built spec level | `7662A0 -> 777F40` |
| AddPoke/Gift | script gift record | yes | `7662A0 -> 777F40` |
| NPC Trade | trade record | yes | `7662A0 -> 777F40` |

Important rule:

> Do not search every system using one “global seed”. Search or replay the upstream RNG domain when natural slot/policy behavior matters; use the inner fixedSeed only after the GenerationSpec has been fixed.

---

# 17. PokeDocs-assisted localization strategy

PokeDocs is best used as a semantic schema reference, not as a Sword symbol table.

Across Switch Pokémon formats, recurring concepts include:

```text
species / form
ability1 / ability2 / ability3
gender
egg groups / hatch species
base height / base weight
```

This helps classify candidate accessors and record readers in the ELF.

For every encounter subsystem, the recommended workflow is:

```text
resource / table semantic field
    ↓
record reader xrefs
    ↓
GenerationSpec write
    ↓
shared resolver read
    ↓
final PK8 setter
```

Do not confuse:

```text
model rendering height
species base height/weight
individual HeightScalar/WeightScalar
```

They belong to different layers.

---

# 18. Recommended IDA research names

```text
0x7662A0   Pml_CreatePokemonFromGenerationSpec
0x777F40   Pml_SharedPokemonGenerationBoundary
0x7667A0   Pml_ResolveGenerationSpec
0x766F60   Pml_ApplyResolvedSpec

0xD31BB0   Field_SelectEncounter
0xD328A0   Field_SelectWeightedSlot10
0xD329D0   Field_ConvertSelectedEntry
0xD311C0   Field_PrepareEncounterPolicies
0xD31910   Field_BuildGenerationSpec

0xD05080   Fishing_FieldEncounterEntry

0xE4A470   ScriptEvent_GeneratePokemon
0xE4AA30   ScriptEvent_BuildGenerationSpec

0xE48D70   AddPoke_BuildGenerationSpec
0xE48A98   AddPoke_CreatePokemon_Callsite

0xE4D5D0   FieldTrade_CreatePokemon_Callsite

0x1629E30  Zone_InitGenerationRng
0x162AD70  Zone_BuildPokemon

0x15DD6E0  DynamaxAdventure_CreatePokemon_Callsite
```

Raid names should remain cluster-level until the exact individual roles of `EC7CA8`, `EC877C`, and `EC950C` are separated dynamically.

---

# 19. Priority next steps

## P0 — Script fixed encounters

Fully map `main+0xE4AA30` record reads to:

```text
GenerationSpec+0x18 PID/shiny policy
+0x20 trainer/shiny context
+0x30 gender
+0x32 nature
+0x34 ability
+0x35 shiny rolls/policy
+0x36..+0x40 IVs
+0x48 perfect IV count
+0x4A/+0x4C size policies
```

This should expose shiny-lock/fixed-nature/fixed-gender semantics cleanly.

## P1 — Raid/Nest

Separate the `EC7xxx..EC9xxx` builders and determine:

- den/raid seed ownership;
- how star level maps to Pokémon level and perfect-IV count;
- shiny policy;
- ability policy;
- species/form selection;
- whether the same seed is copied to `GenerationSpec.fixedSeed` or transformed first.

## P2 — Dynamax Adventures

Map `15DDxxx` encounter record fields and identify which values are route-node policy vs final GenerationSpec policy.

## P3 — Gift/AddPoke

Map `add_poke.bin -> E48D70 -> E48A98`, then runtime-validate starter/story gift callers.

## P4 — Symbol

Use runtime indirect-call telemetry to close the remaining symbol-manager -> Field generator bridge.

---

# 20. Final conclusion

Sword/Shield Pokémon generation is best modeled as:

```text
source-specific selection and RNG
        ↓
source-specific GenerationSpec builder
        ↓
shared PML constructor
```

The critical research boundary is not “wild vs fixed” at the final constructor. The distinction lives upstream in who owns the RNG and who builds the GenerationSpec.

For deterministic tools:

- use `EncounterStateSearcher` when natural Field slot/level/policy behavior must be preserved;
- use an Egg-specific searcher for breeding;
- use a Zone-specific searcher for Zone/Around;
- use `FixedSpecSearcher` only after the source-specific GenerationSpec has already been fixed;
- classify every hook by source/caller before modifying `main+0x7662A0` or `main+0x777F40`.
