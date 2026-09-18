# Pokémon Sword 1.3.2 — PML/System RNG and Complete Next-Egg Preview

Updated: 2026-09-18

Target:
- Pokémon Sword / Shield 1.3.2
- Sample: `main(3).elf`
- GNU Build ID: `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`
- Address notation: `main + offset`

This document continues the Day Care / egg analysis and answers one specific question:

> Can the ordinary egg PID, HeightScalar and WeightScalar be predicted before pickup by locating a PML/System RNG state?

The important answer is **no: the default PML random provider in this build is not a stateful PRNG stored in the game's main module. It requests fresh bytes from `nn::crypto::GenerateCryptographicallyRandomBytes` on every call.**

That changes the design of a complete next-egg viewer.

---

## 1. Executive result

The RNG ownership for a normal Sword/Shield egg is split into two domains.

### Egg-local deterministic RNG

A pending egg has a cached 64-bit seed at:

```text
slot + 0x2F8
```

Pickup initializes the egg-local xoroshiro state from that seed.

This deterministic stream controls breeding decisions such as:

- gender
- nature
- ability
- IV inheritance
- random IVs
- EC
- PID rerolls when Masuda Method and/or Shiny Charm activates the egg-specific PID loop

### PML/System random provider

The generic Pokémon constructor uses:

```text
main+0x7660B0   PmlRandomRaw32 wrapper
main+0x7660D0   PmlRandomBounded wrapper
```

These wrappers ultimately call:

```text
main+0x7D1320   raw provider
main+0x7D1360   bounded provider
```

Both providers use:

```text
nn::crypto::GenerateCryptographicallyRandomBytes
```

directly.

Therefore there is no Sword-main-memory xoroshiro/LCG state to read and advance for these PML calls.

For egg construction, PML is responsible for:

- the single PID roll of an ordinary same-language/no-Shiny-Charm egg
- HeightScalar
- WeightScalar

---

## 2. What `main+0x2617358` actually is

Earlier analysis treated `main+0x2617358` as a possible RNG-state anchor. Relocation analysis shows that this is not correct.

ELF relocation:

```text
0x2617358  R_AARCH64_RELATIVE  0x28F5BA0
```

So the instruction:

```asm
main+0x766030:
    ADRP X8, main+0x2617000
    LDR  X8, [X8,#0x358]
    LDR  X0, [X8]
    RET
```

resolves conceptually to:

```cpp
PmlCore* GetPmlCore()
{
    return *(PmlCore**)(main + 0x28F5BA0);
}
```

Thus:

```text
main+0x2617358
```

is the relocated pointer/GOT entry used to reach the actual global PML Core pointer variable at:

```text
main+0x28F5BA0
```

It is **not** RNG state.

The PML Core object is dynamically allocated; the allocation size visible at `main+0x7659DC` is:

```text
0x298 bytes
```

---

## 3. PML random callback fields

The two wrappers are:

### Raw 32-bit random

```asm
main+0x7660B0:
    BL   main+0x766030
    LDR  X0,[X0,#0x280]
    BR   X0
```

Equivalent:

```cpp
uint32_t PmlRandomRaw32()
{
    PmlCore* pml = GetPmlCore();
    return pml->randomRaw32();
}
```

Callback field:

```text
PmlCore + 0x280
```

### Bounded random

```asm
main+0x7660D0:
    ...
    BL   main+0x766030
    LDR  X1,[X0,#0x288]
    MOV  W0,W19
    BR   X1
```

Equivalent:

```cpp
uint32_t PmlRandomBounded(uint32_t bound)
{
    PmlCore* pml = GetPmlCore();
    return pml->randomBounded(bound);
}
```

Callback field:

```text
PmlCore + 0x288
```

---

## 4. Where the callbacks are installed

The PML setup path constructs a template/config object on the stack and installs the RNG callbacks.

At:

```text
main+0x7D2568
```

the function loads the relocation slot:

```text
main+0x2617B88 -> main+0x7D1320
```

and stores it to stack offset `+0x2D8`.

The config object passed to `main+0x765230` starts at:

```text
SP + 0x58
```

Therefore:

```text
0x2D8 - 0x58 = 0x280
```

which matches `PmlCore+0x280`.

Likewise:

```text
main+0x2617B90 -> main+0x7D1360
```

is stored at stack `+0x2E0`:

```text
0x2E0 - 0x58 = 0x288
```

The object-copy path at `main+0x765C38..0x765C50` copies these callback fields into the live PML Core object.

So the callback mapping is verified as:

| PML Core field | Callback |
|---:|---:|
| `+0x280` | `main+0x7D1320` |
| `+0x288` | `main+0x7D1360` |

---

## 5. Raw provider `main+0x7D1320`

Disassembly:

```asm
main+0x7D1320:
    SUB  SP,SP,#0x20
    ...
    SUB  X0,X29,#4
    MOV  W1,#4
    BL   nn::crypto::GenerateCryptographicallyRandomBytes
    LDUR W0,[X29,#-4]
    ...
    RET
```

Equivalent:

```cpp
uint32_t PmlCryptoRandom32()
{
    uint32_t value;
    nn::crypto::GenerateCryptographicallyRandomBytes(&value, 4);
    return value;
}
```

There is no local persistent state and no game-visible transition such as xoroshiro/LCG advancement.

Each raw request asks the Nintendo crypto API for a new 32-bit value.

---

## 6. Bounded provider `main+0x7D1360`

Special cases:

```text
bound == 1 -> return 0
bound == 0 -> return fresh crypto u32
```

For `bound > 1`, the code constructs an all-ones bit mask large enough to cover the requested range, then performs rejection sampling.

Equivalent model:

```cpp
uint32_t PmlCryptoBounded(uint32_t bound)
{
    if (bound == 1)
        return 0;

    if (bound == 0)
        return PmlCryptoRandom32();

    uint32_t mask = NextPowerOfTwo(bound) - 1;

    for (;;)
    {
        uint32_t raw;
        nn::crypto::GenerateCryptographicallyRandomBytes(&raw, 4);

        uint32_t value = raw & mask;
        if (value < bound)
            return value;
    }
}
```

This matches the important egg-size cases:

```text
PmlRandomBounded(129):
    mask = 0xFF
    accept 0..128
    reject 129..255

PmlRandomBounded(128):
    mask = 0x7F
    accept 0..127
    no rejection after masking
```

Important consequence:

> There is no deterministic “advance count” for `PmlRandomBounded(129)`; it may request multiple crypto words before one is accepted.

---

## 7. Why there is no PML RNG state to read

The default callback implementation does **not** contain:

- a pair of xoroshiro state qwords
- an LCG seed
- MT state
- a game-owned seed buffer
- a deterministic update function

Instead every request reaches:

```text
nn::crypto::GenerateCryptographicallyRandomBytes
```

through the imported SDK function.

Therefore a read-only tool based only on `main.elf` cannot do:

```text
read PML state
-> emulate next()
-> know future PID/Height/Weight
```

because that state does not exist in the game's PML Core.

There may be internal implementation state inside Nintendo's crypto subsystem / OS, but it is outside this `main.elf` and is not exposed as a usable game RNG state.

This is the central correction to the previous research direction.

---

## 8. Ordinary egg PID path

Egg-specific PID handling occurs at:

```text
main+0x789C20..0x789C88
```

For:

```text
same-language parents
AND
no Shiny Charm
```

the egg-specific reroll count is zero.

Therefore `main+0x789770` leaves:

```text
EggCreateParam+0x18 = 0x3FFFFFFFF
EggCreateParam+0x35 = 1
```

The generic constructor enters:

```text
main+0x7667A0
```

and at:

```text
main+0x766864
```

recognizes the PID sentinel.

Because the egg has:

```text
param+0x08 == 0
```

there is no constructor-local xoroshiro pointer.

So at:

```text
main+0x7668C0
```

it calls:

```text
main+0x7660B0
 -> PmlCore+0x280
 -> main+0x7D1320
 -> nn::crypto::GenerateCryptographicallyRandomBytes(4)
```

and stores that 32-bit value as PID.

The normal egg has exactly one PID attempt because:

```text
param+0x35 = 1
```

So the unmodified ordinary PID is fresh system cryptographic randomness generated at pickup/construction time.

---

## 9. EC remains egg-seed deterministic

EC is different.

At:

```text
main+0x789C0C
```

the breeding routine calls the Egg Context RNG callback:

```text
RNG(0xFFFFFFFF)
```

and stores the result at:

```text
EggCreateParam+0x10
```

Therefore:

```text
eggSeed -> egg-local xoroshiro -> ... -> EC
```

is deterministic once all earlier branch-dependent egg RNG calls are reproduced.

This gives the split:

| Field | Ordinary same-language/no-Charm egg |
|---|---|
| EC | eggSeed RNG |
| PID | PML crypto RNG |
| Height | PML crypto RNG |
| Weight | PML crypto RNG |

---

## 10. Height / Weight path

Egg initialization uses:

```text
HeightParam = 0x0100
WeightParam = 0x0100
```

The generic constructor reaches:

```text
main+0x766EFC
    LDRH W0,[param+0x4C]
main+0x766F04
    BL main+0x7672E0
```

for Height, followed by:

```text
main+0x766F0C
    LDRH W0,[param+0x4A]
main+0x766F14
    BL main+0x7672E0
```

for Weight.

For mode `0x0100` with null local RNG, `main+0x7672E0` falls through to:

```text
main+0x767490
```

which performs:

```asm
MOV W0,#129
BL  main+0x7660D0

MOV W0,#128
BL  main+0x7660D0

ADD result0,result1
```

Therefore:

```text
HeightScalar = PmlBounded(129) + PmlBounded(128)
WeightScalar = PmlBounded(129) + PmlBounded(128)
```

and the final write path is:

```text
HeightScalar -> main+0x775700 -> PK8+0x50
WeightScalar -> main+0x775850 -> PK8+0x51
```

---

## 11. Exact PML call sequence for a standard ordinary egg

The egg-specific routine has already resolved the breeding-specific values before the generic constructor:

- gender
- nature
- ability
- six IVs
- EC
- other breeding policy fields

So for a standard same-language/no-Shiny-Charm egg, the target PML consumers are:

```text
#1  main+0x7668C0
    PmlRaw32()
    -> PID

#2  main+0x767490
    PmlBounded(129)
    -> Height part A

#3  main+0x76749C
    PmlBounded(128)
    -> Height part B

#4  main+0x767490
    PmlBounded(129)
    -> Weight part A

#5  main+0x76749C
    PmlBounded(128)
    -> Weight part B
```

Conceptually:

```text
fresh crypto32 -> PID

fresh crypto32 [possibly repeated by rejection]
    -> Height A 0..128
fresh crypto32
    -> Height B 0..127
Height = A+B

fresh crypto32 [possibly repeated by rejection]
    -> Weight A 0..128
fresh crypto32
    -> Weight B 0..127
Weight = A+B
```

These are ordered calls, but **not a deterministic game RNG stream**.

---

## 12. Masuda / Shiny Charm case

When Masuda Method and/or Shiny Charm activates the egg-specific PID loop, PID is already set in:

```text
main+0x789C60..0x789C88
```

using the egg-local RNG.

Then the generic constructor does **not** call `main+0x7668C0` for PID.

PML use relevant to the requested preview becomes only:

```text
Height A
Height B
Weight A
Weight B
```

So:

| Egg mode | PID source | Height/Weight source |
|---|---|---|
| ordinary same-language/no Charm | PML crypto | PML crypto |
| Shiny Charm | eggSeed RNG | PML crypto |
| Masuda | eggSeed RNG | PML crypto |
| Masuda + Charm | eggSeed RNG | PML crypto |

---

## 13. Why a pure read-only complete pre-pickup predictor is impossible

For the egg-local domain:

```text
slot+0x2F8
```

is already known before pickup, so deterministic replay is possible.

For the PML domain, the values do not exist yet. They are produced by future calls to the OS/SDK cryptographic RNG when the generic constructor runs.

Therefore an unmodified-game read-only tool cannot know exact future:

```text
ordinary PID
HeightScalar
WeightScalar
```

before those crypto calls happen.

This is not a matter of finding one more pointer in PML; the default provider itself is cryptographic-random-by-request.

---

## 14. Three implementation modes

### Mode A — read-only predictor

Before pickup:

- read `eggReady`
- read cached `eggSeed`
- emulate `main+0x789770`
- show deterministic egg-local fields

Output:

```text
Species/Form
Gender
Nature
Ability
IVs
EC
PID/Shiny when egg-local PID loop is active
```

For ordinary PID / Height / Weight display:

```text
not generated yet
```

This is the only mode that exactly preserves unmodified future system randomness.

### Mode B — generation-time observer

Hook:

```text
main+0x7660B0
main+0x7660D0
```

or the concrete providers:

```text
main+0x7D1320
main+0x7D1360
```

and log outputs while the egg is being constructed.

This gives the exact natural values, but only after pickup generation has started.

### Mode C — reserve/replay complete preview

For a UI that must show **all fields before acceptance**, use reservation rather than prediction.

At egg-ready time:

1. compute all eggSeed-dependent fields normally;
2. reserve a cryptographically distributed ordinary PID if needed;
3. reserve HeightScalar;
4. reserve WeightScalar;
5. cache them keyed by `slot + eggSeed`;
6. display the complete preview.

At pickup, inject/replay the reserved values before the generic Pokémon constructor.

This changes *which exact future crypto outputs* the unmodified game would have received, so it must be described as deterministic reservation/replay, not pure prediction.

The statistical distributions remain identical when the values are generated through the original PML callbacks.

---

## 15. Recommended reserve/replay design

A clean implementation does **not** need to globally hook the PML provider during normal gameplay.

Use the egg parameter hook window:

```text
main+0x789664  BL main+0x789770
main+0x789668  <return point>
...
main+0x7896F0  BL main+0x777F40
```

At `main+0x789668`, the egg-specific parameter generation is complete but the generic constructor has not yet run.

In `main+0x789500`, EggCreateParam is at:

```text
SP + 0x18
```

So the hook can patch:

```text
param+0x18 = reserved PID         // only ordinary PID path
param+0x4C = reserved Height      // u16, value 0..255
param+0x4A = reserved Weight      // u16, value 0..255
```

When Height/Weight are below `0x0100`, `main+0x7672E0` treats them as exact scalars and returns them unchanged.

When PID is no longer the `0x3FFFFFFFF` sentinel, the generic constructor skips the PML PID fallback.

This makes the displayed preview and received egg match exactly without needing to alter PML globally.

---

## 16. How to reserve statistically correct values

Use the game's own PML callbacks at preview/reservation time.

### Ordinary PID

```cpp
uint32_t pid = PmlRandomRaw32(); // main+0x7660B0
```

### Height

```cpp
uint8_t height =
    PmlRandomBounded(129) +
    PmlRandomBounded(128);
```

### Weight

```cpp
uint8_t weight =
    PmlRandomBounded(129) +
    PmlRandomBounded(128);
```

Then cache:

```cpp
struct ReservedEggPml
{
    uint64_t eggSeed;
    uint32_t pid;        // valid only when ordinary PID uses PML
    uint8_t  height;
    uint8_t  weight;
    bool     hasPid;
    bool     valid;
};
```

The cache must be invalidated when:

- `eggReady` becomes 0 without matching pickup;
- `slot+0x2F8` changes;
- parent state changes in a way that changes the predicted egg;
- a new egg replaces the pending egg.

---

## 17. Safer hook scoping

Avoid replacing `PmlCore+0x280/+0x288` globally unless required.

Those callbacks are shared by many Pokémon/PML operations, not only eggs.

Preferred order:

1. **Best:** inject exact PID/Height/Weight into EggCreateParam at `main+0x789668`.
2. **Second:** hook only egg-specific PML call sites / use an `inEggConstruct` guard.
3. **Last:** globally replace `PmlCore+0x280/+0x288`.

If global callbacks are hooked, always scope replay by:

- current thread
- egg constructor active flag
- expected call type/bound
- expected eggSeed / Day Care slot
- call count

Expected ordinary call signature:

```text
RAW
BOUND(129)
BOUND(128)
BOUND(129)
BOUND(128)
```

Expected Masuda/Charm signature for the requested fields:

```text
BOUND(129)
BOUND(128)
BOUND(129)
BOUND(128)
```

---

## 18. Predictor architecture

Recommended complete tool architecture:

```text
Day Care watcher
    |
    +-- read ready flag
    +-- read cached eggSeed
    |
    v
Egg-local emulator
    |
    +-- Species/Form
    +-- Gender
    +-- Nature
    +-- Ability
    +-- IVs
    +-- EC
    +-- egg-local PID/Shiny where applicable
    |
    v
PML reservation layer
    |
    +-- ordinary PID if required
    +-- HeightScalar
    +-- WeightScalar
    |
    v
Next Egg UI
    |
    +-- PID
    +-- EC
    +-- IV
    +-- Shiny
    +-- Height
    +-- Weight
    |
    v
Pickup hook @ main+0x789668
    |
    +-- verify eggSeed
    +-- inject reserved PML fields
    |
    v
main+0x777F40
    |
    v
Final PK8 matches preview
```

---

## 19. Address reference

| Purpose | Sword 1.3.2 offset |
|---|---:|
| PML Core getter | `0x766030` |
| raw PML wrapper | `0x7660B0` |
| bounded PML wrapper | `0x7660D0` |
| PML global relocation/GOT entry | `0x2617358` |
| actual PML global pointer variable | `0x28F5BA0` |
| PML raw callback field | `PmlCore+0x280` |
| PML bounded callback field | `PmlCore+0x288` |
| raw callback implementation | `0x7D1320` |
| bounded callback implementation | `0x7D1360` |
| raw callback relocation slot | `0x2617B88` |
| bounded callback relocation slot | `0x2617B90` |
| ordinary PID PML call site | `0x7668C0` |
| Height parameter resolve | `0x766F04` |
| Weight parameter resolve | `0x766F14` |
| size resolver | `0x7672E0` |
| normal-size PML fallback | `0x767490` |
| egg parameter generator | `0x789770` |
| post-egg-param hook point | `0x789668` |
| final Pokémon construction call | `0x7896F0 -> 0x777F40` |
| egg EC generation | `0x789C0C` |
| egg PID reroll loop | `0x789C20..0x789C88` |

---

## 20. Final conclusion

The PML/System RNG question is now resolved at the `main.elf` level:

> **There is no deterministic PML RNG state to recover for the default provider. The PML Core stores callbacks, and the default callbacks obtain fresh bytes from Nintendo's cryptographic random API.**

Therefore:

- cached Day Care eggSeed can be replayed deterministically;
- EC and breeding RNG can be predicted;
- Masuda/Charm PID can be predicted from eggSeed;
- ordinary PID, Height and Weight are future PML crypto draws and cannot be read-only predicted before generation;
- a complete before-acceptance UI is still practical through **reserve/replay + EggCreateParam injection** at `main+0x789668`.

This is a cleaner architecture than attempting to locate a nonexistent PML xoroshiro state.
