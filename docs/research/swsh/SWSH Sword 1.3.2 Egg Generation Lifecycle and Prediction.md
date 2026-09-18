# Pokémon Sword 1.3.2 — Egg Generation Lifecycle, Cached Seed, Shiny and Size Prediction

Updated: 2026-09-18

Target:
- Game: Pokémon Sword / Shield 1.3.2
- Sample: `main(3).elf`
- GNU Build ID: `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`
- Address notation: `main + offset`

## 1. Executive conclusion

The Day Care system is split into two distinct stages:

1. **Egg becomes ready before pickup**
   - The game does **not** build and cache a complete PK8 here.
   - It sets an egg-ready flag and generates/caches an **8-byte egg seed**.
2. **Player receives the egg**
   - The game loads the cached egg seed.
   - It initializes the egg-local RNG from that seed.
   - It reads the current parent Pokémon and trainer context.
   - It then generates the egg parameters and constructs the actual Pokémon object.

Therefore:

> Before pickup, the next egg is not present as a finished PK8 in the Day Care cache. What is cached is the readiness state plus a deterministic 64-bit seed.

This seed is enough to reconstruct/predict most breeding attributes before pickup, provided the parent/context state used at pickup is known.

However, **HeightScalar / WeightScalar are a special case**: in the current Sword 1.3.2 path they are resolved by the generic Pokémon constructor using the PML/System RNG at pickup time, not directly by the cached egg-local seed.

---

## 2. Day Care slot state and runtime address

The offsets `+0x2F8` / `+0x300` are **not fixed main-module addresses**. They are offsets inside a dynamically allocated Day Care save/runtime object.

The script-facing callers resolve the Day Care base through the following pointer chain:

```text
p1          = *(u64 *)(main + 0x2610798)
saveRoot    = *(u64 *)(p1 + 0x0)
daycareBase = *(u64 *)(saveRoot + 0x270)
```

This is visible around `main+0x14BB890` and `main+0xDC3D20`.

Common pointer notation:

```text
daycareBase = [[main+0x2610798]+0x0]+0x270
```

The Day Care entry stride is:

```text
0x2B8 bytes per Day Care slot
```

So:

```text
slot = daycareBase + slotIndex * 0x2B8
slotIndex = 0 or 1
```

The currently identified control fields are:

| Offset in Day Care slot | Meaning |
|---:|---|
| `+0x2F4` | egg-check counter |
| `+0x2F8` | cached 64-bit egg seed |
| `+0x300` | egg-ready flag |

Concrete offsets from `daycareBase`:

| Slot | counter | egg seed | ready |
|---:|---:|---:|---:|
| 0 | `+0x2F4` | `+0x2F8` | `+0x300` |
| 1 | `+0x5AC` | `+0x5B0` | `+0x5B8` |

The final absolute RAM address changes between runs because `daycareBase` is a pointer, not a fixed `main+offset` object.

---

## 3. Stage A — egg readiness generation

Main function:

```text
main+0x1401530
```

This routine is the important **pre-pickup Day Care update path**.

### 3.1 Step/check counter

The routine increments:

```text
[slot + 0x2F4]
```

Relevant instructions:

```asm
main+0x140158C  LDR  W9, [X8,#0x2F4]
main+0x1401590  ADD  W9, W9, #1
main+0x1401594  CMP  W9, #0xB5
main+0x1401598  STR  W9, [X8,#0x2F4]
```

When the counter reaches the threshold, it is reset and the game performs an egg-production roll.

The exact gameplay-facing step cadence should be validated dynamically; statically, the threshold constant used here is `0xB5`.

### 3.2 Egg-rate modifier flag

The routine queries flag/item ID:

```text
0x277
```

through:

```text
main+0x1420EF0
```

This flag is used as an index into the egg-generation probability table.

The adjacent ID `0x278` is used later by the egg construction path for Shiny Charm handling.

### 3.3 Compatibility / egg probability roll

The routine first verifies that the Day Care contains a valid pair and obtains a compatibility category through the parent objects.

The actual egg-ready chance uses a global xoroshiro-style RNG state. The core transition is visible around:

```text
main+0x14016A8
```

The result is reduced to the expected bounded range and compared against a table indexed by compatibility and the `0x277` flag state.

### 3.4 What happens when the egg roll succeeds

On success:

```asm
main+0x14016E8  MOV  W9,#1
main+0x14016EC  STRB W9,[X8]
```

where `X8 = slot + 0x300`.

Therefore:

```text
[slot + 0x300] = 1
```

means:

> an egg is ready for pickup.

Immediately afterwards:

```asm
main+0x14016F0  BL  main+0xF28FE0
main+0x14016F4  ...
main+0x14016F8  STR X0,[X8,#0x2F8]
```

`main+0xF28FE0` calls:

```text
nn::crypto::GenerateCryptographicallyRandomBytes(..., 8)
```

and returns an 8-byte random value.

That 64-bit value is stored at:

```text
[slot + 0x2F8]
```

This is the critical **cached egg seed**.

### 3.5 Important result

At the moment the egg becomes ready, the game has cached:

```text
eggReady = 1
eggSeed  = random 64-bit value
```

It has **not** yet called the complete egg Pokémon construction chain.

There is no evidence here of a complete PK8 object being stored in the Day Care slot.

---

## 4. Checking whether an egg is waiting

Small helper:

```text
main+0x1401500
```

It directly reads:

```asm
LDRB W8,[slot + 0x300]
```

and returns whether the value is non-zero.

So `+0x300` is a clean runtime watchpoint for:

> “is there currently a pending egg?”

---

## 5. Stage B — receiving the egg

Main pickup/receive path:

```text
main+0x14018C0
```

### 5.1 Pickup first checks the pending flag

The routine calculates:

```text
slot = daycareBase + slotIndex * 0x2B8
```

then reads:

```asm
main+0x14018F4  LDRB W9,[X8,#0x300]
```

If no egg is pending, it exits.

If an egg exists, it immediately clears the flag:

```asm
main+0x140190C  ADD  X8,X8,#0x300
main+0x1401910  STRB WZR,[X8]
```

So the waiting state is consumed at the start of pickup.

### 5.2 Pickup loads the previously cached seed

Later in the same routine:

```asm
main+0x1401A18  MOV  W8,#0x2B8
main+0x1401A1C  MADD X8,X28,X8,X25
main+0x1401A20  LDR  X1,[X8,#0x2F8]
```

The value loaded from `+0x2F8` is exactly the seed generated when the egg became ready.

### 5.3 Cached seed initializes the egg-local RNG

The cached value is passed to:

```text
main+0xF28FC0
```

This initializes an RNG object/state using that seed.

Relevant behavior:

```text
state[0] = eggSeed
state[1] = 0x82A2B175229D6A5B
state[2] = eggSeed
```

This state is subsequently exposed to the egg-generation code through the RNG callback stored in the Egg Context.

### 5.4 Pickup reads trainer context

The receive path obtains:

```text
Trainer ID32
Shiny Charm state
egg-local RNG callback/state
```

Relevant calls:

```text
main+0x1424D00    Get ID32
main+0x1420EF0    query 0x278
```

The `0x278` result is written into the Egg Context and is consumed later by the PID/shiny reroll logic.

### 5.5 Pickup reads the current parents

The receive routine calls:

```text
main+0x1401160
```

to obtain the two parent Pokémon objects, and then passes them into:

```text
main+0x7893F0
```

This is important:

> The complete child Pokémon parameters are calculated from the cached seed plus the parent objects read at pickup time.

So the Day Care cache is not a frozen serialized child Pokémon.

---

## 6. Actual egg Pokémon creation chain

The receive path enters:

```text
main+0x14018C0
  -> main+0x7893F0
      -> main+0x789500
          -> main+0x789770
          -> main+0x777F40
```

### `main+0x789500`

Builds the approximately `0x80`-byte Pokémon generation parameter structure.

### `main+0x789770`

Egg-specific breeding logic.

This is where the cached egg-local RNG is consumed for the major breeding attributes.

Confirmed generated/resolved here include:

- Species
- Form
- Gender
- Nature
- Ability slot
- IV inheritance selection
- random IV values
- EC
- PID
- shiny rerolls
- parent-dependent special breeding rules

### `main+0x777F40`

Generic Pokémon constructor.

It consumes the completed parameter structure and creates the actual Pokémon object / PK8 data.

---

## 7. Cached egg seed: lifetime and RNG state

The value at `slot+0x2F8` is a **seed**, not a continuously advancing RNG state.

When the egg-ready roll succeeds:

```text
main+0x14016F0 -> main+0xF28FE0
```

calls:

```text
nn::crypto::GenerateCryptographicallyRandomBytes(..., 8)
```

and writes a fresh random 64-bit value to:

```text
slot+0x2F8
```

### Stored seed lifetime

The stored value behaves as follows:

1. New egg becomes ready -> a fresh 64-bit value overwrites `slot+0x2F8`.
2. While the egg waits for pickup -> the stored value does not advance.
3. On pickup -> the game reads the value, but does not clear or mutate `slot+0x2F8`.
4. After pickup -> `slot+0x300` becomes 0, while the old seed can remain in `slot+0x2F8`.
5. The next successful egg-generation event overwrites it with another cryptographically generated 64-bit value.

Therefore a seed read is meaningful as the pending egg seed only when:

```text
slot+0x300 == 1
```

or when a runtime tool observes the `0 -> 1` ready transition.

### Pickup-time egg RNG initialization

At pickup:

```text
main+0x1401A20  LDR X1,[slot+0x2F8]
main+0x1401A2C  BL  main+0xF28FC0
```

The deterministic egg-local RNG object is at:

```text
main+0x25A83A8
```

and `main+0xF28FC0` initializes it as:

```text
state0       = eggSeed
state1       = 0x82A2B175229D6A5B
originalSeed = eggSeed
```

The third qword is retained as the original seed; the active RNG transition uses `state0/state1`.

### Egg-local RNG transition

The callback used by breeding code is:

```text
main+0x1400C70
```

Its raw xoroshiro128+-style transition is equivalent to:

```cpp
uint32_t nextRaw32()
{
    uint64_t s0 = state0;
    uint64_t s1 = state1;

    uint32_t out = (uint32_t)(s0 + s1);

    uint64_t t = s0 ^ s1;
    state0 = rotl64(s0, 24) ^ t ^ (t << 16);
    state1 = rotl64(t, 37);

    return out;
}
```

For bounded requests `RNG(n)`, `main+0x1400C70` uses mask/rejection logic and returns:

```text
0 <= result < n
```

Examples used by the egg generator include `RNG(25)`, `RNG(100)`, `RNG(6)`, `RNG(2)`, `RNG(32)`, and `RNG(0xFFFFFFFF)`.

So the cached 64-bit seed does not get incremented. Instead, it initializes a separate two-qword xoroshiro state that advances every time the breeding generator requests a random value.

---

## 8. EC and PID: exact relationship to the cached egg seed

### EC

The Encryption Constant is directly generated by the egg-local RNG.

At:

```text
main+0x789C0C
```

the code calls the Egg Context RNG callback with:

```text
RNG(0xFFFFFFFF)
```

and stores the result at:

```text
EggCreateParam + 0x10
```

which is later written into the Pokémon object by the generic constructor.

Therefore:

> Given the same cached `eggSeed`, the same parent/context data, and the same branch-dependent RNG consumption before `main+0x789C0C`, EC is deterministic.

EC is **not simply** `low32(seed + constant)`, because many breeding decisions consume the egg RNG before EC, including gender/nature/ability/IV inheritance and random IV generation.

### PID has two different RNG paths

The egg-specific PID/shiny section is:

```text
main+0x789C20 .. main+0x789C88
```

It computes the egg-specific reroll count:

| Situation | egg-local PID attempts |
|---|---:|
| same language, no Shiny Charm | 0 |
| same language + Shiny Charm | 2 |
| different-language parents | 6 |
| different-language + Shiny Charm | 8 |

This distinction is important.

#### Path A — Masuda and/or Shiny Charm

When the calculated reroll count is non-zero, each candidate PID is generated by the **egg-local RNG**:

```text
RNG(0xFFFFFFFF)
```

then checked by:

```text
main+0x777350
```

using the trainer ID32 and the standard shiny XOR condition.

Therefore in these cases:

```text
eggSeed -> egg-local RNG stream -> EC -> candidate PID(s)
```

and PID/shiny can be predicted from the cached seed once all earlier egg-RNG consumption is reproduced.

#### Path B — ordinary same-language egg without Shiny Charm

When the egg-specific reroll count is zero, `main+0x789770` leaves the PID sentinel:

```text
EggCreateParam+0x18 = 0x3FFFFFFFF
EggCreateParam+0x35 = 1
```

The later generic constructor `main+0x7667A0` resolves that sentinel.

Because the egg parameter has:

```text
param+0x08 = 0
```

there is no constructor-local RNG pointer, so `main+0x7668C0` falls back to:

```text
main+0x7660B0
```

the generic PML/System raw RNG.

Therefore:

> For an ordinary same-language/no-Shiny-Charm egg, the final single PID roll is **not determined by the cached Day Care egg seed alone**.

This corrects the earlier simplified statement that every egg PID can be reconstructed solely from `slot+0x2F8`.

### PID / EC relationship

- EC always comes from the egg-local RNG seeded by `slot+0x2F8`.
- With Masuda and/or Shiny Charm, PID candidates also come from that same egg-local stream after EC.
- With neither condition, EC comes from the egg-local stream but PID comes from the separate PML/System RNG.

So there is no universal formula `PID = f(EC)`; their relationship depends on which PID path is active.

---

## 9. Height / Weight and their relationship to PID

The egg parameter structure is initialized with:

```text
WeightParam = 0x0100
HeightParam = 0x0100
```

These values mean normal random scalar generation.

The generic constructor processes:

```text
main+0x7667A0
  -> main+0x7672E0
```

For `0x0100`, because the egg parameter has no constructor-local RNG pointer (`param+0x08 == 0`), the size resolver falls back to the generic PML/System bounded RNG:

```text
main+0x7660D0
```

The order is:

```text
HeightScalar:
    Random(129)
    Random(128)

WeightScalar:
    Random(129)
    Random(128)
```

So:

```text
HeightScalar = Random(129) + Random(128)
WeightScalar = Random(129) + Random(128)
```

Both are `0..255` with the triangular distribution.

The parameter/write path confirms:

```text
param+0x4C -> HeightScalar -> main+0x775700 -> PK8 +0x50
param+0x4A -> WeightScalar -> main+0x775850 -> PK8 +0x51
```

### Cached egg seed relationship

There is no direct `eggSeed -> Height/Weight` mapping in this path.

Height/Weight consume the **PML/System RNG**, not the egg-local RNG initialized from `slot+0x2F8`.

### Relationship to PID

There are two cases:

1. **ordinary same-language/no-Shiny-Charm egg**
   - PID is also resolved by the PML/System RNG.
   - The generic constructor consumes the PID raw roll before resolving Height and Weight.
   - Thus PID and Height/Weight are positions in the same PML RNG stream (with bounded-rejection behavior for the size calls).

2. **Masuda and/or Shiny Charm**
   - PID is already resolved by the egg-local RNG before the generic constructor.
   - The generic constructor skips the PID fallback.
   - Height/Weight still use the PML/System RNG.
   - In this case PID and Height/Weight are from separate RNG streams.

The generic PML/System RNG entry points are:

```text
main+0x7660B0   raw 32-bit RNG
main+0x7660D0   bounded RNG
```

The PML system object is obtained through the global slot at:

```text
main+0x2617358
```

but the internal provider state still needs runtime tracing/hooking if an exact pre-pickup Height/Weight predictor is required.

---

## 10. What can actually be predicted before pickup?

### Directly readable before pickup

```text
egg-ready flag             slot+0x300
cached 64-bit egg seed     slot+0x2F8
both parent Pokémon
trainer ID/context
Shiny Charm state
```

### Not directly cached as a finished child

```text
child PK8
child EC
child PID
child IV block
child HeightScalar
child WeightScalar
```

### Deterministic from cached egg seed + parent/context

The egg-local stream can reproduce:

- Species/Form branches that use the egg RNG
- Gender
- Nature
- Ability
- IV inheritance selections
- random IVs
- EC
- PID candidates when Masuda and/or Shiny Charm activates the egg-local PID loop

### Requires the separate PML/System RNG state

- ordinary same-language/no-Charm final PID
- HeightScalar
- WeightScalar

Therefore a truly exact “next egg before pickup” tool needs **two RNG models**:

```text
A. cached egg seed -> egg-local xoroshiro stream
B. current PML/System RNG state -> generic constructor fallback/size stream
```

If the goal is only IV/Nature/Ability/EC, the cached egg seed path is sufficient once the parent/context rules are reproduced. If the goal includes every PID case plus exact Height/Weight, the PML RNG state must also be captured or controlled.

---

## 11. Recommended runtime implementation

A practical pre-pickup Egg Viewer should use two phases.

### Phase A — detect pending egg

Watch:

```text
slot + 0x300
```

When it changes:

```text
0 -> 1
```

read:

```text
slot + 0x2F8
```

as the 64-bit egg seed.

### Phase B — emulate the egg-local generator

Snapshot:

- parent 1 PK8
- parent 2 PK8
- trainer ID32
- Shiny Charm state

Then emulate:

```text
main+0x789770
```

with the same bounded-RNG behavior and branch-dependent consumption.

Display a predicted next egg:

```text
Species / Form
Gender
Nature
Ability
IVs
EC
PID
Shiny
Shiny XOR
```

### Height / Weight display

Mark these separately as either:

```text
unknown until pickup
```

or:

```text
predicted from captured PML RNG state
```

Do not derive them from `eggSeed` alone.

---

## 12. Best hook points

### Read-only next-egg viewer

No code patch is required.

Recommended data watch:

```text
DayCare slot + 0x300   egg-ready
DayCare slot + 0x2F8   cached egg seed
```

Then reproduce the egg-local RNG externally.

### Capture exact generated parameters

Hook immediately after:

```text
main+0x789770
```

returns and before:

```text
main+0x777F40
```

At this point Species, Form, Gender, Nature, Ability, IVs, EC and PID have been resolved in the parameter block.

### Override exact Height / Weight

Before `main+0x777F40`, replace:

```text
param+0x4C = exact HeightScalar 0..255
param+0x4A = exact WeightScalar 0..255
```

as 16-bit values.

Any value below `0x0100` is passed through directly by `main+0x7672E0`.

---

## 13. Final lifecycle model

```text
Day Care step/update
    |
    |  main+0x1401530
    |  compatibility / egg chance
    v
egg roll succeeds
    |
    +--> [slot+0x300] = 1
    |
    +--> CryptoRandom(8)
          |
          +--> [slot+0x2F8] = eggSeed

        [ egg is now waiting ]

player talks to NPC / receives egg
    |
    |  main+0x14018C0
    |
    +--> verify [slot+0x300]
    +--> clear [slot+0x300]
    +--> load [slot+0x2F8]
    +--> main+0xF28FC0 : initialize egg RNG
    +--> read current parents
    +--> read TID / charm state
    |
    v
main+0x7893F0
    |
    v
main+0x789500
    |
    v
main+0x789770
    |
    +--> Species / Form
    +--> Gender
    +--> Nature
    +--> Ability
    +--> IVs / inheritance
    +--> EC
    +--> PID / shiny
    |
    v
main+0x777F40
    |
    +--> generic constructor
    +--> HeightScalar / WeightScalar via PML/System RNG
    |
    v
final egg PK8
```

## 14. Key takeaway

The most accurate model is:

> **The Day Care caches a pending-egg flag and a 64-bit seed, not a complete PK8.**

Runtime address:

```text
daycareBase = [[main+0x2610798]+0]+0x270
slot        = daycareBase + slotIndex*0x2B8

eggSeed     = slot+0x2F8
eggReady    = slot+0x300
```

The cached seed itself does not advance. It is replaced by a fresh cryptographic 64-bit value only when a new egg becomes ready. At pickup it initializes the egg-local RNG state at `main+0x25A83A8`, whose xoroshiro state then advances as breeding logic consumes random values.

RNG ownership is split:

```text
cached eggSeed / egg-local RNG:
    gender / nature / ability / IV inheritance / IVs / EC
    + PID rerolls for Masuda and/or Shiny Charm

PML/System RNG:
    ordinary same-language/no-Charm single PID fallback
    HeightScalar
    WeightScalar
```

So a pre-pickup predictor is feasible, but an exact predictor for **all** of PID + Height + Weight must capture both RNG domains rather than treating `slot+0x2F8` as the one universal seed.
