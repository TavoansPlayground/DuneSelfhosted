# Fix: The Old Quarry (CB_Dungeon_ThePit) Not Loading on Self-Hosted Server

## Symptom

Players cannot enter **The Old Quarry** from the Overland map. The map never loads,
or the server spins up and exits ~5 seconds after becoming ready.

## Root Cause

The Old Quarry maps to the internal name **`CB_Dungeon_ThePit`** (display label:
`PitDungeon`). It was added to the `initialize_partitions_full_battlegroup()` stored
function by Funcom's `DA-11609` upgrade script, but the operator does not re-run
partition initialisation for an existing battlegroup, so the partition stays absent
until the CRD is patched manually.

Confirmed via Overland server travel-destination log:
```
Register TravelDestination(Overland_to_OldQuarry) for Map(PitDungeon)
```

## Prerequisites

```bash
NS=funcom-seabass-sh-YOURSLUG-aecbuo
BG=sh-YOURSLUG-aecbuo

# Copy the JWT from any existing server set
TOKEN=$(kubectl get battlegroup $BG -n $NS \
  -o jsonpath="{.spec.serverGroup.template.spec.sets[0].arguments[1]}" \
  | sed "s/-ini:engine:\[FuncomLiveServices\]:ServiceAuthToken=//")
```

## Step 1 — Verify the partition is missing

```bash
kubectl exec -n $NS \
  $(kubectl get pod -n $NS -l igw.funcom.com/role=database \
    -o jsonpath='{.items[0].metadata.name}') -- \
  psql -U dune -d dune -h 127.0.0.1 -p 15432 -t \
  -c "SELECT map, label FROM world_partition WHERE map='CB_Dungeon_ThePit'"
```

No rows = partition is missing, apply the fix below.

## Step 2 — Find the next partition ID

```bash
kubectl exec -n $NS \
  $(kubectl get pod -n $NS -l igw.funcom.com/role=database \
    -o jsonpath='{.items[0].metadata.name}') -- \
  psql -U dune -d dune -h 127.0.0.1 -p 15432 -t \
  -c "SELECT MAX(partition_id) FROM world_partition"
```

Use `result + 1` as the `id` value in Step 3 (standard install = **30**).

## Step 3 — Patch the BattleGroup CRD

```bash
kubectl patch battlegroup $BG -n $NS --type=json -p "[
  {
    \"op\": \"add\",
    \"path\": \"/spec/database/template/spec/deployment/spec/worldPartitions/-\",
    \"value\": {
      \"map\": \"CB_Dungeon_ThePit\",
      \"partitions\": [{\"dimension\": 0, \"disable\": false, \"id\": 30,
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
      \"map\": \"CB_Dungeon_ThePit\",
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
      \"resources\": {\"limits\": {\"memory\": \"2Gi\"}},
      \"restartMode\": \"Individual\",
      \"runCommand\": \"/home/dune/run.sh\",
      \"schedulerName\": \"memory-focused-scheduler\",
      \"storageMode\": \"Combined\",
      \"terminationGracePeriodSeconds\": 120
    }
  }
]"
```

The operator reconciles immediately and inserts the `world_partition` row.

## Step 4 — Add to director.ini

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

if "CB_Dungeon_ThePit" in ini:
    print("Already present — nothing to do.")
else:
    new_ini = ini + "\n\n[ CB_Dungeon_ThePit ]\nNumExtraServers = 0"
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

## Step 5 — Restart the BattleGroup Director

```bash
kubectl rollout restart deployment ${BG}-bgd-deploy -n $NS
kubectl rollout status  deployment/${BG}-bgd-deploy -n $NS
```

## Step 6 — Verify

```bash
kubectl exec -n $NS \
  $(kubectl get pod -n $NS -l igw.funcom.com/role=database \
    -o jsonpath='{.items[0].metadata.name}') -- \
  psql -U dune -d dune -h 127.0.0.1 -p 15432 -t \
  -c "SELECT partition_id, map, label FROM world_partition WHERE map='CB_Dungeon_ThePit'"
```

Expected: `30 | CB_Dungeon_ThePit | PitDungeon_0`

## Expected behaviour after the fix

1. Player uses the Overland portal to The Old Quarry.
2. Gateway finds partition 30 (`PitDungeon_0`).
3. Director spawns a `CB_Dungeon_ThePit` server pod (~30–60 s cold start on first visit).
4. Player is registered as *in transit* — server stays alive and player loads in.
