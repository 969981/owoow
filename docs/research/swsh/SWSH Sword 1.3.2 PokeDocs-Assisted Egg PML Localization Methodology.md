# Pokémon Sword 1.3.2 — PokeDocs-Assisted Egg / PML Localization Methodology

Updated: 2026-09-18

Target:
- Pokémon Sword / Shield 1.3.2
- ELF sample: `main(3).elf`
- GNU Build ID: `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`
- Address notation: `main + offset`

Reference repository:
- `pkZukan/PokeDocs`

Related local research:
- `SWSH Sword 1.3.2 Egg Generation Lifecycle and Prediction.md`
- `SWSH Sword 1.3.2 PML System RNG and Complete Next Egg Preview.md`
- `SWSH Sword 1.3.2 Egg RNG to PML Crypto Trigger Timeline.md`

---

## 1. Purpose

This document records a repeatable localization methodology for Sword 1.3.2 that combines:

1. PokeDocs metadata / FlatBuffer schemas as a semantic dictionary;
2. ELF control-flow and data-flow analysis;
3. xrefs from final PK8 field writes;
4. callback installation and indirect-call expansion;
5. imported Nintendo SDK functions as hard anchors;
6. runtime breakpoints for final confirmation.

The main target chain is:

```text
Day Care Egg Generator
      │
      ├── eggSeed RNG
      │      ├─ Gender
      │      ├─ Nature
      │      ├─ Ability
      │      ├─ IV
      │      ├─ EC
      │      └─ Masuda / Shiny-Charm PID
      │
      ▼
EggCreateParam
      │
      ▼
PML Pokémon Constructor
      │
      ├─ unresolved PID
      │      ↓
      │   PML Crypto Provider
      │      ↓
      │     PID
      │
      ├─ Height
      │      ↓
      │   PML Crypto Provider
      │
      └─ Weight
             ↓
          PML Crypto Provider
```

The goal is not merely to assign names to functions, but to justify each name through multiple independent evidence classes.

---

## 2. What PokeDocs is useful for

PokeDocs should not be treated as a Sword 1.3.2 native symbol database.

Its value is primarily semantic:

```text
PokeDocs
  -> known Pokémon data concepts
  -> field clusters
  -> namespace families
  -> cross-generation structural continuity
```

Examples from PokeDocs show recurring Pokémon data concepts such as:

```text
species / form
ability1 / ability2 / ability3
gender
egg group 1 / egg group 2
hatch species / hatch steps
height / weight
base stats
moves / egg moves
```

These concepts can then be mapped back into anonymous Sword functions through their callers and data access patterns.

Important rule:

> PokeDocs provides semantic hypotheses; Sword ELF data-flow provides address-level proof.

---

## 3. Cross-generation semantic anchors from PokeDocs

### 3.1 SV `pml/personal.fbs`

The SV schema groups Pokémon personal-data concepts under `pml` and includes:

```text
SpeciesInfo:
    species_index
    species_form
    height
    weight

PersonalData:
    ability1
    ability2
    ability3
    gender
    group_1
    group_2
    hatch
    hatch_steps
    stats
    egg_moves
```

This is strong semantic evidence that ability, gender ratio, egg-group/hatch data and species base dimensions belong to the same broad Pokémon personal-data layer.

### 3.2 Legends Arceus `PokemonLib/personal.fbs`

LA exposes a similar field cluster:

```text
ability1 / ability2 / ability3
gender
height / weight
group_1 / group_2
hatch_species
base_friendship
```

The namespace changed to `Hayabusa.PokemonLib`, but the conceptual grouping remains recognizable.

### 3.3 SWSH `gfbpokecfg.fbs`

SWSH itself contains model/configuration-side fields such as:

```text
namespace GFLib.Pokemon

height: float
adjust_height: float
field_adjust: float
inframe_height: float
bounding box fields
```

These are model/rendering or presentation dimensions, not the individual PK8 `HeightScalar` field.

This distinction is critical for localization.

---

## 4. Three different meanings of “Height”

A common reverse-engineering error is to find any field named `height` and assume it is the egg's individual size scalar.

There are at least three separate concepts:

```text
A. Model / rendering height
   - GFLib.Pokemon config
   - float
   - model scale / frame placement / bounding-box context

B. Species base height
   - Personal / PML data
   - uint16-like species metadata
   - species-level datum

C. Individual HeightScalar
   - PK8 +0x50
   - uint8
   - generated per individual
```

Likewise:

```text
Species base weight != PK8 WeightScalar
```

Therefore individual size localization should begin from the final PK8 field write, not from a textual `height` search.

---

## 5. Recommended localization strategy

Use this order:

```text
PokeDocs semantic field
        ↓
known final PK8 field / behavior
        ↓
setter / writer
        ↓ xref
parameter resolver
        ↓
random/value provider
        ↓
callback dispatcher
        ↓
callback installation
        ↓
concrete provider
        ↓
Nintendo SDK import
```

This produces a defensible function chain instead of a guessed name.

---

## 6. HeightScalar localization from the final PK8 write

Current Sword 1.3.2 result:

```text
PK8 +0x50 = HeightScalar
```

The confirmed setter is:

```text
main+0x775700
```

Reverse slice:

```text
PK8+0x50
    ↑
main+0x775700          SetHeightScalar
    ↑
main+0x766F04          Height parameter resolve call
    ↑
main+0x7672E0          SizeParam -> Scalar resolver
    ↑
main+0x767490          normal-random-size branch
    ↑
main+0x7660D0          PML bounded-random wrapper
```

The constructor-side load is:

```text
EggCreateParam +0x4C = HeightParam
```

For eggs, the default value is:

```text
0x0100
```

This is not an already-resolved scalar; it is a mode/sentinel telling the generic constructor to generate a normal random size.

---

## 7. WeightScalar localization

Current Sword 1.3.2 result:

```text
PK8 +0x51 = WeightScalar
```

Confirmed setter:

```text
main+0x775850
```

Reverse slice:

```text
PK8+0x51
    ↑
main+0x775850          SetWeightScalar
    ↑
main+0x766F14          Weight parameter resolve call
    ↑
main+0x7672E0          shared SizeParam resolver
    ↑
main+0x767490          normal-random-size branch
    ↑
main+0x7660D0          PML bounded-random wrapper
```

Constructor-side field:

```text
EggCreateParam +0x4A = WeightParam
```

Default egg value:

```text
0x0100
```

This symmetry between Height and Weight is an important confirmation that `0x7672E0` is a generic size-mode resolver rather than a height-specific function.

---

## 8. Recognizing `main+0x7672E0` as a size resolver

The function behavior itself identifies its role.

Observed semantic pattern:

```text
0x0000..0x00FF
    -> return exact scalar directly

0x0100
    -> normal random size

0x0101
    -> very-small range

0x0102
    -> small range

0x0103
    -> middle range

0x0104
    -> large range

0x0105
    -> very-large range
```

So an IDA pseudocode pattern such as:

```cpp
if (param < 0x100)
    return param;

switch (param) {
case 0x100:
    ...
case 0x101:
    ...
case 0x102:
    ...
}
```

combined with callers that feed its result into `PK8+0x50/+0x51`, is enough to assign a high-confidence semantic name such as:

```text
PmlResolveSizeScalar
```

or:

```text
ResolveSizeParamToScalar
```

---

## 9. Localizing the PML bounded-random interface

The normal-size branch around:

```text
main+0x767490
```

calls the same function with constants:

```text
129
128
```

and adds the results.

Conceptually:

```cpp
scalar = Random(129) + Random(128);
```

That makes:

```text
main+0x7660D0
```

a strong candidate for a bounded random interface.

Following its indirect call shows:

```text
main+0x7660D0
    ↓
GetPmlCore()
    ↓
[PmlCore + 0x288]
    ↓
indirect branch
```

So `main+0x7660D0` is not the RNG algorithm itself; it is a PML callback dispatcher/wrapper.

Recommended name:

```text
PmlRandomBounded
```

---

## 10. Localizing the PML raw-random interface

The corresponding raw wrapper is:

```text
main+0x7660B0
```

It dispatches through:

```text
PmlCore + 0x280
```

This path is used by the generic PID fallback.

Recommended name:

```text
PmlRandomRaw32
```

The paired offsets `+0x280` and `+0x288`, together with paired raw/bounded behavior, strongly support treating them as one random-provider interface family.

---

## 11. Localizing PML Core

Getter:

```text
main+0x766030
```

The ELF relocation chain resolves:

```text
main+0x2617358
    ↓ relocation/GOT
main+0x28F5BA0
    ↓
PmlCore*
```

Therefore:

```text
main+0x2617358
```

should not be named as RNG state.

It is a relocation/GOT-side anchor used to obtain the global PML Core pointer.

The runtime PML Core object was observed with an allocation size around:

```text
0x298 bytes
```

with the relevant callback slots:

```text
+0x280  raw random callback
+0x288  bounded random callback
```

Recommended name:

```text
main+0x766030 = GetPmlCore
```

---

## 12. Callback-installation localization

Once an indirect callback field is known, search for **writes into that field**, not just calls through it.

This is one of the strongest general techniques in this analysis.

For Sword 1.3.2:

```text
PmlCore+0x280 -> main+0x7D1320
PmlCore+0x288 -> main+0x7D1360
```

The associated relocation slots are:

```text
main+0x2617B88 -> main+0x7D1320
main+0x2617B90 -> main+0x7D1360
```

The PML initialization/config-copy path places these callbacks into the live PML object.

This resolves the indirect-call chain completely:

```text
PmlRandomRaw32
    -> [PmlCore+0x280]
    -> main+0x7D1320

PmlRandomBounded
    -> [PmlCore+0x288]
    -> main+0x7D1360
```

---

## 13. Hard confirmation from Nintendo SDK imports

`main+0x7D1320` directly calls:

```text
nn::crypto::GenerateCryptographicallyRandomBytes
```

with a 4-byte output buffer.

Conceptual reconstruction:

```cpp
uint32_t PmlCryptoRandom32()
{
    uint32_t value;
    nn::crypto::GenerateCryptographicallyRandomBytes(&value, 4);
    return value;
}
```

This is a hard anchor because it proves that the default PML raw provider is obtaining fresh cryptographic random bytes rather than advancing a game-owned xoroshiro/LCG state.

The bounded implementation at:

```text
main+0x7D1360
```

uses the same crypto source plus masking/rejection to produce a value below a requested bound.

Recommended names:

```text
main+0x7D1320 = PmlCryptoRandom32
main+0x7D1360 = PmlCryptoRandomBounded
```

---

## 14. Height / Weight randomization chain

The resulting complete chain is:

```text
EggCreateParam.HeightParam = 0x0100
        ↓
main+0x766F04
        ↓
main+0x7672E0
        ↓
main+0x767490
        ↓
PmlRandomBounded(129)
PmlRandomBounded(128)
        ↓
PmlCore+0x288
        ↓
main+0x7D1360
        ↓
nn::crypto::GenerateCryptographicallyRandomBytes
        ↓
HeightScalar
        ↓
main+0x775700
        ↓
PK8+0x50
```

Weight is identical except:

```text
EggCreateParam.WeightParam = 0x0100
main+0x766F14
main+0x775850
PK8+0x51
```

Formula:

```text
HeightScalar = PmlBounded(129) + PmlBounded(128)
WeightScalar = PmlBounded(129) + PmlBounded(128)
```

---

## 15. Why PokeDocs helps with Personal accessors

The egg generator needs multiple properties that PokeDocs places in the same Pokémon personal-data concept family:

```text
species/form
gender ratio
ability slots
egg groups
hatch species
species base dimensions
```

Therefore when analyzing:

```text
main+0x789770
```

calls that:

- accept species/form-like values;
- access the same global table/object;
- return small `u8/u16` values;
- cluster together in address space;

should be grouped and tested as likely Personal accessor functions.

Suggested workflow:

```text
main+0x789770
  ↓ enumerate calls
  ↓ classify inputs/return width
  ↓ group same backing table/object
  ↓ compare usage against PokeDocs PersonalData concepts
  ↓ assign provisional names
  ↓ confirm through additional callers
```

This is much safer than naming a function from one call site alone.

---

## 16. eggSeed localization does not primarily depend on PokeDocs

The eggSeed chain is best found directly through data flow.

Generation:

```text
main+0xF28FE0
    ↓
nn::crypto::GenerateCryptographicallyRandomBytes(..., 8)
    ↓
main+0x14016F8
    ↓
[slot+0x2F8] = eggSeed
```

Pickup read:

```text
main+0x1401A20
    ↓
LDR X1,[slot+0x2F8]
    ↓
main+0xF28FC0
    ↓
initialize egg-local RNG
```

Then:

```text
eggSeed
    ↓
main+0x1400C70 egg RNG callback
    ↓
main+0x789770
```

This is a good example of where imported SDK calls and memory data-flow are stronger anchors than metadata schemas.

---

## 17. PID localization methodology

Start with the parameter field, not a string search.

Known parameter:

```text
EggCreateParam +0x18 = PID / PID sentinel
```

### Egg-specific PID path

Inside:

```text
main+0x789C20 .. main+0x789C88
```

Masuda / Shiny Charm uses the egg-local RNG to produce PID candidates.

Shiny check:

```text
main+0x777350
```

### Generic PID fallback

For same-language/no-Charm eggs:

```text
EggCreateParam+0x18 = 0x3FFFFFFFF
```

The generic constructor detects this unresolved sentinel in:

```text
main+0x7667A0
```

and reaches:

```text
main+0x7668C0
    ↓
main+0x7660B0
    ↓
PML raw callback
    ↓
main+0x7D1320
    ↓
Nintendo Crypto
```

This yields a robust semantic name:

```text
main+0x7668C0 = GenerateFallbackPid
```

---

## 18. Exact egg/PML boundary

The egg-specific parameter generator is called at:

```text
main+0x789664
    BL main+0x789770
```

After return:

```text
main+0x789668
```

`EggCreateParam` is complete.

Later:

```text
main+0x7896F0
    BL main+0x777F40
```

enters generic Pokémon construction.

Therefore:

```text
main+0x789668
```

is the best conceptual boundary between:

```text
eggSeed deterministic breeding domain
```

and:

```text
generic PML construction / Nintendo Crypto domain
```

For Masuda + Shiny Charm specifically, at this boundary:

```text
PID      fixed
EC       fixed
IV       fixed
Nature   fixed
Ability  fixed
Gender   fixed

Height   not generated yet
Weight   not generated yet
```

---

## 19. Runtime validation plan

Recommended breakpoints:

```text
main+0x14016F8   eggSeed write
main+0x1401A20   eggSeed pickup read
main+0x789668    EggCreateParam complete
main+0x7668C0    ordinary PID PML fallback
main+0x766F04    Height resolver
main+0x766F14    Weight resolver
main+0x7D1320    raw Nintendo Crypto-backed provider
main+0x7D1360    bounded Nintendo Crypto-backed provider
main+0x775700    HeightScalar final setter
main+0x775850    WeightScalar final setter
```

Expected Masuda+Charm trace:

```text
0x789668
  PID already valid
      ↓
(no 0x7668C0 PID fallback)
      ↓
0x766F04 Height
      ↓
0x7D1360 crypto provider
      ↓
0x766F14 Weight
      ↓
0x7D1360 crypto provider
```

Expected ordinary same-language/no-Charm trace:

```text
0x789668
  PID sentinel remains
      ↓
0x7668C0
      ↓
0x7D1320
      ↓
PID generated
      ↓
0x766F04 Height
      ↓
0x7D1360
      ↓
0x766F14 Weight
      ↓
0x7D1360
```

---

## 20. Recommended IDA names

Current high-confidence working names:

```text
0x1401530  DayCare_UpdateEggAvailability
0xF28FE0   GenerateEggSeed64
0xF28FC0   InitEggLocalRng

0x7893F0   CreateDayCareEgg
0x789500   BuildEggPokemon
0x789770   FillEggCreateParam

0x789C0C   GenerateEggEC
0x789C20   EggPidRollPolicy_Begin
0x777350   IsShinyByPidXor

0x777F40   Pml_CreatePokemon
0x7667A0   Pml_NormalizeInitialSpec

0x766030   GetPmlCore
0x7660B0   PmlRandomRaw32
0x7660D0   PmlRandomBounded

0x7668C0   GenerateFallbackPid
0x7672E0   ResolveSizeParamToScalar
0x767490   GenerateNormalSizeScalar

0x775700   SetHeightScalar
0x775850   SetWeightScalar

0x7D1320   PmlCryptoRandom32
0x7D1360   PmlCryptoRandomBounded
```

These are semantic research names, not claimed original Game Freak symbol names.

---

## 21. Evidence-strength rules

Use the following confidence hierarchy when naming Sword functions:

### Highest confidence

```text
Nintendo SDK import
+ exact data-flow
+ final field write
```

Example:

```text
0x7D1320 -> nn::crypto::GenerateCryptographicallyRandomBytes
```

### High confidence

```text
known PK8 field write
+ xref chain
+ stable input/output behavior
```

Example:

```text
0x775700 -> PK8+0x50
```

### Medium-high confidence

```text
PokeDocs semantic field cluster
+ multiple Sword callers
+ common backing table/object
```

Useful for Personal accessor families.

### Low confidence if used alone

```text
string/name similarity
single caller
one constant
one decompiler guess
```

Do not finalize names from these alone.

---

## 22. Generalized workflow for other Pokémon Switch titles

The same technique can be reused for Sword, Shield, PLA, SV and later titles:

```text
1. Use PokeDocs to learn the semantic data model.
2. Identify the final serialized/runtime field of interest.
3. Reverse from the writer/setter upward.
4. Identify resolver/default-policy functions.
5. Follow random/value providers.
6. Expand virtual/callback/indirect calls.
7. Find callback initialization writes.
8. Anchor concrete implementations on SDK imports.
9. Cross-check against sibling fields and modes.
10. Confirm with runtime breakpoints.
```

For RNG research, this prevents a common failure mode: assuming every random-looking value must come from one global PRNG state.

Sword 1.3.2 instead clearly separates:

```text
Day Care eggSeed deterministic RNG
```

from:

```text
PML callback -> Nintendo Crypto random provider
```

---

## 23. Final localization map

```text
PokeDocs semantic model
        │
        ├─ PersonalData concepts
        │   species/form/ability/gender/hatch/groups
        │
        └─ distinguish model height vs base height vs individual scalar
                    │
                    ▼
Sword main.elf
                    │
        ┌───────────┴────────────┐
        │                        │
Day Care / Egg               PK8 field writes
        │                        │
0x1401530                      +0x50 / +0x51
        │                        │
0xF28FE0                        ▼
        │                   0x775700 / 0x775850
slot+0x2F8                      │
        │                        ▼
0x1401A20                   0x766F04 / 0x766F14
        │                        │
0xF28FC0                        ▼
        │                   0x7672E0
Egg RNG                         │
        │                        ▼
0x789770                    0x767490
        │                        │
        │                        ▼
        │                   0x7660D0
        │                        │
        │                        ▼
        │                   PmlCore+0x288
        │                        │
        │                        ▼
        │                   0x7D1360
        │                        │
        │                        ▼
        │               Nintendo Crypto
        │
        ├─ EC @ 0x789C0C
        └─ Masuda/Charm PID @ 0x789C20..0x789C88
```

The most important methodological conclusion is:

> Use PokeDocs to understand *what type of Pokémon data concept to expect*, then use Sword's actual field writes, xrefs, callback installation and SDK imports to prove *which concrete function/address implements it*.
