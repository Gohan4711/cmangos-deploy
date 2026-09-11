# Custom Changes

This document explains the custom behavior currently layered on top of upstream CMaNGOS PlayerBots.

## LLM chat changes

### Faction gate

Normal LLM generation paths are blocked for Alliance bots. This includes RPG AI chat trigger/action paths and the main chat reply path.

Purpose: keep the current LLM experiment intentionally limited instead of generating dialogue for both factions.

### Real-player `/say` routing

For real-player `/say` messages:

- If the message names a bot, only the named bot(s) may reply through the LLM path.
- If the message names no bot, each eligible bot gets a 10% chance to reply.
- Whispers are not restricted by this `/say` routing logic.

The name detection normalizes individual words and resolves them against actual player/bot names instead of relying only on substring matching.

### Response pacing

The chat cooldown was changed from the upstream 5-25 second range to 35-55 seconds.

Purpose: prevent the nearby bot population from feeling like an instant-response conference call.

### LLM context

The prompt context now includes:

- combat state (`in combat` / `not in combat`)
- current target name when available

This is appended to the pre-prompt so the language model can better align dialogue with what the bot is currently doing.

### Bot-to-bot LLM chat

Bot-to-bot LLM replies in `/say` are constrained by extra context checks:

- same map
- within 15 yards of the speaking bot
- at least one nearby friendly real player within 50 yards
- configured `llmBotToBotChatChance` still applies

This keeps bot-to-bot chatter local to situations where a real player is actually around to experience it.

For high bot-count production use, the intended configuration is to set the bot-to-bot LLM chance to zero unless deliberately testing the feature.

## First Aid progression helper

File pair:

```text
playerbot/strategy/actions/FirstAidProgressionAction.cpp
playerbot/strategy/actions/FirstAidProgressionAction.h
```

Registered action name:

```text
first aid progression
```

It is run from the non-combat maintenance strategy through the existing `random` trigger.

### Threshold behavior

- Skill >= 125 and missing spell `7924`: add item `16084` (`Expert First Aid - Under Wraps`).
- Skill >= 180 and missing spell `7929`: add item `16112` (`Manual: Heavy Silk Bandage`).
- Skill >= 210 and missing spell `10840`: add item `16113` (`Manual: Mageweave Bandage`).
- Skill >= 225, level >= 35, and missing spell `10846`: cast original teaching spell `10847` as a triggered spell.

### Design intent

The helper only solves hard gates that autonomous bots cannot reliably clear themselves.

It does **not**:

- grant cloth
- grant finished bandages
- directly raise First Aid skill
- bypass all trainer progression
- make crafting instant

The normal PlayerBots systems still handle recipe use, crafting, skill-ups, and choosing the best usable bandage.

The Artisan step deliberately skips the Triage quest but uses the original teaching spell so the core applies the normal First Aid spell and skill-step behavior.

### Full inventory behavior

Before inserting a manual, the helper checks `CanStoreNewItem`. If the inventory is full, it returns false and does not force the item into the inventory. The maintenance action can try again later.

## Deploy/runtime integration change

PlayerBots config loading originally failed because:

- the module was compiled with `SYSCONFDIR=../etc/`
- the running `mangosd` process had cwd `/home/cmangos`
- therefore it looked for `/home/etc/aiplayerbot.conf`
- the real runtime config was mounted at `/opt/cmangos/config/aiplayerbot.conf`

The runtime image now creates:

```text
/home/etc -> /opt/cmangos/config
```

This fixed PlayerBots startup without hardcoding Docker-specific paths into PlayerBots source.

## Current build integration

The custom Dockerfile now:

1. clones the pinned upstream CMaNGOS core revision,
2. clones `Gohan4711/playerbots` at pinned commit `2adf08264f5e387c598713595c7942a29f90a21e`,
3. copies that exact PlayerBots tree into the core modules directory,
4. forces CMake `FetchContent` to use the pinned in-tree PlayerBots source,
5. compiles the combined image.

No generated PlayerBots patch is required in the current build path.
