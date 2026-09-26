# Brutal Doom v22 — Monsters Only

Brutal Doom's monsters, gore, and enemy vehicles, with none of its weapons,
HUD, or menus. Pair it with whatever weapon mod you like, or play with
vanilla weapons, and still get Brutal Doom enemies and deaths.

## What you get

- **Brutal Doom v22 monsters.** They replace every vanilla Doom enemy (zombiemen through the Spider Mastermind), and each gets BD's attacks, behavior, and variants. Stealth monsters, extra bosses, and enemy tanks, mechs, and helicopters come along too.
- **The full gore system.** You get blood sprays and decals, gibs, severed limbs, burning and frozen deaths, and colored blood for the right monsters.
- **Weapon-aware deaths with any weapon.** Brutal Doom picks special death animations (chainsaw splits, super shotgun blowouts, plasma melts) based on which weapon you're holding. This mod works out what your weapon *is* from its name, ammo, and damage type, so those deaths also trigger with non-Brutal-Doom weapons.
- **Megawad compatibility.** The mod recognizes several popular megawads by their title screen and swaps in suitable enemies for them. Supported megawads include Eviternity, Ancient Aliens, Scythe II, Doom Zero, Wormwood, and Not Even Remotely Fair.

## How to play

You need GZDoom or UZDoom and a Doom II IWAD. Load this folder the same way
you would a `.pk3`:

```bash
uzdoom -iwad DOOM2.WAD -file /path/to/bdv22monsters
```

To add a weapon mod, list it in the same `-file` argument:

```bash
uzdoom -iwad DOOM2.WAD -file /path/to/MyWeaponMod.pk3 /path/to/bdv22monsters
```

## Settings

There is no options menu. To change a setting, open the console with `~`
and type its name followed by a value, for example `bd_bloodamount 3`.

| Setting | What it does |
|---|---|
| `bd_bloodamount` | Amount of blood (default 2) |
| `zdoombrutaljanitor` | Clean up corpses and gibs over time (0 = off) |
| `BD_DestructibleBodies` | Corpses can be blown apart (default on) |
| `bd_AgileZombies` | More agile zombie behavior |
| `bd_classicmonsters` | Swap Brutal Doom monsters back to vanilla versions |
| `BD_NOBOSSES` | Keep vanilla bosses instead of Brutal Doom's custom boss replacements on specific maps |
| `bd_goreshim` | Weapon-aware deaths for non-Brutal-Doom weapons (default on) |

## For modders

The mod is a plain folder with no build step. DECORATE, ZScript, and asset
changes take effect the next time you launch the game. The one exception is
ACS: scripts in `src/` must be recompiled with `zt-bcc` into `acs/` before
the game sees them. See [AGENTS.md](AGENTS.md) for a full technical tour of
how the pieces fit together.

## Credits

The monsters, gore, and assets come from Brutal Doom v22 by Sergeant_Mark_IV
and its contributors.
