# Fix: Server stuck in "Modifying" after update — registry.funcom.com DNS failure

## Symptoms

- You ran `battleground update`
- The server is now stuck in `Modifying` / refuses to start
- `kubectl get pods` shows `ImagePullBackOff` or `ErrImagePull` on MQ and/or DB utility pods
- The error message in pod events says:
  ```
  failed to resolve reference "registry.funcom.com/...": dial tcp: lookup registry.funcom.com: no such host
  ```

## Root cause

`registry.funcom.com` — Funcom's container image registry — has gone NXDOMAIN (doesn't resolve
in DNS, globally, including Google's 8.8.8.8). The update bumped all image tags to the new build
but the new images can't be pulled.

**The good news:** `battleground update` already downloaded all the new images as tarballs via
SteamCMD before the registry was needed. They're sitting in
`/home/dune/.dune/download/images/battleground/`. The fix is to import them into containerd
correctly — bypassing the broken registry entirely.

---

## The fix (preferred) — import from local tarballs

`battleground update-from-downloads` imports the tarballs but uses the wrong `ctr` variant,
leaving images invisible to the kubelet. Use `k3s ctr` instead.

### Step 1 — find your namespace and BattleGroup name

```bash
sudo kubectl get igwbg -A
```

Output looks like:
```
NAMESPACE                                          NAME
funcom-seabass-sh-<hostid>-<worldid>               sh-<hostid>-<worldid>
```

Set these variables (replace with your actual values):
```bash
NS=funcom-seabass-sh-<hostid>-<worldid>
BG=sh-<hostid>-<worldid>
```

### Step 2 — confirm the tarballs are present

```bash
ls /home/dune/.dune/download/images/battleground/
cat /home/dune/.dune/download/images/battlegroup/version.txt
```

You should see `server.tar`, `server-rabbitmq.tar`, `server-db-utils.tar`, etc., and the version
file should show the new build number (e.g. `1968181-0-shipping`).

If the tarballs are missing, skip to the **Fallback: roll back to old build** section below.

### Step 3 — import all tarballs via k3s ctr

This is the critical difference from `update-from-downloads` — `k3s ctr` properly registers
images into the CRI snapshotter so kubelet can actually use them.

```bash
IMGDIR=/home/dune/.dune/download/images/battlegroup

for tarfile in server-db-utils server-bg-director server-gateway server-text-router server-rabbitmq; do
  echo "=== Importing ${tarfile}.tar ==="
  sudo k3s ctr -n k8s.io images import "${IMGDIR}/${tarfile}.tar"
done

echo "=== Importing server.tar (4GB — takes a while) ==="
sudo k3s ctr -n k8s.io images import "${IMGDIR}/server.tar"
```

Verify all images are now visible to the kubelet:
```bash
sudo crictl images | grep funcom | grep <new-build>
# e.g. grep 1968181
```

You should see 6 images: `seabass-server`, `seabass-server-rabbitmq`, `seabass-server-db-utils`,
`seabass-server-bg-director`, `seabass-server-gateway`, `seabass-server-text-router`.

### Step 4 — delete the stuck pods

```bash
sudo kubectl get pods -n $NS | grep -E 'ImagePullBackOff|ErrImagePull' | awk '{print $1}' | \
  xargs sudo kubectl delete pod -n $NS
```

Wait ~20 seconds then check:
```bash
sudo kubectl get pods -n $NS | grep -E 'mq-|db-dbdepl-util'
```

MQ pods should show `1/1 Running`. The DB util pod will show `Completed` once the migration finishes.

### Step 5 — start the server

```bash
battlegroup start
```

Check status (give it 60–90 seconds for maps to load):
```bash
battlegroup status
```

You should see `Healthy` across the board and `Running` for your world maps.
The server will now advertise the correct new build revision to FLS and appear in the server browser.

---

## Fallback — roll back to the previous cached build

Use this if the tarballs in `images/battlegroup/` are missing. This gets the server running on
the old build immediately — **clients must also be on the old build to connect**.

### Step 1 — identify cached build

```bash
sudo crictl images | grep seabass-server-rabbitmq
```

Note the most recent cached build (e.g. `1963158-0-shipping`). If you see nothing, you cannot
roll back — wait for Funcom to fix their registry.

Set variables:
```bash
NS=funcom-seabass-sh-<hostid>-<worldid>
BG=sh-<hostid>-<worldid>
BROKEN="1968181-0-shipping"   # the build that can't be pulled
CACHED="1963158-0-shipping"   # the build you have locally
```

### Step 2 — patch the BattleGroup CRD

```bash
sudo kubectl get igwbg $BG -n $NS -o json > /tmp/bg_full.json

python3 << EOF
import json

BROKEN = "${BROKEN}"
CACHED = "${CACHED}"

with open('/tmp/bg_full.json') as f:
    d = json.load(f)

spec_str = json.dumps(d['spec'])
count = spec_str.count(BROKEN)
spec_new = json.loads(spec_str.replace(BROKEN, CACHED))

with open('/tmp/bg_patch.json', 'w') as f:
    json.dump({'spec': spec_new}, f)

print(f"Replaced {count} occurrences of {BROKEN} with {CACHED}")
EOF

sudo kubectl patch igwbg $BG -n $NS --type=merge --patch-file=/tmp/bg_patch.json
```

### Step 3 — patch the database deployment

```bash
DBDEPL=$(sudo kubectl get igwdbdepl -n $NS -o jsonpath='{.items[0].metadata.name}')
sudo kubectl patch igwdbdepl $DBDEPL -n $NS --type=merge \
  -p "{\"spec\":{\"utilityImage\":\"registry.funcom.com/funcom/self-hosting/seabass-server-db-utils:${CACHED}\"}}"
```

### Step 4 — delete stuck pods and start

```bash
sudo kubectl get pods -n $NS | grep -E 'ImagePullBackOff|ErrImagePull' | awk '{print $1}' | \
  xargs sudo kubectl delete pod -n $NS

sleep 15
battlegroup start
battlegroup status
```

### Step 5 — upgrade once Funcom fix their registry

```bash
# Confirm DNS is back:
nslookup registry.funcom.com 8.8.8.8

# Then update normally:
battleground update
```

---

## Notes

- **No data loss risk.** If the DB util pods never started (which is typical in this scenario),
  no migration ran and rollback is completely safe.
- **Version mismatch.** If you roll back to the old build, clients on the new game version
  won't see the server in the browser. You'll need to either have everyone opt into the old
  Steam branch, or use the preferred tarball-import fix instead.
- **Why `k3s ctr` and not `ctr`?** k3s bundles its own containerd tooling. Plain `ctr` imports
  into the containerd content store but skips the CRI snapshotter registration. `k3s ctr` does
  the full import that kubelet can actually use.
- **The `update-from-downloads` bug.** `battlegroup update-from-downloads` uses plain `ctr`
  internally, which is why it appears to succeed but images still can't be pulled. This is a
  Funcom tooling issue that presumably only matters when the registry is unreachable.
