# Star Conflict — AMP Generic Module Template

Self-host a private Star Conflict dedicated server managed through [AMP (Application Management Portal) by CubeCoders](https://cubecoders.com/AMP).

---

## What this template does

This folder contains **two AMP Generic Module templates** that work together:

| Template | Manages | Install first? |
|---|---|---|
| `star-conflict-mongodb.kvp` | MongoDB 2.6.x database | **Yes** |
| `star-conflict-generic.kvp` | Hub.exe + all game services | Second |

- The MongoDB instance downloads MongoDB 2.6.12 and manages `mongod` as its own process.
- The Star Conflict instance downloads game files via SteamCMD and runs Hub.exe (via Wine on Linux).
- AMP's **startup dependency** system ensures MongoDB is always running before Hub.exe starts.

---

## Prerequisites

### On every host

| Requirement | Notes |
|---|---|
| AMP ≥ 2.5.0.8 with ADS | Generic Module + Application Deployment Service |
| SteamCMD | AMP can install this automatically |

### On Linux hosts only (Star Conflict instance only)

| Requirement | Notes |
|---|---|
| Wine (stable, winehq.org) | Or use the recommended Docker image `cubecoders/ampbase:wine-stable` |

> **Why Docker on Linux for the Star Conflict instance?**  
> All Star Conflict server binaries are Windows `.exe` files. The Docker image bundles a correct Wine version and avoids host-side Wine installation and prefix management. The MongoDB instance does **not** need Wine — it uses a native Linux binary.

---

## Setting up both AMP instances

### Step 1 — Copy template files

Copy **all files** from this folder into your AMP templates directory (typically `~/.ampdata/templates/` or the path shown in AMP's Generic Module settings).

### Step 2 — Create the MongoDB instance

1. In AMP ADS, create a new instance → **Generic Module** → search for **Star Conflict MongoDB**.
2. Name it something clear, e.g. `StarConflict-DB`.
3. Run **Update / Install** — AMP downloads MongoDB 2.6.12 and writes the default `mongod.cfg`.
4. Start the instance and confirm it reaches the **Running** state. Check the console for:
   ```
   [initandlisten] waiting for connections on port 27017
   ```

### Step 3 — Create the Star Conflict instance

1. In AMP ADS, create a new instance → **Generic Module** → search for **Star Conflict**.
2. Name it e.g. `StarConflict-Server`.
3. Run **Update / Install** — AMP downloads the game via SteamCMD and (on Linux) initialises Wine.

### Step 4 — Link the instances (startup dependency)

This is what makes AMP automatically start MongoDB before Star Conflict.

1. In AMP ADS, open the **StarConflict-Server** instance → **Instance Settings** (or **Configuration** tab depending on your AMP version).
2. Find the **Startup Dependencies** / **Application Dependencies** section.
3. Add `StarConflict-DB` (the MongoDB instance) as a dependency.
4. Save. AMP will now start `StarConflict-DB` automatically whenever `StarConflict-Server` is started, and will wait for MongoDB to report ready before launching Hub.exe.

> **How AMP knows MongoDB is ready:** The MongoDB template sets `Console.AppReadyRegex` to match the `waiting for connections on port` log line. AMP waits for this before considering the dependency satisfied.

---

## Port forwarding

Open the following TCP ports in your router/firewall to allow clients to connect:

| Port | Service | Required for clients? |
|---|---|---|
| 3801 | Load Balancer | **Yes** — primary game port |
| 3802 | Shard | **Yes** — match sessions |
| 3815 | Chat Server | **Yes** — in-game chat |
| 3850 | Hub | No — internal only |
| 2222 | SSH Admin | Optional — admin access |
| 27017 | MongoDB | **No** — localhost only, managed by MongoDB instance |

---

## Configuration

### Via AMP Settings UI

The **Network**, **Performance**, and **Administration** sections in the AMP Settings tab let you change the most common values without editing files manually:

| Setting | Config file | Default |
|---|---|---|
| Load Balancer Port | `cloud/data/loadbalancer.cfg` | 3801 |
| Shard Port | `cloud/data/shard.cfg` | 3802 |
| Chat Server Port | `cloud/data/chat_server.cfg` | 3815 |
| Hub Management Port | `cloud/data/hub.cfg` | 3850 |
| Server FPS Cap | `cloud/data/dediserver.cfg` | 32 |
| SSH Admin Port | `cloud/data/dediserver.cfg` | 2222 |

### Manual config editing

All service config files live in `<AMP instance root>/star-conflict/212070/cloud/data/`. Each file uses the format:

```
set variable_name "value"
```

Lines beginning with `//` are comments. Changes take effect on the next server restart. Each service has a `*_local.cfg` companion (e.g. `hub_local.cfg`) which is loaded last and is the safest place for custom overrides — it will not be overwritten by game updates.

### Matchmaking

Matchmaking queues, MMV weights, and mode-specific behaviour are configured in `cloud/data/default_platform_config.lua`. Refer to `cloud/data/matchmaking.md` (included with the game) for the full parameter reference.

---

## Server architecture overview

Hub.exe is the **process manager**. When started, it reads `cloud/data/cloud.xml` and launches all child services on the same node:

| Service | Type ID | Default port |
|---|---|---|
| Load Balancer | 1 | 3801 |
| Shard | 2 | 3802 |
| MatchMaker | 3 | — |
| Game Reward Dispenser | 7 | — |
| Authenticator | 8 | — |
| Chat Server | 9 | 3815 |
| Config Server | 11 | — |
| Universe | 12 | — |
| Quest Server | 13 | — |
| Zone Coordinator | 14 | — |
| Lobby Server | 16 | — |
| Achievement Server | 17 | — |
| Leaderboard Server | 18 | — |
| Clan Server | 19 | — |
| Tournament | 20 | — |
| Early MatchMaker | 21 | — |
| Scripts Server | 22 | — |
| Adventure Server | 23 | — |

All services run within the single Hub process tree on a single node (`127.0.0.1:3850`).

---

## Troubleshooting

**Server starts but clients cannot connect**
- Verify ports 3801 and 3802 are open and forwarded.
- Check that MongoDB is running: `mongo --eval "db.runCommand({ping:1})"`.

**Wine errors on Linux**
- Use the recommended Docker container (`cubecoders/ampbase:wine-stable`) instead of a host Wine installation.
- If running without Docker, ensure Wine `stable` from winehq.org is installed, not a distro package.

**SteamCMD prompts for Steam credentials**
- This is expected. Star Conflict requires a Steam account login to download even though it is free-to-play. AMP will prompt for your Steam username, password, and Steam Guard code (if enabled) when you run Update/Install. The account just needs to own or have launched Star Conflict at least once.

**SteamCMD download fails / times out**
- The download is the full game client including server files — expect several GB. Check AMP's SteamCMD log for network errors or a Steam maintenance window.

**Services crash immediately after start**
- The MongoDB instance is not running or the startup dependency is not configured. Check that `StarConflict-DB` is in the Running state, then verify the dependency link under the Star Conflict instance settings.
- Check `cloud/data/cloud.xml` — the node address (`127.0.0.1:3850`) must be reachable by Hub.

**MongoDB instance fails to start**
- The `fastdl.mongodb.org` download URLs for MongoDB 2.6.12 are legacy endpoints. If they are unavailable, manually place `mongod.exe` (Windows) or `mongod` (Linux) into the `bin/` directory of the MongoDB instance and run Update again skipping the download stage.
- On Linux, ensure the `mongod` binary has execute permission: `chmod +x bin/mongod`.

**Config changes not applied**
- After editing any `.cfg` file, restart the AMP instance. Live setting changes are not supported.
