# Build and Run

This document records the current known-good workflow for the custom CMaNGOS Classic + PlayerBots image.

## Current build model

The current Docker image still builds from upstream PlayerBots at a pinned revision and then applies `docker/server/llm-status-context.patch`.

Pinned revisions used for the known-good build:

- CMaNGOS core: `8ec338a1704e7dcb1c0213eb7ed58f9231ade40f`
- PlayerBots base: `89a4e5aebd6aa41ee87f6e65d89b66fff5c5c7c1`

Custom PlayerBots source is also versioned separately in `Gohan4711/playerbots` on branch `llm-status-context`; the current custom checkpoint is `2adf08264f5e387c598713595c7942a29f90a21e`.

The patch-based Docker path is temporary and should eventually be replaced with a direct build from the custom PlayerBots fork.

## Build command

Run from the deploy repository root:

```bash
cd ~/cmangos-deploy && docker build \
  -f docker/server/Dockerfile.llmstatus \
  --build-arg CMANGOS_EXPANSION=classic \
  --build-arg CMANGOS_CORE_REVISION=8ec338a1704e7dcb1c0213eb7ed58f9231ade40f \
  --build-arg CMANGOS_PLAYERBOTS_REVISION=89a4e5aebd6aa41ee87f6e65d89b66fff5c5c7c1 \
  -t cmangos-server-classic-llmstatus:local \
  .
```

Known-good image tag:

```text
cmangos-server-classic-llmstatus:local
```

Known-good manifest from the current combined build:

```text
sha256:1844b0debea185e6595c652b64006d0774b33025b803a46a59256a9bbaac955a
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

## Temporary patch detail

`docker/server/llm-status-context.patch` contains CRLF-derived lines from upstream PlayerBots source. A generic `git diff --check` reports trailing-whitespace warnings against the patch file itself. The patch has separately passed `git apply --check` and produced a successful build; do not normalize its line endings casually while it remains the active build artifact.
