# How to Check for Missing Maps on Self-Hosted Server

Run this after any game update to find maps that are accessible from the Overland
but not yet registered in your battlegroup.

## Prerequisites

```bash
NS=funcom-seabass-sh-YOURSLUG-aecbuo

DB_POD=$(kubectl get pod -n $NS -l igw.funcom.com/role=database \
  -o jsonpath='{.items[0].metadata.name}')

OVERMAP_POD=$(kubectl get pod -n $NS -l igw.funcom.com/map=Overmap \
  -o jsonpath='{.items[0].metadata.name}')
```

## Step 1 — Get all maps the Overland server can route to

```bash
kubectl exec -n $NS $OVERMAP_POD -- \
  grep "Register TravelDestination" \
  /home/dune/server/DuneSandbox/Saved/Logs/DuneSandbox_PIDX-2.log \
  | sed "s/.*TravelDestination(\([^)]*\)) for Map(\([^)]*\)).*/\2/" \
  | sort -u
```

This lists the display-name of every map the Overland has a portal to.

## Step 2 — Get all maps currently registered in the DB

```bash
kubectl exec -n $NS $DB_POD -- \
  psql -U dune -d dune -h 127.0.0.1 -p 15432 -t \
  -c "SELECT upgrade_map_name(map) FROM world_partition ORDER BY partition_id"
```

## Step 3 — Compare

Any name from Step 1 that does not appear in Step 2 is a missing map.

To look up the internal `CB_*` map name needed for the battlegroup patch:

```bash
kubectl exec -n $NS $DB_POD -- \
  psql -U dune -d dune -h 127.0.0.1 -p 15432 -t \
  -c "SELECT downgrade_map_name('DisplayNameHere')"
```

Then follow the patch procedure in the relevant fix file, or use the general
template below.

## General patch template for any missing map

```bash
BG=sh-YOURSLUG-aecbuo
MAP=CB_InternalMapName        # from downgrade_map_name()
DISPLAY=DisplayNameHere       # from Step 1
NEXT_ID=32                    # SELECT MAX(partition_id)+1 FROM world_partition
MEMORY=2Gi                    # 3Gi for CB_Overland_*, 2Gi for everything else

TOKEN=$(kubectl get battlegroup $BG -n $NS \
  -o jsonpath="{.spec.serverGroup.template.spec.sets[0].arguments[1]}" \
  | sed "s/-ini:engine:\[FuncomLiveServices\]:ServiceAuthToken=//")

kubectl patch battlegroup $BG -n $NS --type=json -p "[
  {
    \"op\": \"add\",
    \"path\": \"/spec/database/template/spec/deployment/spec/worldPartitions/-\",
    \"value\": {
      \"map\": \"$MAP\",
      \"partitions\": [{\"dimension\": 0, \"disable\": false, \"id\": $NEXT_ID,
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
      \"map\": \"$MAP\",
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
      \"resources\": {\"limits\": {\"memory\": \"$MEMORY\"}},
      \"restartMode\": \"Individual\",
      \"runCommand\": \"/home/dune/run.sh\",
      \"schedulerName\": \"memory-focused-scheduler\",
      \"storageMode\": \"Combined\",
      \"terminationGracePeriodSeconds\": 120
    }
  }
]"
```

Then add the map to director.ini and restart the director:

```bash
python3 - <<PYEOF
import subprocess, json
NS, BG, MAP = "$NS", "$BG", "$MAP"
r = subprocess.run(["kubectl","get","battlegroup",BG,"-n",NS,"-o","json"], capture_output=True, text=True)
bg = json.loads(r.stdout)
ini = bg["spec"]["utilities"]["director"]["spec"]["configFiles"]["files"]["director.ini"]
if MAP in ini:
    print("Already present")
else:
    patch = json.dumps([{"op":"replace","path":"/spec/utilities/director/spec/configFiles/files/director.ini","value":ini+f"\n\n[ {MAP} ]\nNumExtraServers = 0"}])
    r2 = subprocess.run(["kubectl","patch","battlegroup",BG,"-n",NS,"--type=json","-p",patch], capture_output=True, text=True)
    print(r2.stdout.strip() or r2.stderr.strip())
PYEOF

kubectl rollout restart deployment ${BG}-bgd-deploy -n $NS
```
