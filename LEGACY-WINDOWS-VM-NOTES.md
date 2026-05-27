# Legacy Funcom Windows VM — Troubleshooting Notes

Footnote-style notes for the **original Funcom self-hosting setup**:
**Windows host (often Server 2022) → Hyper-V → Alpine Linux VM → k3s → game pods.**

This is *not* the borgcube install. The borgcube path is documented in [CLAUDE.md](CLAUDE.md) and [DUNE-UBUNTU-HOSTING.md](DUNE-UBUNTU-HOSTING.md), and nothing in this file applies there.

These notes exist for diagnosing issues on **other people's installs** that are still on the legacy path. Append new findings as dated entries; keep them short and symptom-first so they can be skimmed when a friend pastes a screenshot.

---

## 2026-05-17 — `ctr: error reading from server: EOF` / `connection reset by peer` during prerequisite image import

### Symptom
During k3s startup, `ctr` import of `prerequisites/*.tar` images (coredns, igw-postgres, jetstack/cert-manager, etc.) fails with one of:

```
ERRO[0006] progress stream failed to recv ... error="error reading from server: EOF"
ctr: error reading from server: EOF
Import of images/prerequisites/<name>.tar failed (attempt 1/3)...
k3s/containerd not responding, restarting
```

or the stronger variant:

```
ERRO[0000] progress stream failed to recv ... error="error reading from server: read unix @->/run/k3s/containerd/containerd.sock: read: connection reset by peer"
ERRO[0000] send stream ended without EOF                 error="error reading from server: read unix @->/run/k3s/containerd/containerd.sock: read: connection reset by peer"
```

k3s restarts and the import loop never converges.

### Interpretation
**`connection reset by peer` ≠ bare `EOF`.** Reset means containerd was alive and got killed mid-import by something external — not a crash, not a corrupt tarball. Cainjector imports fine then postgres dies → not the data, it's the environment hitting a ceiling on the bigger image.

### Causes, in observed-frequency order
1. **Windows Defender / AV on the host.** Server 2022 has Defender on by default; it scans the Alpine VM's vhdx and any tar/container layer it can see, holds files open, and surfaces to containerd as "connection reset". **Fix:** exclude the entire Dune install folder *and* the VM's vhdx path (Hyper-V VM directory, or `%USERPROFILE%\AppData\Local\Packages\*WSL*\LocalState\*.vhdx` if WSL2). Also exclude `containerd.exe` / k3s process images if reachable.
2. **WSL2 memory cap** (if the "Alpine VM" is actually running under WSL2 rather than full Hyper-V). Default is 50 % of host RAM up to 8 GB; importing postgres on top of running k3s + already-loaded images OOM-kills containerd. **Fix:** `%USERPROFILE%\.wslconfig`:
   ```
   [wsl2]
   memory=24GB
   processors=8
   swap=8GB
   ```
   then `wsl --shutdown`, retry.
3. **VM disk full / vhdx out of space.** Containerd snapshotter needs several GB free during import; host volume can be full even if the vhdx is sparse.
4. **Nested virtualization not enabled** if the Server 2022 host is itself a VM (Proxmox/ESXi/Hyper-V guest). WSL2 silently misbehaves under load without it; explicitly enable nested virt on the hypervisor.

### Triage order
1. AV exclusion on install folder + VM vhdx path.
2. Bump WSL2 / Hyper-V VM memory to ≥16 GB (24 GB comfortable).
3. `wsl --shutdown` (if WSL) or restart the VM. Retry.
4. **If a later/larger image now fails** → still memory; keep raising.
   **If the same image fails at the same byte** → still AV, or genuinely corrupt — re-verify the install (SteamCMD `app_update <id> validate`, or re-extract the archive with 7-Zip, not Windows Explorer's built-in zip).

### Why this doesn't happen on borgcube
Bare-metal Ubuntu: no AV scanning the snapshotter, no WSL memory cap, no nested-virt requirement, no vhdx layer. The whole class of failure goes away with the migration in [DUNE-UBUNTU-HOSTING.md](DUNE-UBUNTU-HOSTING.md).

---

<!-- Add new dated entries above this line, newest first. Keep entries symptom-first so a screenshot paste can match quickly. -->
