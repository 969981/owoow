# SWSH Encounter Taxonomy and Hook Boundaries

Updated: 2026-09-16. Sword 1.3.2 addresses below were verified against the real `main(3).elf` (GNU Build ID `a3b75bcd3311385aeed67fbeeb79cbb7bf02f471`).

## Common architecture

```text
Encounter source / outer RNG
        -> species/form/level + policies
        -> GenerationSpec
        -> shared Pokémon generation
        -> final Pokémon
```

There is no single global RNG stream for Field, Egg and Zone.

| Type | Sword 1.3.2 chain | RNG owner | Preferred hook |
|---|---|---|---|
| ordinary Field / hidden | `D31BB0 -> D328A0/D329D0 -> D311C0 -> D31910 -> 7662A0 -> 777F40` | shared Field context (`+0x98` in owning object) | slot at `D328A0`; spec at `D31910/7662A0` |
| Fishing | `DA4720/DA5550` controller/resource; runtime `DAF350 -> D05080 -> D31BB0 -> ...` | Field outer RNG | source guard `D05080` + common spec hook |
| Symbol | resources at `EA24B0`, `E51380`, `F14210`; runtime managers `1484780`, `14B2F50`; common generation sink confirmed but exact indirect bridge open | Field outer RNG | symbol source telemetry + guarded `7662A0` |
| Egg | `14018C0 -> 7893F0 -> 789500 -> 789770 -> 777F40` | breeding state `25A83A8` | breeding state or post-`789770` spec |
| Zone/Around | `1629E30 -> 162AD70 -> 777F40` | local Zone xoroshiro state | state passed to `162AD70`; spec before `162B07C` |
| Script keyed | `DCEFB0/DCF310 -> E4A470 -> E4A340/E4AA30 -> 7662A0` | keyed/script context | guarded `7662A0` |

## Shared sink callers

```text
7662B4 -> 777F40 : generic Field/Script
7896F0 -> 777F40 : Egg
162B07C -> 777F40: Zone/Around
```

This LR split is the recommended safety classifier at the final shared boundary.

## GenerationSpec confirmed fields

```text
+0x00 fixedSeed
+0x08 useFixedSeed
+0x28 species
+0x2C form
+0x2E level
+0x30 gender policy (high confidence)
+0x32 nature policy (high confidence)
+0x34 ability policy (high confidence)
+0x35 shiny rolls/policy count (high confidence)
+0x36..+0x40 six IV sentinels/values
+0x48 perfect-IV count
```

The local fixed stream uses `(fixedSeed, 0x82A2B175229D6A5B)`.

## Seed Searcher rule

Do not collapse all searching into one fixed seed.

`FixedSpecSearcher` searches the local inner generation after the spec policy is fixed. `EncounterStateSearcher` searches the outer subsystem RNG and is required for natural slot/level/policy/mark behavior. Egg and Zone need their own state searchers because they do not own the ordinary Field stream.
