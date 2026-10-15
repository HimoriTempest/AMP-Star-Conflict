# AMP Generic Module template: Star Conflict

Hosts the self-contained Star Conflict server stack in CubeCoders AMP.

The server files come from either a **server tarball** downloaded from a URL (default: [`star-conflict-server.tar.gz`](https://nx.himori.dev/public.php/dav/files/wyxJddcXopyeJDW)) or **SteamCMD** (Steam app 212070). Choose with **Server Files Source** under the Updates settings; see [Server files](#server-files). The tarball contains `cloud/` (including the game's own MongoDB 2.6.12 in `cloud/mongodb/`, which the template only uses to migrate old data), all of `data/` (including every `.pak` archive) and `sys_data/`, about 12 GB. Both sources install into `star-conflict/instance/server/`.

- `Hub.exe` is the supervisor. It reads `cloud/data/cloud.xml` and spawns about 17 services: LoadBalancer, Shard, MatchMaker, Chat, Universe, ZoneCoordinator and others.
- `ZoneCoordinator` spawns one `DedicatedServer.exe` per battle.
- Everything persists to MongoDB on `127.0.0.1:27017`. The services connect as soon as they start, so **MongoDB must be up before Hub starts**. `hub-wrapper.sh` starts the [bundled MongoDB](#bundled-mongodb) and waits until it accepts connections before it starts Hub (up to 60 seconds, otherwise the start is aborted), and shuts it down cleanly after Hub exits.

The instance folder. AMP's file manager root (and what AMP backs up) is `instance/`; the Wine prefix and the generated scripts sit above it, out of the way:

```
star-conflict/
├── instance/                file manager root
│   ├── server/              game files (tarball or SteamCMD install target)
│   │   ├── cloud/  data/  sys_data/
│   │   └── server-logs/     all game logs and crash dumps
│   └── database/            bundled MongoDB 3.4, native Linux build
│       ├── mongodb/         binaries (downloaded on update)
│       ├── data/            the database
│       └── mongod.log
├── .wine/                   Wine prefix
└── hub-wrapper.sh, mongodb.sh, amp-settings.sh   written by the template
```

Server-file updates never touch `database/` or `server/server-logs/` (neither the tarball nor the Steam files contain them).

Earlier layouts are moved over automatically by "Move Into Instance Folder", the first update and start stage: `star-conflict/server/` (before 2.1) and `star-conflict/database/` (2.0) are renamed into `instance/`. If both the old and the new folder hold files, the new one is used and the old one is left alone, with a warning in the console. Before 2.0 the database was MongoDB 2.6 under Wine in `server/mongo-data/` (before 1.12: `star-conflict/mongo-data/`) and SteamCMD installed into `212070/`; see [Bundled MongoDB](#bundled-mongodb) and [Server files](#server-files).

The game's binaries are Windows-only (Hub and services are 32-bit). The template is Linux only: they run under Wine inside AMP's `cubecoders/ampbase:wine-stable` container. MongoDB runs natively.

## Files

| File | Purpose |
|---|---|
| `star-conflict.kvp` | Main template. On Linux it runs `star-conflict/hub-wrapper.sh` (written by a start stage), which starts `wine Hub.exe` from `instance/server/cloud`, streams Hub's log into the AMP console and forwards stop signals to Hub. |
| `star-conflictupdates.json` | Gets the server files into `server/`: either downloads and extracts the tarball from Server Tarball URL, or installs Steam app 212070 with SteamCMD. It also installs MongoDB 3.4 into `database/` (`mongodb.sh install`) and initialises the Wine prefix. |
| `star-conflictstart.json` | Pre-start: writes the `*_local.cfg` files, the starter pack, the AMP block in `default_platform_config.lua` (via `amp-settings.sh`) and the `cloud.xml` node address. Then `mongodb.sh prepare` stops anything left from the last run, migrates old MongoDB 2.6 data once and restarts the Wine server. |
| `star-conflictports.json` | Ports exposed to players. |
| `star-conflictconfig.json` | AMP settings: public address, accounts (starting credits/GS, premium), economy (XP/credit multipliers, prices, bonuses), battles, matchmaking. See [Settings](#settings). |
| `star-conflictmetaconfig.json` | Empty. Config is written by start stages. |
| `CONSOLE.md` | All Hub and Shard console commands, with what they do. |

## Server files

**Server Files Source** (Updates settings) decides where **Update** gets the server files:

| | Tarball URL (default) | SteamCMD |
|---|---|---|
| Source | **Server Tarball URL**: a direct `.tar.gz` link with `./cloud`, `./data` and `./sys_data` at the top. For a Nextcloud share, use `https://<host>/public.php/dav/files/<token>` or append `/download` to the share link. | Steam app 212070, Windows depot, through AMP's SteamCMD |
| Download | about 10 GB, `wget`, on every update | about 12 GB the first time, then only Steam's changes |
| Lands in | extracted into `instance/server/` | installed into `instance/server/` |
| Login | none | **Steam login required.** Anonymous download is refused (`ERROR! Failed to install app '212070' (No subscription)`, tested with SteamCMD). AMP asks for username, password and Steam Guard code during the update (`App.SteamUpdateAnonymousLogin=False`). The account needs Star Conflict in its library; it's free to play, so add it once in the Steam client. A separate Steam account for the server is recommended. |
| Version | whatever the tarball was built from | the current Steam version of the game |

Either way, files that exist only in `instance/server/` are kept, such as `common_local.cfg` and the `*_local.cfg` files. Files from the download overwrite the template's edits (`default_platform_config.lua`, `starter_pack.xml`, `cloud.xml`, `store.xml`), and the start stages reapply those on the next start. Switching from Tarball URL to SteamCMD later works the same way: Steam installs over the extracted files.

Before 2.0, SteamCMD installed into `star-conflict/212070/` and the files were hard-linked into `server/`. On the first SteamCMD update after upgrading, "Move Steam Install" links everything into `instance/server/` once more, including Steam's manifest in `steamapps/` (so Steam sees it as its install), and removes `212070/`.

## Bundled MongoDB

The game ships MongoDB 2.6 for Windows. The template doesn't use it: it runs **MongoDB 3.4.24 for Linux** natively, everything in `instance/database/`.

**Why 3.4.** The game's database driver dates from the 2.6 era and relies on behaviour later versions removed: `aggregate` without a `cursor` option (3.6 rejects it; Authenticator, MatchMaker and Shard use it for stats), index creation by inserting into `system.indexes` (removed in 4.2; there is no `createIndexes` in the binaries), and the legacy wire protocol with `getLastError` (removed in 5.1). 3.4 is the newest version that does all of it. Both old-style operations were tested against 3.4.24. It brings WiredTiger, and it's MongoDB's generic Linux build, which only needs glibc, so the container needs nothing extra.

**Install.** Update runs `mongodb.sh install`: downloads `mongodb-linux-x86_64-3.4.24.tgz` from `fastdl.mongodb.org` into `database/mongodb/`, checked against MongoDB's SHA-256. Skipped when 3.4.24 is already there. `database/data/` is never touched.

**Running.** `hub-wrapper.sh` starts `mongod` (`--bind_ip 127.0.0.1`, the instance's MongoDB Port, `--wiredTigerCacheSizeGB` from **MongoDB Cache Size (GB)**, default 1) before Hub and stops it with SIGTERM after Hub has exited, so a stopped instance has no running database and AMP backups of `database/data/` are consistent. If the wrapper itself gets killed, the next start's `mongodb.sh prepare` stops the left-over `mongod` (found by `database/mongod.pid`). See [Starting and stopping](#starting-and-stopping) for the full order.

**Migrating from 2.6.** On the first start after upgrading from a version before 2.0, `mongodb.sh prepare` finds the old 2.6 data (`server/mongo-data/`, or `mongo-data/` from before 1.12) and copies it into `database/data/` once:

1. It starts the game's `mongod.exe` 2.6 under Wine on the old data, on a spare port.
2. It starts a temporary MongoDB 3.4 on `database/data/`, on another spare port.
3. `mongodump --archive | mongorestore --archive` (3.4 tools), skipping the `admin` database (2.6's users and auth schema, which the game doesn't use).
4. It stops both, and `database/data/.migrated` records the source, time and databases.

The old folder is left as it is; delete it once the server runs fine. If anything fails, the start is aborted, `database/data/` is emptied again and the end of `database/migrate.log` is shown. It only runs into an empty `database/data/`.

This was tested with the game's own `mongod.exe` 2.6 under Wine: 20,000 accounts, a unique index and a TTL index came over intact, and the game's `aggregate` returned the right result afterwards.

**Check the old data before relying on it.** While testing, MongoDB 2.6 under Proton 10's Wine silently lost records as a collection grew (an index build right after 20,000 inserts `scanned 3424 total records`; reading on failed with `BSONObj size: -286331154 (0xEEEEEEEE) is invalid`). Under Proton 11 the same writes were fine. Whether AMP's `wine-stable` is affected wasn't tested. If the migration fails with that error, the 2.6 data was already damaged: the migration can't copy damaged records, and the old folder still holds what's there.

## Starting and stopping

AMP's Generic module has pre-start stages but nothing that runs on stop: it sends SIGTERM (`App.ExitMethod=OS_CLOSE`) and kills the process after `App.ExitTimeout`. So `hub-wrapper.sh`, the process AMP runs, does both ends, in a fixed order.

**Start:**
1. Pre-start stages: config files, `amp-settings.sh`, the node address, then `mongodb.sh prepare` (stops a MongoDB left from a killed wrapper, migrates old 2.6 data once, restarts the Wine server, which clears services and sockets left from the last run).
2. `hub-wrapper.sh` starts MongoDB and waits until it accepts connections.
3. It starts Hub, which starts the services. AMP marks the instance ready when the load balancer has registered.

**Stop** (SIGTERM from AMP), each step logged in the console as `[AMP] Stopping: …`:
1. SIGTERM to Hub; wait until it has exited.
2. The services and battle servers shut themselves down once Hub is gone. The wrapper waits for every game process in the Wine prefix (`wineserver -w`) for up to 30 seconds, then kills what's left (`wineserver -k`) and names it in a warning.
3. Only now, with nothing left that could write, MongoDB is stopped (SIGTERM, clean shutdown).
4. The wrapper stops its own helpers (the `tail -F` log followers and the accounts loop, including their child processes), prints `[AMP] Stopped` and exits with Hub's exit code. AMP only considers the application stopped once nothing holds the console output open any more, so a single leftover `tail` would keep the instance in "Stopping" (as in 2.2.0 before this was fixed).

`App.ExitTimeout` is 120 seconds, enough for the whole sequence. If AMP kills the wrapper anyway, the next start's pre-start stages clean up (step 1 of the start).

Tested with stand-ins for Hub and the services under Proton's Wine: a service that exited 17 seconds after Hub was waited for, one that hung was killed after 30 seconds, and MongoDB stopped last in both cases.

## Ports

| Port | Proto | Service | cvar |
|---|---|---|---|
| 3801 | TCP | Load balancer (clients connect here) | `lb_listenTo` |
| 3802 | TCP | Shard | `shard_listenTo` |
| 3815 | TCP | Chat | `chat_server_port` |
| 35000–35049 | TCP+UDP | Battle servers (`DedicatedServer.exe`) | `net_gamePort` |

Internal only, never forward or open these to players. Each instance gets its own from AMP:

| Port (default) | Proto | Service | Set through |
|---|---|---|---|
| 3850 | TCP | Hub, which every service connects to | `hub_local.cfg` (`hub_listenTo`) and the `cloud.xml` node address |
| 9001 | TCP | Hub's JSONP status listener | `hub_local.cfg` (`jsonp_listenPort`) |
| 27017 | TCP | Bundled MongoDB 3.4, bound to `127.0.0.1`, no auth | `mongod --port`, and `db_mongoHosts` in the AMP block of `common_local.cfg` |
| 3851 | TCP | Hub remote console (SSH), only with a Remote Console Password | `hub_local.cfg` (`ssh_port`) |
| 3852 | TCP | Shard remote console (SSH), only with a Remote Console Password | `shard_local.cfg` (`ssh_port`) |

### Several servers on one host

AMP containers share the host's network here, so two instances clash on any port they both use. With 1.9.0 every port an instance opens comes from AMP's port allocation, so several instances can run side by side:

- **Players' ports:** load balancer, shard, chat, and the battle port range.
- **Internal ports:** Hub, Hub status, MongoDB and the two console ports, as in the table above.

Each instance has its own Wine prefix, `database/` and server files. AMP assigns different ports to new instances. For instances created before 1.9.0, check that their Hub/Hub status/MongoDB ports differ, because both may show the defaults 3850/9001/27017. Players pick a server by its **load balancer port** on the login screen.

Before 1.9.0, Hub (3850), the JSONP listener (9001) and MongoDB (27017) were fixed. The second instance to start couldn't open them, and its services could connect to the first instance's Hub and database.

Not separable: two instances pointed at the **same external MongoDB** share their data. The database names come from the Wine user (`cosmosim_amp` and so on), which is `amp` in every container. Give each instance its own MongoDB server, or use the bundled one.

### IP binding and keeping the internal ports local

**The game can't listen on a chosen IP.** Every listener binds to all addresses (`0.0.0.0`):
- `hub_listenTo`, `lb_listenTo` and `shard_listenTo` take a port only. `hub_listenTo "127.0.0.1:38999"` was read as port 127 (`Failed to listen on port 127`), and Hub didn't start.
- ChatServer has `0.0.0.0:%u` hard-coded.
- Battle servers only take `net_gamePort`.

That's why the template keeps `Meta.AppIsMultiIPAware=False`. Marking it multi-IP aware would let AMP give two instances the same ports on different IPs, and since both would listen on all addresses, they'd clash.

The **bundled MongoDB** is the only internal service that can be restricted, and it is: `--bind_ip 127.0.0.1`.

**Hub and Hub status** stay reachable on every address of the host. Two things keep them private:
1. Keep their ports **outside every forwarded range**, as in [Several servers on one host](#several-servers-on-one-host). Then nothing outside the host reaches them through port forwarding.
2. For the host's other networks (NetBird, LAN, the Incus bridge), add a firewall rule in the `amp` container that only allows the internal ports on loopback. The services reach Hub through the node address, which is one of the host's own IPs, so that traffic goes over `lo` and is still allowed:
   ```sh
   # in the amp Incus container, as root (make it persistent there); ports = Hub, Hub status, MongoDB and the console ports of each instance
   nft add table inet sc_internal
   nft add chain inet sc_internal input '{ type filter hook input priority 0; }'
   nft add rule inet sc_internal input iifname != "lo" tcp dport '{ 10055, 10056, 10057, 10058, 10059 }' drop
   ```

**Keep the ports out of Linux's ephemeral range** (`/proc/sys/net/ipv4/ip_local_port_range`, usually 32768–60999). If ADS auto-assigns ports in that range (for example 60059–60061 and 60008+), an outgoing connection can grab one first, and that service then fails with `Failed to listen on port …`. Give each instance its own ports below 32768 in its port settings, for example the defaults for the first server, and 4801/4802/4815, 4850, 10001, 28017 and 36000–36049 for a second.

## Host requirements (Docker)

The template always runs in a container (`Meta.ContainerPolicy=Required`), using `cubecoders/ampbase:wine-stable`, which provides the 64-bit Wine with WoW64/32-bit support the server needs. AMP therefore needs Docker on the host. On NixOS:

```nix
# configuration.nix
virtualisation.docker.enable = true;
users.users.amp.extraGroups = [ "docker" ];   # whatever user runs ADS

networking.firewall = {
  allowedTCPPorts = [ 3801 3802 3815 ];
  allowedTCPPortRanges = [ { from = 35000; to = 35049; } ];
  allowedUDPPortRanges = [ { from = 35000; to = 35049; } ];
};
```

## Adding the template to AMP

1. Push this repo to GitHub (`HimoriTempest/AMPStarConflict`, per `manifest.json`). The repository name must not contain hyphens. AMP rebuilds the template source from `<owner>-<repo>-<branch>`, and a hyphenated repo name makes upgrades fail with `source --main`.
2. In ADS go to **Configuration → Instance Deployment → Configuration Repositories** and add `HimoriTempest/AMPStarConflict:main`. Then click **Fetch latest**.
3. Create an instance, choose *Generic*, then *Star Conflict*. AMP always creates it in a container.
4. Pick **Server Files Source** (Updates settings), then update the instance. *Tarball URL* downloads the tarball from **Server Tarball URL** (about 10 GB, no Steam login). *SteamCMD* installs Steam app 212070 (about 12 GB) and asks for a Steam login.
5. Set the address players connect to. By default the template uses **AMP's Advertised Connection Address** (instance settings → Network and Ports), as long as that IP is present on the host. **Public Address** overrides it. Hub only starts if the address is one of the host's own IPs. For internet players behind NAT, add the public IP to the host on a **dummy network interface** in the `amp` container (`ip link add sc-public type dummy; ip addr add <public IP>/32 dev sc-public; ip link set sc-public up`). Don't use `lo`: Hub ignores addresses on the loopback interface (`there is no my node in xml`), even though Wine lists them. Keep AMP's instance binding (`Webserver.IPBinding`) on a normal address such as the NetBird IP, because it is the last fallback, and keep forwarding the player ports as usual. The console shows the chosen address (`Cloud node address: … (source)`).

## Pointing clients at the server

Enter the server's address and the **load balancer port** in the server/port fields on the client's login screen. Don't use the battle port.

Steam starts the client with `-launchLocalServer`, which also starts a MongoDB and Hub on the player's machine. To stop that, empty the `executable` attributes of `<Mongo>` and `<Hub>` in `win64\local_server.xml`; its own comments say an empty executable skips startup. On Linux/Proton you can instead strip the flag with the launch option `bash -c 'exec "${@/-launchLocalServer/}"' -- %command%`.

## Settings

Every option is reversible. The default is the game's original value, and setting it back restores the original behaviour on the next start.

**How settings reach the game.** ConfigServer pushes `cloud/data/default_platform_config.lua` to every service *after* the service has read its `.cfg` files. So for anything that file defines, a `set` in a `*_local.cfg` is silently overwritten. In 1.3.x this is why Infinite Premium, Bonus GS, the player limit and starting credits had no effect. The template therefore writes most settings as an AMP block appended to that file: `-- >>> AMP managed settings` … `-- <<< AMP managed settings`, with one `if( service_type == "…" )` section per service. Only values that differ from vanilla are written. With everything at vanilla there is no block, and the file is byte-identical to the original.

| Setting | Writes | Vanilla | Applies to |
|---|---|---|---|
| Starting Credits | `credits_on_start` (Shard) | 0 | Accounts created afterwards |
| Starting Galactic Standards | `gold_on_start` (Shard) | 300 | Accounts created afterwards |
| Starting Premium Days | `premium_on_start` = days × 24 (Shard; the game's unit is **hours**), capped at 20000 days | 0 | Accounts created afterwards |
| Premium Hours Per Account Rank | `premiumTimeForAccountRank` (Shard, in seconds) | 24 | Rank-ups afterwards |
| Infinite Premium For All Players | `auth_infinitePremiumForRoles` = `gm;support;player` (Shard) | `gm;support` | All accounts with the default `player` role |
| XP Multiplier | `rate_exp` (MatchMaker) and `sv_expRate` (open space), all 18 ranks | 1 | Battles afterwards |
| Credit Multiplier | `rate_cred` (MatchMaker), all 18 ranks | 1 | Battles afterwards |
| First Win Of The Day XP Multiplier | `sv_dailyWin{Pve,Pvp,Coop,Brawl}ExpRewardCf` (battles, open space) | 3 | Battles afterwards |
| Bonus GS Per Win / Per Loss | `reward_bonusGoldForWinner` / `reward_bonusGoldForLooser` (reward service) | 0 / 0 | Battles afterwards |
| Daily Token Credit Limit | `sv_tokenCreditsDailyLimit` (Shard, reward service) | 1000 | Immediately |
| Ship / Module / Ammo Price Multiplier | `store_vesselPriceMod` / `store_modulePriceMod` / `store_munitionPriceMod` (Shard) | 1 | Store |
| AFK Kick Time (seconds) | `sv_afkCheckDuration` (battle servers) | 120 | Battles |
| Open Space Resurrection Cost | `sv_resurrectionCreditsPrice` (open space) | 100000 | Open space |
| Max Players (AMP's own player limit, `$MaxUsers`) | `lb_limitPlayersOnShardTo` (load balancer). The server has one shard, so this is the server-wide limit. | 800 | Logins |
| Ships In Arena Matches | **Total** ships per arena match, bots filling up to it (6 gave player + 5 bots). Capped by the queue: 16 in the newbie arena queue, 24 otherwise. `cvars.mm_botsCount` line, plus `botsCount` in both arena queue templates (`special = { arcade = 1 }` becomes `special = { arcade = 1, botsCount = N }`), edited in place like SCPatcher. 24 fills a regular arena match. SCPatcher also turns Kick Excessive Bots off. | 6 | Matchmaking |
| Kick Excessive Bots | `cvars.sv_kickExcessiveBotsEnabled` line, edited in place. Game comment: kick when there are more bots than players. The battle server then removes the bot with a `DMG_KICK_EXCESSIVE_BOT` damage. Probably makes room when players join a running match (undocumented). | 1 | Battles |
| Coop vs AI Team Size (experimental) | `botsCount = 2N-1, botsInTeam1 = N-1` added to the three `coop_vs_ai = 1` special entries (both coop templates and the newbie coop queue), edited in place. Sized for one player: N-1 allies, N enemies. Capped at 8 ships in the newbie coop queue, 12 otherwise. Not yet confirmed that the matchmaker uses them; check `botsCount`/`botsInTeam1` in the battle log. | 0 (off) | Coop matchmaking |
| Buy Everything | `cvars.store_allowAllItemsForBying` line, edited in place. Buying obsolete or dev items can break an account. | 0 | Store |
| DLC Packs In Shop (experimental) | Adds `flags="15"` to the 14 `<Item … defName="Bundle_Arc_DLC_*">` lines in `cloud/data/store.xml`, which sets the store's IS_SHOW flag (plus the other three item flags, whose bit values are undocumented). The client then shows these hidden bundles in the shop. Off removes the attribute again. | off | Shop, from the next start |
| Destroyer Pack Items In Shop (experimental) | Removes `cantBeBought="1"` from the 8 Mk1 `<Item>` lines of the former destroyer packs' unique weapons and modules in `cloud/data/store.xml` (see below). They keep their existing GS prices (110–180) and still only fit their destroyer. Off puts the flag back, and the file is byte-identical to vanilla again. | off | Shop, from the next start |
| DLC Portraits / Titles / Taunts / Decals Price (GS, experimental) | One shop entry per category for the cosmetic DLC-only content (71 portraits, 15 ship titles, 10 taunts, 2 Pikabu decals), carried by an existing bundle in `cloud/data/store.xml` (see below). 0 restores the original lines. | 0 (off) | Shop, from the next start |
| Starter Pack: Credits / Galactic Standards / Token Credits | `credits` / `goldCredits` / `tokenCredits` attributes in `cloud/data/starter_pack.xml`, on top of the starting values above | 0 / 0 / 0 | Accounts created afterwards |
| Battle Pass / Battle Pass Start / Battle Pass End | In place: `local battlePassStart`, `local battlePassEndTable` (one stage) at the top of the file, and `cvars.store_battlePassUnlockLvlBtnStart`. Off writes back the original, expired 2023 values. Dates are taken as UTC in any common format (AMP itself stores them as `10/13/2026 00:00:00`) and converted to the game's `YYYY-MM-DDThh:mm:ssZ`. An unparseable date leaves the battle pass off and logs why. In 1.5.0/1.5.1 only the ISO format was accepted, so dates entered through AMP were always rejected. | off | Shard and QuestServer, from the next start |

**Bots and game modes.** The arena settings only affect PvP (Arena) matches. Coop vs AI sizes the enemy team to the number of players, so solo it is one bot, unless Coop vs AI Team Size is set. The special/brawl modes need 2–5 real players to start, and only "one against all" adds bots.

Starting values and the starter pack are only granted when an account is created. Changing them does not touch existing balances.

Player progress in the battle pass is stored per start time. Changing **Battle Pass Start** begins a new battle pass and existing progress is lost; DLC-bought premium is kept. `battle_pass.xml` defines one stage with 41 rewards, so the template runs exactly that stage.

**Former DLC packs.** Steam DLC purchases can't reach a self-hosted server: ownership is checked through Steam's publisher web API, which needs the publisher's key. In the Steam client, the ship window's DLC button opens the Steam store. The 14 packs still exist as GS-priced bundles (`Bundle_Arc_DLC_*` in `data/gamedata/def/objects/bundles.lua`: ships, titles, account auras, decals, open-world paints), but they're marked `hiddenInShop`. That flag is only checked by the client, and the client still shows a bundle whose store item carries `IS_SHOW`. **DLC Packs In Shop** sets that flag. Players can earn GS for the packs without paying through Starting Galactic Standards and Bonus GS Per Win / Per Loss. Not yet confirmed in the client.

**Former destroyer packs.** The destroyer packs (Procyon, Vigilant, Tyrant, Sirius, Emperor, Albireo) came with the ship and its unique weapon and modules. The destroyers are craftable ships here, built in the shipyard from parts that every destroyer (and, for the generic parts, every crafted ship) shares, so they can't be sold for GS on their own without also selling parts for all the others. **Destroyer Pack Items In Shop** therefore only unblocks the packs' unique items, Mk1, at the GS prices `store.xml` already has: `Weapon_DestrResonatingLaser_T5` (Vigilant's Vacuum-resonant laser), `Weapon_KineticFloatingCrit_T5`, `Module_IsotopeHarvester_T5`, `Module_ShieldHealOnEnemyDeath_T5` (Tyrant's Thermoactive weapon, Isotope Harvester, Warp stream detector), `Weapon_OmegaB_T5`, `Module_EmperorsRobe_T5` (Emperor's Omega-B launcher, Gyges emitter), `Weapon_PlasmaWebLaser_T5`, `Module_DroneField_T5` (Albireo's Fort-4 launcher, Autonomous barrier torpedo). The names were matched through each item's `required_ship` and the pack descriptions. Procyon's and Sirius's packs only had an ordinary Coilgun, and the packs' other modules are regular shop items. Not yet confirmed in the client.

**DLC extras.** Apart from the 14 DLC packs, 83 more bundles in `bundles_dlc.lua` hold content that nothing else in the server data hands out: 71 pilot portraits, the ship titles and taunts of the `*_Stuff` packs (Dragonfly, Wasp, Cyning, Mjolnir, Kraken, Peregrine, Jaguar, Octopus, Caltrop, Saw One, Ronin), more taunts, and the Pikabu decals. They are hidden in the client, `cantBeBought` and priced 99999999 GS. Instead of 83 shop entries, each category becomes one: the client draws a shop entry from its own data, so an existing bundle carries the category and `store.xml` lists the whole category as its contents.

| Setting | Carrier (what the shop shows) | Delivers |
|---|---|---|
| DLC Portraits Price | `Bundle_Avatar_117` (one portrait) | all 71 portraits |
| DLC Titles Price | `Bundle_Mjolnir_Stuff` (the Mjolnir title) | all 15 ship titles |
| DLC Taunts Price | `Bundle_Confessor_Stuff` (the Confessor taunt) | all 10 taunts |
| DLC Decals Price | `Bundle_Pikabu` | the 2 Pikabu decals |

A price above 0 sets the IS_SHOW flag (`flags="15"`, as for the DLC packs), removes `cantBeBought`, sets the price and rewrites the carrier's `<Bundle>` contents from the other DLC bundles. Every start first puts all 83 bundles back to vanilla, so 0 restores the file byte for byte (this also cleans up 2.6.0's per-bundle prices). Not included: bundles with boosts, premium time or random rewards, and `Bundle_Avatar_221`, whose portrait is obtainable elsewhere. All DLC ships were already buyable for GS. Untested in the client: that the carrier appears at the new price, and that buying it delivers the whole category (the server takes bundle contents from `store.xml`, the client only shows the carrier's own name, icon and preview). Adds about 10 seconds to each start.

Bots, Kick Excessive Bots and Buy Everything come from SCPatcher (Star Conflict Mod Manager) by Furebel, limited to the parts that are plain config values. Its other mods are not included because they edit game data (loot tables, quests, ore, iridium) and need `prepare_shared` to rebuild the server XML: arena queue timing, arena search fix, canister loot, enemy rarity, iridium multiplier, ore drops and quest GS rewards.

**Where the files are.** Every setting's description in AMP ends with `Edits: …`, the path as AMP's file manager shows it. All of them are in `instance/server/cloud/data/`:

| File | Settings |
|---|---|
| `default_platform_config.lua` | AMP block at the end: Accounts, Economy, Battles, Max Players. In place: Bots, Kick Excessive Bots, Buy Everything, Battle Pass |
| `starter_pack.xml` | Starter Pack |
| `cloud.xml` | Public Address (node address) |
| `common_local.cfg` | External MongoDB (AMP block only) |
| `loadbalancer_local.cfg`, `shard_local.cfg`, `chat_server_local.cfg`, `dediserver_local.cfg` | Ports, Public Address |

The scripts that write them are regenerated in `star-conflict/` on every start: `amp-settings.sh`, `mongodb.sh` (also on every update) and `hub-wrapper.sh`. Before 2.0 there was a `mongo-start.sh`; it isn't used any more.

## Admin and moderator roles

**Admin Account IDs** and **Moderator Account IDs** (Administration settings) give accounts a role, by account ID (uid): the number in the console's `[AMP] Player joined: name (uid N)` lines. An account exists after its first login.

| Role | Can |
|---|---|
| Admin | Everything the server knows: the Punish menu (mute/ban), GM tools (`gmView`: player info, renaming), destructive edits (`adminEdit`: resetting stats and accounts), every admin task, logging in during maintenance. |
| Moderator | Mute and ban, temporary and permanent, and lifting them; logging in during maintenance; observing games. No GM tools. |

In game, a moderator or admin right-clicks a player in chat and gets **Punish** next to Report: the report window with Mute and Ban buttons, a duration and the rule points.

How it works, worked out against a running server here: an account's role is its `admRole` field (`player` by default). What a role may do is one document in a separate database, `web_adm.admRoles`: `{ roles: { <role>: { <privilege>: true, ... } } }`. Privileges are the admin task names (`mute`, `ban`, `playerInfo`, ... 61 of them) plus general ones; tested: `gmView` covers `playerInfo` and `changeNickName`, `adminEdit` covers `resetStats`, `mute` only `mute`. Shard reads the roles once at startup, so `hub-wrapper.sh` runs `mongodb.sh roles` after MongoDB is up and before Hub: it writes the `admin` and `moderator` roles and sets `admRole` on the listed accounts. Accounts that are no longer listed become `player` again; other roles (for example a `gm` set by hand) are left alone. An account has only one role: open-world access ("Free Space") is the role `alpha-tester-dlc`, so listing such an account as Admin or Moderator replaces it. Changes therefore take effect on the next start. With Infinite Premium on, admins and moderators get it too (`auth_infinitePremiumForRoles`).

Not tested yet: the Punish menu in the client (the client checks its own privilege numbers, which aren't readable; mute and ban were verified server-side by running the admin tasks as admin and moderator accounts).

## Remote console

Every service has an SSH server built in. With a **Remote Console Password** (Administration settings), the template switches it on for Hub and Shard, on their own internal ports (Hub Console Port, Shard Console Port), with an RSA host key it creates once in `star-conflict/ssh_host_rsa_key`. AMP's console then becomes the game's console: type a command to run it on Hub, or put `shard ` in front to run it on Shard. Useful on Hub: `help <command>`, `cmdList`, `cvarList`, `cloud_status` (every service with load and pid), `hub_status`, `uptime`, `enterMaintenanceMode [minutes]` / `leaveMaintenanceMode`, `cron_addCmd <dow> <hours> <minutes> <cmd>`, `set <cvar> <value>`; on Shard: `shard sm_dumpClients` (connected players), `shard store_dump`, `shard cvarList`. Empty password = off (vanilla). All commands with descriptions: [CONSOLE.md](CONSOLE.md). The console also has the template's own account commands, which work without a password: `player <uid or name>`, `give <uid> gs|credits|tokens <amount>`, `give <uid> premium <days>`, `give <uid> item|ship <shop id>`, `give <uid> title <id>` and `find <text>` (see [CONSOLE.md](CONSOLE.md#account-commands); every item and title ID is listed in [ITEMS.md](ITEMS.md)).

The SSH server is old and fragile, which the console works around: it only speaks `diffie-hellman-group1-sha1`, `ssh-rsa`, `aes128-cbc` and `hmac-sha1` (the client re-enables them), handles one session at a time and wedges for good if a client gives up before it answers (so no timeouts, one command at a time, Shard only once it has finished starting), its key exchange fails now and then, sometimes it ends a session before running the command, and its reply over SSH is often empty. Every command it runs is logged with its output, though, so the console checks the service's log: a command is sent again until it shows up there (up to 5 times), Hub's output appears through the Hub log stream, and Shard's is read from its log and printed with `[Shard]`. Tested here: 30 commands, each run exactly once, every reply complete; a command takes a few seconds.

Needs `ssh` and `ssh-keygen` in the container (`Meta.ExtraContainerPackages=["openssh-client"]`; an existing container may need recreating to get it). Keep the console ports internal: anything that opens a connection and leaves (a port scanner) can block the console until the next start. The password is stored in `hub_local.cfg`/`shard_local.cfg`, and failed logins are logged with the password that was tried. No quotes or backslashes in the password.

## Player tracking

Logins and logouts only appear in the shard's log, not Hub's. `hub-wrapper.sh` follows `…/cloud/Hub/logs/*_shard_YYYY_MM_DD.log` as well and turns it into console lines that AMP's user tracking reads:

```
[AMP] Player joined: himori (uid 1)      <- "Granting authorization ( uid 1 … nick: 'himori' )" + "logged in as 1"
[AMP] Players online: 1
[AMP] Player left: himori (uid 1)        <- "SessionManager::OnDisconnect, uid: 1 …"
[AMP] Stats: accounts 7, cpu 14.2%, memory 2310 MB   <- every minute, see Metrics below
```

`Console.UserJoinRegex` / `UserLeaveRegex` match the first and third lines, so AMP's user list, user count and history work. Connections that never log in (uid 0, such as port checks) are ignored. AMP clears its user list when the application stops.

## Metrics

AMP's built-in **CPU Usage** and **Memory Usage** tiles measure one process: the one AMP started, `hub-wrapper.sh`, which is idle, so they show about 0. A template can point them at one other process (`App.MonitorChildProcessName`), but never at a sum, and the load here is spread over about 17 services, the battle servers and MongoDB. The memory total they show is the host's RAM.

So the whole server is reported as custom metrics instead. Once a minute `hub-wrapper.sh` prints `[AMP] Stats: accounts N, cpu X%, memory Y MB`, and `Console.MetricsRegex` turns it into three tiles:

| Tile | Source |
|---|---|
| **Registered Accounts** | `db.accounts.count()` in `cosmosim_<user>`, queried every 5 minutes (the last value is repeated in between). AMP shows custom metrics as "value / highest so far". |
| **Total CPU Usage** | The container's cgroup (`cpu.stat`, `usage_usec`): CPU time of every process of the instance over the last minute, in % of all cores (0–100, like AMP's own tile). |
| **Total Memory Usage** | The container's cgroup: `memory.current` minus `inactive_file`, the same figure `docker stats` shows. Reclaimable file cache (MongoDB's data files, the game's `.pak` files) doesn't count. |

**Scales.** AMP shows custom metrics as "value / highest value seen", because a template can't give them a maximum. So the very first stats line reports CPU at 100% and memory at the limit (the container's `memory.max`, which is AMP's memory limit for the instance, or the host's RAM if there is none), and the real reading follows in the same instant. From then on Total CPU Usage reads against 100 and Total Memory Usage against the limit. That one priming line shows up once per start in the console and in AMP's metric history.

Every process of the instance runs in its container, so the cgroup totals cover Hub, all services, battle servers, MongoDB and Wine's helpers. Without cgroup v2 files, the console says so once and the figures are summed over the container's processes instead (`/proc/*/stat`, `ps` RSS). The first line comes about a minute after the start.

Tested outside AMP in a separate cgroup: one busy core of 8 read as 12.0% (the busy process's own CPU time gave the same 0.96 cores), and a process holding 300 MB raised Total Memory Usage from 2 to 315 MB and back. Not yet confirmed: that AMP turns the three named groups of one regex into three tiles, since the official templates only ever use one.

## External MongoDB

Set **MongoDB** to *External* and **External MongoDB Hosts** to the server, under the Database settings:

- **Several servers:** two instances pointed at the same external MongoDB share their data (see [Several servers on one host](#several-servers-on-one-host)).
- **Format** (the game's `db_mongoHosts`): `host:port`, or a replica set as `host1:port,host2:port,host3:port`. The game asks the servers which one is primary (`ismaster`). Options may follow after `|`, for example `|connpool=5`. The port defaults to 27017.
- **How it's applied:** every service and every battle server executes `common.cfg`, and with it `common_local.cfg`, before connecting to the database. The template therefore writes `set db_mongoHosts "…"` there, in a block between `// >>> AMP managed` and `// <<< AMP managed`. Everything else in that file stays yours. Switching back to *Bundled* removes the block.
- **On start:** the bundled MongoDB is not started. "Wait For External MongoDB" waits up to 60 seconds for one of the hosts to accept connections, otherwise the start is aborted. "Prepare MongoDB" still restarts the Wine server, but it only stops the *bundled* `mongod` (by its pid file), never whatever else answers on port 27017, and doesn't migrate old data.
- **No authentication.** The game's driver has no login support (no SCRAM or MONGODB-CR in the binaries), so the external MongoDB must accept unauthenticated connections from this host. Keep it on a private network or behind a firewall.
- **Version: MongoDB 3.4.** Same reasons as for the [bundled MongoDB](#bundled-mongodb): 3.6+ breaks the game's aggregations, 4.2+ its index creation, 5.1+ everything. The [AMPStarConflictDB](https://git.himori.dev/Himori/AMPStarConflictDB) template runs one in its own AMP instance.
- **One MongoDB per instance.** See [Several servers on one host](#several-servers-on-one-host).
- **Data is not migrated.** Database names get the Wine user as a suffix (`cosmosim_amp`, `stats_cosmosim_amp`, `achievements_amp`, `logs_amp`, `games_amp`, `delivered_amp`). Copy them as described below before switching.
- The "Registered accounts" metric queries the first host of the list (reads from a secondary are allowed).

### Moving the bundled database to an external MongoDB

The bundled MongoDB and the external one are both 3.4, so the data moves over as it is:

- **With AMPStarConflictDB:** stop both instances, then copy the contents of this instance's `database/data/` into the database instance's `data/` (in AMP's file manager or on the host) before its first start. Or use its **Import From** setting with `127.0.0.1:<this instance's MongoDB Port>` while this instance runs with players kept off; see its README.
- **Any other MongoDB 3.4:** dump with `database/mongodb/bin/mongodump` while this instance runs (players off) and restore with `mongorestore` 3.4 into the external server.

Then set **MongoDB** to *External* and **External MongoDB Hosts** to the server, and start the instance. `database/data/` stays as it is: switching back to *Bundled* uses it again.

## Custom cvars

Files the template owns are regenerated on every start: `loadbalancer_local.cfg`, `shard_local.cfg`, `chat_server_local.cfg`, `dediserver_local.cfg` and `game_reward_dispencer_local.cfg`. It also rewrites the AMP block and three cvar lines in `default_platform_config.lua`, the currency attributes in `starter_pack.xml` and the node address in `cloud.xml`.

Your own `.cfg` settings go in `server/cloud/data/common_local.cfg`. Every service and battle server execs it. The template only manages the `// >>> AMP managed` … `// <<< AMP managed` block in it, used for an external MongoDB, and never touches the rest. They only take effect for cvars that `default_platform_config.lua` doesn't set. For anything that file defines, change the file itself, outside the AMP block, which is rewritten on every start.

## Status / unverified assumptions

Verified against build 1.99.0.167293:
- The process layout and ports listed above.
- The cvar names (taken from the binaries).
- Each service looks for `*_local.cfg` overrides. Its logs show `failed to read file '<svc>_local.cfg'` when the file is missing.
- The stack runs under Wine/Proton: Hub, all services and the game's own mongod 2.6 run when launched by the game client.

Not yet tested end-to-end in AMP:
- That default accounts have `admRole` `player`. The string sits next to `admRole` in `Shard.exe` (`user is not a player`), but no account record has been inspected. If premium does not appear, check an account document's `admRole` in MongoDB.
- `premium_on_start` is in **hours**, confirmed: the game multiplies it by 3600 into a 32-bit number. 1.9.1/1.9.2 wrote days × 86400, so 3650 days became 315360000 "hours", which overflowed to about 16488 days (a new account showed 16490d). From 1.9.3 it is days × 24, capped at 20000 days.
- That `tokenCredits` is the daily-capped currency shown in game. The name and the `sv_tokenCreditsDailyLimit` cvar suggest it.
- That ConfigServer applies `default_platform_config.lua` cvars to MatchMaker, battle servers and the store the same way the game normally does. This is the mechanism SCPatcher relies on, but it has not been tested here.
- The load balancer hands clients the host of the `cloud.xml` node address plus each service's port. With the default `127.0.0.1`, remote clients are told to connect to themselves (client log: `received shard address: (127.0.0.1:…)`, then error 10061). `net_overrideShardAddr`, `net_overrideChatServerAddr` and `net_overrideDedicatedServerAddr` are still written but had no visible effect, so the template also writes the node address: Public Address, or the IP binding when that is empty. Hub only starts if that address belongs to one of its own interfaces (`NetAddress::IsOfMyIface`). This works here because the AMP container shares the host network. It has not been tried with a bridged container.
- That battle servers pick ports from `net_gamePort` upward, and whether they use TCP or UDP. Both are forwarded.
- The XP/credit multipliers, price multipliers, AFK time and resurrection cost were written from the cvar descriptions in `default_platform_config.lua` and haven't been observed in a match yet. Starting credits/GS and Infinite Premium use the values the server demonstrably reads: a new account got exactly `gold_on_start` 300 and `credits_on_start` 0.
- The battle pass dates are written the way the config file's own comments describe, but no battle pass has been seen running in the client yet. Player tracking was tested against log lines copied from this server's shard log, not yet against live AMP.
- Stopping works: Hub exits on `SIGTERM` and its services shut themselves down when Hub disconnects. The order around it is in [Starting and stopping](#starting-and-stopping); it was tested with stand-ins, not yet with the real services, so how long they take to exit is not known yet. A leftover socket on the chat port once stopped ChatServer from starting, which the Wine server restart on every start prevents.
- Hub and its services against MongoDB 3.4 in a real game session. The two old-style operations the driver needs were tested against 3.4.24, but a full session hasn't been played on it yet.
- Which `.pak` archives the server really needs. Services load scripts such as `scripts/ai/adventure_server.lua` and `scripts/scripts_server/main.lua` from inside them (their file index is obfuscated), so the tarball ships all of `data/`. A slimmer tarball without the paks made ScriptsServer and AdventureServer fail.
- The instance is marked Ready when Hub logs that the load balancer registered (`registered itself as 810000000000001`). If that service ID changes, AMP will stay on "Starting" while the server works fine. Adjust `Console.AppReadyRegex` if that happens. Hub ignores `-logFileName` for its location, but it honours `-outputDir`. The template starts it with `-outputDir server-logs/`, and Hub passes that on to every service and battle server. Logs are then in `server-logs/logs/` (`hub_YYYY_MM_DD.log`, one file per service, battle logs in a dated subfolder), crash dumps in `server-logs/cloud_exceptions/`. The wrapper follows the Hub and shard logs there.

## Rebuilding the server tarball

From a Steam install of Star Conflict:

```sh
cd "$HOME/.local/share/Steam/steamapps/common/star conflict"
tar --sort=name --owner=0 --group=0 --numeric-owner \
  --exclude='./cloud/Exceptions' --exclude='./cloud/mongodb/data/*' --exclude='*_local.cfg' \
  --exclude='./cloud/mongodb/'{bsondump,mongoexport,mongofiles,mongoimport,mongooplog,mongoperf,mongos,mongostat,mongotop}.exe \
  -czf star-conflict-server.tar.gz ./cloud ./data ./sys_data
```

`cloud/mongodb/mongo-r2.6.12.zip` is MongoDB's source code. Keep it in the archive: MongoDB 2.6 is AGPL-3.0, and redistributing the binary requires providing the source.
