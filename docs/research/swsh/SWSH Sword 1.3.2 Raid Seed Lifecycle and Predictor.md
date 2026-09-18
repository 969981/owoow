# Pokémon Sword 1.3.2 — Raid Seed Lifecycle, Persistence and Predictor

Updated: 2026-09-18

Target:
- Pokémon Sword 1.3.2
- ELF: `main(3).elf`
- GNU Build ID: `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`
- Address notation: `main+offset`

This document closes the lifecycle of the 64-bit Max Raid / G-Nest seed stored at `NestEntry+0x68`: where it lives, how a fresh seed is created, how an existing seed advances, how bulk refresh code touches the den records, and how the value is consumed by the Raid builder and shared Pokémon generator.

Research names are semantic reconstruction names, not claimed original Game Freak symbols.

---

## 1. Executive conclusion

The per-den Raid seed has two fundamentally different transitions:

```text
Fresh activation / reseed:
    Nintendo Crypto64
        ↓
    NestEntry+0x68

Existing-den advance:
    oldSeed
      ↓
    initialize xoroshiro128+ with
      s0 = oldSeed
      s1 = 0x82A2B175229D6A5B
      ↓
    first output = s0 + s1
      ↓
    NestEntry+0x68 = oldSeed + 0x82A2B175229D6A5B (mod 2^64)
```

Therefore, once a den already has a stored seed, its persistent seed frame progression is:

```cpp
static constexpr uint64_t RAID_ADVANCE = 0x82A2B175229D6A5BULL;

uint64_t NextRaidSeed(uint64_t seed)
{
    return seed + RAID_ADVANCE;
}
```

The important distinction is:

- the first/new seed is not predictable before the Nintendo crypto call;
- every ordinary deterministic advance after that is exactly predictable from the stored seed;
- the seed is stored in save-backed Nest data and later copied directly into `GenerationSpec.fixedSeed`.

---

## 2. Save-backed Nest storage

The game obtains the common save root through the global pointer entry around:

```text
main+0x2610798
```

Raid/Nest code repeatedly does:

```asm
ADRP X8,0x2610000
LDR  X8,[X8,#0x798]
LDR  X8,[X8]
LDR  X19,[X8,#0x248]
```

Semantic reconstruction:

```cpp
saveRoot = *(void***)(main + 0x2610798);
nestBlock = *(void**)((uint8_t*)saveRoot + 0x248);
```

So the Nest data is not only a temporary field object. `saveRoot+0x248` points to mutable save-backed Nest state.

This does **not** mean each setter immediately writes the cartridge/save file. The setters mutate the save-backed in-memory block; normal game save serialization is responsible for disk persistence.

---

## 3. NestEntry addressing and layout

The common address formula appears repeatedly:

```asm
MOV    W10,#0xA68
SMADDL X10,W1,W10,X0
MOV    W11,#0x18
MADD   X9,X9,X11,X10
```

Semantic form:

```cpp
NestEntry* entry =
    nestBlock
    + areaIndex * 0xA68
    + denIndex  * 0x18;
```

A scan routine at `main+0x1426260` walks `0xA50` bytes in `0x18` byte increments. Since:

```text
0xA50 / 0x18 = 110
```

one area contains 110 den records.

Current reconstructed fields around each record are:

| Relative field | Type | Meaning |
|---:|---|---|
| `+0x60` | `u64` | den/event identifier or key, high confidence |
| `+0x68` | `u64` | persistent Raid seed, confirmed |
| `+0x70` | `u8` | den state/grade field A, semantic name provisional |
| `+0x71` | `u8` | den state/grade field B, semantic name provisional |
| `+0x72` | packed state | low byte + additional bits in the following byte |

The exact user-facing semantics of `+0x70/+0x71/+0x72` are still being separated. The `+0x68` seed interpretation is confirmed by both writers and Raid-builder consumers.

---

## 4. Exact Raid seed setter: `main+0x14260F0`

The per-den seed write has a tiny dedicated accessor:

```asm
main+0x14260F0  LDRSW X9,[X2]
main+0x14260F4  MOV   W10,#0xA68
main+0x14260F8  SMADDL X10,W1,W10,X0
main+0x14260FC  MOV   W11,#0x18
main+0x1426100  MADD  X9,X9,X11,X10
main+0x1426104  LDR   X8,[X3]
main+0x1426108  STR   X8,[X9,#0x68]
main+0x142610C  RET
```

Semantic prototype:

```cpp
void SetNestSeed64(
    void* nestBlock,
    int areaIndex,
    const int* denIndex,
    const uint64_t* seed)
{
    auto* entry = (uint8_t*)nestBlock
                + areaIndex * 0xA68
                + (*denIndex) * 0x18;

    *(uint64_t*)(entry + 0x68) = *seed;
}
```

A direct BL audit finds exactly three callers:

```text
main+0xF0A6EC
main+0x14B9094
main+0x14B9200
```

This is extremely useful because the complete seed lifecycle can be classified by examining only these writer paths.

---

## 5. Writer class A: fresh Crypto64 initialization / reseed

The function around:

```text
main+0x14B8ED0
```

selects a den and gives it a fresh 64-bit seed.

It retrieves the save/Nest block:

```asm
main+0x14B8F2C  ADRP X8,0x2610000
main+0x14B8F30  LDR  X8,[X8,#0x798]
main+0x14B8F34  LDR  X24,[X8]
...
main+0x14B8F44  LDR  X22,[X24,#0x248]
```

It obtains the current area through:

```text
main+0xEC6BE0
```

and eventually selects a den index saved at:

```text
SP+0x34
```

The fresh seed is generated here:

```asm
main+0x14B907C  BL  main+0xF28FE0
main+0x14B9080  STR X0,[SP,#0x10]
main+0x14B9084  ADD X2,SP,#0x34
main+0x14B9088  ADD X3,SP,#0x10
main+0x14B908C  MOV X0,X21
main+0x14B9090  MOV W1,W20
main+0x14B9094  BL  main+0x14260F0
```

`main+0xF28FE0` is the previously analyzed wrapper that requests eight random bytes from:

```text
nn::crypto::GenerateCryptographicallyRandomBytes
```

The null/fallback path performs the same operation:

```asm
main+0x14B91E8  BL  main+0xF28FE0
main+0x14B91EC  STR X0,[SP,#0x10]
...
main+0x14B9200  BL  main+0x14260F0
```

Therefore:

```text
Fresh selected-den seed
    = Nintendo cryptographic random 64-bit value
```

It is not computed from the previous den seed and is not derived from a visible game-side global PRNG.

The precise player-visible action represented by `0x14B8ED0` should still be dynamically labeled before naming it specifically as a particular UI action such as a Wishing Piece. The safe current name is:

```text
Selected-den fresh activation/reseed path
```

---

## 6. Post-reseed state derivation

Immediately after writing the fresh seed, the same path reads `entry+0x68` back and initializes a local xoroshiro context:

```asm
main+0x14B90C4  LDRSW X8,[SP,#0x34]
main+0x14B90C8  MOV   W9,#0xA68
main+0x14B90CC  SMADDL X9,W20,W9,X22
main+0x14B90D0  MOV   W10,#0x18
main+0x14B90D4  MADD  X8,X8,X10,X9
main+0x14B90D8  LDR   X1,[X8,#0x68]
main+0x14B90DC  ADD   X0,SP,#0x18
main+0x14B90E0  BL    main+0xF28FC0
```

The following inline bounded RNG performs two independent `0..999` draws using mask/rejection:

```asm
; first draw
main+0x14B90F4  LDP X8,X9,[SP,#0x18]
...
main+0x14B910C  AND W23,W10,#0x3FF
main+0x14B9110  CMP W23,#0x3E7
main+0x14B9114  B.HI retry

; second draw
main+0x14B911C  ...
main+0x14B9130  AND W25,W10,#0x3FF
main+0x14B9134  CMP W25,#0x3E7
main+0x14B9138  B.HI retry
```

This proves that the newly created seed is immediately used to derive additional den state. The exact mapping of those two values to rare/common presentation/state fields is left provisional until runtime verification.

---

## 7. Writer class B: deterministic existing-seed advance

The other writer is inside:

```text
main+0xF0A5E0
```

This function loads the existing seed, initializes the standard local xoroshiro context, takes the first output, and writes that output back to the same `+0x68` field.

Relevant code:

```asm
main+0xF0A674  MOV   X8,#0x8960
main+0xF0A678  MOVK  X8,#0x79F1,LSL#16
main+0xF0A67C  MOVK  X8,#0x17A5,LSL#32
main+0xF0A680  MOVK  X8,#0x0F4B,LSL#48
main+0xF0A684  STR   X8,[SP,#0x8]

main+0xF0A688  MOV   X8,#0x6A5B
main+0xF0A68C  MOVK  X8,#0x229D,LSL#16
main+0xF0A690  MOVK  X8,#0xB175,LSL#32
main+0xF0A694  MOVK  X8,#0x82A2,LSL#48
main+0xF0A698  STP   X8,XZR,[SP,#0x10]

main+0xF0A69C  LDRSW X8,[X20]
main+0xF0A6A0  MOV   W9,#0xA68
main+0xF0A6A4  SMADDL X9,W21,W9,X19
main+0xF0A6A8  MOV   W10,#0x18
main+0xF0A6AC  MADD  X8,X8,X10,X9
main+0xF0A6B0  LDR   X1,[X8,#0x68]  ; oldSeed
main+0xF0A6B4  ADD   X0,SP,#0x8
main+0xF0A6B8  BL    main+0xF28FC0
```

`main+0xF28FC0` initializes:

```text
s0 = oldSeed
s1 = 0x82A2B175229D6A5B
```

Then the function executes the xoroshiro transition. Critically, the value saved back as the new persistent seed is the **pre-transition xoroshiro128+ output**:

```asm
main+0xF0A6BC  LDP X8,X9,[SP,#0x8]
main+0xF0A6C0  EOR X10,X9,X8
...
main+0xF0A6D0  ADD X8,X9,X8       ; output = s0 + s1
...
main+0xF0A6E8  STR X8,[SP,#0x28]
main+0xF0A6EC  BL  main+0x14260F0
```

Therefore the stored seed recurrence collapses to the very simple equation:

```text
seed[n+1] = seed[n] + 0x82A2B175229D6A5B mod 2^64
```

The state transition instructions are real, but the advanced two-word xoroshiro state is **not** what gets persisted into `NestEntry+0x68`. The first output is persisted, and the next invocation reinitializes from that 64-bit value again.

---

## 8. Why the persistent sequence is an additive frame sequence

Let:

```text
C = 0x82A2B175229D6A5B
```

Each call to the persistent-den advance function starts from:

```text
s0 = currentSeed
s1 = C
```

The xoroshiro128+ output before state transition is:

```text
output = s0 + s1
```

and the game stores that output as the new seed.

So:

```text
seed1 = seed0 + C
seed2 = seed1 + C = seed0 + 2*C
seed3 = seed0 + 3*C
...
```

General form:

```cpp
uint64_t RaidSeedAtFrame(uint64_t seed0, uint64_t frame)
{
    return seed0 + frame * 0x82A2B175229D6A5BULL;
}
```

All operations are modulo `2^64` by normal unsigned 64-bit overflow.

Backward step:

```cpp
seed_prev = seed_now - 0x82A2B175229D6A5BULL;
```

Because `C` is odd, addition by `C` is a full-period permutation of the 64-bit space. Over `2^64` advances, every 64-bit value appears exactly once before the sequence repeats.

For solving a frame difference directly from two known seeds, the multiplicative inverse is:

```text
C^-1 mod 2^64 = 0xC855099EEB5DB5D3
```

so:

```cpp
uint64_t FrameDelta(uint64_t fromSeed, uint64_t toSeed)
{
    return (toSeed - fromSeed) * 0xC855099EEB5DB5D3ULL;
}
```

This is modulo `2^64`. In normal use, a bounded forward-frame search is safer when user-facing event semantics matter.

---

## 9. `main+0xF0A5E0` is more than a numeric increment

The same function also updates den state fields around the seed transition.

Direct callees include:

```text
main+0x14261D0  read byte/state at entry+0x72
main+0x1426130  write entry+0x71
main+0x1426110  write entry+0x70
main+0x1426150  write entry+0x72 and related packed bit handling
```

So a faithful “advance one den frame” implementation should distinguish:

```text
mathematical seed advance
```

from:

```text
full game den-refresh operation
```

The first is exactly `seed += C`. The second additionally refreshes den state/presentation/table-related fields.

For an offline Pokémon outcome searcher, the seed equation may be sufficient if the relevant Raid table/policies are known. For exact in-game state emulation, the surrounding state updates must also be modeled.

---

## 10. Direct callers of the advance routine

Direct BL callers of `main+0xF0A5E0` are:

```text
main+0xF0ABEC
main+0xF0AC4C
main+0x10270B4
main+0x1027108
main+0x14B928C
main+0x14B92CC
```

A wrapper around:

```text
main+0x1027020
```

invokes `main+0xF0A5E0` with the control argument `W5=1`.

Known callers of that wrapper include:

```text
main+0x10233FC
main+0x1026BF4
```

These are useful runtime breakpoints for separating field/UI-triggered single-den advance operations from bulk refreshes.

Exact user-facing action names remain provisional until runtime traces attach these functions to specific interactions.

---

## 11. Bulk refresh orchestration: `main+0xF0A950`

The larger routine:

```text
main+0xF0A950
```

retrieves the same save-backed Nest block and iterates den records for an area.

It calls `main+0xF0A5E0` from:

```text
main+0xF0ABEC
main+0xF0AC4C
```

while iterating den indices through the 110-entry area.

There are many direct callers of `F0A950`, including:

```text
main+0xF06FDC
main+0xF06FEC
main+0xF06FFC
main+0xF0729C
main+0xF072AC
main+0x1027204
main+0x102722C
main+0x1426498
main+0x14264BC
main+0x1426500
main+0x1426524
main+0x1426568
main+0x142658C
```

The orchestration around `main+0xF06F60` calls `F0A950` for area indexes `0`, `1`, and `2`. Similar multi-area orchestration exists around `main+0xF07280` and `main+0x1426440`.

This is strong evidence for a global/area Nest refresh operation. However, the exact real-world trigger — for example day rollover versus another save/event transition — should not be assigned purely from static evidence. The current safe semantic name is:

```text
BulkNestAreaRefresh
```

---

## 12. Additional per-area entropy/history fields in the bulk refresh

`main+0xF0A950` also contains this sequence:

```asm
main+0xF0AA6C  MOV   W8,#0xA68
main+0xF0AA70  SMADDL X20,W21,W8,X19
main+0xF0AA74  LDR   X1,[X20,#0xAB0]
main+0xF0AA78  ADD   X0,SP,#0x50
main+0xF0AA7C  BL    main+0xF28FC0
main+0xF0AA80  LDR   X8,[X20,#0xAB8]
main+0xF0AA84  STR   X8,[X20,#0xAB0]
main+0xF0AA88  BL    main+0xF28FE0
main+0xF0AA8C  STR   X0,[X20,#0xAB8]
```

Conceptually:

```text
old area value at +0xAB0
      ↓ initialize local RNG
previous +0xAB8
      ↓ copied into +0xAB0
new Crypto64
      ↓ written to +0xAB8
```

These values appear to be area-level entropy/history participating in bulk Nest refresh logic.

Do **not** confuse these with the per-den Raid seed. The per-den Pokémon-generation seed remains `NestEntry+0x68`.

The exact structural ownership of `+0xAB0/+0xAB8` needs more work because these offsets exceed the nominal `0xA68` area stride and may belong to an adjacent/shared area metadata region.

---

## 13. Raid seed consumption by the encounter builder

The later Raid runtime path reads the active den entry and passes the seed to the Raid builder.

The indexing is visible around:

```asm
main+0x1029810  LDPSW X8,X9,[X20]
main+0x1029814  MOV   W11,#0xA68
main+0x1029818  MADD  X11,X8,X11,X21
main+0x102981C  MOV   W10,#0x18
main+0x1029820  MADD  X10,X9,X10,X11
...
main+0x1029868  MADD  X24,X9,X10,X8
main+0x102986C  LDRB  W8,[X24,#0x70]!
...
main+0x10298A0  LDUR  X2,[X24,#-0x8]  ; entry+0x68
main+0x10298A4  LDRB  W3,[X24,#1]
main+0x10298B8  BL    main+0xEC7080
```

Thus:

```text
NestEntry+0x68
    ↓ X2
main+0xEC7080
```

At Raid-builder entry:

```asm
main+0xEC70B4  MOV X20,X2
```

and a representative `GenerationSpec` initialization writes:

```asm
main+0xEC776C  ... useFixedSeed = 1
main+0xEC7770  STR X20,[X10]       ; spec+0x00 = raidSeed
```

Therefore:

```text
persistent Nest seed
     ↓
Raid builder
     ├─ local raid-builder RNG initialized from same seed
     └─ GenerationSpec.fixedSeed = same seed
             ↓
          main+0x7662A0
             ↓
          main+0x777F40
             ↓
          main+0x7667A0
             ↓
      fixed-seed Pokémon RNG
```

---

## 14. Important two-stream/reinitialization detail

The Raid builder does **not** merely hand one continuously advanced RNG object through the whole pipeline.

It explicitly initializes local contexts from the same 64-bit raid seed, including:

```asm
main+0xEC78B0  SUB X0,X29,#0x70
main+0xEC78B4  MOV X1,X20
main+0xEC78B8  BL  main+0xF28FC0
```

and also writes the original seed to the final `GenerationSpec.fixedSeed`.

So a faithful predictor should model:

```text
raidSeed frame N
    │
    ├─ Raid-builder deterministic table/policy logic
    │      [explicit local initialization(s)]
    │
    └─ GenerationSpec.fixedSeed
           ↓
       shared PML fixed generator
           [separate local initialization]
```

Do not implement this as one giant xoroshiro stream with a single advance counter.

---

## 15. Complete lifecycle

The current closed lifecycle is:

```text
[Fresh den activation/reseed]
main+0x14B8ED0
      ↓
main+0xF28FE0
Nintendo Crypto64
      ↓
main+0x14260F0
      ↓
NestEntry+0x68
      ↓
additional den-state draws

          OR later

[Existing den frame/refresh advance]
NestEntry+0x68 = oldSeed
      ↓
main+0xF0A5E0
      ↓
init (oldSeed, 0x82A2B175229D6A5B)
      ↓
first xoroshiro128+ output
      ↓
newSeed = oldSeed + 0x82A2B175229D6A5B
      ↓
main+0x14260F0
      ↓
NestEntry+0x68 = newSeed
      ↓
state fields refreshed

          ↓ later encounter construction

main+0x10298A0
      ↓
load NestEntry+0x68
      ↓
main+0xEC7080
      ↓
Raid table/policy construction
      +
GenerationSpec.fixedSeed = raidSeed
      ↓
main+0x7662A0
      ↓
main+0x777F40
      ↓
final Raid Pokémon
```

---

## 16. Predictor architecture

A Raid predictor can now be divided into three deterministic layers.

### Layer A: persistent den frame

Read:

```text
saveRoot
  -> saveRoot+0x248
  -> areaIndex*0xA68
  -> denIndex*0x18
  -> +0x68
```

Then for target frame `n`:

```cpp
seed_n = seed_0 + n * 0x82A2B175229D6A5BULL;
```

No iterative xoroshiro simulation is required merely to jump the persistent den seed.

### Layer B: Raid source builder

For each `seed_n`, reproduce the appropriate `main+0xEC7080` branch and the relevant Raid encounter table/policies:

```text
species
form
level/star-dependent policy
gender
nature
ability
perfect-IV policy
PID/shiny policy
other Raid-specific flags
```

This step matters because the seed alone does not encode every source policy; the encounter record and current Raid context also participate.

### Layer C: shared Pokémon fixed generator

Once the Raid GenerationSpec is known:

```text
GenerationSpec.fixedSeed = seed_n
GenerationSpec.useFixedSeed = 1
        ↓
main+0x7667A0
```

replay the shared fixed RNG exactly to obtain the final Pokémon fields.

---

## 17. Searcher design

The natural Raid searcher can use O(1) seed jumps:

```cpp
for (uint64_t frame = begin; frame < end; ++frame)
{
    uint64_t raidSeed = seed0 + frame * RAID_ADVANCE;

    RaidSpec spec = BuildRaidSpec(raidSeed, raidTable, context);
    PokemonResult p = ResolveFixedSpec(spec);

    if (MatchesTarget(p))
        emit(frame, raidSeed, p);
}
```

For very large sparse queries, frame seeds can be generated in vector/SIMD/GPU batches because the persistent seed equation is affine.

The bottleneck becomes the Raid table/policy simulation and Pokémon generation, not seed stepping.

---

## 18. Predictability boundary

There are two different answers to “can the next Raid seed be predicted?”

### Existing initialized den

Yes.

If the current saved seed is known:

```text
next = current + 0x82A2B175229D6A5B
```

and arbitrary future deterministic frames can be jumped directly.

### Not-yet-created fresh den seed

No, not as a read-only prediction before the call.

Fresh initialization uses:

```text
main+0xF28FE0
 -> nn::crypto::GenerateCryptographicallyRandomBytes(...,8)
```

So the new random seed does not exist in game-owned state before Nintendo Crypto produces it.

Once that value has been written into `NestEntry+0x68`, however, subsequent additive frame progression is deterministic until another fresh reseed/replacement occurs.

---

## 19. Recommended runtime watchpoints / hooks

For verification, the highest-value breakpoints are:

```text
main+0x14260F0  every persistent per-den seed write
main+0x14B907C  fresh Crypto64 request
main+0x14B9094  fresh seed commit
main+0xF0A6B0   load old seed for deterministic advance
main+0xF0A6D0   compute oldSeed + constant
main+0xF0A6EC   deterministic seed commit
main+0xF0A950   bulk area refresh
main+0x10298A0  Raid generation reads active stored seed
main+0xEC7080   Raid builder entry
main+0xEC7770   representative fixedSeed write into GenerationSpec
```

A useful telemetry record is:

```cpp
struct RaidSeedEvent {
    uint32_t area;
    uint32_t den;
    uint64_t oldSeed;
    uint64_t newSeed;
    enum Reason {
        FreshCrypto,
        DeterministicAdvance,
        BulkRefresh
    } reason;
};
```

For `main+0x14260F0`, classify the write from LR/caller:

```text
LR from 0x14B9098 / 0x14B9204 area -> FreshCrypto
LR from 0xF0A6F0                -> DeterministicAdvance
```

This avoids global guessing and immediately validates real gameplay transitions.

---

## 20. Current confidence / remaining open items

### CONFIRMED

- save-backed Nest block is reached via `saveRoot+0x248`;
- per-den addressing uses `area*0xA68 + den*0x18`;
- each area contains 110 records;
- `NestEntry+0x68` is the 64-bit Raid seed;
- `main+0x14260F0` is the exact seed setter;
- only three direct callers write through that setter;
- fresh selected-den seed comes from Nintendo Crypto64 via `main+0xF28FE0`;
- existing-den persistent advance writes the first xoroshiro128+ output back;
- persisted recurrence is `seed += 0x82A2B175229D6A5B mod 2^64`;
- `main+0x10298A0` reads the same stored seed for Raid generation;
- `main+0xEC7080` receives it as `X2` and copies it into `GenerationSpec.fixedSeed`;
- Raid builder and shared PML resolver use separately initialized deterministic contexts from the seed.

### HIGH / needs runtime naming

- exact meanings of `entry+0x70/+0x71/+0x72`;
- which specific UI/gameplay action owns the `0x14B8ED0` fresh-reseed path;
- exact external trigger names for each caller of `BulkNestAreaRefresh`.

### OPEN

- full field-by-field model of every Raid encounter table variant and event table;
- exact lifecycle role of area-level values around `+0xAB0/+0xAB8`;
- complete generic save-to-disk call chain after save-backed RAM mutation;
- dynamic classification of every `F0A5E0` caller into player action / day change / network/event refresh semantics.

---

## 21. Practical conclusion

The important result for tooling is that the Max Raid seed is now no longer an opaque RNG state.

For an already initialized den:

```text
current saved Raid seed
        ↓
seed[n] = seed[0] + n * 0x82A2B175229D6A5B
        ↓
Raid source builder
        ↓
GenerationSpec.fixedSeed
        ↓
shared Pokémon generator
        ↓
future Raid Pokémon
```

So the next implementation target is a `RaidFrameSearcher` that reads the stored `NestEntry+0x68`, performs constant-time frame jumps, reproduces the `EC7080` Raid policy builder, then runs the shared fixed-seed Pokémon resolver.