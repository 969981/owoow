## Introduction
Pokémon Sword and Shield carried over wandering overworld Pokémon from the Let's Go games. Unlike LGPE, these overworld wanderers have identifiable origins, and thus it is possible to legality check them by verifying the correlation between certain values in their data similar to gen 3-4 PIDIV.

Overworld data is stored in the save file in the overworld block.  This is why you can save in front of a strong spawn or Galarian bird, reset the game, and encounter exactly the same Pokémon.

## Can Overworld RNG be abused or manipulated?
It was recently discovered in October 2021 that this is possible to do.  Current methods can be done with or without CFW.  Additionally, the presence of overworld data allows for the creation of CFW bots that simply scan overworld data for shiny or marked Pokémon without ever engaging in battle.

## What Pokémon does Overworld RNG apply to?
The categorization can seem complex, so let me break it down:

### These encounters are confirmed to use Overworld RNG.  
In general, you'll be able to run into these while on your bike or walking.  These Pokémon use the forced shiny mechanic, which causes 65521/65536 of all shinies to be square shiny.  These encounters have marks, Synchronize, and use Shiny Charm.
- Encounter slots*: Pokémon that spawn in the grass, water, or wandering in caves with a range of levels.
- Hidden ! encounters: e.g. everything in the grass in Glimwood Tangle.
- Static encounters on the overworld: e.g. Galarian birds, Swords of Justice, and "strong spawns".
- Static encounters scattered around the map that you can run into: e.g. Route 5 Eldegoss and the Galar Mine Carkol.
- Fishing spots*: Any Pokémon that you can fish from a pool.

_* Starred encounters can have Brilliant Aura._  Brilliant Aura increases their minimum 31 IVs from 0 to 2-3 and sets their level to the maximum possible for that encounterslot prior to upscaling to 60.  Encounters that do not have any guaranteed IVs (e.g. through Brilliant Aura or as a legendary) cannot have 6 perfect IVs through Overworld RNG.

### These encounters do not use Overworld RNG.
These are spawned with cryptographically-secure RNG, and are generally 15/16 star shiny.  Other than trees, these encounters tend to be similar in that they do not have marks, do not Synchronize, and do not use Shiny Charm.
- Tree encounters: Any Pokémon that falls out of a tree when shaken. These encounters **do** have marks, Synchronize, and use Shiny Charm.
- Scripted static encounters: Think of this as anything you have to click on to spawn, such as the Regis, Zacian/Zamazenta, Eternatus, Calyrex, Spectrier/Glastrier, and the Impidimp/Morgrem in Glimwood Tangle.
- Gift Pokémon: e.g. Cara Liss's fossils, Kubfu, Type: Null, Cosmog.
- Regina's in-game trades.
- Motostoke Gym encounters: Even though they imitate the ! of hidden encounters, these are actually scripted encounters.
- Curry encounters: the rare Pokémon that appear in camp when making curry.

## What traits are controlled by Overworld RNG?  
When a Pokémon is generated in the overworld block, it is stored as a collection of values for different traits along with a 32-bit seed.  Currently, we cannot verify any correlation between the separate values and the 32-bit seed.  How these values are initially determined is still mysterious.

The basic values that are stored separately include: species, form, level, shiny (random, forced shiny, forced nonshiny), nature, gender, ability (1/2/HA), item if present, guaranteed IVs, mark, and Brilliant Aura.

Some Pokémon use additional fields to add in extra moves that they normally would not have at their encounter level, such as Galarian Zapdos's Thunderous Kick.  The move fields are also used to store the egg move for Brilliant Aura Pokémon.

Additionally, there are (as far as I know) unused fields that can lock in a Pokémon's IVs or EVs.

Each Pokémon has a 32-bit seed that is used to generate the PID, EC, IVs, and height/weight.  (Height and Weight are unused values in Sword/Shield, but are still generated through this RNG.)

## How is the 32-bit seed used to generate the last few values?  
SWSH's RNG uses xoroshiro128+, which has already been described in greater detail by Admiral Fish on his [Raid Generation write-up](https://github.com/Admiral-Fish/RNGWriteups/blob/master/Gen%208/Raid%20Generation.md).  Overworld RNG uses a simplified version of this, and only follows the steps for EC, PID, IVs, height, and weight in that order.

This RNG is the basis for legality checking that PKHeX uses for its [Overworld8RNG class](https://github.com/kwsch/PKHeX/blob/master/PKHeX.Core/Legality/RNG/Methods/Overworld8RNG.cs).

## You referenced a guide about Raid Generation.  What about Max Raids, Max Lair, and Eggs?
- Max Raid generation is already linked above.  It is actually verifible, but intensive to do so since they use 64-bit seeds.
- Max Lair generates Pokémon similarly to Max Raids.  All of them are generated as never-shiny and forced shiny to star (xor1) if you succeed the [final shiny roll at the end](https://twitter.com/SciresM/status/1319878015581523968).
- [Egg generation](https://github.com/Admiral-Fish/RNGWriteups/blob/master/Gen%208/Egg%20Generation.md) has also been described by Admiral Fish.

## Resources  
- If you're curious about looking at your own Overworld block and can dump it from a save file, check out theSlayer's [SWSH Overworld Checker](https://projectpokemon.org/home/files/file/4423-swsh-overworld-checker/).
- All of the information about which encounters use Overworld RNG and which ones can have Brilliant Aura was checked on my dataset, which I've uploaded to the [Anubirb](https://github.com/Lusamine/Anubirb) repository.