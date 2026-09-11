# Build and Run

This document records the current known-good workflow for the custom CMaNGOS Classic + PlayerBots image.

## Current build model

The Docker image builds directly from the custom `Gohan4711/playerbots` fork at a pinned commit. No generated PlayerBots patch is applied.

Pinned revisions used for the known-good build:

- CMaNGOS core: `8ec338a1704e7dcb1c0213eb7ed58f9231ade40f`
- Custom PlayerBots: `2adf08264f5e387c598713595c7942a29f90a21e`

PlayerBots source is versioned in `Gohan4711/playerbots` on branch `llm-status-context`.

## Build command

Run from the deploy repository root:

```bash
cd ~/cmangos-deploy && docker build \
  -f docker/server/Dockerfile.llmstatus \
  --build-arg CMANGOS_EXPANSION=classic \
  --build-arg CMANGOS_CORE_REVISION=8ec338a1704e7dcb1c0213eb7ed58f9231ade40f \
  --build-arg CMANGOS_PLAYERBOTS_REPOSITORY_URL=https://github.com/Gohan4711/playerbots.git \
  --build-arg CMANGOS_PLAYERBOTS_REVISION=2adf08264f5e387c598713595c7942a29f90a21e \
  -t cmangos-server-classic-llmstatus:local \
  .
```

Known-good image tag:

```text
cmangos-server-classic-llmstatus:local
```

Known-good image ID/manifest from the direct custom-fork build:

```text
sha256:bb189b65bb583e6cdf9dd1866ada1b56089b674aed071c61e3551425b62d82df
```

## Runtime config path

PlayerBots is compiled with `SYSCONFDIR=../etc/`. `mangosd` runs with working directory `/home/cmangos`, so relative config reads resolve through `/home/etc`.

The runtime image therefore creates:

```text
/home/etc -> /opt/cmangos/config
```

This is required so PlayerBots can load `aiplayerbot.conf` from the mounted runtime config directory.

## Start / recreate mangosd

After rebuilding the image, recreate the `mangosd` container so it definitely uses the new image:

```bash
cd ~/cmangos-deploy && docker compose up -d --force-recreate mangosd
```

The database and `realmd` may remain running.

## Verify PlayerBots startup

```bash
cd ~/cmangos-deploy && docker compose logs mangosd --tail=200 | grep -Ei "Playerbot|aiplayerbot|random bot|randombot"
```

Known-good startup output includes:

```text
Initializing AI Playerbot by ike3, based on the original Playerbot by blueboy
Initializing random bot names...
Creating random bot accounts...
Creating random bot characters...
No new random bots needed. Accounts: 200, bots: 1800.
AI Playerbot initialized
```

The previous failure looked like:

```text
AI Playerbot is Disabled. Unable to open configuration file aiplayerbot.conf
```

If that line returns, verify the runtime symlink and process working directory before changing code.

## Service-state guidance

- Git operations / source inspection: server may stay running.
- PlayerBots source edits: server may stay running until rebuild time.
- Image rebuild: preferably stop `mangosd`; database and `realmd` can stay up.
- In-game tests: database, `realmd`, and `mangosd` must be running.
- LLM tests additionally need the LLM shim/Ollama path.
- First Aid/profession tests do not need the LLM shim.

## Safety rule

Do not use this for routine work:

```text
docker compose down -v
```

The `-v` option can remove persistent volumes and is not part of the normal development workflow.

## PlayerBots source pinning

The build uses `Gohan4711/playerbots` directly at the exact pinned commit above. Keep the revision pinned for reproducible builds; update it deliberately after new PlayerBots changes are committed and tested.
