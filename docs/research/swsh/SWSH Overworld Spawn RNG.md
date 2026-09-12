## Introduction  
This will be a step-by-step description of the RNG process used to generate spawns in Pokémon Sword and Shield. Encounter generation RNG uses a [xoroshiro128+](https://prng.di.unimi.it/xoroshiro128plus.c) state at `HEAP+0x4C2AAC18`.

All addresses are for Shield 1.3.2. Sword addresses are slightly different but the logic is the same.

## Determining if a Hidden Encounter is Spawned  
This part pertains only to hidden grass encounters, which are generated when the player walks in the grass. As long as no hidden encounter is on the overworld, the game will attempt to spawn one at increasingly higher chance until one is spawned.

The chance of a hidden encounter being generated is in `sub_7100D30C10`, which follows:
```c
__int8 __fastcall get_hidden_encounter_rate(encount_info *info, unsigned int encounter_rate)
{
  int lead_item; // w8
  unsigned int v3; // w8
  int lead_ability_flags; // w8
  unsigned int v5; // w9
  int encounter_flags; // w9
  unsigned int v7; // w10
  unsigned int v8; // w8
  
  lead_item = info->lead_item;
  if ( lead_item == 320 || lead_item == 224 )   // Pure Incense, Cleanse Tag
  {
    v3 = encounter_rate >> 3;
  }
  else
  {
    lead_ability_flags = info->lead_ability_flags;
    v5 = encounter_rate >> 3;
    if ( (lead_ability_flags & 2) == 0 )        // stench, quick feet, white smoke, infiltrator
      v5 = encounter_rate;
    if ( (lead_ability_flags & 1) != 0 )        // illuminate, arena trap, no guard
      v3 = 2 * encounter_rate;
    else
      v3 = v5;
  }
  encounter_flags = info->encounter_flags;
  v7 = v3 >> ((encounter_flags & 0x20) != 0);   // 1/2 the encounter rate if biking at base speed
  v8 = v3 / 5;                                  // 1/5 the encounter rate if biking at boosted speed
  if ( (encounter_flags & 0x40) == 0 )
    v8 = v7;
  if ( v8 >= 100 )
    LOBYTE(v8) = 100;
  if ( (encounter_flags & 0x14) != 0 )
    return 0;
  else
    return v8;
}
```
The algorithm is essentially:  
1. `rand.NextInt(100)` for the lead ability check. This happens even though all of the encounter rate modifying abilities are 100% chance to activate.
1. Determine the encounter rate based on `sub_7100D30C10`.  
   - `initial_encounter_rate = attempt * 22`, up to a maximum of 100. For walking, the rate of a spawn is 22%, 44%, 66%, 88%, and 100% for each subsequent failed attempt.  
   - As with previous games, if Pure Incense or Cleanse Tag is held by the lead Pokémon, its ability is ignored. If the lead Pokémon holds no item, then its ability takes effect if applicable.  
   - Note that encounter reducing abilities and items now modify the encounter rate by 1/8x. Encounter boosting abilities still modify the encounter rate by 2x.  
   - Riding the bike at normal speed modifies the final encounter rate by 1/2x. Riding the bike at boosted speed modifies the final encounter rate by 1/5x.
1. `rand.NextInt(100)` to determine if a spawn occurs based on the encounter rate calculated in step #2. If not, then go back to step #1 and check lead ability, then attempt the check for `attempt + 1`.
1. Once a hidden spawn is generated, the game performs more checks and attempts to place it. Please see [this writeup](https://gist.github.com/Lincoln-LM/28aa7865ae4ff738ab2a7feae3fba65c) by [Lincoln-LM](https://gist.github.com/Lincoln-LM) explaining this process in more detail.

## Determining if a Symbol Encounter is Spawned  
I did not look into this in great detail, but it looks like it triggers `sub_7100CE0100` as well. If you have more information and want to contribute, please let me know so I can fill in this part!

## Determining the Encounter Slot  
This now pertains to both hidden and symbol encounters. SWSH uses many spawn-boosting mechanics to alter the potential spawns. The following are performed in order.

### Lead Ability Activation  
A `rand.NextInt(100)` is performed to check the lead ability activation. The lead ability is checked in `sub_7100D30AB0`. These are the flags that are set:  

       1 Illuminate, Arena Trap, No Guard
       2 Stench, Quick Feet, White Smoke, Infiltrator
       4 Compound Eyes, Super Luck
       8 Synchronize
     x20 Cute Charm+
     x40 Magnet Pull*
     x80 Lightning Rod, Static*
    x100 Flash Fire*
    x200 Storm Drain*
    x400 Harvest*

\+ Cute Charm activates if `rand + 1 <= 66`.  
\* These type-pulling abilities activate if `rand >= 49`, which is exactly 51% to activate! GF has an off-by-one error.

### KO Respawning  
If the player has knocked out a symbol encounter prior to the next spawn, the game performs a `rand.NextInt(100)`. This is checked in `sub_7100D31EA0`. If `rand > 49`, the game then attempts to see if the KO'd species/form can be found in the current encounter table. This step is skipped for hidden encounters.
1. If the roll fails, or if the species/form combination is not in the current encounter table, then KO respawning fails.
1. If a roll succeeds and the slot is found, then that slot is used for the spawn.

### Type Pulling Abilities  
If a slot still has not been chosen, the game then checks the lead ability flags determined in step #1. This is handled in `sub_7100D31BE0`.  
- If a type-pulling ability is found, and multiple Pokémon of that type exist in the current encounter table, then a `rand.NextInt(total)` is performed to pick one of them. This smaller list of Pokémon of a particular type is in order from index 0 to index 9.

### Pokédex Recommendations   
If a slot still has not been chosen, the game then checks to see if the Pokédex has any recommendations that can boost particular species. This is handled in `sub_7100D32530`.
1. The game always performs a `rand.NextInt(100)` and checks if this `rand > 49`. If this check is satisfied, then the recommendation fails and the bonus does not activate.
1. If the bonus activates and any Pokédex Recommendations are active, then the game uses a `rand.NextInt(4)` to pick one of the 4 recommendations.
1. If this species exists in the current encounter table, then the first match is chosen as the slot.

#### A few things to note about Pokédex Recommendations...  
* Recommendations are global; even if a Pokémon is shown in a particular area, it affects all encounter tables with that species.
* They only bonus the first species match for a Pokémon in the encounter table. For example, only Average Size Pumpkaboo and Phony Sinistea can be chosen, because they have the lowest index.
* If a day advances, all of the recommendations are cleared. New ones are not generated unless the player checks them in the Pokédex. This step is skipped if recommendations are not set.
* The Pokédex Recommendations block has a seed used to generate each day's recommendations. This seed is advanced and recommendations are generated the first time the player looks at the Pokédex that day. This also means the recommendations are deterministic, but because the player has to view the Pokédex repeatedly to advance the seed, knowing in advance doesn't save any work.

### Normal Encounter Slots  
At this point, if no previous slot was chosen, the game uses a `rand.NextInt(100)` to select from the normal encounter table. The details are handled in `sub_7100D328D0`.

### Pokémon Generation  
From here, SWSH generates initial values in `sub_7100D32A00`.  
* Form, unused because it is for Minior only.
* Level, generated as `rand.NextInt(level_delta) + level_min`. There is code for a particular lead ability flag `0x10` to force maximum level, but nothing sets it in the previous lead ability check. If the encounter slot Pokémon is higher than the player's maximum captureable level, it becomes shiny-locked.
* Marks are generated via [this algorithm](https://gist.github.com/SciresM/e50648f8e30136c4f99883fe1652d407) by SciresM. If the player owns the Mark Charm, up to 3 rolls are attempted.

Further overworld values are generated in `sub_7100D311F0`.  
* This includes checks for Brilliant Pokémon (if not a hidden spawn), fishing chain, and all values for the Pokémon itself.
* This function contains another mark check, which will overwrite the previous one from `sub_7100D32A00` if this function is called. Presumably, this is because the previous function is used for basic Pokémon such as with trees, which don't have overworld placement.
* I won't cover this in more detail since it is already known by the RNG manipulation community.