# AMP Generic Module template: Star Conflict

Hosts the self-contained Star Conflict server stack in CubeCoders AMP.

The server files come from [`star-conflict-server.tar.gz`](https://nx.himori.dev/public.php/dav/files/wyxJddcXopyeJDW). It contains `cloud/` (with the bundled MongoDB 2.6.12 in `cloud/mongodb/`), all of `data/` (including every `.pak` archive) and `sys_data/`, about 12 GB. The archive is extracted to `star-conflict/server/`.

- `Hub.exe` is the supervisor. It reads `cloud/data/cloud.xml` and spawns about 17 services: LoadBalancer, Shard, MatchMaker, Chat, Universe, ZoneCoordinator and others.
- `ZoneCoordinator` spawns one `DedicatedServer.exe` per battle.
- Everything persists to MongoDB on `127.0.0.1:27017`. The services connect as soon as they start, so **MongoDB must be up before Hub starts**. The pre-start stages enforce this: they start mongod and wait until port 27017 accepts connections. If it doesn't come up within 90 seconds, the start is aborted.

MongoDB data lives outside the extracted files, in `star-conflict/mongo-data/`, so it survives updates.

All binaries are Windows-only (Hub and services are 32-bit, mongod is 64-bit). On Linux they run under Wine inside AMP's `cubecoders/ampbase:wine-stable` container.

## Files

| File | Purpose |
|---|---|
| `star-conflict.kvp` | Main template. On Linux it runs `star-conflict/hub-wrapper.sh` (written by a start stage), which starts `wine Hub.exe` from `server/cloud`, streams Hub's log into the AMP console and forwards stop signals to Hub. |
| `star-conflictupdates.json` | Downloads and extracts the server tarball, creates `mongo-data/`, initialises the Wine prefix. |
| `star-conflictstart.json` | Pre-start: writes the `*_local.cfg` files, the starter pack, the AMP block in `default_platform_config.lua` (via `amp-settings.sh` / `.ps1`) and the `cloud.xml` node address. Then it restarts the Wine server and MongoDB and waits for port 27017 before Hub launches. |
| `star-conflictports.json` | Ports exposed to players. |
| `star-conflictconfig.json` | AMP settings: public address, accounts (starting credits/GS, premium), economy (XP/credit multipliers, prices, bonuses), battles, matchmaking. See [Settings](#settings). |
| `star-conflictmetaconfig.json` | Empty. Config is written by start stages. |

## Ports

| Port | Proto | Service | cvar |
|---|---|---|---|
| 3801 | TCP | Load balancer (clients connect here) | `lb_listenTo` |
| 3802 | TCP | Shard | `shard_listenTo` |
| 3815 | TCP | Chat | `chat_server_port` |
| 35000–35049 | TCP+UDP | Battle servers (`DedicatedServer.exe`) | `net_gamePort` |

Internal only, never forward these: Hub `3850` and MongoDB `27017` (bound to `127.0.0.1`, no auth).

**Keep the ports out of Linux's ephemeral range** (`/proc/sys/net/ipv4/ip_local_port_range`, usually 32768–60999). If ADS auto-assigns ports in that range (for example 60059–60061 and 60008+), an outgoing connection can grab one first, and that service then fails with `Failed to listen on port …`. Set them back to the defaults above in the instance's port settings.

## Installing on a NixOS host

The Linux side needs a 64-bit Wine with WoW64/32-bit support at `/usr/bin/wine`. That doesn't exist on NixOS, so **run the instance in a container**. In ADS, when creating the instance, enable *Use Containers* (AMP needs Docker on the host).

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
3. Create an instance, choose *Generic*, then *Star Conflict*, with containers enabled.
4. Update the instance. This downloads the server tarball (about 12 GB). No Steam login is needed.
5. Optionally set **Public Address**. If it's empty, the server advertises the application IP binding, or the instance IP binding (`Webserver.IPBinding`) when the application binding is `0.0.0.0`. Whatever is used must be an IP of one of the host's own network interfaces, such as its NetBird, Tailscale or LAN address. A DNS name or a NAT-forwarded public IP stops Hub from starting.

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
| Bots In Arena Matches | `cvars.mm_botsCount` line, plus `botsCount` in both arena queue templates (`special = { arcade = 1 }` becomes `special = { arcade = 1, botsCount = N }`), edited in place like SCPatcher. 23 fills a 24-player arena. Needs Kick Excessive Bots **off** to have any effect with few players. | 6 | Matchmaking |
| Kick Excessive Bots | `cvars.sv_kickExcessiveBotsEnabled` line, edited in place. When on, battles drop bots until there are no more bots than players, so a solo player always gets a 1v1. | 1 | Battles |
| Buy Everything | `cvars.store_allowAllItemsForBying` line, edited in place. Buying obsolete or dev items can break an account. | 0 | Store |
| Starter Pack: Credits / Galactic Standards / Token Credits | `credits` / `goldCredits` / `tokenCredits` attributes in `cloud/data/starter_pack.xml`, on top of the starting values above | 0 / 0 / 0 | Accounts created afterwards |
| Battle Pass / Battle Pass Start / Battle Pass End | In place: `local battlePassStart`, `local battlePassEndTable` (one stage) at the top of the file, and `cvars.store_battlePassUnlockLvlBtnStart`. Off writes back the original, expired 2023 values. Dates are taken as UTC in any common format (AMP itself stores them as `10/13/2026 00:00:00`) and converted to the game's `YYYY-MM-DDThh:mm:ssZ`. An unparseable date leaves the battle pass off and logs why. In 1.5.0/1.5.1 only the ISO format was accepted, so dates entered through AMP were always rejected. | off | Shard and QuestServer, from the next start |

Starting values and the starter pack are only granted when an account is created. Changing them does not touch existing balances.

Player progress in the battle pass is stored per start time. Changing **Battle Pass Start** begins a new battle pass and existing progress is lost; DLC-bought premium is kept. `battle_pass.xml` defines one stage with 41 rewards, so the template runs exactly that stage.

Bots, Kick Excessive Bots and Buy Everything come from SCPatcher (Star Conflict Mod Manager) by Furebel, limited to the parts that are plain config values. Its other mods are not included because they edit game data (loot tables, quests, ore, iridium) and need `prepare_shared` to rebuild the server XML: arena queue timing, arena search fix, canister loot, enemy rarity, iridium multiplier, ore drops and quest GS rewards.

**Where the files are.** Every setting's description in AMP ends with `Edits: …`, the path as AMP's file manager shows it. All of them are in `star-conflict/server/cloud/data/`:

| File | Settings |
|---|---|
| `default_platform_config.lua` | AMP block at the end: Accounts, Economy, Battles, Max Players. In place: Bots, Kick Excessive Bots, Buy Everything, Battle Pass |
| `starter_pack.xml` | Starter Pack |
| `cloud.xml` | Public Address (node address) |
| `loadbalancer_local.cfg`, `shard_local.cfg`, `chat_server_local.cfg`, `dediserver_local.cfg` | Ports, Public Address |

The scripts that write them are regenerated in `star-conflict/` on every start: `amp-settings.sh` (or `.ps1` on Windows), `mongo-start.sh` and `hub-wrapper.sh`.

## Player tracking

Logins and logouts only appear in the shard's log, not Hub's. `hub-wrapper.sh` follows `…/cloud/Hub/logs/*_shard_YYYY_MM_DD.log` as well and turns it into console lines that AMP's user tracking reads:

```
[AMP] Player joined: himori (uid 1)      <- "Granting authorization ( uid 1 … nick: 'himori' )" + "logged in as 1"
[AMP] Players online: 1
[AMP] Player left: himori (uid 1)        <- "SessionManager::OnDisconnect, uid: 1 …"
[AMP] Registered accounts: 7             <- db.accounts.count() in cosmosim_<user>, every 5 minutes
```

`Console.UserJoinRegex` / `UserLeaveRegex` match the first and third lines, so AMP's user list, user count and history work. `Console.MetricsRegex` turns the last line into a **Registered Accounts** metric. Connections that never log in (uid 0, such as port checks) are ignored. AMP clears its user list when the application stops.

## Custom cvars

Files the template owns are regenerated on every start: `loadbalancer_local.cfg`, `shard_local.cfg`, `chat_server_local.cfg`, `dediserver_local.cfg` and `game_reward_dispencer_local.cfg`. It also rewrites the AMP block and three cvar lines in `default_platform_config.lua`, the currency attributes in `starter_pack.xml` and the node address in `cloud.xml`.

Your own `.cfg` settings go in `server/cloud/data/common_local.cfg`. Every service execs it, and the template never touches it. They only take effect for cvars that `default_platform_config.lua` doesn't set. For anything that file defines, change the file itself, outside the AMP block, which is rewritten on every start.

## Status / unverified assumptions

Verified against build 1.99.0.167293:
- The process layout and ports listed above.
- The cvar names (taken from the binaries).
- Each service looks for `*_local.cfg` overrides. Its logs show `failed to read file '<svc>_local.cfg'` when the file is missing.
- The stack runs under Wine/Proton: Hub, all services and the bundled mongod run when launched by the game client.

Not yet tested end-to-end in AMP:
- That default accounts have `admRole` `player`. The string sits next to `admRole` in `Shard.exe` (`user is not a player`), but no account record has been inspected. If premium does not appear, check an account document's `admRole` in MongoDB.
- That `tokenCredits` is the daily-capped currency shown in game. The name and the `sv_tokenCreditsDailyLimit` cvar suggest it.
- That ConfigServer applies `default_platform_config.lua` cvars to MatchMaker, battle servers and the store the same way the game normally does. This is the mechanism SCPatcher relies on, but it has not been tested here.
- The load balancer hands clients the host of the `cloud.xml` node address plus each service's port. With the default `127.0.0.1`, remote clients are told to connect to themselves (client log: `received shard address: (127.0.0.1:…)`, then error 10061). `net_overrideShardAddr`, `net_overrideChatServerAddr` and `net_overrideDedicatedServerAddr` are still written but had no visible effect, so the template also writes the node address: Public Address, or the IP binding when that is empty. Hub only starts if that address belongs to one of its own interfaces (`NetAddress::IsOfMyIface`). This works here because the AMP container shares the host network. It has not been tried with a bridged container.
- That battle servers pick ports from `net_gamePort` upward, and whether they use TCP or UDP. Both are forwarded.
- The XP/credit multipliers, price multipliers, AFK time and resurrection cost were written from the cvar descriptions in `default_platform_config.lua` and haven't been observed in a match yet. Starting credits/GS and Infinite Premium use the values the server demonstrably reads: a new account got exactly `gold_on_start` 300 and `credits_on_start` 0.
- The battle pass dates are written the way the config file's own comments describe, but no battle pass has been seen running in the client yet. Player tracking was tested against log lines copied from this server's shard log, not yet against live AMP.
- Stopping works: Hub exits on `SIGTERM` and its services shut themselves down when Hub disconnects. On every start, `mongo-start.sh prepare` (a foreground stage) cleanly stops the previous MongoDB, waits for the process to exit (not just its port to close) and restarts the Wine server (`wineserver -k`). Then `mongo-start.sh run` (a background stage) starts MongoDB fresh. These must stay separate stages: in 1.3.3 the restart ran in the background stage, so "Wait For MongoDB" saw the old MongoDB, started Hub, and the Wine server restart then killed Hub. If a clean shutdown was ever cut off after its journal cleanup, leaving a non-empty `mongod.lock` with no journal files, the script clears that stale lock. With journal files present, it leaves the lock alone so MongoDB recovers from the journal. This clears services and sockets left over from the last run. A leftover socket on the chat port once stopped ChatServer from starting.
- Which `.pak` archives the server really needs. Services load scripts such as `scripts/ai/adventure_server.lua` and `scripts/scripts_server/main.lua` from inside them (their file index is obfuscated), so the tarball ships all of `data/`. A slimmer tarball without the paks made ScriptsServer and AdventureServer fail.
- The instance is marked Ready when Hub logs that the load balancer registered (`registered itself as 810000000000001`). If that service ID changes, AMP will stay on "Starting" while the server works fine. Adjust `Console.AppReadyRegex` if that happens. Hub ignores `-logFileName` for its location and always writes `hub_YYYY_MM_DD.log` under `.wine/drive_c/users/<user>/AppData/Local/Targem/StarConflict/cloud/Hub/logs/`. The wrapper follows that file. The other services log to the same folder.

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
