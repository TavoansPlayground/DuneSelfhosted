# Setup: Dual Deep Desert — one PvP instance, one PvE instance

## What this does

Registers a second Deep Desert dimension (`dimension=1`) alongside the default one
(`dimension=0`). When a player enters Deep Desert they can choose between the two via the
SELECT INSTANCE dialog. The server only spins up whichever instance is actually entered
(`dedicatedScaling=true`), so both instances don't cost RAM simultaneously.

- **Dimension 0 (partition 8 by default)** — PvP enabled
- **Dimension 1 (new partition)** — PvE by default (not listed in `m_PvpEnabledPartitions`)

PvP/PvE routing is controlled by `UserGame.ini`:
```ini
[/Script/DuneSandbox.PvpPveSettings]
m_bShouldForceEnablePvpOnAllPartitions=False
+m_PvpEnabledPartitions=<pvp_partition_id>
; any partition NOT listed here is PvE when ForceEnableAll=False
```

### Known limitation — SELECT INSTANCE Kanly badge

The SELECT INSTANCE dialog shows a "Kanly" column (PvP/PvE icon). For self-hosted servers
this badge is **hardcoded to PvE by Funcom's FLS backend** regardless of any server-side
configuration. It cannot be changed.

- `m_bShouldForceEnablePvpOnAllPartitions=True/False` — no effect on the badge (confirmed)
- The badge is set at world-creation time on Funcom's portal, not by server config

**The actual in-game PvP behaviour IS controlled correctly** by `m_PvpEnabledPartitions`.
Players on a partition listed there can fight each other; players on an unlisted partition
cannot. The badge just cosmetically lies.

Keep `m_bShouldForceEnablePvpOnAllPartitions=False`. Setting it to `True` forces ALL
partitions to full PvP and would break your PvE instance — and it still won't change the badge.

---

## Requirements

- Ubuntu bare-metal or VM with k3s and the Funcom battlegroup operator
- `DeepDesert_1` must already be registered in your battlegroup (it is by default)
- `python3`, `kubectl` available as `sudo kubectl`
- The battlegroup must be **running** (operator needs to accept the patch)

---

## Step 1 — Run the setup script

Save `dual-deepdesert-setup.sh` (see companion file), make executable, run as the `dune` user.
It handles:
1. Adding the dimension=1 partition to the `igwbg` CRD
2. Setting `NumExtraServers = 1` in `director.ini` (via `igwbg`, **not** `igwbgd` — see below)
3. Setting `ServerSetScale` to `replicas: 2` so both pods can run
4. Patching `UserGame.ini` to mark the PvP partition

## Step 2 — Bootstrap player routing (one-time, per player)

After the setup script runs, **new players will be routed correctly automatically**. However,
any player who has previously entered Deep Desert (on dimension=0 only) will be stuck in the
director's "grace period" and always land on partition 8 regardless of which instance they pick.

**Why**: The director's SRVR module derives `HomeDimension` from live server heartbeat history,
not from the database. A player whose only history is dimension=0 has `HomeDimension=0`
permanently until they physically land on partition 34 at least once. The SRVR grace-period
grant overrides the INST module's correct partition-34 selection every time.

**Things that do NOT fix this:**
- Waiting 3+ minutes (HomeDimension is persistent state, not a timer)
- Changing `player_travel_state.login_target_dimension_index` in the DB (director ignores it)
- Abandoning the sietch on partition 8

**The 3-minute rule (ongoing):**

After the bootstrap, SELECT INSTANCE works correctly — but only if you wait ~3 minutes after
leaving one instance before entering a different one. The director issues a 3-minute grace
period grant when you leave any Deep Desert partition. If you enter a different instance within
that window, the grace period overrides SELECT INSTANCE and sends you back to where you just
were. Entering the *same* instance immediately is fine (grace period routes you there anyway).

**The fix — delete pod-8 once (first-time bootstrap only):**

```bash
# Replace with your actual namespace and battlegroup ID
NS=funcom-seabass-sh-<id>-<suffix>
kubectl delete pod <bg>-sg-deepdesert-1-pod-8 -n $NS
```

With pod-8 gone (~45 s respawn), the SRVR module has nothing to fall back to, INST wins, and
the player lands on partition 34. The game then heartbeats `HomeDimension=1` naturally, and
pod-8 comes back on demand. **After this one-time bootstrap, SELECT INSTANCE works correctly
for all future sessions.**

Pod-8 respawns within ~45 seconds automatically — anyone entering Abbir right after it comes
back will route normally.

---

## Critical implementation details

### Patch `igwbg`, NOT `igwbgd`

The Funcom battlegroup operator **reconciles `igwbgd` from the parent `igwbg` CRD**. Any
manual patch to `igwbgd` is silently overwritten by the operator within seconds.

The authoritative `director.ini` lives at:
```
igwbg spec.utilities.director.spec.configFiles.files["director.ini"]
```

The setup script patches this via `--type=merge` on the `igwbg` resource.

### `NumExtraServers = 1` is required

Without this, the director only manages 1 server slot for DeepDesert_1 (the default) and the
SELECT INSTANCE dialog only shows one option. Set it in `director.ini`:

```ini
[ DeepDesert_1 ]
NumExtraServers = 1
MinServers = 0
```

### `ServerSetScale replicas: 2` is required

The `ServerSetScale` CRD controls the **total pod count** across all partitions for a map.
With `partitions: [8, 34]` and `replicas: 1`, only one pod total is created. Set to 2:

```yaml
spec:
  partitions: [8, 34]
  replicas: 2
```

The script patches this with:
```bash
kubectl patch serversetscale <name> -n $NS --type=merge \
  -p '{"spec":{"partitions":[8,34],"replicas":2}}'
```

---

## To revert

Remove the dimension=1 entry from the battlegroup and remove the `m_PvpEnabledPartitions` line
from `UserGame.ini`. Also reset `NumExtraServers` and `ServerSetScale`:

```bash
# Find the index of the dimension=1 partition
NS=funcom-seabass-<id>
BG=sh-<id>

sudo kubectl get igwbg $BG -n $NS -o json | python3 -c "
import json,sys
d=json.load(sys.stdin)
wps=d['spec']['database']['template']['spec']['deployment']['spec']['worldPartitions']
for i,wp in enumerate(wps):
    if wp['map']=='DeepDesert_1':
        for j,p in enumerate(wp['partitions']):
            if p['dimension']==1:
                print(f'Remove path: /spec/database/template/spec/deployment/spec/worldPartitions/{i}/partitions/{j}')
"

# Then patch with op=remove at that path, remove the PvP line from UserGame.ini,
# remove NumExtraServers from director.ini, and set ServerSetScale replicas back to 1
```

---

## Confirmed working on

| Date | Build | Platform | Notes |
|------|-------|----------|-------|
| 2026-05-27 | `1973075-0-shipping` | Ubuntu 24.04 bare-metal, k3s | PvP activates in outer bands (beyond a1-a9) on partition 8; partition 34 fully PvE confirmed |
