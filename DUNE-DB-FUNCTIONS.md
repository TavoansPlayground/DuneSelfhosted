# Dune Awakening DB — Notable Functions Reference

Discovered by inspecting the live `dune` schema on a self-hosted dedicated server running the Funcom battleground operator on k3s.
Cross-referenced against a PTC backup where noted.
All functions live in the `dune` schema. Connect via the SSH tunnel on port 15432.

**Connecting:**
```powershell
# From Windows, tunnel first:
ssh -i ~/.ssh/id_ed25519 -L 15432:localhost:15432 dune@<server-ip>
# Then in another terminal (requires psql locally, or use dune-admin's raw SQL tab):
psql -h localhost -p 15432 -U dune -d dune
```

**Getting player identifiers** (needed by most functions):
```sql
-- FLS ID, account_id, actor_id (pawn), controller_id all in one:
SELECT
    convert_from(e.encrypted_funcom_id, 'UTF8') AS fls_id,
    e.id                                         AS account_id,
    a.id                                         AS actor_id,
    ps.player_controller_id                      AS controller_id,
    ps.character_name,
    ps.online_status
FROM dune.actors a
JOIN dune.encrypted_accounts e ON e.id = a.owner_account_id
JOIN dune.player_state ps ON ps.account_id = e.id
WHERE a.class ILIKE '%PlayerCharacter%'
ORDER BY ps.character_name;
```

---

## Account Takeover / Character Transfer

Three functions form a full character-transfer pipeline. Funcom uses this internally to move characters between Funcom accounts — useful for us if a player loses access to their Funcom account and creates a new one but wants to keep their character.

### `set_account_as_takeoverable(old_fls_id text, new_fls_id text)`
Reassigns the `encrypted_accounts.user` field to a new FLS ID and sets `takeoverable = true`. Step 1 of a transfer. The new FLS ID must already exist as a registered but empty account (i.e. the player must have logged in at least once with the new account so a row exists).

```sql
SELECT dune.set_account_as_takeoverable('old-fls-id', 'new-fls-id');
```

> **Guess:** The intended flow is: player logs in once with new account → row created → admin calls `set_account_as_takeoverable` → player calls `takeover_account` from the game client, or admin calls it directly. The `takeoverable` flag may be what the game client checks before showing a "claim character" UI option.

### `can_takeover_account(fls_id text) → boolean`
Sanity check — returns whether the account is currently marked as takeover-eligible. Run before calling `takeover_account` to confirm the flag is set.

```sql
SELECT dune.can_takeover_account('new-fls-id');
-- Should return true before proceeding
```

### `takeover_account(user_to_takeover text, current_user text)`
Full account swap — exchanges `encrypted_funcom_id` between two `encrypted_accounts` rows. Character data stays in place; only the FLS identity is swapped. After this call, the character previously owned by `user_to_takeover` is now accessible to `current_user`.

```sql
SELECT dune.takeover_account('old-fls-id', 'new-fls-id');
```

**Full workflow for transferring a character to a new account:**
```sql
-- 1. Verify new account exists (player must have logged in once):
SELECT id FROM dune.encrypted_accounts WHERE convert_from(encrypted_funcom_id, 'UTF8') = 'new-fls-id';

-- 2. Mark old account as transferable to new FLS ID:
SELECT dune.set_account_as_takeoverable('old-fls-id', 'new-fls-id');

-- 3. Confirm the flag is set:
SELECT dune.can_takeover_account('new-fls-id');

-- 4. Execute the swap:
SELECT dune.takeover_account('old-fls-id', 'new-fls-id');
```

> **Warning:** No undo. Take a DB backup before running. Both accounts must exist in `encrypted_accounts`.

---

## Character Management

### `set_character_name(account_id bigint, name text)`
Renames a character. Writes to `encrypted_player_state` using `encrypt_user_data()` — names are stored encrypted at rest. Player must relog for the new name to appear in-game.

```sql
-- Get account_id from the reference query at the top, then:
SELECT dune.set_character_name(<account_id>, 'NewName');
```

> **Guess:** There is likely no uniqueness check in this function — it just writes the name. If the game enforces unique names at login time it may conflict, but since this is a private server with a small population it's probably fine. Worth testing on a non-peak time.

### `delete_character(actor_id bigint)`
Hard-deletes a character actor and associated rows (properties, FGL entities, transforms). Irreversible. Does **not** delete inventory, items, buildings — those are linked via `actor_id` FK and will orphan. Only use this if you're also cleaning up the related tables, or if the character never had any world presence.

```sql
SELECT dune.delete_character(<actor_id>);
```

> **Guess:** For a clean full character wipe, `delete_account` is likely safer as it cascades via FK constraints. `delete_character` appears to be a lower-level tool that Funcom's migration scripts call after already handling inventory/building cleanup separately.

### `delete_account(account_id bigint)`
Removes a full account and cascades via FK constraints across most player tables. Intended for full player removal. The player would need to log back in from scratch if they ever return (new character, no progress).

```sql
SELECT dune.delete_account(<account_id>);
```

> **Warning:** This is irreversible and broad. Confirm the `account_id` twice before running. Does not remove world-placed structures (buildings, totems) — those are owned by actors, not accounts directly. You may need to separately clean up orphaned structures.

---

## Journey / Story Progression

### `complete_journey_story_nodes_for_player(fls_id text, node_ids text[])`
Bulk-completes a list of journey story nodes for a player by FLS ID. Has a built-in offline check — raises an exception if the player is online. More surgical than dune-admin's per-node approach since you pass an array.

```sql
-- Player must be offline. Get their FLS ID from the reference query.
SELECT dune.complete_journey_story_nodes_for_player(
    'player-fls-id',
    ARRAY['Journey.Act1.NodeA', 'Journey.Act2.NodeB']
);
```

> **Guess on node IDs:** Node ID strings are visible in `player_tags` rows that start with `Journey.` and `Contract.Tracking.`. The pattern appears to be the tag name itself, but without `complete_condition_state` as a prefix. Query `player_tags` for a player who has already completed the quest to see what the completed state looks like, then use those node IDs for players who are stuck.

```sql
-- Find all journey nodes a player has completed (for reference):
SELECT tag FROM dune.player_tags
WHERE account_id = <account_id>
AND tag LIKE 'Journey.%'
ORDER BY tag;
```

### `reveal_journey_story_nodes_for_player(fls_id text, node_ids text[])`
Sets `reveal_condition_state = true` for the given nodes without marking them complete. Useful for making a quest appear in the journal for a player who should have unlocked it but didn't.

```sql
SELECT dune.reveal_journey_story_nodes_for_player(
    'player-fls-id',
    ARRAY['Journey.Act2.AssassinsHandbook']
);
```

### `reset_journey_story_nodes_for_player(fls_id text, node_ids text[])`
Resets listed nodes to incomplete — clears `complete_condition_state` and `has_pending_reward`. Use when a player wants to replay a quest or when a node got erroneously auto-completed.

```sql
SELECT dune.reset_journey_story_nodes_for_player(
    'player-fls-id',
    ARRAY['Journey.Act1.NodeA']
);
```

> **Guess:** Resetting a node doesn't remove the associated `player_tags` that were set as a consequence of completing it. If the quest completion set tags like `DialogueFlags.Quest.XCompleted`, those remain. For a truly clean re-run you'd also need to remove those tags via `update_player_tags`.

### `fix_broken_harkonnen_players_due_to_fooled_thufir()`
A shipped hotfix (DA-6358) for a specific Harkonnen story bug where the `FooledThufir` dialogue tag combined with `Hark_ThufirBetrayedComplete` at Tier 4/5 permanently broke the questline. Handles three sub-cases across build versions (pre-1.2.40, 1.2.40, 1.3.0):
- Removes `DialogueFlags.Faction.FooledThufir` tag
- Force-promotes affected players to Harkonnen Tier 5 (2000 rep)
- Backs up affected actor rows into temp tables (`da_6358_*`) before patching

```sql
-- Safe to run at any time — only affects players matching the broken tag combo:
SELECT dune.fix_broken_harkonnen_players_due_to_fooled_thufir();

-- Check if anyone was affected:
SELECT * FROM dune.da_6358_broken_players_12400;
SELECT * FROM dune.da_6358_broken_players_1300;
SELECT * FROM dune.da_6358_pre_broken_players;
```

> **Key insight:** This function reveals that `player_tags` is the canonical source of truth for all story/dialogue/faction state. Diagnosing any stuck quest should start with querying `player_tags` for the affected player, not `journey_story_node`.

---

## Player Tags (Story State Engine)

`player_tags` stores every story flag, faction tier, contract state, and dialogue choice as `(account_id, tag text)` rows. It is the most important table for diagnosing progression issues.

### Reading tags
```sql
-- All tags for a player:
SELECT tag FROM dune.player_tags WHERE account_id = <account_id> ORDER BY tag;

-- Tags matching a pattern (e.g. all faction tags):
SELECT tag FROM dune.player_tags
WHERE account_id = <account_id>
AND tag LIKE 'Faction.%'
ORDER BY tag;

-- Using the admin function (same result, cleaner):
SELECT * FROM dune.admin_read_player_tags(<account_id>);
```

### `update_player_tags(account_id bigint, tags_to_add text[], tags_to_remove text[])`
Adds and removes tags atomically. This is the correct way to surgically repair story state.

```sql
-- Add a tag:
SELECT dune.update_player_tags(<account_id>,
    ARRAY['Faction.Harkonnen.Tier5'],
    ARRAY[]::text[]
);

-- Remove a broken tag:
SELECT dune.update_player_tags(<account_id>,
    ARRAY[]::text[],
    ARRAY['DialogueFlags.Faction.FooledThufir']
);

-- Add one, remove one simultaneously:
SELECT dune.update_player_tags(<account_id>,
    ARRAY['Contract.Tracking.FactionStory.R4C6Completed'],
    ARRAY['Contract.Tracking.Active.FactionStory.R4C6']
);
```

> **Guess:** The tag strings are case-sensitive and must match exactly what the game binary writes. Use `player_tags` on a player who has correctly completed the content as a reference for the exact tag names before writing them for another player.

**Common tag prefixes observed:**
| Prefix | Meaning |
|---|---|
| `Journey.` | Main story quest nodes |
| `JourneySets.` | Fremkit / starting gear unlock flags |
| `Faction.Harkonnen.TierN` | Harkonnen tier level |
| `Faction.Atreides.TierN` | Atreides tier level |
| `DialogueFlags.Factions.*` | Dialogue choices within faction stories |
| `Contract.Tracking.Active.*` | Currently active contracts |
| `Contract.Tracking.Completed.*` | Completed contract flags |
| `Exploration.POI.*` | Points of interest discovered |

---

## Landsraad

### `landsraad_force_end_term(term_id bigint)`
Forces the current Landsraad voting cycle to end immediately by setting its end time to `now()`. The Director picks this up on its next polling cycle (likely within seconds).

```sql
-- Step 1: Get the current active term ID:
SELECT term_id, start_time, end_time, reigning_faction_id
FROM dune.landsraad_decree_term
ORDER BY term_id DESC LIMIT 1;

-- Step 2: Force it to end:
SELECT dune.landsraad_force_end_term(<term_id>);
```

> **Guess:** This triggers the Director to run `landsraad_determine_winner`, tally votes, apply the winning decree, and start a new term. If no votes have been cast the outcome will be decided by whatever the default winner logic is (likely the current reigning faction wins by default). Useful for unsticking a Landsraad that got into a bad state, or for testing decree cycling on a fresh world.

### `landsraad_change_term_end_time(term_id bigint, new_end_time timestamp, notify boolean)`
Adjusts the end time of the current term to a specific future timestamp — less aggressive than force-ending. Use this to extend a cycle or schedule it to end at a specific real-world time.

```sql
-- Extend the current term by 7 days from now:
SELECT dune.landsraad_change_term_end_time(
    <term_id>,
    (NOW() AT TIME ZONE 'UTC' + INTERVAL '7 days')::TIMESTAMP,
    false
);
```

### Viewing Landsraad state
```sql
-- Current term, reigning faction, active decree, vote counts:
SELECT
    t.term_id,
    f_reign.name  AS reigning_faction,
    d_active.decree_name AS active_decree,
    t.start_time,
    t.end_time,
    f_win.name    AS winning_faction,
    d_elect.decree_name  AS elected_decree
FROM dune.landsraad_decree_term t
LEFT JOIN dune.factions f_reign ON f_reign.id = t.reigning_faction_id
LEFT JOIN dune.landsraad_decrees d_active ON d_active.id = t.active_decree_id
LEFT JOIN dune.factions f_win ON f_win.id = t.winning_faction_id
LEFT JOIN dune.landsraad_decrees d_elect ON d_elect.id = t.elected_decree_id
ORDER BY t.term_id DESC LIMIT 1;

-- Current vote counts:
SELECT faction_id, COUNT(*) AS votes
FROM dune.landsraad_decree_votes
WHERE term_id = <term_id>
GROUP BY faction_id;
```

---

## World Seeding (Coriolis)

Coriolis drives loot spawns, resource field placement, and environmental reset cycles. Seeds are stored at three levels: farm (global), per-map, and per-partition. Changing a seed takes effect on the next Coriolis reset — it does not immediately respawn the world.

### `debug_get_coriolis_seeds()`
Returns all current seeds in one call — farm seed, every map seed, every partition seed.

```sql
SELECT * FROM dune.debug_get_coriolis_seeds();
```

### `debug_set_farm_seed(seed integer)`
Pushes one seed value to all levels (farm + all maps + all partitions) simultaneously. Effectively reseeds the entire world for the next reset cycle.

```sql
-- Use any integer. Negative values are valid (-1 is the "unset" sentinel).
SELECT dune.debug_set_farm_seed(99999);
```

### `debug_set_map_seed(map text, seed integer)`
Reseeds a single map. Map name must match the `map` column in `world_partition` exactly.

```sql
-- Get valid map names:
SELECT DISTINCT map FROM dune.world_partition ORDER BY map;

-- Reseed one map:
SELECT dune.debug_set_map_seed('CB_Overland_S_01', 42);
```

### `debug_set_partition_seed(partition_id bigint, seed integer)`
Reseeds a single partition. Most granular level.

```sql
-- Get partition IDs:
SELECT partition_id, map FROM dune.world_partition ORDER BY partition_id;

-- Reseed partition 5:
SELECT dune.debug_set_partition_seed(5, 12345);
```

> **Guess on what reseeding affects:** Loot node contents, resource field positions, NPC/encounter spawn variants, and shifting sand patterns are all seeded. Player bases, structures, inventory, and character state are NOT seeded — those are stored explicitly. A reseed should be safe to run live without harming player progress. The Coriolis cycle timer determines when the new seed actually takes effect; you can force the cycle by restarting the relevant server pod, though this is untested.

> **Guess on finding a "good" seed:** Run `debug_get_coriolis_seeds()` on a fresh world before players build anything, record that seed, and you can restore that loot distribution later with `debug_set_farm_seed()`.

---

## Server Lifecycle

### `set_battlegroup_close_date(close_date timestamp)`
Schedules a graceful server shutdown. The Director reads `farm_variables.battlegroup_close_date` and begins its wind-down sequence at the specified UTC time. Returns the stored timestamp as confirmation.

```sql
-- Schedule a shutdown for June 1st at 20:00 UTC:
SELECT dune.set_battlegroup_close_date('2026-06-01 20:00:00');

-- Check what's currently scheduled:
SELECT dune.get_battlegroup_close_date();

-- Cancel a scheduled shutdown:
UPDATE dune.farm_variables SET battlegroup_close_date = NULL WHERE one_row = true;
```

> **Guess on wind-down behaviour:** Based on the function name and Director's role, this likely triggers a server announcement to connected players (e.g. "Server closing in X minutes") followed by a clean shutdown rather than an immediate kill. The exact warning interval is unknown — it's likely configurable in the Director config, not the DB.

> **Use case:** Scheduling game-update maintenance windows. Set the close date, patch the server, then restart. Players get a graceful warning rather than a sudden disconnect.

---

## Economy

### `migrate_clamp_max_allow_solaris(pawn_id bigint, max_solaris bigint)`
Collapses all `SolarisCoin` item stacks in a player's backpack into a single stack capped at `max_solaris`. Funcom shipped this as a migration tool for enforcing a Solaris cap. Also useful for cleaning up duped coins.

```sql
-- Cap a player's Solaris at 500,000:
SELECT dune.migrate_clamp_max_allow_solaris(<actor_id>, 500000);

-- Check current Solaris stacks first:
SELECT id, stack_size FROM dune.items i
JOIN dune.inventories inv ON i.inventory_id = inv.id
WHERE inv.actor_id = <actor_id>
AND i.template_id = 'SolarisCoin';
```

> **Guess:** `pawn_id` here is the actor ID (PlayerCharacter pawn), same as what the reference query at the top returns as `actor_id`. The function only touches `inventory_type = 0` (backpack). Solaris in other inventory slots (if any exist) would not be affected. Player should be offline when this runs.

### Taxation system
Building taxes are fully implemented in the DB. Invoices accrue against totems, and payment/removal fires `pg_notify('taxation_notify_channel', ...)` so the game server updates in real time without a relog.

```sql
-- View all tax invoices for a player (by controller_id):
SELECT * FROM dune.taxation_get_all_invoices_for_player(<controller_id>);

-- View all invoices for a specific totem:
SELECT * FROM dune.taxation_get_all_invoices_for_server(<totem_id>);

-- Force-pay an invoice (frees the base from tax debt):
-- invoice_status values are smallints; 1 is likely "paid" based on typical patterns
SELECT dune.taxation_pay_invoice(<invoice_id>, 1);

-- Wipe all invoices for a totem (clean slate, no payment required):
SELECT dune.taxation_remove_invoices_from_totem(<totem_id>);

-- Wipe all invoices globally (use with caution — resets tax state server-wide):
-- No single function for this; run per-totem or directly:
DELETE FROM dune.tax_invoice WHERE totem_id = <totem_id>;
```

> **Guess on invoice_status values:** The `invoice_status` smallint likely maps to an enum in the game binary. Common patterns suggest 0 = unpaid, 1 = paid, 2 = cancelled/forgiven. Calling `taxation_pay_invoice(<id>, 1)` is the safe bet. Avoid writing values you haven't confirmed — the `pg_notify` fires regardless and may confuse the server if the status value is unexpected.

> **Use case:** A player's base is being taxed into demolition and they've been offline. You can force-pay their outstanding invoices to save the base until they return.

---

## Dungeons

### `delete_all_dungeon_completions_for_all_dungeons_by_player(player_id bigint, keep_for_others boolean)`
Wipes a player's dungeon completion history. The `keep_for_others` flag is important:
- `true` — removes only this player's record from shared runs. Groupmates keep their completions.
- `false` — nukes the entire run record, removing it for every player who was in the same group.

```sql
-- Safe wipe — only affects this player, not their groupmates:
SELECT dune.delete_all_dungeon_completions_for_all_dungeons_by_player(<player_id>, true);

-- Nuclear option — removes the run record for everyone in the group:
SELECT dune.delete_all_dungeon_completions_for_all_dungeons_by_player(<player_id>, false);
```

> **Guess on use cases:**
> - Player had a bugged dungeon run that's blocking rewards → wipe with `true`, let them rerun
> - Entire group got a bugged completion nobody could claim → wipe with `false`, group reruns together
> - Resetting dungeon leaderboard for a specific player on a fresh season

> **Guess:** `player_id` here is `player_controller_id` (the controller actor ID), not `account_id`. Use the reference query at the top to get the right value.

---

## Returning Player Awards

The server tracks when a player last logged in and when they last received a "welcome back" reward. These functions let you manually control that state.

### `update_returning_player_status(fls_id text, min_seconds integer)`
Called by the server on login to determine if a player qualifies for a returning player award. You can call it manually to force-evaluate a player's eligibility right now.

```sql
-- Evaluate with a 7-day (604800 second) absence threshold:
SELECT dune.update_returning_player_status('player-fls-id', 604800);
```

> **Guess:** If the player's `last_login_time` is more than `min_seconds` ago AND they haven't received an award recently, this sets `last_returning_player_event_time = now()`, which the server reads on next login to trigger the award popup. You could use this to manually flag a player for a returning reward even if the server hasn't evaluated them yet.

### `returning_player_award_given(account_id bigint)`
Marks the award as claimed — sets `last_returning_player_awarded_time = now()` and clears `last_returning_player_event_time`. Call this after manually giving someone a returning player gift via dune-admin to prevent the server from issuing a duplicate on their next login.

```sql
SELECT dune.returning_player_award_given(<account_id>);
```

> **Guess on triggering the award for a specific player:**
> ```sql
> -- 1. Force-flag them as eligible (0 seconds = always qualifies):
> SELECT dune.update_returning_player_status('player-fls-id', 0);
> -- 2. Give them the items manually via dune-admin
> -- 3. Mark the award as given to prevent double-grant:
> SELECT dune.returning_player_award_given(<account_id>);
> ```

---

## PTC Progress Unlock — Investigation Finding

**The progress unlock tool was NOT DB-driven.** Confirmed by comparing a PTC world backup against the live world.

| Table | PTC backup | Live world | Conclusion |
|---|---|---|---|
| `demo_users` | Empty | Empty | Never used |
| `player_access_codes` | Empty | Empty | Never used |
| `farm_variables` | Farm state only | Farm state only | No feature flags |

The `demostate` enum (`Demo`, `DbMigratedToRetail`, `Retail`) and `demo_users` table exist in the schema but were never populated — even during PTC. The unlock tool was almost certainly a command-line argument in the battlegroup ServerSet `arguments` array that was present during the PTC world and not carried forward to the live world spec.

**Why we can't recover it:** The PTC battlegroup CRD was deleted without being archived. The arg name is unknown.

**What to try if someone finds the arg name:** Add it to the `arguments` list for every ServerSet in the battleground CRD and redeploy:
```bash
sudo kubectl edit battleground <world-id> -n <namespace>
# Add the arg to each set's arguments array, save, operator will rolling-restart pods
```

---

## Quick Reference — Getting IDs

```sql
-- Everything you need for a player by character name:
SELECT
    convert_from(e.encrypted_funcom_id, 'UTF8') AS fls_id,
    e.id          AS account_id,
    a.id          AS actor_id,
    ps.player_controller_id AS controller_id,
    ps.character_name,
    ps.online_status::text,
    a.map
FROM dune.actors a
JOIN dune.encrypted_accounts e ON e.id = a.owner_account_id
JOIN dune.player_state ps ON ps.account_id = e.id
WHERE a.class ILIKE '%PlayerCharacter%'
AND ps.character_name ILIKE '%<name>%';

-- Get a totem_id for a base by base name:
SELECT pa.actor_id AS totem_id, pa.actor_name
FROM dune.permission_actor pa
WHERE pa.actor_name ILIKE '%<base name>%';

-- Get current Landsraad term_id:
SELECT term_id FROM dune.landsraad_decree_term ORDER BY term_id DESC LIMIT 1;

-- Get all partition IDs and maps:
SELECT partition_id, map FROM dune.world_partition ORDER BY partition_id;
```

---

*Last updated: 2026-05-22*
