# Probably deprecated : See DUNE-DB-FUNCTIONS.md
# Character Relink: Restore an old character from a backup



Restores a player's character from an old-server DB backup into a fresh-install world,
preserving all progress, inventory, buildings, and land claims.

**The approach:** While the fresh DB is still up, note the IDs it assigned to the player.
Then drop the fresh DB, restore the backup as the main DB, and update the old IDs to match.
No cross-DB copying, no Python scripts — just SQL.

**When you need this:**
- You're migrating players from an old world into a fresh install
- A player wants their old character from before a world reset
- You rolled the DB back and players need their pre-rollback characters restored

---

## Tools

### On your admin machine (Windows)

| Tool | Purpose | Get it |
|------|---------|--------|
| **SSH client** | Connect to your server | Built into Windows 10+, or use PuTTY |

### On the server

Everything is already present:
- `psql`, `pg_restore` — available inside the DB pod
- `sudo kubectl` — installed with k3s
- DB accessible at `localhost:15432` on the host and port `15432` inside the pod — no port-forwarding needed

---

## Finding the DB pod

The pod name includes the world ID and changes if the world is recreated. Run this at the
start of each session:

```bash
NS=funcom-seabass-sh-<hostid>-<worldid>   # check with: sudo kubectl get ns | grep funcom-seabass
DB_POD=$(sudo kubectl get pods -n $NS -o name | grep db-dbdepl-sts | cut -d/ -f2)
echo $DB_POD
```

Opening a shell inside the pod:
```bash
sudo kubectl exec -it -n $NS $DB_POD -- bash
```

Superuser psql (for DROP/CREATE/session_replication_role):
```bash
sudo kubectl exec -it -n $NS $DB_POD -- psql -U postgres -p 15432
```

---

## Prerequisites

- The fresh server is installed and running
- The player has logged in at least once on the fresh server (so the DB has assigned them IDs)
- You have the `.backup` file from the old server
  (e.g. `/home/dune/backups/<date>/<backup>.backup`)
- You have the player's **Steam ID** (64-bit)
  — ask the player, or look it up at [steamid.io](https://steamid.io) (paste their profile URL)

---

## Step 1 — note the player's fresh IDs

While the fresh DB is still up, open a psql shell and run:

```sql
SELECT
  ea.id                    AS account_id,
  eps.player_controller_id AS ctrl_id,
  eps.player_pawn_id       AS pawn_id,
  eps.player_state_id      AS state_id
FROM dune.encrypted_accounts ea
JOIN dune.encrypted_player_state eps ON eps.account_id = ea.id
WHERE ea.platform_id = '<STEAM_ID>';
```

**Write these four values down.** You need them in Step 4.

---

## Step 2 — stop the server and copy the backup into the pod

```bash
battlegroup stop

sudo kubectl cp /home/dune/backups/<date>/<backup>.backup $NS/$DB_POD:/tmp/restore.backup
```

---

## Step 3 — drop the fresh DB and restore the backup

Open a **superuser** psql shell inside the pod:

```bash
sudo kubectl exec -it -n $NS $DB_POD -- psql -U postgres -p 15432
```

```sql
-- Kick any remaining connections
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'dune' AND pid <> pg_backend_pid();

DROP DATABASE dune;
CREATE DATABASE dune OWNER dune;
\q
```

Restore the backup:
```bash
sudo kubectl exec -n $NS $DB_POD -- \
  pg_restore -U postgres -p 15432 -d dune /tmp/restore.backup
```

This takes 1–5 minutes depending on DB size.

---

## Step 4 — find the player's OLD IDs in the restored backup

```bash
sudo kubectl exec -it -n $NS $DB_POD -- psql -U postgres -p 15432 -d dune
```

```sql
SELECT
  ea.id                    AS old_account_id,
  eps.player_controller_id AS old_ctrl,
  eps.player_pawn_id       AS old_pawn,
  eps.player_state_id      AS old_state
FROM dune.encrypted_accounts ea
JOIN dune.encrypted_player_state eps ON eps.account_id = ea.id
WHERE ea.platform_id = '<STEAM_ID>';
```

You now have both sets of IDs — fresh (from Step 1) and old (from this step).

---

## Step 5 — swap the IDs

Open a **superuser** psql shell, substitute your actual values into the `\set` lines at the
top, and run the whole block:

```sql
SET session_replication_role = replica;  -- disables FK checks so we can update PKs freely

-- ── Fill in your values ──────────────────────────────────────────────
\set old_acct  123     -- old account_id  (from Step 4)
\set new_acct  1       -- new account_id  (from Step 1)
\set old_ctrl  456     -- old ctrl actor
\set new_ctrl  2       -- new ctrl actor
\set old_pawn  457     -- old pawn actor
\set new_pawn  3       -- new pawn actor
\set old_state 458     -- old state actor
\set new_state 4       -- new state actor
-- ─────────────────────────────────────────────────────────────────────

-- Account references (update before changing the PK)
UPDATE dune.encrypted_player_state      SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.actors                      SET owner_account_id = :new_acct WHERE owner_account_id = :old_acct;
UPDATE dune.player_tags                 SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.player_respawn_locations    SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.map_areas                   SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.mnemonic_recall             SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.building_progression        SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.building_favorites          SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.communinet_player           SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.player_access_codes         SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.journey_story_node          SET account_id = :new_acct WHERE account_id = :old_acct;
UPDATE dune.journey_story_node_cooldown SET account_id = :new_acct WHERE account_id = :old_acct;

-- Account PK
UPDATE dune.encrypted_accounts SET id = :new_acct WHERE id = :old_acct;

-- Actor references (update before changing the PKs)
UPDATE dune.actor_state SET actor_id = :new_ctrl  WHERE actor_id = :old_ctrl;
UPDATE dune.actor_state SET actor_id = :new_pawn  WHERE actor_id = :old_pawn;
UPDATE dune.actor_state SET actor_id = :new_state WHERE actor_id = :old_state;

UPDATE dune.actor_fgl_entities SET actor_id = :new_ctrl  WHERE actor_id = :old_ctrl;
UPDATE dune.actor_fgl_entities SET actor_id = :new_pawn  WHERE actor_id = :old_pawn;
UPDATE dune.actor_fgl_entities SET actor_id = :new_state WHERE actor_id = :old_state;

UPDATE dune.encrypted_player_state SET
  player_controller_id = :new_ctrl,
  player_pawn_id       = :new_pawn,
  player_state_id      = :new_state
WHERE account_id = :new_acct;

UPDATE dune.inventories SET actor_id = :new_ctrl  WHERE actor_id = :old_ctrl;
UPDATE dune.inventories SET actor_id = :new_pawn  WHERE actor_id = :old_pawn;
UPDATE dune.inventories SET actor_id = :new_state WHERE actor_id = :old_state;

UPDATE dune.buildings           SET owner_id  = :new_pawn WHERE owner_id  = :old_pawn;
UPDATE dune.permission_actor    SET actor_id  = :new_pawn WHERE actor_id  = :old_pawn;
UPDATE dune.permission_actor_rank SET player_id = :new_pawn WHERE player_id = :old_pawn;

-- Specialization / progress / social tables (keyed on pawn actor)
UPDATE dune.specialization_tracks               SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.purchased_specialization_keystones  SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.player_faction                      SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.player_faction_reputation           SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.consumed_per_player_lore            SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.dialogue_met_npcs                   SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.dialogue_taken_nodes                SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.tutorial_per_player                 SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.player_markers                      SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.overmap_players                     SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.dungeon_completion_players          SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.journey_tracked_cards               SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.guild_members                       SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.landsraad_task_player_contributions SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.landsraad_task_progress_player      SET player_id = :new_pawn WHERE player_id = :old_pawn;
UPDATE dune.landsraad_house_rewards             SET player_id = :new_pawn WHERE player_id = :old_pawn;

-- Actor PKs (last, after all references are fixed)
UPDATE dune.actors SET id = :new_ctrl  WHERE id = :old_ctrl;
UPDATE dune.actors SET id = :new_pawn  WHERE id = :old_pawn;
UPDATE dune.actors SET id = :new_state WHERE id = :old_state;

SET session_replication_role = DEFAULT;
```

---

## Step 6 — verify

```sql
SELECT
  ea.id, ea.platform_id,
  eps.player_controller_id,
  eps.player_pawn_id,
  eps.player_state_id
FROM dune.encrypted_accounts ea
JOIN dune.encrypted_player_state eps ON eps.account_id = ea.id
WHERE ea.platform_id = '<STEAM_ID>';
```

The IDs should now match what you wrote down in Step 1.

---

## Step 7 — clean up and start

Inside the DB pod:
```bash
rm /tmp/restore.backup
```

On the host:
```bash
battlegroup start
battlegroup status
```

The player logs in and finds their old character exactly as it was at the time of the backup.

---

## Notes

- **Buildings and land claims carry over automatically.** They're stored as actors with
  `owner_account_id` — updating that column in Step 5 covers the whole base. No extra steps.
- **Repeat per player.** Do one player at a time: note IDs → swap → verify → next player.
  After the first restore (Step 3), the DB is already in place — subsequent players only need
  Steps 1, 4, and 5.
- **If a table doesn't exist** (e.g. `guild_members`, `landsraad_*` on a vanilla install),
  the UPDATE will fail harmlessly with `relation does not exist`. Just skip those lines.
- **Character names are encrypted** in the DB — always identify players by their Steam ID.
- **If something goes wrong**, restore the backup again (Step 3) and start over. The backup
  file is never modified.
