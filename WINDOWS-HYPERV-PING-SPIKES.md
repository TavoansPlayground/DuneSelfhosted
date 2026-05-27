# Fix: Ping spikes to ~1100 ms after server has been running for a while (Hyper-V host)

## Symptom

Clients report normal ping on connect, then latency climbs to ~1100 ms after minutes or hours
of uptime. Restarting the server temporarily fixes it. Happens on Windows hosts running the
game server inside Hyper-V or WSL2.

---

## Fixes — apply in this order

### 1. Windows network throttling (most common — apply first)

Windows throttles network traffic for background processes by default. Set these two registry
values on the **Windows host** and reboot:

```reg
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile]
"NetworkThrottlingIndex"=dword:ffffffff
"SystemResponsiveness"=dword:00000000
```

`NetworkThrottlingIndex=0xFFFFFFFF` disables the throttle entirely (default is `10`).
`SystemResponsiveness=0` tells Windows to prioritise background services over foreground apps.

Apply via `.reg` file, or run in an elevated PowerShell:
```powershell
$path = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile"
Set-ItemProperty -Path $path -Name "NetworkThrottlingIndex" -Value 0xffffffff -Type DWord
Set-ItemProperty -Path $path -Name "SystemResponsiveness"   -Value 0          -Type DWord
```

### 2. Hyper-V timer coalescing

Hyper-V coalesces timer interrupts to save power, causing periodic ~1 s latency bursts.
Fix on the **Windows host**:

```reg
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\kernel]
"GlobalTimerResolutionRequests"=dword:00000001
```

Also run in an elevated CMD (persists across reboots):
```cmd
bcdedit /set useplatformtick yes
bcdedit /set disabledynamictick yes
```

Requires a reboot.

### 3. NIC power management

The network adapter enters a low-power state after light traffic and takes ~1 s to wake.
In **Device Manager → Network Adapters → [your NIC] → Properties → Power Management**:
- Uncheck **"Allow the computer to turn off this device to save power"**

Also check the advanced adapter settings for "Energy Efficient Ethernet" or "Green Ethernet"
and disable those too.

### 4. Windows Defender real-time scanning

Defender scans game server data and VM disk writes in real-time, adding latency under load.
Add exclusions on the **Windows host**:

- The entire Dune install folder (e.g. `C:\DuneServer\` or wherever SteamCMD put it)
- The Hyper-V VM / WSL2 vhdx path:
  - Hyper-V: your VM's directory (e.g. `C:\ProgramData\Microsoft\Windows\Hyper-V\`)
  - WSL2: `%LOCALAPPDATA%\Packages\*WSL*\LocalState\*.vhdx`
- The k3s / containerd binary if accessible from the host

PowerShell (elevated):
```powershell
Add-MpPreference -ExclusionPath "C:\DuneServer"
Add-MpPreference -ExclusionPath "C:\ProgramData\Microsoft\Windows\Hyper-V"
```

Note: Defender exclusions also fix the `ctr: connection reset by peer` image import failure.
Same root cause — AV interrupting low-level I/O.

---

## Confirmed working on

| Date | Platform | Notes |
|------|----------|-------|
| 2026-05-27 | Windows Server 2022 + Hyper-V | All four fixes applied; ping stabilised |
