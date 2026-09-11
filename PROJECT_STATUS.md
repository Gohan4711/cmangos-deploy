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

- Deploy checkpoint: `a5bfae1f568b54efa4187741d51912506c068bb1`
- Custom PlayerBots checkpoint: `2adf08264f5e387c598713595c7942a29f90a21e`
- CMaNGOS core revision used for the current build: `8ec338a1704e7dcb1c0213eb7ed58f9231ade40f`
- Upstream PlayerBots base revision used by the current Docker build: `89a4e5aebd6aa41ee87f6e65d89b66fff5c5c7c1`
- Local image tag: `cmangos-server-classic-llmstatus:local`
- Known-good image manifest from the current combined build: `sha256:1844b0debea185e6595c652b64006d0774b33025b803a46a59256a9bbaac955a`

## Current runtime state

The current combined image builds successfully and starts the server. PlayerBots configuration loading is fixed and PlayerBots initializes successfully.

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
- Combined Docker image build completes successfully.
- PlayerBots patch applied cleanly before build.
- `mangosd`, `realmd`, and database containers start successfully.
- `aiplayerbot.conf` is now found and PlayerBots initializes.
- Both custom repositories and working branches are pushed to GitHub.

### Still pending

- In-game runtime test of the First Aid helper at skill thresholds 125 / 180 / 210 / 225.
- Re-test the full LLM flow on the latest combined image after the First Aid changes.
- Replace the temporary Docker patch workflow with direct builds from `Gohan4711/playerbots` at a pinned custom commit.
- Remove `docker/server/llm-status-context.patch` once the Docker build consumes the custom PlayerBots repository directly.
- Profession assignment/distribution implementation and profession hard-gate audit.

## Important operational rules

- Never use `docker compose down -v` for routine work; the database volume must not be destroyed accidentally.
- Source inspection and Git operations can be done while the server is running.
- Stop `mangosd` before rebuilding/replacing its image when practical.
- Database and `realmd` do not need to be stopped for PlayerBots source builds.
- LLM shim/Ollama are not required for PlayerBots startup or profession/First Aid tests.

## Known repository detail

`docker/server/llm-status-context.patch` contains CRLF-derived lines from upstream source. `git diff --check` therefore reports trailing-whitespace warnings when checking the patch file itself. The active patch was separately verified with `git apply --check` and has produced a successful build. Do not normalize it casually while it remains part of the known-good build path.
