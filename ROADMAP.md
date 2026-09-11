# Roadmap

This roadmap tracks the custom CMaNGOS Classic + PlayerBots project in practical execution order.

## 1. Runtime verification

### First Aid

Test the new helper in-game at the exact hard-gate thresholds:

- 125 -> Expert First Aid book appears and is used
- 180 -> Heavy Silk Bandage manual appears and is used
- 210 -> Mageweave Bandage manual appears and is used
- 225 + level 35 -> Artisan First Aid teaching spell applies correctly

Also verify:

- full inventory does not lose/force items
- normal crafting continues after each gate
- bots eventually progress to Runecloth / Heavy Runecloth bandages through trainers/crafting

### LLM

Re-test on the latest direct-build image:

- named `/say` routes to the named bot only
- unnamed `/say` does not cause a bot pile-on
- whispers still work
- Alliance normal LLM paths remain blocked
- combat state/current target reach the prompt correctly
- bot-to-bot behavior respects configured chance and proximity rules

## 2. Remove patch-based PlayerBots build — COMPLETE

Completed on 2026-09-11:

- Docker now clones `Gohan4711/playerbots` directly
- build pins custom commit `2adf08264f5e387c598713595c7942a29f90a21e`
- no generated PlayerBots patch is applied
- `docker/server/llm-status-context.patch` has been removed from the active deploy tree
- the direct build completed successfully
- the recreated `mangosd` container was verified to use image `sha256:bb189b65bb583e6cdf9dd1866ada1b56089b674aed071c61e3551425b62d82df`
- PlayerBots initialized successfully from that image

## 3. Profession assignment system

Goal: bots begin naturally, level normally, and end up with a believable server-wide profession mix instead of every bot converging on the same choices.

Initial target for roughly 500 active bots:

- Mining + Blacksmithing: ~18%
- Mining + Engineering: ~12%
- Skinning + Leatherworking: ~18%
- Herbalism + Alchemy: ~18%
- Tailoring + Enchanting: ~18%
- Mining + Skinning: ~6%
- Herbalism + Skinning: ~5%
- Mining + Herbalism: ~5%

Approximate gathering coverage:

- Mining: ~205 bots
- Skinning: ~145 bots
- Herbalism: ~140 bots

Design principles:

- stable assignment per bot
- class may influence probability, but no rigid min-max mapping
- respect the core two-primary-profession limit
- avoid auction-house dependency for mandatory progression
- solve hard gates only where natural bot behavior cannot

## 4. Profession hard-gate audit

Audit each profession end-to-end from a fresh low-level bot.

### Known areas to inspect

- profession tool acquisition
- blacksmith hammer / mining pick / skinning knife
- engineering tools
- enchanting rods, especially higher ranks
- trainer travel and rank transitions
- specialization requirements
- recipes that require quests, books, vendors, or unusual travel

The existing inventory factory can provide some profession tools during full random-bot initialization, but that is not enough to assume normal level-1 progression is safe.

## 5. Secondary professions

After primary professions are stable:

- Cooking progression audit
- Fishing progression audit
- remaining First Aid edge cases

Use the same philosophy: preserve normal gameplay and only bridge progression gates that autonomous bots cannot reasonably solve.

## 6. Scale and behavior test

Target: roughly 500 active autonomous bots without turning starter zones or chat into noise.

Validate:

- CPU/memory impact
- login/logout churn
- travel behavior
- profession distribution in practice
- economy/resource pressure
- LLM request volume
- bot-to-bot LLM disabled or tightly controlled for scale

## 7. Documentation and release hygiene

Keep these files current after meaningful checkpoints:

- `PROJECT_STATUS.md`
- `BUILD_AND_RUN.md`
- `CUSTOM_CHANGES.md`
- `ROADMAP.md`

For each known-good milestone, record:

- core commit
- PlayerBots commit
- deploy commit
- image tag / digest when useful
- tests actually performed
- tests still pending

Do not label something as verified merely because it compiles; distinguish build verification from in-game runtime verification.
