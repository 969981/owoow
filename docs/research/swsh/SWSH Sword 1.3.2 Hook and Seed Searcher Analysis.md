# Pokémon Sword 1.3.2 Encounter Hook & Seed Searcher Reverse Engineering

Date: 2026-09-16  
Target: Pokémon Sword 1.3.2  
Title ID: `0100ABF008968000`  
Build ID (short): `A3B75BCD3311385A`  
GNU Build ID: `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`  
Analyzed ELF: Library `main(3).elf`, SHA-256 `689db76a880c6ca5b20308343e6f49268c67f75fd009b7d644a166e18adc0652`

## 1. Executive conclusion

Sword 1.3.2 does **not** have one global RNG `Next()` that should be hooked for every encounter. The binary confirms multiple RNG ownership domains and a shared Pokémon generation boundary:

```text
Field encounter selection / policies  --\
Egg breeding RNG -----------------------> 0x777F40 -> 0x766660 -> 0x7667A0 -> 0x766F60 -> Pokémon
Zone/Around local RNG ------------------/
```

The correct design is therefore two-layered:

1. **EncounterStateSearcher**: search/replay the outer encounter RNG state (`s0/s1`) when natural slot, level, Brilliant, shiny policy, gender, nature, ability, item and mark must all remain naturally produced by the encounter system.
2. **FixedSpecSearcher**: hold/override the already-built `GenerationSpec`, then search the inner `fixedSeed` for deterministic EC/PID/IV/size output. This is the narrowest and safest way to generate a target Pokémon without patching the finished PK8.

A fixed seed by itself does **not** determine the whole encounter. In ordinary overworld generation, many policy fields are resolved by the outer field RNG before the fixed-seed stream is created. Mark generation also remains on the outer stream.

## 2. Sword/Shield address migration rule

A global `Sword = Shield - 0x30` rule is wrong.

The Sword ELF verifies a `-0x30` shift in several late field/fishing/script regions, while the shared lower Pokémon-generation code keeps identical addresses.

| Role | Shield 1.3.2 research | Sword 1.3.2 verified |
|---|---:|---:|
| ordinary field upstream | `0xD03ED0` | `0xD03EA0` |
| overworld policy | `0xD311F0` | `0xD311C0` |
| field spec builder | `0xD31940` | `0xD31910` |
| encounter selection core | `0xD31BE0` | `0xD31BB0` |
| weighted ten-slot selector | `0xD328D0` | `0xD328A0` |
| selected-slot conversion | `0xD32A00` | `0xD329D0` |
| fishing controller | `0xDA4750` | `0xDA4720` |
| fishing resource loader | `0xDA5580` | `0xDA5550` |
| script keyed entry | `0xDCEFE0` | `0xDCEFB0` |
| script keyed entry 2 | `0xDCF340` | `0xDCF310` |
| keyed generator | `0xE4A4A0` | `0xE4A470` |
| keyed helper | `0xE4A370` | `0xE4A340` |
| keyed helper 2 | `0xE4AA60` | `0xE4AA30` |
| generic field generation | `0x7662A0` | `0x7662A0` |
| shared generation boundary | `0x777F40` | `0x777F40` |
| egg spec builder | `0x789770` | `0x789770` |

Never port a patch by a fixed delta without validating the target function fingerprint and call graph.

## 3. Shared Pokémon generation core

### 3.1 `0x777F40` caller classification

Direct BL scan in Sword 1.3.2 finds exactly three callers:

```text
0x7662B4 -> 0x777F40   generic Field / Script path
0x7896F0 -> 0x777F40   Egg path
0x162B07C -> 0x777F40  Zone/Around path
```

This makes `0x777F40` an excellent telemetry boundary. It is not, however, the whole deterministic generator by itself.

Observed downstream:

```text
0x777F40
  -> 0x766660
     -> 0x7667A0    GenerationSpec resolver / local RNG consumer
        -> 0x766F60 GenerationSpec application / Pokémon object population
```

### 3.2 `0x7667A0`: critical resolver

This is currently the most valuable low-level hook/research function. It copies the incoming spec, decides whether to create a local fixed-seed RNG, then resolves sentinel/policy fields.

Confirmed fixed-seed handling:

```text
GenerationSpec +0x00 : fixedSeed (u64)
GenerationSpec +0x08 : useFixedSeed (bool)
```

At `0x766808`, the function tests `spec+0x08`; when non-zero it loads `spec+0x00` and calls `0x767700`.

`0x767700` initializes the local xoroshiro stream with:

```text
s0 = fixedSeed
s1 = 0x82A2B175229D6A5B
```

The helper stores an additional seed copy in its local context, but the RNG transition consumes the first two 64-bit words.

The resolver also contains logic for EC/PID policy, perfect-IV count, random IVs, ability policy, form policy, gender policy, nature policy and size-related fields. Exact field names below are intentionally confidence-tagged.

## 4. GenerationSpec layout

Confirmed from Sword writer/reader evidence:

```cpp
struct GenerationSpec {
    uint64_t fixedSeed;       // +0x00 CONFIRMED
    uint8_t  useFixedSeed;    // +0x08 CONFIRMED
    // ...
    uint64_t ecPolicy;        // +0x10 PROBABLE: sentinel/random resolution at 0x7667A0
    uint64_t pidPolicy;       // +0x18 HIGH: PID/shiny policy logic
    uint64_t trainerValue;    // +0x20 HIGH: trainer-derived value consumed by PID/shiny logic
    uint32_t species;         // +0x28 CONFIRMED
    uint16_t form;            // +0x2C CONFIRMED
    uint16_t level;           // +0x2E CONFIRMED
    uint16_t genderPolicy;    // +0x30 HIGH
    int16_t  naturePolicy;    // +0x32 HIGH
    uint8_t  abilityPolicy;   // +0x34 HIGH
    uint8_t  shinyRolls;      // +0x35 HIGH
    int16_t  iv[6];           // +0x36..+0x40 CONFIRMED sentinel/random-IV path
    uint16_t itemPolicy;      // +0x44 PROBABLE/HIGH
    uint8_t  perfectIVCount;  // +0x48 CONFIRMED
    uint16_t sizePolicyA;     // +0x4A PROBABLE
    uint16_t sizePolicyB;     // +0x4C PROBABLE
};
```

`0xD31910` proves the main field layout directly:

```text
[x1 + 0x28] <- dword [x2 + 0x00]   species
[x1 + 0x2C] <- halfword [x2 + 0x04] form
[x1 + 0x2E] <- byte [x2 + 0x06]     level
```

It also copies six IV policy halfwords, perfect-IV count and, when the upstream fixed-seed flag is set, writes `useFixedSeed=1` and copies the seed into `+0x00`.

## 5. Ordinary wild / hidden field encounter

### Verified core chain

```text
0xD31BB0  common encounter selection dispatcher
  -> 0xD328A0 weighted fallback slot selector
  -> 0xD329D0 selected-entry conversion
  -> 0xD311C0 overworld generation policy preparation
  ...
0xD30FE0 field GenerationSpec bridge
  -> 0xD31910 BuildFieldGenerationSpec
  -> 0x7662A0 generic field Pokémon generation entry
  -> 0x777F40
```

`0xD31BB0` has three verified upstream callers:

```text
0xD044E0 (inside 0xD03EA0..0xD04678)
0xD04DE8 (inside 0xD04750..0xD04E54)
0xD0535C (inside 0xD05080..0xD053C0)
```

At least one `0xD04750` caller is in the shake-tree flow (`APPEAR_POKEMON`, `fi0111_shaketreeswait01_loop`, `fi_shaketree_off` references). `0xD05080` is confirmed below as the fishing encounter entry. Therefore `0xD31BB0` is a genuine common selection core rather than one encounter type.

### `0xD328A0`: ten-slot weighted selector

The slot selector uses the shared Field RNG context and an inline xoroshiro128+ transition. For a 0..99 roll it does **rejection sampling**:

```cpp
do {
    r = Next32() & 0x7F;
} while (r > 99);
```

It then accumulates ten probabilities at six-byte stride and returns slot `0..9`, or failure if total probability does not cover the roll.

This matters to a Seed Searcher: replacing this with `Next() % 100` is wrong and can shift all later advances when the rejected value is 100..127.

### Recommended hooks

Natural target already present in encounter table:

- weighted-fallback-only hook: function `0xD328A0`;
- complete post-special-selection hook: inline boundary after final slot is established in `0xD31BB0` and before `0xD329D0` (more complete, but more fragile than a function hook).

Target not present / direct target generation:

- ordinary field: hook the spec produced by `0xD31910`, before the `0x7662A0` call at `0xD310D8`;
- generic field/script: hook `0x7662A0` entry and modify the incoming GenerationSpec, with caller/context filtering.

## 6. Fishing

### Resource/controller path

Sword addresses:

```text
0xDA4720..0xDA5550  fishing controller/state machine
0xDA5550..0xDA5A3C  fishing resource loader
```

`0xDA55F0/0xDA55F4` directly forms the address of the string:

```text
script/fishing.dat
```

The same subsystem contains direct references to `FISHING_OK` and `FISHING_NG`.

### Pokémon generation path

A separate fishing runtime function is now statically connected to the common encounter selector:

```text
0xDAF350..0xDAF4D0
    0xDAF428 -> 0xD05080
0xD05080..0xD053C0
    0xD0535C -> 0xD31BB0
```

This closes the previously missing “fishing runtime -> common encounter selection” side substantially. The exact asynchronous/script object edge between `DA4720/DA5550` and `DAF350` is still indirect (virtual/callback driven), so it is not claimed as a direct BL edge.

### Recommended Fishing hooks

For a natural fishing target already in the table:

```text
source guard: 0xD05080 / fishing runtime context
slot selection: 0xD31BB0 / 0xD328A0
```

For a target not in the table or for deterministic fixed generation:

```text
set Fishing-active guard at source
hook common field GenerationSpec at 0x7662A0
apply species/form/level and/or fixedSeed only while guard is active
```

Do **not** hook `0xDA5550` to choose species; it is the `fishing.dat` loader/state boundary, not the species generator.

## 7. Symbol / visible overworld encounter

### Confirmed resources and runtime support

The Sword ELF contains and references:

```text
bin/field/param/symbol_encount_mons_param/symbol_encount_mons_param.bin
bin/field/param/symbol_encount/ai.blua
bin/field/param/encount/encount_symbol_k.bin
SymbolEncount
```

Verified references:

```text
0xEA2538/0xEA253C -> symbol_encount_mons_param.bin
0xE513A8/0xE513AC -> symbol encounter AI resource
0xF1424C/0xF14250 -> encount_symbol_k.bin
0xD5DEAC/0xD5DEB0 -> "SymbolEncount"
```

The Shield research addresses `0x14847B0` and `0x14B2F80` map to Sword functions beginning at `0x1484780` and `0x14B2F50` respectively, confirming the same `-0x30` shift in this region. These functions operate on symbol runtime objects/managers.

### Open edge

Static evidence currently confirms both ends:

```text
symbol resources -> runtime managers
common field selection/generation -> 0xD31BB0 -> GenerationSpec -> 0x7662A0
```

but the exact indirect runtime callback that joins the symbol manager to the common generator has not yet been proven by a direct call edge. This is intentionally left OPEN.

Existing `owoow` `Symbol.Generate()` consumes two placement rolls before lead/slot selection (`NextInt(361)` then raw `Next()`), unlike Fishing. That implementation remains consistent with the known game behavior, but the two placement rolls have not yet been assigned to a single Sword function address in this static pass.

### Safe current Symbol hook design

Use a two-stage dispatcher:

1. symbol runtime hook/telemetry at the symbol manager layer to set a thread-local/source flag;
2. modify the actual GenerationSpec at common `0x7662A0` only while the Symbol flag is active.

This avoids guessing a fake “Symbol Generate Pokémon” function while still keeping the modification isolated from Egg/Zone/other scripted encounters.

## 8. Egg / breeding

Egg generation is a separate RNG subsystem.

### Breeding RNG

Verified global breeding RNG state:

```text
0x25A83A8  Breeding xoroshiro state
```

Function:

```text
0x1400C70..0x1400D3C
```

is the breeding bounded RNG function. The previously used label `0x1400CA0` points inside this function; a function-level hook must use `0x1400C70`.

It performs the same xoroshiro128+ transition and an unbiased bounded/rejection path.

`0x14018C0..0x1401B6C` is the breeding controller. It loads a daycare-slot seed at `slot + 0x2F8`, initializes the breeding state via `0xF28FC0`, then proceeds to the Egg path.

`0xF28FC0` initializes:

```text
s0 = seed
s1 = 0x82A2B175229D6A5B
```

### Egg generation chain

```text
0x14018C0 controller
  -> 0x7893F0
     -> 0x789500
        -> 0x789770 BuildEggGenerationSpec
        -> 0x7896F0 -> 0x777F40
```

`0x789770` resolves egg-specific data before the shared Pokémon generator: species/form, level 1, nature, ability, item, inherited/random IVs and shiny/PID logic are driven by the breeding RNG callback/context.

### Recommended Egg hooks

Faithful breeding RNG replay:

- inject/read state at `0x25A83A8` immediately after the controller initializes it, or hook `0x1400C70` for telemetry;
- do **not** modify the overworld Field RNG `+0x98`.

Final generation override:

- hook after `0x789770` returns and before `0x7896F0 -> 0x777F40`.

Setting `useFixedSeed` in the resulting Egg spec can force the shared downstream resolver, but this is a **controlled generation override**, not faithful replay of the natural breeding stream because many egg fields have already been decided by `0x789770`.

## 9. Zone / Around visible objects

Zone/Around owns a local RNG stream.

Verified chain:

```text
0x1629E30..0x162A984  local Zone setup
  -> seed acquisition
  -> 0xF28FC0 local xoroshiro initialization
0x162AD70..0x162B39C  Zone object selection/build
  -> 0x162B07C -> 0x777F40
```

Inside `0x162AD70`:

```text
ZoneData entry +0x170 -> GenerationSpec species
ZoneData entry +0x174 -> GenerationSpec form
```

The function receives/updates its own RNG state pointer and uses that state for local random/shuffle decisions. It is not the shared overworld RNG at `FieldRngContext + 0x98`.

Recommended hooks:

- natural Zone state replay: inject `s0/s1` at the RNG pointer passed to `0x162AD70` (or immediately after the parent initializes the local stack RNG);
- species/form override: after the species/form writes near `0x162AFA0` / `0x162B004`, before `0x162B07C`;
- fixed generation: modify that stack GenerationSpec before the `0x777F40` call.

## 10. Seed Searcher architecture

### 10.1 Tier A: `FixedSpecSearcher`

Use when encounter/spec policy fields are already known or intentionally overridden.

Input:

```text
GenerationSpec policy snapshot
Trainer value / shiny policy
Target EC/PID/IV/size constraints
Seed range
```

For each candidate seed:

```cpp
rng.s0 = seed;
rng.s1 = 0x82A2B175229D6A5B;

Resolve only fields whose spec policy is sentinel/random,
in the exact order used by 0x7667A0.

Reject candidate as soon as a target constraint fails.
```

For ordinary overworld specs that already have nature/gender/ability/shiny policy resolved, the existing `owoow.Core/RNG/Generators/Overworld/Fixed.cs` model is already close to the effective inner stream:

```text
EC -> PID -> guaranteed IV slots -> remaining IVs -> height/weight/scale
```

The existing `SpreadFinder` also already seeds the inner RNG as `(FixedSeed, 0x82A2B175229D6A5B)`.

Important limitations:

- species/form/level are not chosen by fixedSeed;
- natural shiny state/policy is decided on the outer stream, although the inner PID is then forced/anti-forced according to policy;
- nature/gender/ability are normally outer-policy results;
- Mark remains on the outer stream;
- therefore a “perfect target Pokémon” search cannot be modeled as `Pokemon = Generate(seed)` alone.

### 10.2 Tier B: `EncounterStateSearcher`

Use when natural encounter legality/context should be preserved.

For each outer state/frame:

```text
clone (s0,s1)
apply encounter-type-specific pre-rolls
perform lead ability / type pulling / DexRec / weighted slot
choose level
consume discarded mark roll
resolve Brilliant/shiny/gender/nature/ability/item
resolve aura-specific values when applicable
fixedSeed = outer.Next32()
run inner fixed-seed generator
consume final Mark on outer RNG
match complete target
```

Subtype differences already represented in `owoow`:

- Hidden: encounter-rate loop/step failures before slot selection.
- Symbol: placement `NextInt(361)` + one raw `Next()` before lead/slot.
- Fishing: no Symbol placement pair; Fishing mark condition is enabled.
- Zone: separate local stream and separate builder; do not reuse Field outer stream.
- Egg: breeding state and inheritance algorithm; do not reuse Field outer stream.

### 10.3 Search strategy

Fast deterministic search:

```text
Target IV/EC/PID/size
 -> FixedSpecSearcher
 -> candidate fixedSeed
 -> inject spec +0x00/+0x08
```

Natural overworld search:

```text
Target species + nature + ability + shiny + mark + IV...
 -> EncounterStateSearcher
 -> candidate outer (s0,s1) / advance
 -> inject/restore outer RNG state
 -> let game build original GenerationSpec and fixedSeed
```

Hybrid search:

```text
search outer state only for species/level/policies/mark
then override fixedSeed for exact inner spread
```

Hybrid mode is useful experimentally but should be labeled as an override because the fixedSeed is no longer the one naturally derived from that outer state.

## 11. Recommended runtime Hook dispatcher

Do not scatter behavior patches across every xoroshiro transition. Keep a few narrow hooks:

```text
Field source/selection telemetry : 0xD31BB0
Field GenerationSpec entry       : 0x7662A0
Shared generation telemetry      : 0x777F40
Egg spec boundary                : after 0x789770, before 0x7896F0
Zone spec boundary               : before 0x162B07C
Breeding RNG telemetry/state     : 0x1400C70 / state 0x25A83A8
Fishing source guard             : 0xD05080
```

Suggested dispatcher model:

```cpp
enum class EncounterSource {
    Unknown,
    Field,
    Symbol,
    Fishing,
    Egg,
    Zone,
    ScriptKeyed,
};

void OnGenerationSpec(EncounterSource source, GenerationSpec* spec) {
    Trace(spec, source);

    if (!Target.Enabled || !Target.AppliesTo(source))
        return;

    if (Target.OverrideSpecies) {
        spec->species = Target.Species;
        spec->form = Target.Form;
    }
    if (Target.OverrideLevel)
        spec->level = Target.Level;

    if (Target.UseFixedSeed) {
        spec->fixedSeed = Target.FixedSeed;
        spec->useFixedSeed = 1;
    }
}
```

At `0x777F40`, LR directly distinguishes Generic Field (`0x7662B4`), Egg (`0x7896F0`) and Zone (`0x162B07C`). This is ideal for telemetry and a final safety filter.

## 12. Hook matrix

| Encounter | Best natural-state hook | Best target/spec hook | FixedSeed hook | Status |
|---|---|---|---|---|
| ordinary wild/hidden | shared Field RNG before `0xD31BB0` | `0xD31910` output / `0x7662A0` | spec `+0/+8` | core closed |
| Fishing | source `0xD05080`, then Field RNG | guarded `0x7662A0` | spec `+0/+8` | generation path closed; controller callback edge indirect |
| Symbol | Symbol runtime telemetry + common Field RNG | guarded `0x7662A0` | spec `+0/+8` | resources/runtime confirmed; direct bridge OPEN |
| Egg | breeding state `0x25A83A8` | post-`0x789770` | optional controlled override | closed to shared sink |
| Zone/Around | local state passed to `0x162AD70` | pre-`0x162B07C` | stack spec `+0/+8` | closed to shared sink |

## 13. What is now proven vs still open

**CONFIRMED:** Sword 1.3.2 identity; region-dependent Sword/Shield address shifts; ordinary slot selection core; ten-slot rejection sampling; GenerationSpec species/form/level offsets; fixedSeed/useFixedSeed offsets; fixed RNG initialization constant; shared 777F40 three-caller split; independent breeding RNG; Egg builder/sink chain; independent Zone RNG and Zone species/form offsets; Fishing resource loader and Fishing runtime path into D31BB0.

**OPEN:** exact Symbol runtime callback that bridges symbol resource/runtime tables to the common field generator; exact semantic names for `GenerationSpec +0x4A/+0x4C`; complete dynamic validation of every `0x7667A0` policy/sentinel combination.

These OPEN points should be resolved with runtime caller telemetry rather than guessed static names.
