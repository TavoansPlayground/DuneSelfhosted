# Fix: WindPass (CB_Overland_S_08) Not Loading on Self-Hosted Server

## Symptom

Players cannot travel to the **WindPass** overland location. The map never loads, or
the server spins up and immediately exits (~5 seconds after becoming ready).

## Root Cause

WindPass maps to the internal map name **`CB_Overland_S_08`**. Funcom shipped a DB
upgrade script (`DA-11609_add_map_WindPassserver_maps_setup.sql`) to add it, but the
battlegroup CRD was created before that migration ran, so the map was never registered.

Three things are missing on affected deployments:

| What | Where |
|---|---|
| `CB_Overland_S_08` world-partition entry | `world_partition` DB table |
| `CB_Overland_S_08` server set | BattleGroup CRD `serverGroup.sets` |
| `CB_Overland_S_08` in director config | BattleGroup CRD `utilities.director` `director.ini` |

## Prerequisites

- `kubectl` access to the server host
- Know your namespace — substitute `$NS` below with yours (e.g.
  `funcom-seabass-sh-7d7f67607ffcf8f2-aecbuo`)
- Know your battlegroup name — substitute `$BG` (usually same slug as the namespace
  suffix, e.g. `sh-7d7f67607ffcf8f2-aecbuo`)
- Know the exact `ServiceAuthToken` JWT already in your battlegroup (copy it from an
  existing server set — all sets share the same token)

```bash
NS=funcom-seabass-sh-YOURSLUG-aecbuo
BG=sh-YOURSLUG-aecbuo
TOKEN=eyJhbGci...   # copy from existing set in: kubectl get battlegroup $BG -n $NS -o yaml
```

## Step 1 — Verify the problem

```bash
kubectl exec -n $NS $(kubectl get pod -n $NS -l igw.funcom.com/role=database \
  -o jsonpath='{.items[0].metadata.name}') -- \
  psql -U dune -d dune -h 127.0.0.1 -p 15432 -t \
  -c "SELECT map, label FROM world_partition WHERE map='CB_Overland_S_08'"
```

If this returns **no rows**, the partition is missing and you need the fix below.

## Step 2 — Patch the BattleGroup CRD

This single `kubectl patch` command adds the world-partition entry **and** the server
set in one operation. The operator reconciles immediately and inserts the DB row.

Replace `$TOKEN` with the JWT from your battlegroup before running.

```bash
kubectl patch battlegroup $BG -n $NS --type=json -p "[
  {
    \"op\": \"add\",
    \"path\": \"/spec/database/template/spec/deployment/spec/worldPartitions/-\",
    \"value\": {
      \"map\": \"CB_Overland_S_08\",
      \"partitions\": [{\"dimension\": 0, \"disable\": false, \"id\": 29,
                        \"maxX\": 1, \"maxY\": 1, \"minX\": 0, \"minY\": 0}]
    }
  },
  {
    \"op\": \"add\",
    \"path\": \"/spec/serverGroup/template/spec/sets/-\",
    \"value\": {
      \"arguments\": [
        \"-FarmRegion=Europe Test\",
        \"-ini:engine:[FuncomLiveServices]:ServiceAuthToken=$TOKEN\",
        \"-RMQGameTlsEnabled=true\"
      ],
      \"connectDirector\": false,
      \"connectionMode\": \"AutoDiscovery\",
      \"dedicatedScaling\": true,
      \"envSecretSelectors\": [{\"matchLabels\": {\"igw.funcom.com/env-for-server\": \"default\"}}],
      \"ignoreUnreadyDatabase\": false,
      \"image\": \"registry.funcom.com/funcom/self-hosting/seabass-server:1960494-0-shipping\",
      \"map\": \"CB_Overland_S_08\",
      \"messageQueues\": [
        {\"argumentNames\": {\"hostname\": \"--RMQGameHostname\", \"port\": \"--RMQGamePort\"},
         \"mode\": \"Internal\", \"port\": \"amqp\",
         \"serviceLabelSelector\": {\"matchLabels\": {\"messagequeue\": \"game\"}}},
        {\"argumentNames\": {\"hostname\": \"--RMQAdminHostname\", \"port\": \"--RMQAdminPort\"},
         \"mode\": \"Internal\", \"port\": \"amqp\",
         \"serviceLabelSelector\": {\"matchLabels\": {\"messagequeue\": \"admin\"}}}
      ],
      \"readinessPollMode\": \"ServerStats\",
      \"replicas\": 0,
      \"resources\": {\"limits\": {\"memory\": \"3Gi\"}},
      \"restartMode\": \"Individual\",
      \"runCommand\": \"/home/dune/run.sh\",
      \"schedulerName\": \"memory-focused-scheduler\",
      \"storageMode\": \"Combined\",
      \"terminationGracePeriodSeconds\": 120
    }
  }
]"
```

> **Note on partition `id: 29`** — this assumes your battlegroup has the standard 28
> partitions (ids 1–28). If you have added custom partitions, check the current max
> with `SELECT MAX(partition_id) FROM world_partition` and use `max+1` instead.

## Step 3 — Add CB_Overland_S_08 to director.ini

The director.ini controls on-demand server scaling. Run this Python snippet on the
host to append the new section and patch it back:

```bash
python3 - <<'PYEOF'
import subprocess, json

NS = "funcom-seabass-sh-YOURSLUG-aecbuo"
BG = "sh-YOURSLUG-aecbuo"

r = subprocess.run(
    ["kubectl", "get", "battlegroup", BG, "-n", NS, "-o", "json"],
    capture_output=True, text=True
)
bg = json.loads(r.stdout)
ini = bg["spec"]["utilities"]["director"]["spec"]["configFiles"]["files"]["director.ini"]

if "CB_Overland_S_08" in ini:
    print("Already present — nothing to do.")
else:
    new_ini = ini + "\n\n[ CB_Overland_S_08 ]\nNumExtraServers = 0"
    patch = json.dumps([{
        "op": "replace",
        "path": "/spec/utilities/director/spec/configFiles/files/director.ini",
        "value": new_ini
    }])
    r2 = subprocess.run(
        ["kubectl", "patch", "battlegroup", BG, "-n", NS,
         "--type=json", "-p", patch],
        capture_output=True, text=True
    )
    print(r2.stdout.strip() or r2.stderr.strip())
PYEOF
```

## Step 4 — Restart the BattleGroup Director

The director caches director.ini at startup; restart it to pick up the new section:

```bash
kubectl rollout restart deployment ${BG}-bgd-deploy -n $NS
```

Wait ~20 seconds for it to come back:

```bash
kubectl rollout status deployment/${BG}-bgd-deploy -n $NS
```

## Step 5 — Verify

```bash
# DB row should exist with label WindPass_0
kubectl exec -n $NS $(kubectl get pod -n $NS -l igw.funcom.com/role=database \
  -o jsonpath='{.items[0].metadata.name}') -- \
  psql -U dune -d dune -h 127.0.0.1 -p 15432 -t \
  -c "SELECT partition_id, map, label FROM world_partition WHERE map='CB_Overland_S_08'"

# Map name functions should translate correctly
kubectl exec -n $NS $(kubectl get pod -n $NS -l igw.funcom.com/role=database \
  -o jsonpath='{.items[0].metadata.name}') -- \
  psql -U dune -d dune -h 127.0.0.1 -p 15432 -t \
  -c "SELECT upgrade_map_name('CB_Overland_S_08'), downgrade_map_name('WindPass')"
# Expected:  WindPass  |  CB_Overland_S_08
```

## Expected behaviour after the fix

1. Player travels to WindPass on the Overmap.
2. Gateway finds partition 29 (`WindPass_0`).
3. Director spawns a `CB_Overland_S_08` pod (~30–60 s cold start on first visit).
4. Player is registered as *in transit* — the server sees
   `InGameOrInTransitPlayerCount ≥ 1` and stays alive.
5. Player loads into WindPass.

## Background: why this happened

The game server includes DB upgrade scripts under
`/home/dune/server/DuneSandbox/Database/Upgrade/`. The relevant file is:

```
DA-11609_add_map_WindPassserver_maps_setup.sql
```

This script redefines `initialize_partitions_full_battlegroup()` to include
`CB_Overland_S_08` and updates the `upgrade_map_name` / `downgrade_map_name`
stored functions. On a fresh install the function definitions are updated
automatically, but the operator does **not** re-run partition initialisation for an
existing battlegroup — it only reconciles what the CRD describes. Hence the
partition stays absent until you patch the CRD as above.
