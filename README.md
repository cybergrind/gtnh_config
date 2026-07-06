# GTNH Server Config Tracking

This repository tracks **what we change in the GregTech: New Horizons server config and
why**, so every change is reviewed and recorded *before* it is applied to the live server.

The live server lives at `/mnt/extra/1000/games/minecraft/server`. This repo is the
written record of changes made to the config files under that directory
(`config/`, `serverutilities/`, start scripts, etc.).

## How we work (change workflow)

Every change follows three steps, in order:

1. **Find** what needs to change — locate the exact config file (or the relevant GTNH
   wiki page) and the exact key/line.
2. **Document** it here — record *where* (file + line), *what* (before → after value),
   and *why*. This happens **before** the change is applied.
3. **Apply** the change to the live config, then mark it ✅ Applied below.

Status legend: 📝 Planned · ✅ Applied · ↩️ Reverted

---

## Environment

| Item | Value |
|------|-------|
| Server directory | `/mnt/extra/1000/games/minecraft/server` |
| Modpack | GregTech: New Horizons (Minecraft 1.7.10, Forge) |
| Java runtime | **Java 21 LTS** (`/usr/lib/jvm/java-21-openjdk/bin/java`) — see Change 0 |
| Launcher | `lwjgl3ify-forgePatches.jar` via `startserver-java9.sh` |
| Listen port | 25565 |

> ⚠️ The system default `java` is **Java 26**, which the modpack cannot use. Always
> launch the server through `startserver-java9.sh`, which pins Java 21 explicitly.

---

## Change 0 — Fix server boot (Java + Galacticraft) ✅ Applied

The server would not boot. Two stacked problems:

**0a. Wrong Java version.**
- **Symptom:** crash at early mixin transform —
  `Class file major version 70 is not supported by active ASM (version 9.0 supports class version 69)`.
- **Cause:** the system default `java` is Java 26 (class file major version 70). GTNH's
  bundled ASM 9.0 only supports up to class version 69 (Java 25).
- **Where:** `startserver-java9.sh`
- **What:** launch with Java 21 instead of the bare `java` on PATH.
  ```diff
  +JAVA="/usr/lib/jvm/java-21-openjdk/bin/java"
  -   java -Xms24G -Xmx24G -Dfml.readTimeout=180 @java9args.txt -jar lwjgl3ify-forgePatches.jar nogui
  +   "$JAVA" -Xms24G -Xmx24G -Dfml.readTimeout=180 @java9args.txt -jar lwjgl3ify-forgePatches.jar nogui
  ```
  (`startserver-java9.bat` got an explanatory comment only — Windows path differs.)

**0b. Galacticraft native crash.**
- **Symptom:** `SIGSEGV` in `ld-linux` during `dlopen` of the JPEG codec, triggered by
  `micdoodle8.mods.galacticraft.core.GalacticraftCore.postInit` calling `ImageIO`.
- **Cause:** Galacticraft writes a JPEG at post-init; the JDK JPEG native codec crashes
  when loaded inside the modded process. (A standalone Java 21 JPEG write works fine, so
  it is modpack-specific, not a broken JDK.)
- **Where:** `java9args.txt`
- **What:** force headless AWT.
  ```diff
   -Dfile.encoding=UTF-8
  +-Djava.awt.headless=true
   -Djava.system.class.loader=com.gtnewhorizons.retrofuturabootstrap.RfbSystemClassLoader
  ```

**Result:** server boots cleanly on Java 21, passes Galacticraft post-init, and binds
port 25565.

---

## Change 1 — Disable pollution ✅ Applied

GregTech's global pollution mechanic (fog, recolor, debuffs).

- **Where:** `config/GregTech/Pollution.cfg`, line 5, section `pollution`
- **What:**
  ```diff
  -    B:"Activate Pollution"=true
  +    B:"Activate Pollution"=false
  ```
- **Why:** disable pollution entirely as requested.
- **Note:** the master `Activate Pollution` switch turns the whole system off; the
  per-machine pollution amounts below it are left untouched (they become irrelevant
  while the system is off).

---

## Change 2 — Raise claimable chunks (5×) ✅ Applied

Max number of chunks a player/team can **claim** (protect), per rank.

- **Where:** `serverutilities/server/ranks.txt`, key `serverutilities.claims.max_chunks`
- **What:**

  | Rank | Before | After |
  |------|-------:|------:|
  | `[player]` | 100 | **500** |
  | `[vip]` | 500 | **2500** |
  | `[admin]` | 1000 | **5000** |

- **Why:** allow much larger claimed areas (5× the current limits).
- **Note:** `[admin]` also has `serverutilities.claims.bypass_limits: true`, so the admin
  number is mostly cosmetic, but is raised for consistency.

---

## Change 3 — Raise active / loadable chunks → 2000 ✅ Applied

Max number of chunks that can be **chunk-loaded** (kept active/ticking when no player is
nearby), per rank.

- **Where:** `serverutilities/server/ranks.txt`, key `serverutilities.chunkloader.max_chunks`
- **What:**

  | Rank | Before | After |
  |------|-------:|------:|
  | `[player]` | 50 | **2000** |
  | `[vip]` | 100 | **2000** |
  | `[admin]` | 1000 | **2000** |

- **Why:** requested target of 2000 active loadable chunks.
- **⚠️ Note:** you can only chunk-load chunks you have **claimed**, so the *effective*
  number of active chunks for a rank is capped by that rank's
  `claims.max_chunks` (Change 2). Example: a `[player]` can load at most 500 (their claim
  cap), even though the chunkloader limit is 2000. Active/ticking chunks are the heaviest
  thing on server performance — watch TPS after raising this.

---

## Change 4 — Disable explosions (force off, server-wide) ✅ Applied

Creeper / TNT / other explosions across the whole server.

- **Where:** `serverutilities/serverutilities.cfg`, line 371, section `world`
- **What:**
  ```diff
  -    S:enable_explosions=TRUE
  +    S:enable_explosions=FALSE
  ```
- **Why:** disable explosions everywhere to protect bases. `FALSE` forces explosions off
  for everyone (overrides per-team settings). `DEFAULT` would instead let each team
  decide for their own claim — not what we want here.

---

## Change 5 — Extra hearts → boost passive health regen ✅ Applied

GTNH has **no direct "set max hearts" config**. Permanent extra hearts come from
Tinkers' Construct **heart canisters** (a gameplay item you craft/fill — see note). To
reduce early-boss-fight instant-death frustration via config, we make passive health
**regenerate faster and stay fast when low**.

- **Where:** `config/HungerOverhaul/HungerOverhaul.cfg` (the active preset; `default.cfg`
  and `blankslate.cfg` are unused templates).
- **What:**
  ```diff
  -    I:healthRegenRatePercentage=100
  +    I:healthRegenRatePercentage=200
  ...
  -    B:modifyRegenRateOnLowHealth=true
  +    B:modifyRegenRateOnLowHealth=false
  ```
  | Key | Before | After | Effect |
  |-----|-------:|------:|--------|
  | `healthRegenRatePercentage` | 100 | **200** | Heals 2× faster |
  | `modifyRegenRateOnLowHealth` | true | **false** | Low health no longer *slows* regen — you recover from near-death at full speed |
- **Why:** the user wants to avoid instant-death frustration in early Thaumcraft /
  Twilight Forest boss fights. Faster, low-health-friendly regen makes survival more
  forgiving without changing game balance as much as flat extra max HP.
- **Note (real extra hearts):** for actual additional max hearts, use Tinkers' Construct
  **heart canisters** — craft them and they permanently raise max HP when filled. That is
  the GTNH-intended progression and needs no config change. `TinkersConstruct.cfg` →
  `"Drop heart canisters on death"=false` means you won't lose them on death.

---

## Change 6 — FindIt search radius (16 → 128) ✅ Applied

How far the FindIt item/block search reaches.

- **Where:** `config/findit.cfg`, section `general`
- **What:**
  ```diff
  -    S:SearchRadius=16
  +    S:SearchRadius=128
  ```
- **Why:** large sprawling multiblock bases need a much wider search range.
- **Note:** `S:MaxResponseSize=20` still caps how many results are returned per search;
  raise that too if 20 hits feel too few across a 128-block radius.

---

## Change 7 — Mob griefing off ✅ Applied (config part) + 📝 console command

Stop mobs from destroying/altering the world.

**7a. Comprehensive (all mobs) — world gamerule.** No config file; run once in the live
server console (persists in the world save):
```
/gamerule mobGriefing false
```
This stops creeper terrain damage, enderman block pickup, zombies breaking doors,
sheep eating grass, etc.

**7b. Creeper-only backstop — config file.**
- **Where:** `config/CodeChickenCore.cfg`
- **What:**
  ```diff
  -	environmentallyFriendlyCreepers=false
  +	environmentallyFriendlyCreepers=true
  ```
- **Why:** ensures creepers never grief terrain even if the gamerule is ever reset. Note
  this is creeper-only; the gamerule (7a) is the comprehensive switch.

---

## Change 8 — Keep inventory on 📝 console command

Players keep their items/XP on death (protects hard-earned backpacks and tools).

- **Where:** world gamerule — **no config file**.
- **What:** run once in the live server console (persists in the world save):
  ```
  /gamerule keepInventory true
  ```
- **Why:** requested to avoid losing gear on death. (Debatable for balance, but chosen.)

---

## Change 9 — LAN listen on 0.0.0.0 ✅ Already correct (no change)

Make the server reachable on the local network.

- **Where:** `server.properties`
- **State:** `server-ip=0.0.0.0` is **already set**, and the server binds `*:25565`
  (all interfaces) — verified with `ss -ltn`. The server is already LAN-reachable.
- **If a LAN client still can't connect, the listener is not the problem — check:**
  - `white-list=true` → the player must be on the whitelist (`/whitelist add <name>`),
    or set `white-list=false`.
  - `online-mode=true` → clients must be logged into a valid Mojang/Microsoft account on
    the same auth as the server; set `false` only for a trusted offline LAN.
  - OS firewall must allow inbound TCP **25565**.

---

## Change 10 — Enable chunk claiming + ranks (master switches) ✅ Applied

The in-game `/claim` command was rejected with a **"chunk claiming is disabled"**
server message. Root cause: the master on/off switches were off, which made the per-rank
limits raised in Changes 2 and 3 inert.

**10a. Enable chunk claiming.**
- **Where:** `serverutilities/serverutilities.cfg`, section `world`
- **What:**
  ```diff
  -    B:chunk_claiming=false
  +    B:chunk_claiming=true
  ```
- **Why:** master switch for claiming. While `false`, ServerUtilities rejects every
  `/claim` attempt regardless of per-rank `claims.max_chunks` (Change 2). It also makes
  `chunk_loading` inert — the config comment states *"If chunk_claiming is set to false,
  changing this won't do anything,"* so Change 3 (active/loadable chunks) was dead too.

**10b. Enable ranks.**
- **Where:** `serverutilities/serverutilities.cfg`, section `ranks`
- **What:**
  ```diff
  -    B:enabled=false
  +    B:enabled=true
  ```
- **Why:** the per-rank limits from Changes 2 & 3 live in `ranks.txt` and only apply when
  the ranks system is active. With ranks off, everyone falls back to default behavior and
  the raised `[player]/[vip]/[admin]` caps never take effect.

**Result:** `/claim` works, and the raised claim/loader caps from Changes 2 & 3 are now
live. Requires a server restart.

---

## Change 11 — Forestry builder's backpack: accept decorative blocks ✅ Applied

Let the **Forestry builder's (woven) backpack** pick up decorative blocks it does not accept
by default. Started as a handful of specific blocks and grew to **all decorative blocks from
Ztones, Chisel, and ArchitectureCraft** (plus one ProjectRed lamp).

- **Where:** `config/forestry/backpacks.cfg`, section `backpacks` → `builder`, key
  `item.stacks`
- **What:** appended one specific stack plus **264 wildcard stacks**:
  ```diff
       S:item.stacks <
       ...
  +        ProjRed|Illumination:projectred.illumination.lamp:29
  +        Ztones:auroraBlock:*
  +        ...            (41 Ztones blocks)
  +        chisel:acacia_planks:*
  +        ...            (221 Chisel blocks)
  +        ArchitectureCraft:shape:*
  +        ArchitectureCraft:shapeSE:*
        >
  ```
- **Why:** requested — accept all decorative blocks from these three mods.
- **No name-glob exists.** Forestry's parser (`forestry.core.utils.Stack.parseStackString`)
  splits each entry on `:+` and requires **exactly** `<modId>:<name>`, `<modId>:<name>:<meta>`,
  or `<modId>:<name>:*`. The `*` is the **only** wildcard and applies **only to metadata**;
  the name must resolve to a real registered block/item. So `chisel:*` is impossible — every
  block is listed individually, with `:*` to cover all of its metadata variants in one line.
- **`*` verified externally** against the GTNH Forestry source
  ([`GTNewHorizons/ForestryMC`](https://github.com/GTNewHorizons/ForestryMC), `Stack.java`):
  ```java
  meta = parts[2].equals("*") ? OreDictionary.WILDCARD_VALUE
          : NumberFormat.getIntegerInstance().parse(parts[2]).intValue();
  ```
  `:*` → `OreDictionary.WILDCARD_VALUE` (all metadata). Matches the shipped jar bytecode.
- **Block list source:** the live world registry (`World/level.dat`, the persisted FML
  `ItemData`) — the authoritative list of what is actually registered on this server. Counts:
  Ztones 41, Chisel 221, ArchitectureCraft 2 (`shape`, `shapeSE`).
- **Scope = decorative only** (per request). Excluded as tools/machines/items:
  `chisel:{chisel,diamondChisel,netherStarChisel,obsidianChisel,autoChisel,upgrade,cubit,tallow,cloudinabottle}`,
  `Ztones:{booster,minicoal,minicharcoal}`, and all ArchitectureCraft non-shape entries
  (`sawbench`, `largePulley`, `hammer`, `sawblade`, `glowbrush`, `cladding`, `chisel`).
- **Format note:** the earlier request wrote `modid:name/meta` (slashes); Forestry uses
  **colons**, so all entries use `:`. The lone ProjectRed lamp keeps its specific meta `29`.
- **Note:** Forge re-sorts these lists alphabetically when it rewrites the file, so final
  on-disk ordering will differ after first load. A sibling `ore.dict` list (ore-dictionary
  names) exists in the same section — unused here since we target concrete block IDs.

---

## Applying changes

Config changes require a **server restart** to take effect (1.7.10 Forge reads these at
load). `ranks.txt` can be reloaded in-game with `/serverutilities reload` for some keys,
but a restart is the reliable path.
