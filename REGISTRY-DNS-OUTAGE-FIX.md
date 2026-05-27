# Fix: Server stuck in ErrImagePull after update — registry.funcom.com DNS failure

## Symptoms

- You ran `battlegroup update`
- Pods (`mq-admin-sts-0`, `mq-game-sts-0`, `db-dbdepl-util-*`) are stuck in `ErrImagePull` or `ImagePullBackOff`
- `kubectl describe pod` events show:
  ```
  failed to resolve reference "registry.funcom.com/...": dial tcp: lookup registry.funcom.com: no such host
  ```
- `battlegroup status` shows the battlegroup stuck in `Modifying` or services not `Healthy`

## Root cause

`registry.funcom.com` — Funcom's container image registry — periodically goes NXDOMAIN (DNS
lookup fails globally, including against 8.8.8.8). This happens after game updates that bump the
image tags to a new build number.

**The images are already on your machine.** `battlegroup update` downloads all new images as
tarballs via SteamCMD before the registry is needed. They live in:

```
/home/dune/.dune/download/images/battlegroup/
```

The problem is that the update script imports them using plain `ctr`, which writes to containerd's
content store but **skips the CRI snapshotter** — so the kubelet cannot see or use the images and
falls back to pulling from the now-dead registry. `ctr images ls` will show the images as present,
but `crictl images` will not, because they were never registered in the CRI layer.

The fix is to re-import the same tarballs using `k3s ctr`, which does the full import that kubelet
can actually use.

---

## Quick check — confirm this is the problem

```bash
# Confirm registry is NXDOMAIN:
nslookup registry.funcom.com 8.8.8.8

# Confirm tarballs are present:
ls /home/dune/.dune/download/images/battlegroup/

# Show the new build version:
cat /home/dune/.dune/download/images/battlegroup/version.txt

# Confirm kubelet CANNOT see the new build (this is the core symptom):
sudo crictl images | grep funcom | grep "$(cat /home/dune/.dune/download/images/battlegroup/version.txt)"
# If the above returns nothing, proceed with the fix below.
# If it returns 6 images, the fix has already been applied — just delete the stuck pods (Step 4).
```

---

## The fix (preferred) — import from local tarballs via k3s ctr

### Step 1 — find your namespace and BattleGroup name

```bash
sudo kubectl get ns | grep funcom-seabass
```

Output looks like:
```
funcom-seabass-sh-<hostid>-<worldid>   Active   ...
```

Set these variables (replace with your actual values):
```bash
NS=funcom-seabass-sh-<hostid>-<worldid>
BG=sh-<hostid>-<worldid>
```

### Step 2 — confirm the tarballs are present

```bash
ls /home/dune/.dune/download/images/battlegroup/
cat /home/dune/.dune/download/images/battlegroup/version.txt
```

You should see `server.tar`, `server-rabbitmq.tar`, `server-db-utils.tar`, etc., and the version
file should show the new build number (e.g. `1973075-0-shipping`).

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

echo "=== Importing server.tar (4GB — takes 1-2 minutes) ==="
sudo k3s ctr -n k8s.io images import "${IMGDIR}/server.tar"
```

> **Note:** Each import may report `total: 0.0 B (0.0 B/s)`. This is normal — it means the
> content blobs were already written to the store by the earlier plain `ctr` import. The
> `k3s ctr` re-import still registers the images correctly with the CRI snapshotter even when no
> new bytes are transferred.

### Step 4 — verify kubelet can now see the images

```bash
NEW_BUILD=$(cat /home/dune/.dune/download/images/battlegroup/version.txt)
sudo crictl images | grep funcom | grep "$NEW_BUILD"
```

You should see **6 images**: `seabass-server`, `seabass-server-rabbitmq`, `seabass-server-db-utils`,
`seabass-server-bg-director`, `seabass-server-gateway`, `seabass-server-text-router`.

If you see 0 results, check that k3s is running (`systemctl status k3s`) and retry Step 3.

### Step 5 — delete stuck pods

```bash
sudo kubectl get pods -n $NS | grep -E 'ImagePullBackOff|ErrImagePull' | awk '{print $1}' | \
  xargs -r sudo kubectl delete pod -n $NS
```

Wait ~30 seconds then verify the MQ pods came back up:
```bash
sudo kubectl get pods -n $NS | grep -E 'mq-|db-dbdepl-util'
```

`mq-admin-sts-0` and `mq-game-sts-0` should show `1/1 Running`. The `db-dbdepl-util` pod will
show `Completed` once the migration finishes (or may not appear at all if no migration was needed).

### Step 6 — confirm full recovery

```bash
battlegroup status
```

You should see `Healthy` across the board and `Running` for your world maps. The server is now on
the new build and will advertise correctly to FLS and appear in the server browser.

---

## Fallback — roll back to the previous cached build

Use this **only if the tarballs in `images/battlegroup/` are missing** (e.g. SteamCMD didn't
finish before the registry went down). This gets the server running on the old build immediately.

> **Warning:** Clients on the new game version won't be able to connect to a server on the old
> build. Use the tarball-import fix above whenever possible.

### Step 1 — identify cached builds

```bash
sudo crictl images | grep seabass-server-rabbitmq
```

Note all the build tags shown. Pick the most recent one that isn't the broken new build. If you
see nothing at all, you cannot roll back — wait for Funcom to restore their registry.

```bash
NS=funcom-seabass-sh-<hostid>-<worldid>
BG=sh-<hostid>-<worldid>
BROKEN="<new-build-that-cant-pull>"    # e.g. 1973075-0-shipping
CACHED="<previous-build-you-have>"     # e.g. 1968181-0-shipping
```

### Step 2 — patch the BattleGroup CRD to the cached build

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

### Step 4 — delete stuck pods and check status

```bash
sudo kubectl get pods -n $NS | grep -E 'ImagePullBackOff|ErrImagePull' | awk '{print $1}' | \
  xargs -r sudo kubectl delete pod -n $NS

sleep 20
battlegroup status
```

### Step 5 — upgrade once Funcom restore their registry

```bash
# Confirm DNS is back:
nslookup registry.funcom.com 8.8.8.8

# Then update normally:
battlegroup update
```

---

## Why `k3s ctr` and not `ctr`

k3s ships its own bundled `containerd` and tooling. The key difference when importing images:

| Tool | Writes content blobs | Registers with CRI snapshotter | Kubelet can use? |
|------|:---:|:---:|:---:|
| `ctr` (system package) | ✅ | ❌ | ❌ |
| `k3s ctr` | ✅ | ✅ | ✅ |
| `crictl pull` | ✅ | ✅ | ✅ |

`battlegroup update-from-downloads` uses plain `ctr` internally — this is a Funcom tooling bug
that only surfaces when the registry is unreachable. Normally, images pulled live from the
registry go through the CRI path automatically and the distinction doesn't matter.

**Diagnostic tell:** if `ctr -n k8s.io images ls | grep funcom` shows the new build but
`crictl images | grep funcom` does not, you're in this exact situation.

---

## Notes

- **No data loss risk.** If the DB util pods never started (typical in this scenario), no database
  migration ran and rollback is completely safe. Your world data is intact.
- **`update-from-downloads` is not the fix.** It uses the same plain `ctr` and will appear to
  succeed while leaving the same problem behind.
- **This is a recurring Funcom infrastructure issue.** When pods fail to pull after an update,
  always check `nslookup registry.funcom.com 8.8.8.8` first.

---

## Confirmed working on

| Date | Build | Platform |
|------|-------|----------|
| 2026-05-27 | `1973075-0-shipping` | Ubuntu 24.04 bare-metal, k3s |
| 2026-05-?? | `1968181-0-shipping` | Ubuntu 24.04 bare-metal, k3s |
