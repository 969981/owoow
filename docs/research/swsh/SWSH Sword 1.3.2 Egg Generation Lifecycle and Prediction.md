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

## 2. Day Care slot state

The relevant Day Care entry stride is:

```text
0x2B8 bytes per Day Care slot
```

The currently identified control fields are:

| Offset in Day Care slot | Meaning |
|---:|---|
| `+0x2F4` | egg-check counter |
| `+0x2F8` | cached 64-bit egg seed |
| `+0x300` | egg-ready flag |

The two Day Care slots are selected using:

```text
slotBase = daycareBase + slotIndex * 0x2B8
```

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

## 7. Which values are already predictable before pickup?

Once:

```text
[slot+0x300] == 1
```

and the cached:

```text
eggSeed = [slot+0x2F8]
```

is available, a predictor can read the current parent data and emulate the same egg-local RNG.

### Predictable from cached egg seed + parent/context state

The current static analysis supports pre-pickup prediction of:

- Species
- Form
- Gender
- Nature
- Ability
- random IVs
- inherited IV positions
- which parent supplies each inherited IV
- Encryption Constant (EC)
- PID
- shiny / non-shiny
- shiny XOR class

The exact RNG consumption sequence must include conditional branches such as:

- fixed-gender species
- Everstone
- both parents holding Everstone
- Destiny Knot
- Power Items
- repeated IV-index selections
- special species/form breeding cases
- Masuda Method
- Shiny Charm

Because those branches change the number/order of RNG calls.

---

## 8. PID / shiny is determined during pickup construction, but is predictable earlier

The PID is not stored as a finished value in the Day Care slot.

Instead, the cached seed is consumed at pickup by the egg-local RNG.

Core PID/shiny section:

```text
main+0x789C20 .. main+0x789C88
```

Shiny check:

```text
main+0x777350
```

The standard Gen VIII shiny XOR is used:

```text
TID ^ SID ^ PID_hi ^ PID_lo < 16
```

The egg path implements the Sword/Shield breeding roll counts:

| Situation | PID attempts |
|---|---:|
| normal | 1 |
| Shiny Charm | 2 |
| different-language parents | 6 |
| different-language + Shiny Charm | 8 |

Therefore:

> Shiny is generated only when the pickup path constructs the egg Pokémon, but after the egg-ready seed has been cached it can be predicted before pickup by replaying the egg-local RNG.

---

## 9. Height / Weight are different

The egg parameter structure is initialized with:

```text
WeightParam = 0x0100
HeightParam = 0x0100
```

These values mean “generate a normal random scalar”.

The generic constructor later calls:

```text
main+0x7667A0
  -> main+0x7672E0
```

For `0x0100`, the normal size path is:

```text
main+0x767320
```

When a local constructor RNG state is supplied, it resolves the scalar from that local state.

However, the egg construction parameter currently sets the constructor RNG selector at `param+0x08` to zero.

That causes the constructor-local RNG pointer to be null and the size resolver falls back to:

```text
main+0x7660D0
```

which dispatches through the generic PML/System RNG.

For normal size:

```text
HeightScalar = Random(129) + Random(128)
WeightScalar = Random(129) + Random(128)
```

The resulting range is `0..255` with the triangular distribution already documented in the broader SWSH analysis.

### Consequence

The cached Day Care egg seed at `+0x2F8` does **not by itself determine HeightScalar / WeightScalar** in this path.

So:

- PID/IV/Nature/Ability/etc. can be reconstructed from the cached egg seed.
- Height/Weight are assigned when the generic Pokémon constructor runs at pickup.
- To predict Height/Weight before pickup, a runtime tool must additionally know/control the relevant PML/System RNG state and account for intervening consumers.
- Alternatively, a hook can override `HeightParam/WeightParam` after `main+0x789770` and before `main+0x777F40`.

---

## 10. Direct-read vs prediction

The distinction is important for tool design.

### Directly readable before pickup

```text
Day Care pending flag
Day Care cached egg seed
both parent Pokémon
trainer ID/context
Shiny Charm state
```

### Not directly present as a finished child Pokémon

```text
child PK8
child PID field
child IV block
child EC
child HeightScalar
child WeightScalar
```

### Can be calculated before pickup

Most breeding attributes can be emulated from:

```text
cached egg seed
+ current parent data
+ trainer ID
+ charm/language/item state
+ exact RNG-consumption model
```

This is a **prediction**, not a direct PK8 read.

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

The most accurate wording is:

> **The next egg has a seed before pickup, not a complete PK8.**

When the Day Care decides that an egg exists, it caches a 64-bit seed at `slot+0x2F8`. The complete Pokémon is generated when the player receives the egg.

This means a pre-pickup next-egg viewer is feasible:

- detect `slot+0x300 == 1`
- read `slot+0x2F8`
- snapshot both parents and trainer context
- emulate the egg-local generator

For the core breeding result, this can reveal the next egg before accepting it.

For exact HeightScalar / WeightScalar, the viewer additionally needs the generic PML/System RNG state or a pickup-time hook, because those two values are not fixed solely by the cached Day Care egg seed.
