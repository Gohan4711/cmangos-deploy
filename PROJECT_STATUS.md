# Project Status

Last updated: 2026-09-11

## Project goal

Run a reproducible CMaNGOS Classic server with PlayerBots, custom LLM chat behavior, and believable bot progression. The long-term direction is a large autonomous bot population with natural leveling/professions, while only solving progression hard-gates that bots cannot reasonably clear on their own.

## Repositories and branches

- Deploy/runtime: `Gohan4711/cmangos-deploy`, branch `llm-status-context`
- PlayerBots code: `Gohan4711/playerbots`, branch `llm-status-context`
- Upstream deploy: `mserajnik/cmangos-deploy`
- Upstream PlayerBots: `cmangos/playerbots`
- Upstream core: `cmangos/mangos-classic`

## Known-good revisions

- Current deploy checkpoint: `b6fc50f5413772104cdcee32fcc363e1ff125656`
- Previous patch-based deploy checkpoint: `a5bfae1f568b54efa4187741d51912506c068bb1`
- Custom PlayerBots checkpoint: `2adf08264f5e387c598713595c7942a29f90a21e`
- CMaNGOS core revision used for the current build: `8ec338a1704e7dcb1c0213eb7ed58f9231ade40f`
- Custom PlayerBots revision used by the current Docker build: `2adf08264f5e387c598713595c7942a29f90a21e`
- Local image tag: `cmangos-server-classic-llmstatus:local`
- Known-good direct-build image ID/manifest: `sha256:bb189b65bb583e6cdf9dd1866ada1b56089b674aed071c61e3551425b62d82df`

## Current runtime state

The current image builds directly from the pinned `Gohan4711/playerbots` custom commit, starts successfully, and initializes PlayerBots correctly. The generated PlayerBots patch is no longer part of the build path.

Observed startup output includes:

- `Initializing AI Playerbot by ike3, based on the original Playerbot by blueboy`
- `No new random bots needed. Accounts: 200, bots: 1800.`
- `AI Playerbot initialized`

The PlayerBots config-path bug was traced to the process working directory `/home/cmangos` combined with compiled `SYSCONFDIR=../etc/`. The runtime image now creates:

```text
/home/etc -> /opt/cmangos/config
```

This allows `aiplayerbot.conf` and other relative `../etc/...` config lookups to resolve to the mounted runtime config directory.

## Custom PlayerBots behavior in the current checkpoint

### LLM chat

- Alliance bots are gated out of normal LLM generation paths.
- Real-player `/say` naming a bot routes the reply to the named bot instead of every nearby bot replying.
- Real-player `/say` without a bot name gives each eligible bot a 10% reply chance.
- Chat response delay was increased from 5-25 seconds to 35-55 seconds.
- LLM prompt context includes combat status and current target.
- Bot-to-bot LLM `/say` is constrained to nearby bots and requires a nearby friendly real player; the configured bot-to-bot chance still applies.
- Direct whispers are intentionally not blocked by the `/say` routing rules.

### First Aid progression helper

The helper is intentionally narrow. It does not grant cloth, bandages, skill points, or arbitrary profession ranks.

- First Aid >= 125 and missing Expert rank: add `Expert First Aid - Under Wraps` (`16084`).
- First Aid >= 180 and missing Heavy Silk Bandage: add manual `16112`.
- First Aid >= 210 and missing Mageweave Bandage: add manual `16113`.
- First Aid >= 225, level >= 35, and missing Artisan rank: cast original teaching spell `10847`, deliberately bypassing the Triage quest while still using core spell/skill-step behavior.
- If inventory is full, book insertion fails cleanly and the maintenance action can retry later.

Existing PlayerBots behavior already handles recipe use, bandage crafting when skill-ups are available, and choosing higher-rank bandages for use.

## Validation status

### Verified

- Custom PlayerBots source compiles with the pinned CMaNGOS core.
- Direct Docker build from the pinned custom PlayerBots commit completes successfully.
- The running `mangosd` container was verified to use image `sha256:bb189b65bb583e6cdf9dd1866ada1b56089b674aed071c61e3551425b62d82df`.
- `mangosd`, `realmd`, and database containers start successfully.
- `aiplayerbot.conf` is now found and PlayerBots initializes.
- Both custom repositories and working branches are pushed to GitHub.

### Still pending

- In-game runtime test of the First Aid helper at skill thresholds 125 / 180 / 210 / 225.
- Re-test the full LLM flow on the latest direct-build image after the First Aid changes.
- Profession assignment/distribution implementation and profession hard-gate audit.

## Important operational rules

- Never use `docker compose down -v` for routine work; the database volume must not be destroyed accidentally.
- Source inspection and Git operations can be done while the server is running.
- Stop `mangosd` before rebuilding/replacing its image when practical.
- Database and `realmd` do not need to be stopped for PlayerBots source builds.
- LLM shim/Ollama are not required for PlayerBots startup or profession/First Aid tests.

## Build source of truth

The Docker build now clones `Gohan4711/playerbots` directly and pins commit `2adf08264f5e387c598713595c7942a29f90a21e`. The former generated `docker/server/llm-status-context.patch` has been removed from the active build path.
