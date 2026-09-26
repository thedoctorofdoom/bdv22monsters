# AGENTS.md

Guidance for AI agents working in this repository.

## What this is

`bdv22monsters` ("BDMO", Brutal Doom Monsters Only) is a GZDoom/UZDoom mod
extracted from **Brutal Doom v22**. It keeps BD's monsters, gore, enemy
vehicles, and map-compatibility patches, and strips out everything
player-facing: weapons, HUD, menus, tips, skills, player classes, and
non-English localization. The point is to play BD-style enemies and gore
alongside **any** weapon mod, or vanilla weapons.

The repo is a **loose-folder mod**, not a packaged `.pk3`. UZDoom loads the
directory directly via `-file`. There is no build step for anything except
ACS (see below).

Many comments mention "Phase N" and "Monsters Only sanitization". They refer
to the staged removal that produced this repo. The audit tooling
("BDMO-tools") is **not** in this repo.

## Load order and entry points

The engine discovers content by lump name at the repo root:

| Lump | Role |
|---|---|
| `DECORATE.txt` | Root of all actor definitions; `#include`s everything under `actors/` in dependency order |
| `zscript.zc` | ZScript root (`version "3.2.3"`); includes `zscript/BD_WCS.zc` and `zscript/BDMO_GoreShim.zc` |
| `MAPINFO.txt` | Registers event handlers `ZC_TMapHack` and `BDMO_GoreShimHandler` |
| `LOADACS.txt` | Autoloads only `BD_Main` (the rest are commented-out legacy libraries) |
| `CVARINFO.txt` | Only the cvars that remaining ACS/DECORATE still read |
| `language.enu` | Only string table (`[enu eng en default]`): monster tags and obituaries |
| `SNDINFO.*`, `GLDEFS.txt`, `DECALDEF.*`, `modeldef.*`, `*.bm` | Sounds, dynamic lights, decals, 3D-model gore, brightmaps |

**Order matters in `DECORATE.txt`.** ZScript compiles first, so DECORATE can
inherit from ZScript classes such as `BDMO_GoreMonster`. Within DECORATE,
base classes must be included before their subclasses. For example,
`actors/Support/*` comes after `actors/MISC/FIRE.dec` because some Support
actors subclass FIRE actors. Commented-out `#include`s (most `Weapons/*`,
`ARACHNORB`, `LedgeGrab`, `BrutalChexQuest`) are intentionally disabled.

## Directory map

- `actors/Enemies/` contains BD's monster replacements. Each one uses `Replaces <VanillaClass>`, for example `Imp : BDMO_GoreMonster Replaces DoomImp`.
  - `Bosses/`, `ST Monsters/` hold extra bosses and monsters.
  - `WCS/` is the Wad Compatibility System: replacement monsters for specific megawads (Eviternity, Ancient Aliens, Doom Zero, Wormwood, NERF, and others).
- `actors/Gore/` holds blood, gibs, limbs, burn/freeze deaths, and colored blood types.
- `actors/MISC/`, `actors/Support/` hold effects (particles, smoke, fire, puffs, casings) and actors that the kept gore/monster code still references.
- `actors/Weapons/` keeps only what monsters and vehicles still use: projectiles, tracers, explosives, melee, plasma, and BFG effects. It holds **no player weapons**.
- `actors/PlayerClasses/` is a historical name. It contains monster variants (stealth, dying, last-stand, underwater, Doom 1 bosses) and footsteps, not player classes.
- `actors/VEHICLES/` holds enemy vehicles (tanks, mech, helicopter, HMG), crashes, and critters.
- `actors/TOKENS.dec` defines BD's inventory tokens (weapon-selected markers and similar).
- `zscript/` holds the only ZScript (see below).
- `src/` holds ACS **source**; `acs/` holds the **compiled** `.o` libraries the engine actually loads.
- `sprites/`, `sounds/`, `models/`, `graphics/`, `brightmaps/` hold assets (about 65 MB, mostly sprites).

## ZScript

### `zscript/BDMO_GoreShim.zc` (the key BDMO-specific logic)

BD monsters choose weapon-specific death animations (chainsaw bisect, SSG
blowapart, plasma melt, and similar) by checking for tokens in the killer's
inventory, such as `A_JumpIfInTargetInventory("HasCutingWeapon", ...)`. Only
BD's own weapons grant those tokens. The shim fakes them for any other
weapon:

1. `BDMO_GoreMonster` overrides `DamageMobj` and calls `BDMO_GoreShim.Process`
   before damage lands. Most BD monsters inherit from it.
2. `BDMO_GoreShimHook` does the same through passive `ModifyDamage`. It covers
   monsters that must stay parented to vanilla classes, listed in
   `NeedsHook()`. `BDMO_GoreShimHandler.WorldThingSpawned` gives it to them.
3. `Classify()` looks at the player's weapon class name, its ammo type, the
   damage type, the inflictor, and the distance. It then picks which tokens
   to grant (`TK_CUT`, `TK_PLASMA`, `TK_FISTS`, `TK_SSG`, `TK_MINIGUN`,
   `TK_HEAVYAUTO`).
4. `BDMO_GoreShimLedger` records which tokens the shim granted and removes
   them after `HOLD_TICS` (10). It never touches tokens a real BD weapon
   granted.
5. The shim does nothing in these cases: explosive or fire damage (those
   deaths depend only on damage type and gibhealth), a player holding real BD
   marker tokens, a source that isn't a player, or `bd_goreshim` set to false.

Tokens are also mirrored onto the victim. BD's headshot paths call damage
again with the monster or its `HeadHitbox` as the source, so the death-state
checks read the victim's own inventory. Token classes are resolved **by
name** at runtime, because DECORATE classes don't exist yet when ZScript
compiles. Keep that pattern when you add new token references.

### `zscript/BD_WCS.zc`

Holds `ZandroCompat`/`ZC_TMapHack`, a lump-reading shim for ACS. ACS calls it
through `acs/zancmpat.o`. The WCS uses it to read the loaded megawad's
`TITLEPIC`.

## ACS

- `src/BD_Main.acs` is the library root: `#library "BD_Main"`, BCS syntax (`#import "zcommon.bcs"`, `#include "libbcs.bcs"`). It mirrors the server/client cvars into globals and holds the preset scripts (`DefaultCvars`, `AllCvarsOn`, and others).
- `src/BDMO_Monsters.acs` holds every named script that DECORATE calls by name:
  - WAD detection: `BD_HashTitlepic` sums `TITLEPIC`, matches the sum against `WadHash[]`, and sets `bd_WadCheck`. `WadChecker` then switches monster states for that megawad.
  - Megawad patches, boss health bars, and `Head_Shot`.
  - Floor-type footstep and splash scripts.
  - Vehicle enter/leave scripts.
- `src/BCSFunc.acs` holds shared math and helper functions and the vehicle world arrays.
- `src/zancmpat.bcs` is the source for `acs/zancmpat.o`.

**Compiler:** `zt-bcc` 0.10.0-alpha.8 (BCS dialect). A comment in
`BCSFunc.acs` works around its `libbcs.bcs` namespace change. The compiler is
**not installed** on this machine and not vendored in the repo. If you edit
`src/*.acs`, you must recompile to `acs/BD_Main.o` or the change has no
effect. Tell the user if you can't recompile; never hand-edit `.o` files.

A named script that DECORATE calls must exist in `BDMO_Monsters.acs` or
`BD_Main.acs`. Removing one silently breaks the actors that call it, because
the engine only reports it when the script runs. Grep `actors/` for
`ACS_NamedExecute` / `CallACS` before you delete any script.

## Cvars

Cvars live in `CVARINFO.txt`, and ACS mirrors them in `BD_Main.acs`. Key
ones: `bd_bloodamount` (per user), `zdoombrutalblood`, `zdoombrutaljanitor`
(body cleanup), `bd_classicmonsters`, `bd_AgileZombies`,
`BD_DestructibleBodies`, `BD_NOBOSSES`, `bd_WadCheck` (usually set
automatically), `bd_zombiemanspawn`, `bd_shotgunguyspawn`, and `bd_goreshim`.
There is no MENUDEF, so users change these from the console. When you add a
cvar that ACS reads, add it to both `CVARINFO.txt` and `ServerCVars` /
`ClientCVars`.

## Conventions and pitfalls

- **Case-insensitive lump names, case-sensitive filesystem tooling.** Class and actor names in DECORATE are case-insensitive, but keep file paths exactly as they are (`actors/VEHICLES`, `ST Monsters` with a space).
- **Removing content:** before deleting an actor, sprite, or sound, grep for every reference: DECORATE `#include`s, `Replaces`, spawn calls, `SNDINFO.*`, `modeldef.*`, `GLDEFS.txt`, `doommonsters.bm`, and ACS. Missing sprites and sounds only show up at runtime.
- **Never ship sprites under vanilla pickup, weapon, or projectile names** (`BON1`, `BON2`, `CLIP`, `SHEL`, `PINS`, `STIM`, `MEDI`, `TRAC`, and similar). Weapon mods reskin those under the vanilla names, so when BDMO loads after them, BDMO's copies override theirs. BDMO's own drop actors use `BM`-prefixed sprites instead (`BMHB`, `BMAB`, `BMCL`, `BMPS`, `BMTR`). For the same reason, don't give BDMO actors vanilla `SpawnID`s.
- **Strings:** add new user-facing text to `language.enu` only. Other languages were removed on purpose. Engine `OB_*` obituaries fall back to UZDoom's built-in English strings.
- Match the surrounding style. DECORATE uses tabs and BD's original naming (including misspellings such as `HasCutingWeapon` and `TehArchvile`). Don't "fix" names; other code references them.

## Testing

Follow the user's UZDoom macOS rule:

```bash
/Applications/uzdoom.app/Contents/MacOS/uzdoom \
  -iwad "/path/to/DOOM2.WAD" \
  -file "/path/to/bdv22monsters" > /tmp/uzdoom_run.log 2>&1 &
# wait ~10s, then kill the pid
```

- Run it outside the sandbox (UZDoom needs to write its config).
- Capture stdout; `+logfile` doesn't work on the macOS build.
- There's no `timeout` command, so background the process and kill it.
- A clean run shows `adding .../bdv22monsters, N lumps` and `Init complete.` with no `Script error`.
- Add `-warp 1` to reach a map and test gore and monster behavior. To exercise the gore shim, add a weapon mod as another `-file` entry. If both mods define the same lump, the one loaded later wins.
