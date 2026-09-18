# Pokémon Sword 1.3.2 — Fixed Encounter / Max Raid / Dynamax Adventure Deep Dive

Updated: 2026-09-18

Target:
- Pokémon Sword 1.3.2
- ELF: `main(3).elf`
- GNU Build ID: `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`
- Address notation: `main+offset`

This document continues the shared `GenerationSpec -> 0x7662A0 -> 0x777F40` analysis and closes several previously open questions, especially the relationship between Max Raid/Nest seed data and `GenerationSpec.fixedSeed`.

Research names in this document are semantic names, not claimed original Game Freak symbols.

---

## 1. New high-value conclusions

### CONFIRMED: Max Raid/Nest seed is passed directly into `GenerationSpec.fixedSeed`

At the caller around `main+0x1029868`, the active Nest runtime entry is addressed, and the 64-bit value at entry `+0x68` is loaded into `X2` before calling the raid builder:

```asm
main+0x1029868  MADD X24, X9, X10, X8
main+0x102986C  LDRB W8,[X24,#0x70]!
...
main+0x10298A0  LDUR X2,[X24,#-0x8]   ; runtime Nest entry +0x68
main+0x10298A4  LDRB W3,[X24,#0x1]    ; +0x71
main+0x10298A8  MOV  X8,SP
main+0x10298AC  MOV  X1,X23            ; entry +0x60 region
main+0x10298B0  MOV  W5,W22
main+0x10298B8  BL   main+0xEC7080
```

`main+0xEC7080` immediately preserves that third argument:

```asm
main+0xEC70B4  MOV X20,X2
```

The local GenerationSpec base is `X29-0xC8`. Multiple branches in the same raid builder then do:

```asm
; GenerationSpec.useFixedSeed = 1
main+0xEC776C  MOV   W9,#1
main+0xEC776C  ...
main+0xEC776C..0xEC7770
               STRB W9,[X29,#-0xC0]   ; spec +0x08
               STR  X20,[X10]         ; spec +0x00
```

and in another branch:

```asm
main+0xEC921C  MOV   W9,#1
main+0xEC9224  STRB  W9,[X29,#-0xC0]  ; spec +0x08
main+0xEC9228  STR   X20,[X10]         ; spec +0x00
```

Because `X20 == original X2`, the data flow is direct:

```text
Nest runtime entry +0x68
        ↓
X2 at main+0x10298B8
        ↓
main+0xEC7080
        ↓
X20
        ↓
GenerationSpec+0x00 = fixedSeed
GenerationSpec+0x08 = 1
        ↓
main+0x7662A0
        ↓
main+0x7667A0
        ↓
local fixed-seed RNG
```

This closes the previous open question: for this Max Raid/Nest path, the runtime raid seed is not merely correlated with the PML seed — it is copied directly into `GenerationSpec.fixedSeed`.

---

## 2. Raid builder entry and argument interpretation

The main high-confidence builder is:

```text
main+0xEC7080
```

Observed entry argument use:

```cpp
// semantic reconstruction
RaidBuildResult BuildRaidPokemon(
    uint32_t arg0,
    void* nestRuntimeRegion,  // X1, around entry +0x60
    uint64_t raidSeed,        // X2, CONFIRMED seed
    uint32_t arg3,
    uint32_t arg4,
    uint32_t arg5);
```

Inside:

```asm
main+0xEC70A0  MOV X9,X8
main+0xEC70A4  MOV W24,W5
main+0xEC70AC  ...
main+0xEC70B0  MOV W28,W3
main+0xEC70B4  MOV X20,X2    ; preserve raidSeed
main+0xEC70B8  MOV X21,X1
```

The exact semantic names of arguments 0/1/3/4/5 still need dynamic labeling, but `X2` as the deterministic seed is now confirmed by its direct write into the GenerationSpec.

---

## 3. Raid GenerationSpec initialization

The raid builder repeatedly creates a GenerationSpec at:

```text
spec = X29 - 0xC8
```

A representative initializer is:

```asm
main+0xEC770C  MOV X22,#-1
main+0xEC7710  MOV X9,#0x3FFFFFFFF
main+0xEC7714  SUB X10,X29,#0xC8
main+0xEC7718  STP X22,X9,[X10,#0x10]   ; EC / PID defaults
main+0xEC7720  STR X22,[X10,#0x20]      ; trainer/context default
...
main+0xEC7748  MOV W9,#0x1000100
...
main+0xEC775C  STUR W9,[X10,#0x4A]     ; size defaults
main+0xEC776C  MOV W9,#1
main+0xEC776C  STRB W9,[X29,#-0xC0]    ; useFixedSeed = 1
main+0xEC7770  STR X20,[X10]            ; fixedSeed = raidSeed
```

Therefore Raid/Nest differs fundamentally from the ordinary Egg path:

```text
Raid:
  fixedSeed = runtime raid seed
  useFixedSeed = 1

Egg:
  eggSeed drives egg-specific generation first
  generic PML local fixed-seed mode is normally not the same mechanism
```

---

## 4. Raid encounter record -> GenerationSpec mapping

A representative record pointer is held in `X28`. The code uses standard FlatBuffer vtable access. The offsets below are **vtable slots**, not direct object byte offsets.

High-confidence mapping around `main+0xEC7A80`:

| FlatBuffer vtable slot | Read form | GenerationSpec target | Interpretation |
|---:|---|---:|---|
| `+0x06` | `u32` | `+0x28` | Species |
| `+0x08` | `u32` -> `u16` | `+0x2C` | Form |
| `+0x0A` | `u16` | `+0x2E` | Level |
| `+0x24` | enum/byte | `+0x30` | Gender policy |
| `+0x2E` | byte | `+0x32` | Nature policy / nature value |
| `+0x1A` | enum/byte | `+0x34` | Ability policy |
| `+0x26` | byte | `+0x48` | Perfect-IV count / perfect-IV policy |
| `+0x28` | enum/byte + runtime condition | `+0x18` | PID / shiny policy |

Representative species/form/level writes:

```asm
; Species -> spec+0x28
... FlatBuffer field +0x06 ...
STUR W8,[X29,#-0xA0]

; Form -> spec+0x2C
... FlatBuffer field +0x08 ...
STURH W8,[X29,#-0x9C]

; Level -> spec+0x2E
... FlatBuffer field +0x0A ...
STURH W19,[X29,#-0x9A]
```

Because `spec = X29-0xC8`:

```text
-X29 offset -0xA0 => spec+0x28
-X29 offset -0x9C => spec+0x2C
-X29 offset -0x9A => spec+0x2E
```

---

## 5. Raid ability policy encoding

The record field selected through vtable slot `+0x1A` is constrained to `0..4` and transformed through a packed constant before being stored at `spec+0x34`.

The packed mapping is equivalent to:

```text
record enum 0 -> 0x00
record enum 1 -> 0x01
record enum 2 -> 0x02
record enum 3 -> 0xFF
record enum 4 -> 0xFE
```

The final semantics of `0xFE/0xFF` are resolved downstream by `main+0x7667A0`; they should be treated as policy sentinels rather than literal ability slot numbers.

---

## 6. Raid PID / shiny policy

The raid path writes special 64-bit PID policy sentinels into `GenerationSpec+0x18`.

Observed values include:

```text
0x1FFFFFFFF
0xFFFF0002FFFFFFFF
0x3FFFFFFFF   (generic random sentinel in initial state)
```

The shared resolver proves at least two downstream meanings:

### `0x1FFFFFFFF`: force non-shiny behavior

At `main+0x7669B8`, a random PID candidate is obtained and passed to:

```text
main+0x777370
```

`main+0x777370` checks shiny XOR and flips PID bit 28 only when the candidate is shiny:

```asm
main+0x777384  EOR W9,W1,#0x10000000
main+0x777388  CMP W8,#0x10
main+0x77738C  CSEL W0,W9,W1,LO
```

So this policy preserves a naturally non-shiny candidate, but alters a shiny candidate into a non-shiny PID.

### `0xFFFF0002FFFFFFFF`: force-shiny behavior

The resolver routes this sentinel through:

```text
main+0x7773A0
```

which preserves an already-shiny PID and otherwise rewrites the PID high half so the trainer/PID XOR falls into the shiny range.

Thus these two special policies can now be named with substantially higher confidence:

```text
0x1FFFFFFFF          -> force / guarantee non-shiny
0xFFFF0002FFFFFFFF   -> force / guarantee shiny
```

`0x3FFFFFFFF` remains the generic unresolved/random PID sentinel used by the normal generation flow.

The exact role of the separate `0x2FFFFFFFF` policy remains to be fully labeled; do not collapse it into either of the above without runtime validation.

---

## 7. Raid uses the same seed in more than one local context

The raid builder not only writes `raidSeed` into `GenerationSpec.fixedSeed`, it also initializes a local xoroshiro context from the same seed during upstream raid construction.

Representative code:

```asm
main+0xEC78B0  SUB X0,X29,#0x70
main+0xEC78B4  MOV X1,X20        ; X20 = raidSeed
main+0xEC78B8  BL  main+0xF28FC0
```

`X29-0x70` corresponds to a local state region later used during raid table/policy selection.

This means the raid builder conceptually has two deterministic uses of the same seed:

```text
raidSeed
   ├─ initialize raid-builder local RNG
   │      -> encounter-table / policy decisions
   │
   └─ GenerationSpec.fixedSeed
          -> shared Pokémon resolver local RNG
```

These are separately initialized streams, not necessarily one continuously advanced RNG object.

This distinction is important for a faithful Raid Seed Searcher: upstream raid-table consumption and downstream PML generation must be modeled as their actual separate initialization points.

---

# 8. Dynamax Adventure: fixedSeed generation is also deterministic, but owned by a different RNG state

The Dynamax Adventure / Underground Exploration builder around:

```text
main+0x15DD330
```

constructs a GenerationSpec at:

```text
spec = SP + 0x10
```

The crucial seed code is:

```asm
main+0x15DD42C  LDR X8,[X21,#0x2AB8]
main+0x15DD430  LDR X9,[X21,#0x2AC0]
main+0x15DD434  ADD X10,X9,X8        ; xoroshiro128+ output
main+0x15DD438  EOR X9,X9,X8
main+0x15DD43C  EOR X8,X9,X8,ROR#40
main+0x15DD440  EOR X8,X8,X9,LSL#16
main+0x15DD444  ROR X9,X9,#0x1B
main+0x15DD448  STR X8,[X21,#0x2AB8]
main+0x15DD450  STR X9,[X21,#0x2AC0]
main+0x15DD454  STR X10,[SP,#0x10]   ; spec.fixedSeed
main+0x15DD458  STRB W8,[SP,#0x18]   ; spec.useFixedSeed = 1
```

The byte written at `SP+0x18` is set to `1` immediately before this sequence.

Therefore:

```text
Underground Exploration object
  RNG state0 = +0x2AB8
  RNG state1 = +0x2AC0
          ↓
      xoroshiro128+
          ↓
 next output = state0 + state1
          ↓
GenerationSpec.fixedSeed
GenerationSpec.useFixedSeed = 1
```

This is a major difference from Max Raid:

```text
Max Raid:
    stored runtime Nest seed -> directly copied to fixedSeed

Dynamax Adventure:
    persistent DA xoroshiro state -> one Next() output -> fixedSeed
```

So a Max Raid seed searcher and a Dynamax Adventure state searcher must be separate implementations.

---

## 9. Dynamax Adventure species/form/level mapping

With `spec = SP+0x10`, the builder writes:

```asm
; species
main+0x15DD3D4  STR W10,[SP,#0x38]    ; spec+0x28

; form
main+0x15DD400  STRH W10,[SP,#0x3C]   ; spec+0x2C

; level
main+0x15DD428  STRH W8,[SP,#0x3E]    ; spec+0x2E
```

The corresponding FlatBuffer record slots are:

```text
vtable +0x14 -> Species
vtable +0x0A -> Form
vtable +0x12 -> Level
```

The same builder initializes normal GenerationSpec defaults including:

```text
PID policy default  = 0x3FFFFFFFF
Ability policy      = sentinel/default
Size policies       = 0x0100 / 0x0100
```

before applying Dynamax Adventure record overrides.

---

## 10. Dynamax Adventure IV handling

The region `main+0x15DD4D0..0x15DD66C` writes directly into:

```text
spec+0x36
spec+0x38
spec+0x3A
spec+0x3C
spec+0x3E
spec+0x40
```

which are the six IV policy fields.

Special record values such as `0xFF`, `-1` and values below `-1` are treated as sentinels rather than literal IVs. The builder can also write `spec+0x48` (PerfectIVCount), so the DA encounter record supports both explicit-IV and guaranteed-perfect-IV style policies.

This should be preserved exactly when implementing an Adventure predictor; normalizing all record IV fields to literal `0..31` would lose generation semantics.

---

## 11. Dynamax Adventure generation boundary

After the spec is finished:

```asm
main+0x15DD6D4  ADD X2,SP,#0x10
main+0x15DD6D8  MOV X0,X23
main+0x15DD6DC  MOV X1,X25
main+0x15DD6E0  BL  main+0x7662A0
```

So the complete deterministic core is:

```text
DA persistent RNG state (+0x2AB8/+0x2AC0)
        ↓ one xoroshiro advance
fixedSeed
        ↓
DA encounter record
        ↓
GenerationSpec
        ↓
main+0x15DD6E0
        ↓
main+0x7662A0
        ↓
main+0x777F40
```

---

# 12. Scripted fixed encounter builder `main+0xE4AA30`

The fixed/event encounter path is now mapped in more detail.

```text
event_encount_data.bin
      ↓
main+0xE4A470
      ↓
main+0xE4AA30
      ↓
GenerationSpec
      ↓
main+0xE4A570 -> main+0x7662A0
```

`main+0xE4AA30` takes:

```text
X1 = FlatBuffer event encounter record
X2 = GenerationSpec destination
```

and copies/transforms fields directly.

---

## 13. Fixed-event record -> GenerationSpec map

Observed map:

| FlatBuffer vtable slot | GenerationSpec | Behavior |
|---:|---:|---|
| `+0x16` | `+0x28` | Species |
| `+0x06` | `+0x2C` | Form |
| `+0x14` | `+0x2E` | Level |
| `+0x18` | `+0x20` | Trainer/shiny context value |
| `+0x26` | `+0x18` | PID policy enum |
| `+0x2A` | `+0x30` | Gender policy enum |
| `+0x28` | `+0x32` | Nature / random-nature sentinel |
| `+0x38` | `+0x34` | Ability policy enum |
| `+0x32` etc. | `+0x36..+0x40`, `+0x48` | IV/perfect-IV encoding |

Species/form/level are direct:

```asm
main+0xE4AA70  STR  W8,[X19,#0x28]
main+0xE4AA9C  STRH W8,[X19,#0x2C]
main+0xE4AAC8  STRH W8,[X19,#0x2E]
```

---

## 14. Fixed-event trainer context special marker

The builder reads a 64-bit event value through FlatBuffer vtable slot `+0x22` and compares it with:

```text
0xCBF29CE484222645
```

This is the well-known FNV-1a/FNV-1 64-bit offset basis constant.

When the event value equals this marker, the builder obtains the current trainer value via:

```text
main+0x1424D00
```

and writes it to:

```text
GenerationSpec+0x20
```

Otherwise it uses the event-record-provided trainer/context value.

Semantically:

```text
fixed event trainer field
      ├─ explicit value -> use record value
      └─ special marker -> use current player's trainer value
```

The exact source schema field name remains open, but the runtime behavior is clear.

---

## 15. Fixed-event PID policy enum

At `main+0xE4ABB8..0xE4ABDC`, an enum `0..2` from event data is transformed as:

```cpp
pidPolicy = 0x3FFFFFFFFULL - (uint64_t(enumValue) << 32);
```

Thus:

```text
enum 0 -> 0x3FFFFFFFF
enum 1 -> 0x2FFFFFFFF
enum 2 -> 0x1FFFFFFFF
```

From the shared resolver we can now confidently identify:

```text
0x1FFFFFFFF -> guarantee non-shiny behavior
```

while `0x3FFFFFFFF` is the generic random/unresolved PID path.

`0x2FFFFFFFF` remains a distinct policy and should not yet be renamed without additional dynamic confirmation.

---

## 16. Fixed-event gender / nature / ability policy encoding

### Gender

Event enum at vtable slot `+0x2A` maps to:

```text
0 -> 0x00FF   ; random/default sentinel
1 -> 0
2 -> 1
```

stored at:

```text
GenerationSpec+0x30
```

### Nature

Event nature field at vtable slot `+0x28` behaves as:

```text
0..24 -> literal nature
25    -> 0xFFFF random-nature sentinel
```

stored at:

```text
GenerationSpec+0x32
```

### Ability

Event ability enum at vtable slot `+0x38` maps as:

```text
0 -> 0xFF
1 -> 0
2 -> 1
3 -> 2
```

stored at:

```text
GenerationSpec+0x34
```

This is a particularly useful direct mapping for reconstructing `event_encount_data.bin` semantics.

---

## 17. Fixed-event IV encoding

The event builder supports two styles:

```text
A. explicit six IV policies/values
B. guaranteed-perfect-IV count encoding
```

For the first IV-related signed byte read through vtable slot `+0x32`:

```text
value == -1       -> IV sentinel 0xFFFF
value <= -2       -> PerfectIVCount = -value - 1
otherwise         -> literal IV policy/value
```

Examples:

```text
-2 -> 1 guaranteed perfect IV
-3 -> 2 guaranteed perfect IVs
-4 -> 3
-5 -> 4
-6 -> 5
-7 -> 6
```

When explicit IV mode is used, subsequent event fields populate:

```text
GenerationSpec+0x36
GenerationSpec+0x38
GenerationSpec+0x3A
GenerationSpec+0x3C
GenerationSpec+0x3E
GenerationSpec+0x40
```

and a record value `0xFF/-1` is converted into the `0xFFFF` random-IV sentinel.

This is strong evidence that fixed-event records can express both “N guaranteed 31 IVs” and per-stat fixed/random IV policies.

---

# 18. Scripted Gift/AddPoke builder `main+0xE48D70`

The AddPoke/Gift path uses a separate record-to-spec builder.

Caller initialization around `main+0xE489C8` creates a default GenerationSpec on the stack and explicitly leaves:

```text
fixedSeed    = 0
useFixedSeed = 0
```

then calls:

```text
main+0xE48D70
```

and finally:

```text
main+0xE48A98 -> main+0x7662A0
```

Therefore random fields in this path normally do **not** have the raid-style deterministic `fixedSeed` source. If unresolved, they fall through to the generic PML provider behavior.

---

## 19. Gift/AddPoke record mapping

Confirmed/high-confidence writes in `main+0xE48D70`:

| FlatBuffer vtable slot | GenerationSpec | Meaning |
|---:|---:|---|
| `+0x18` | `+0x28` | Species |
| `+0x12` | `+0x2C` | Form |
| — | `+0x2E` | **Level is forced to 50** |
| `+0x1C` | `+0x18` | PID policy enum via lookup table |
| `+0x20` | `+0x30` | Gender policy |
| `+0x1E` | `+0x32` | Nature |
| `+0x2E` | `+0x34` | Ability policy |
| `+0x28,+0x24,+0x26,+0x2A,+0x2C,+0x22` | `+0x36..+0x40` | six IV policies |

The fixed level is explicit:

```asm
main+0xE48DC8  MOV  W8,#0x32
main+0xE48DCC  STRH W8,[X2,#0x2E]
```

So this particular AddPoke record format creates level-50 Pokémon regardless of a level field in the record.

---

## 20. Gift PID policy lookup table

The lookup table used by the Gift/AddPoke path contains exactly:

```text
index 0 -> 0x3FFFFFFFF
index 1 -> 0x1FFFFFFFF
index 2 -> 0x2FFFFFFFF
```

This ordering differs from the arithmetic encoding used by `main+0xE4AA30` fixed-event encounters:

```text
Fixed event E4AA30:
0 -> 0x3FFFFFFFF
1 -> 0x2FFFFFFFF
2 -> 0x1FFFFFFFF

Gift E48D70:
0 -> 0x3FFFFFFFF
1 -> 0x1FFFFFFFF
2 -> 0x2FFFFFFFF
```

Therefore enum values from different resource schemas must **not** be assumed to share identical numeric semantics even though they eventually map into the same `GenerationSpec.pidPolicy` domain.

---

## 21. Gift nature / gender / ability / IV behavior

Gift mappings closely mirror the common policy model:

```text
Gender record enum:
0 -> 0xFF
1 -> 0
2 -> 1

Nature:
0..24 -> literal nature
25    -> 0xFFFF random

Ability record enum:
0 -> 0
1 -> 1
2 -> 2
3 -> 0xFF
4 -> 0xFE
```

Each Gift IV field treats `-1` as:

```text
0xFFFF random-IV sentinel
```

while other values are copied into the corresponding spec IV field.

The Gift caller initializes both size parameters to:

```text
0x0100 / 0x0100
```

so, unless overridden later, Height/Weight follow the normal generic random-size path.

Because `useFixedSeed=0` here, those unresolved random fields use the PML default provider rather than a deterministic local fixed-seed stream.

---

# 22. Revised RNG ownership model

The current generation systems can now be separated more precisely:

```text
Ordinary Field
  outer Field RNG
      ↓
  Encounter selection/policies
      ↓
  GenerationSpec
      ↓
  optional inner fixedSeed stream

Egg
  Day Care eggSeed RNG
      ↓
  EggCreateParam
      ↓
  unresolved fields may fall to PML Crypto

Max Raid/Nest
  stored Nest runtime seed (+0x68)
      ├─ raid-builder local RNG initialized from same seed
      └─ GenerationSpec.fixedSeed = same seed

Dynamax Adventure
  persistent Adventure RNG
  state @ object+0x2AB8/+0x2AC0
      ↓ one xoroshiro128+ output
  GenerationSpec.fixedSeed

Script Fixed Encounter
  event record policies
      ↓
  GenerationSpec
  normally no evidence here yet that all events own a separate stored encounter seed

Gift/AddPoke
  gift record policies
      ↓
  GenerationSpec with useFixedSeed=0
      ↓
  unresolved random values can use PML Crypto
```

This confirms that a single universal `NextPokemonSeed()` abstraction would be incorrect for Sword/Shield.

---

# 23. Seed-searcher design consequences

The implementation should be split by source domain.

### Max Raid Seed Searcher

Input should be the stored 64-bit Nest seed from runtime record `+0x68`.

Reproduce:

```text
seed
  ├─ raid-builder local RNG decisions
  └─ shared GenerationSpec fixedSeed RNG
```

Do not advance one stream continuously unless the game does so at that exact site; the builder reinitializes local contexts from the same seed.

### Dynamax Adventure State Searcher

Input is a 128-bit state:

```text
s0 = object+0x2AB8
s1 = object+0x2AC0
```

One `xoroshiro128+` next output becomes the fixed seed for the Pokémon spec.

Therefore searching only a 64-bit Pokémon `fixedSeed` loses the natural relationship to the Adventure route/session RNG state.

### Fixed Encounter Predictor

The main task is not seed search first; it is exact event-record policy decoding:

```text
event record
    ↓
E4AA30
    ↓
GenerationSpec
    ↓
shared resolver
```

### Gift/AddPoke Predictor

Because the observed path leaves `useFixedSeed=0`, a read-only pre-generation predictor cannot derive future PML Crypto outputs from a game-owned PRNG state. Deterministic preview would require either observation at construction time or a reserve/replay design similar to the Egg Height/Weight case.

---

# 24. Recommended hook / breakpoint points

| Purpose | Address |
|---|---:|
| Raid builder entry / seed arg in `X2` | `main+0xEC7080` |
| Raid generic generation branch 1 | `main+0xEC7CA8` |
| Raid generic generation branch 2 | `main+0xEC877C` |
| Raid generic generation branch 3 | `main+0xEC950C` |
| Runtime Nest seed load before builder | `main+0x10298A0` |
| Fixed-event record -> spec | `main+0xE4AA30` |
| Fixed-event generic generation | `main+0xE4A570` |
| Gift record -> spec | `main+0xE48D70` |
| Gift generic generation | `main+0xE48A98` |
| DA persistent RNG advance | `main+0x15DD42C` |
| DA fixedSeed write | `main+0x15DD454` |
| DA generic generation | `main+0x15DD6E0` |
| Generic spec generation entry | `main+0x7662A0` |
| Shared PML boundary | `main+0x777F40` |
| Shared spec resolver | `main+0x7667A0` |

For Raid dynamic validation, the best first breakpoint pair is:

```text
main+0x10298A0
main+0xEC7080
```

Verify:

```text
*(u64*)(NestEntry+0x68) == X2 at EC7080
```

then break before each `0x7662A0` call and confirm:

```text
GenerationSpec+0x00 == X2
GenerationSpec+0x08 == 1
```

This provides a direct runtime proof of the static data-flow result.

---

# 25. Current status

### Newly confirmed in this pass

- Max Raid/Nest runtime seed is loaded from the active Nest entry at `+0x68` and passed as `X2` into `main+0xEC7080`.
- `main+0xEC7080` writes that exact seed into `GenerationSpec.fixedSeed` and sets `useFixedSeed=1` across multiple raid branches.
- Raid builder also initializes a separate local RNG context from the same seed.
- `0x1FFFFFFFF` is a force-non-shiny PID policy with high confidence from `main+0x777370` behavior.
- `0xFFFF0002FFFFFFFF` is a force-shiny PID policy with high confidence from `main+0x7773A0` behavior.
- Dynamax Adventure owns a persistent xoroshiro state at object `+0x2AB8/+0x2AC0`; one Next output becomes `GenerationSpec.fixedSeed`.
- `main+0xE4AA30` fixed-event policy mappings are substantially decoded, including species/form/level, trainer-context marker, PID policy, gender, nature, ability and IV/perfect-IV encoding.
- `main+0xE48D70` Gift/AddPoke mappings are substantially decoded; the observed builder forces level 50 and normally leaves `useFixedSeed=0`.

### Still open

- Exact semantic name/meaning of `0x2FFFFFFFF` PID policy.
- Full Nest runtime struct naming around `+0x60..+0x71` beyond the confirmed `+0x68` seed data flow.
- Exact field names in the source FlatBuffer schemas for every raid/event/gift vtable slot.
- Which individual story gift events use the `E48D70` level-50 AddPoke format versus other gift constructors.
- Exact source and lifecycle/advance rules for the stored Nest seed before `main+0x10298A0` reads it.
- Full mapping from Adventure route/session selection state to every encounter-table choice before `main+0x15DD330`.
