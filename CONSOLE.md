# Remote console commands (Hub and Shard)

All console commands of **Hub** and **Shard**, as reported by the services themselves (`cmdList`, `help <command>`, build 1.99.0.167293), with what they do. Descriptions in quotes are the game's own; the rest come from testing on a running server or, where marked *(untested)*, from the command's name and parameters.

## Using the console

Set a **Remote Console Password** (instance settings, Administration) and restart. Then, in AMP's console:

- `<command>` runs on **Hub**. Its output appears in the console through Hub's log, with timestamps.
- `shard <command>` runs on **Shard**. Its output is printed with `[Shard]` in front, followed by `executed '<command>', result is: success|failure`.

Parameters in `<angle brackets>` are required, `[square brackets]` optional. `help <command>` shows a command's parameters, `help <cvar>` a setting's description. A command takes a few seconds (see [Remote console](README.md#remote-console) for why).

Markers: ⚠ changes or stops something, use with care. ⛔ don't use on a live server.

## Account commands

These are the template's own commands, typed in AMP's console like the others. They work through MongoDB and Shard's admin tasks (the mechanism of the developers' old web panel), so they need no Remote Console Password. Grants are carried out by Shard itself, so they also work while the player is online; the player may need to relog to see them.

| Command | What it does |
|---|---|
| `player <uid or name>` | Shows an account: uid, nickname, username, role, online or offline, GS, credits, tokens, premium, ban and chat mute, ships, number of items and titles. The name is matched case-insensitively. |
| `give <uid> gs <amount>` | Adds Galactic Standards. A negative amount takes them away (`give 3 gs -500`). |
| `give <uid> credits <amount>` | Adds (or takes) credits. |
| `give <uid> tokens <amount>` | Adds (or takes) token credits. |
| `give <uid> premium <days>` | Adds premium time: from now if premium has run out, otherwise on top of it. |
| `give <uid> item <shop id> [amount]` | Adds an item from the shop by its id: module, weapon, consumable, blueprint, resource, also items that aren't sold. |
| `give <uid> ship <shop id>` | Adds a ship by its shop id (the same as `item`). |
| `give <uid> title <title id>` | Adds a title. |
| `find <text>` | Searches the shop and the titles for `<text>` (internal or display name, at least 3 characters, matched as typed): shop id, type, internal name, display name, GS price and whether it's normally sold. |

Example: `find Thor` lists `item 5784 ship Ship_Race1_M_T5_Dlc Thor, 3500 GS`, then `give 3 ship 5784` gives account 3 the Thor. The full list of item and title IDs is in [ITEMS.md](ITEMS.md).

Behind them (tested on a running server): `give` is the admin task `give` with `value: goldCredits|credits|tokenCredits|premiumTill, amount` (premium in hours) or `item: <shop id>, amount`; titles use `giveTitle`; tasks are queued in `adminTasks` and Shard runs them within 5 seconds. The uid is the number in the console's `[AMP] Player joined: name (uid N)` lines, or from `player <name>`.

Not offered on purpose: decals and paints (the admin task `giveVesselCustomElement` rejected every parameter tried) and `giveFreeSpace`, which grants open-world access by changing the account's role to `alpha-tester-dlc`: an account has only one role, so it would take Admin or Moderator away.

## Most useful

| Command | Runs on | What it does |
|---|---|---|
| `cloud_status` | Hub | Every service of the server with its cloud address, host:port, system load, SSH port and process ID. |
| `hub_status` | Hub | The services connected to Hub, and which service owns which part of the work (QS, AS, AU and Adventure hash rings). |
| `uptime` | both | "Log application uptime": how long the service has been running. |
| `shard sm_dumpClients` | Shard | Players currently connected to Shard. |
| `enterMaintenanceMode [minutes]` | Hub | ⚠ Puts the server into maintenance: only accounts with the `playInMaintenance` privilege (Admin and Moderator roles) can log in. *(untested)* |
| `leaveMaintenanceMode [minutes]` | Hub | Ends maintenance mode. *(untested)* |
| `cvarList [filter]` | both | "List of registered cvars": every setting with its current value, e.g. `cvarList db_` or `shard cvarList sm_`. |
| `set <cvar> [value]` | both | ⚠ "Set cvar value": changes a setting until the service restarts. Without a value it prints the current one. Settings the template manages are rewritten on every start. |
| `cron_addCmd <dow> <hours> <minutes> <cmd>` | both | ⚠ "schedule command": runs `<cmd>` every week at that time, see [Scheduling](#scheduling). |

## Hub only

| Command | Parameters | What it does |
|---|---|---|
| `cloud_status` | | See above. Tested. |
| `hub_status` | | See above. Tested. |
| `hub_routes` | | Hub's routing table: Hub itself (`… is myself`) and any further Hubs. On a single server it lists just one line. Tested. |
| `hub_reloadCloudXml` | | ⚠ Re-reads `cloud/data/cloud.xml` (the services and the node address) without a restart. The template rewrites the node address on every start. *(untested)* |
| `enterMaintenanceMode` | `[minutes]` | ⚠ See above. The minutes are probably a countdown shown to players before maintenance begins. *(untested)* |
| `leaveMaintenanceMode` | `[minutes]` | See above. *(untested)* |

## Shard only

| Command | Parameters | What it does |
|---|---|---|
| `sm_dumpClients` | | Players connected to Shard. Tested (empty table without players). |
| `sm_disconnectAll` | | ⚠ Disconnects every player from Shard (they can log in again). *(untested)* |
| `store_dump` | | Writes the whole shop (every item: id, credit, GS, token and event prices, flags, name) to Shard's log: about 57,000 lines. Useful to check what the server really loaded from `store.xml`. Tested. |
| `dbqp_printData` | | Database query statistics: how often each kind of query ran and how long it took. Tested. |
| `dbg_testAccountRankUp` | `<uid>` | ⚠ Test command: raises the account's rank. Changes the account. *(untested)* |
| `dbg_testBattlePassUnlockPremium` | `<uid> <eventStartTime_YYYY-MM-DDThh:mm:ssZ> <stage> [<unlockLevels>]` | ⚠ Test command: unlocks the premium track of a battle pass for an account; the start time must match the battle pass's start (Battle Pass Start setting). *(untested)* |
| `dbg_giveAutogenModule` | `<uid> <rollData>` | ⚠ "verifies given rollData and if success adds autogen item to specified uid". The format of `rollData` is unknown. *(untested)* |
| `dbg_logUserInfo` | `<yuplayId>` | Logs an account by its Yuplay (Gaijin account) ID. Not applicable to a self-hosted server. |
| `dbg_testHangUp` | | ⛔ Test command that deliberately hangs Shard. |
| `arc_testLogin` | `<arc_account> <arc_token>` | Gaijin Arc login test. Doesn't work without Gaijin's services. |
| `arc_testPurchase` | `<arc_id>` | Gaijin Arc purchase test. Doesn't work without Gaijin's services. |
| `steam_testInitTxn` | `<steamid> <en/ru> <RUB/USD/EUR/..>` | Steam microtransaction test (needs the publisher's Steam key). Doesn't work self-hosted. |

## Both (engine commands)

These exist in every service. On Shard, put `shard ` in front.

### Information

| Command | Parameters | What it does |
|---|---|---|
| `help` | `<command/cvar/alias name>` | "Show description for commands and cvars or substituted string for aliases". Tested. |
| `cmdList` | `[filter]` | "List of registered commands". Tested. |
| `cvarList` | `[filter]` | "List of registered cvars", with their values. Tested. |
| `uptime` | | "Log application uptime". Tested. |
| `ctm_status` | | "report task manager status": the service's background task queues and how often each task ran. Tested. |
| `cloud_status_cached` | | "print cached cloud status if available". Printed nothing in testing (Hub uses `cloud_status`). |
| `dumpMemStats` | | "dump memory statistics" of the engine to the log. *(untested)* |
| `stat_endpointDump` | | "log stat endpoint state": the connection to the developers' statistics server (not used here). |
| `echo` | `<text>` | "Print command parameters". |

### Settings

| Command | Parameters | What it does |
|---|---|---|
| `set` | `<cvar> [value]` | ⚠ "Set cvar value", until the service restarts. Tested. |
| `seta` | `<cvar> [value]` | ⚠ Same, and marks the setting to be saved ("CVAR_ARCHIVE"). Whether a server service saves it anywhere is unknown; use the `*_local.cfg` files for lasting changes. |
| `toggle` | `<cvar> [<values...>]` | ⚠ "Cyclic toggle cvar value": switches a setting to its next value (0/1, or through the values given). |
| `incVar` | `<cvar> <startValue> <endValue> [<step>]` | ⚠ "Cyclic increment/decrement cvar value". |
| `exec` | `<filename>` | ⚠ "Execute console script file": runs a `.cfg` file of `set` lines, path relative to `server/cloud/data/` (e.g. `exec common_local.cfg`). |

### Scheduling

| Command | Parameters | What it does |
|---|---|---|
| `cron_addCmd` | `<dow> <hours> <minutes> <cmd>` | ⚠ "schedule command": runs `<cmd>` weekly. **UTC time**, day of week **0 = Monday** (observed: on a Thursday at 17:47 CEST, `cron_dump` showed `dow: 3 h: 15`). Example: `cron_addCmd 0 4 0 enterMaintenanceMode 15`. Entries live in memory: a restart clears them. Tested (adding and listing). |
| `cron_addCvarToggler` | `<dow> <hours> <minutes> <duration> <cvar> <toggle_to> <toggle_back_to>` | ⚠ "schedule cvar toggler": weekly, sets `<cvar>` to `<toggle_to>` for `<duration>` (probably minutes), then back. E.g. a weekend event. *(untested)* |
| `cron_dump` | | "dump schedule": the current time as cron sees it and all entries with their index. Tested. |
| `cron_remove` | `<index>` | "remove entry by idx" (0-based, from `cron_dump`). |
| `cron_removeAll` | | "remove all entries". Tested. |

### Aliases

| Command | Parameters | What it does |
|---|---|---|
| `alias` | `<alias> <string>` | "Add an alias": a short name for a command line, until restart. |
| `aliasList` | `[filter]` | "List of aliases". |
| `unalias` | `<alias>` | "Remove an alias". |

### Lua scripting

| Command | Parameters | What it does |
|---|---|---|
| `scriptRun` | `<lua>` | ⚠ "execute script command": runs Lua inside the service, with the game's script API. Powerful and unchecked. |
| `scriptRunFile` | `<file>` | ⚠ "execute script file". |
| `scriptSaveExports` | `[<filename>] [list parent exports?] [skip empty?]` | "save script exports description to file" (default `ScriptExports.txt` in `server/cloud/`): the list of functions the service's Lua can call. The starting point for anything `scriptRun` could do. |
| `scriptForceGC` / `scriptStepGC` | | "force" / "step" Lua garbage collection. |

### Stopping and testing

| Command | Parameters | What it does |
|---|---|---|
| `quit` | | ⛔ "shutdown application". On Hub it stops the whole server behind AMP's back (AMP then sees the application exit); on Shard it stops Shard alone. Use AMP's Stop/Restart instead. |
| `testCrash` | | ⛔ "Generate exception to test error handling": crashes the service on purpose. |
| `maintainMem` | | "immediately maintain memory managers" (frees cached engine memory). |
| `con_dump` | `<filename>` | "Dump console buffer to file", relative to `server/cloud/`. |
| `clear` | | "Clear the console buffer". |
| `profiler_statsAttach` / `profiler_statsDetach` | `<profiler_name>` | Attach/detach a profiler to the statistics manager (developer tooling). |
| `stat_endpointConnect` / `stat_endpointDisconnect` | | Connect to / disconnect from the developers' statistics server (not used here). |
| `gonalytics_sendEvent` | | "send gonalytics event": the developers' analytics (not used here). |
| `http_get` | `"<url>"` | ⚠ "Do a GET request on url" from the server. |
| `http_uploadFile` | `"<url>" "<filepath>" <fieldname>` | ⚠ "Do a multipart POST containing file on url": uploads a file from the server to any URL. |

### Client leftovers (no effect on a server)

`bind`, `bindList`, `unbindall`, `toggleConsole`, `openConsoleWithText`, `cl_dumpTypeDb`, `wait`: key bindings, the client's console window, the reflection database dump and a one-frame delay. They exist because the services share the engine with the game client.

## Not here

- **Kick, mute and ban** are not console commands (granting is, see [Account commands](#account-commands)). They are admin tasks, used in game through the Punish menu (Admin and Moderator roles, see [Admin and moderator roles](README.md#admin-and-moderator-roles)).
- **Other services** (MatchMaker, ChatServer, …) have their own consoles with the same engine commands, but the template only opens Hub's and Shard's.
