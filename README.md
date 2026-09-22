# SkyMine - Bukkit Mining Plugin and Server Deployment Scripts

Two things living in one folder: `SkyMineCore`, a Paper/Purpur plugin implementing tiered
auto-resetting mines with a rank ladder, and the pile of bash/Python scripts used to deploy and
repair the SkyMine server around it - first as a bare `screen` process, then under Ahurapanel
(a patched `minepanel`) in Docker, and finally talking to a remote Pterodactyl panel. The plugin
is real code; the scripts are one-shot, host-specific operational glue.

**Suggested repo name:** `skymine-minecraft-plugin`
**Stack:** Java (Bukkit/Paper API 1.21, Vault economy API, LuckPerms API), bash, small Python patchers
**Status:** active
**Last modified:** 2026-08-28

## What it does

- `SkyMineCore.java` - plugin entry point: resolves Vault and LuckPerms on enable, loads mines from
  config, runs a 5-second scheduler that resets any mine past its threshold or interval, listens for
  block breaks to count mined blocks, rewrites join/quit messages, hands out a starter kit to new
  players, and implements `/spawn`, `/kit`, `/rankup`, `/mines [name]`, `/setmine <name>`,
  `/skyminewand`, `/reloadmine <name>`, `/skymine`. Rankup withdraws the cost from Vault and swaps
  the `skymine.rank.*` LuckPerms node.
- `Mine.java` - cuboid mine model: weighted block composition parsed from `MATERIAL:chance;...`,
  reset on percent-broken or on an interval, refills only air blocks, 5,000,000-block volume cap on
  wand-created selections.
- `plugin.yml` / `config.yml` - command and permission declarations, MOTD, starter kit, the
  Miner → CoalMiner → IronMiner → DiamondMiner → NetheriteMiner ladder, empty `mines: {}`.
- `deploy_ahurapanel.sh`, `ahurapanel.sh`, `fix_panel.sh`, `fix_frontend.sh`,
  `rebuild_frontend.sh`, `fix_sidebar.sh`, `fix_translations.py` - clone, patch, build and relaunch
  Ahurapanel under `screen`, including new translation keys for the remote-panel page.
- `deploy_ptero.sh`, `deploy_ptero2.sh` - add a Pterodactyl integration to that panel and expose it
  at `/pterodactyl/servers`.
- `migrate_server.sh`, `fix_mcdata.sh`, `fix_server_config.sh`, `fix_cm_config.py` - move the
  existing server into `/app/servers/skymine/mc-data` so Docker manages it, and fix CataMines config.
- `fix_server.sh` … `fix4_server.sh`, `install_spark.sh`, `fix_maven.py`, `fix_settings.py` - install
  spark (built from source with the Gradle module list trimmed to the bukkit chain) and ProtocolLib.
- `server-config.json` - `itzg/minecraft-server` environment for PURPUR 1.21.8, 3 G heap.

## Layout

```
SkyMineCore.java   plugin main class, commands, listeners, rankup
Mine.java          mine region model and reset logic
plugin.yml         commands + skymine.admin permission
config.yml         motd, starter_kit, ranks, mines
deploy_*.sh        panel + Pterodactyl deployments (contain secrets)
fix_*.sh / .py     targeted repairs against one live host
ahurapanel.sh      start|stop|restart|status wrapper
server-config.json Docker/itzg server definition
```

## Notes

- Secrets are committed and must be stripped before publishing: a plaintext sudo password in
  `deploy_ahurapanel.sh` and `fix_panel.sh`, a live Pterodactyl application key in `deploy_ptero.sh`
  and `deploy_ptero2.sh`, a panel administrator login used in most `fix_*.sh` scripts, plus public
  host addresses and the panel domain. Rotate everything, not just delete the lines.
- There is no build file. To compile, put `SkyMineCore.java` and `Mine.java` under
  `src/main/java/me/skymine/core/` and build against Paper API 1.21 plus the Vault and LuckPerms
  API jars; `plugin.yml` is already in the right shape for the jar root.
- The bash/Python scripts assume `/home/developer/ahurapanel`, `/app/servers/skymine`,
  `/home/developer/skymine` and a `screen` session called `skymine`. They will fail harmlessly but
  uselessly anywhere else.
- `onlineMode: false` in `server-config.json` - offline mode, so usernames are unverified.
