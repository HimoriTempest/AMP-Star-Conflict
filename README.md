# AMP Generic Module template: Star Conflict

Hosts the self-contained Star Conflict server stack in CubeCoders AMP.

The server files come from [`star-conflict-server.tar.gz`](https://nx.himori.dev/public.php/dav/files/wyxJddcXopyeJDW). It contains `cloud/` (with the bundled MongoDB 2.6.12 in `cloud/mongodb/`), `data/gamedata/`, `data/levels/` (map/nav/scene data, no textures) and `sys_data/`. The archive is extracted to `star-conflict/server/`.

- `Hub.exe` is the supervisor. It reads `cloud/data/cloud.xml` and spawns about 17 services: LoadBalancer, Shard, MatchMaker, Chat, Universe, ZoneCoordinator and others.
- `ZoneCoordinator` spawns one `DedicatedServer.exe` per battle.
- Everything persists to MongoDB on `127.0.0.1:27017`. The services connect as soon as they start, so **MongoDB must be up before Hub starts**. The pre-start stages enforce this: they start mongod and wait until port 27017 accepts connections. If it doesn't come up within 90 seconds, the start is aborted.

MongoDB data lives outside the extracted files, in `star-conflict/mongo-data/`, so it survives updates.

All binaries are Windows-only (Hub and services are 32-bit, mongod is 64-bit). On Linux they run under Wine inside AMP's `cubecoders/ampbase:wine-stable` container.

## Files

| File | Purpose |
|---|---|
| `star-conflict.kvp` | Main template. On Linux, a small bash wrapper runs `wine Hub.exe -logFileName star-conflict/logs/hub.log` from `server/cloud`. It streams that log into the AMP console and forwards stop signals to Hub. |
| `star-conflictupdates.json` | Downloads and extracts the server tarball, creates `mongo-data/`, initialises the Wine prefix. |
| `star-conflictstart.json` | Pre-start: writes `cloud/data/*_local.cfg` from AMP settings, starts mongod in the background, waits for port 27017 before Hub launches. |
| `star-conflictports.json` | Ports exposed to players. |
| `star-conflictconfig.json` | AMP settings: public address, shard player limit, infinite premium, starter pack (credits / GS / token credits), bonus GS per battle, arena bots, kick excessive bots, buy everything. |
| `star-conflictmetaconfig.json` | Empty. Config is written by start stages. |

## Ports

| Port | Proto | Service | cvar |
|---|---|---|---|
| 3801 | TCP | Load balancer (clients connect here) | `lb_listenTo` |
| 3802 | TCP | Shard | `shard_listenTo` |
| 3815 | TCP | Chat | `chat_server_port` |
| 35000–35049 | TCP+UDP | Battle servers (`DedicatedServer.exe`) | `net_gamePort` |

Internal only, never forward these: Hub `3850` and MongoDB `27017` (bound to `127.0.0.1`, no auth).

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

1. Push this repo to GitHub (`HimoriTempest/AMP-Star-Conflict`, per `manifest.json`).
2. In ADS go to **Configuration → Instance Deployment → Configuration Repositories** and add `HimoriTempest/AMP-Star-Conflict:main`. Then click **Fetch latest**.
3. Create an instance, choose *Generic*, then *Star Conflict*, with containers enabled.
4. Update the instance. This downloads the server tarball (about 250 MB). No Steam login is needed.
5. Set **Public Address** to the host name or IP players will use, then start.

## Pointing clients at the server

The client normally starts its own local stack (`-launchLocalServer`). To use a remote server instead, it must skip that and set `net_masterServerAddress` to `<public-address>:3801`. Do this through a `user_config.cfg` (the client `exec`s that file) or the Steam launch options. This step is **not verified yet**. See below.

## Account, economy and matchmaking settings

Every option is reversible. The original value is the default, and setting it back restores the server's original behaviour on the next start.

| Setting | How it works | Applies to |
|---|---|---|
| Infinite Premium For All Players | `set auth_infinitePremiumForRoles "player"` in `shard_local.cfg` (empty when off) | All accounts with the default `player` role, on login |
| New Account Credits | `sm_newAccountCredits` (default 10000) | Accounts created afterwards |
| Starter Pack: Credits / Galactic Standards / Token Credits | Rewrites the `credits`, `goldCredits` and `tokenCredits` attributes in `cloud/data/starter_pack.xml` (defaults 0/0/0). Any other content in the file is left alone. Setting all three back to 0 gives a byte-identical original. | Accounts created afterwards |
| Bonus GS Per Win / Per Loss | `reward_bonusGoldForWinner` / `reward_bonusGoldForLooser` in `game_reward_dispencer_local.cfg` | Battles finished afterwards |
| Bots In Arena Matches | `cvars.mm_botsCount` in `cloud/data/default_platform_config.lua` (vanilla 6; 23 fills a 24-player arena) | Matchmaking, from the next start |
| Kick Excessive Bots | `cvars.sv_kickExcessiveBotsEnabled` in the same file (vanilla 1) | Battles, from the next start |
| Buy Everything | `cvars.store_allowAllItemsForBying` in the same file (vanilla 0). Buying obsolete or dev items can break an account. | Store, from the next start |

Starter pack and new-account values are only granted when an account is created. Changing them does not touch existing balances.

The three matchmaking/store values live in `default_platform_config.lua`, not a `.cfg`. ConfigServer is the only service that reads that file, and it has no override file of its own, so a `set` in a `*_local.cfg` would be overwritten. Only those three `cvars.… = N` lines are rewritten. Setting the vanilla values back gives a byte-identical file.

These come from SCPatcher (Star Conflict Mod Manager) by Furebel, limited to the parts that are plain config values. Its other mods are not included because they edit game data (loot tables, quests, ore, iridium) and need `prepare_shared` to rebuild the server XML: arena queue timing, arena search fix, canister loot, enemy rarity, iridium multiplier, ore drops and quest GS rewards.

## Custom cvars

Files the template owns are regenerated on every start: `loadbalancer_local.cfg`, `shard_local.cfg`, `chat_server_local.cfg`, `dediserver_local.cfg` and `game_reward_dispencer_local.cfg`. It also rewrites the three currency attributes in `starter_pack.xml` and three cvar lines in `default_platform_config.lua`. Put your own settings in `server/cloud/data/common_local.cfg`. Every service execs it and the template never touches it.

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
- That an empty `net_override*Addr` behaves as "no override".
- That battle servers pick ports from `net_gamePort` upward, and whether they use TCP or UDP. Both are forwarded.
- How the client is pointed at a remote server (`net_masterServerAddress` format and where `user_config.cfg` is read from).
- What happens to Hub on `SIGTERM` under Wine. Child services shut themselves down when Hub disconnects. mongod keeps running between app restarts, and the next start reuses it. It only stops when the container stops.
- That the tarball is complete for a server-only install. It leaves out the client `.pak` archives, `data/ui` and all `data/levels` textures. `DedicatedServer.exe` references `levels/*.map`, `scene.xml` and `pathfinding.nav`, which are included.
- The instance is marked Ready when Hub logs that the load balancer registered (`registered itself as 810000000000001`). If that service ID changes, AMP will stay on "Starting" while the server works fine. Adjust `Console.AppReadyRegex` if that happens. Hub writes to a Windows console, not stdout, so its log reaches AMP only through `star-conflict/logs/hub.log`. Each service still writes its own log to `.wine/drive_c/users/*/AppData/Local/Targem/StarConflict/cloud/Hub/logs/`.

## Rebuilding the server tarball

From a Steam install of Star Conflict:

```sh
cd "$HOME/.local/share/Steam/steamapps/common/star conflict"
tar --sort=name --owner=0 --group=0 --numeric-owner \
  --exclude='./cloud/Exceptions' --exclude='./cloud/mongodb/data/*' --exclude='*_local.cfg' \
  --exclude='./cloud/mongodb/'{bsondump,mongoexport,mongofiles,mongoimport,mongooplog,mongoperf,mongos,mongostat,mongotop}.exe \
  --exclude='./data/levels/*.dds' --exclude='./data/levels/**/*.dds' \
  -czf star-conflict-server.tar.gz ./cloud ./data/gamedata ./data/levels ./sys_data
```

`cloud/mongodb/mongo-r2.6.12.zip` is MongoDB's source code. Keep it in the archive: MongoDB 2.6 is AGPL-3.0, and redistributing the binary requires providing the source.
