# test lab — crash-free driver development

All kernel work is proven in a VirtualBox guest before the host ever sees
it. The host has never taken a test BSOD from this pipeline.

## Guest build (scripted, ~30 min, zero clicks)

1. `VBoxManage createvm` — 4 GB RAM, 24 GB disk, Intel PRO/1000 NIC
   (inbox drivers, no virtio fuss), BIOS boot.
2. Stock ISOs with hostile installers get bypassed: the disk is imaged
   **offline** — `clonehd` to VHD, `diskpart` partition, `DISM /apply-image`
   straight from `install.wim`, `bcdboot`, unattend with autologon admin.
3. First-boot provisioning via `RunOnce`: UAC off, test kit at `C:\test`,
   versioned completion flags (a run never executes twice, a new kit
   version always re-runs).

## Command loop (the whole trick)

No Guest Additions, no SSH, no services in the guest. A 600-byte batch
loop polls `GET http://10.0.2.2:8900/cmd` every 12 seconds:

```
host writes  "20 whoami"  → guest runs it → POSTs output → host reads log
```

Host side is a 100-line Python listener with an outbox file. Result:
interactive-speed control of the guest with **zero guest-side footprint**
beyond `curl.exe` (inbox) and the loop itself. Reboots self-heal via
RunOnce; versioned done-flags prevent stale re-runs.

## Crash cycle (minutes, not hours)

1. Push kit (offline VHD mount, or post-boot file copy) → guest auto-runs.
2. On BSOD: `controlvm poweroff` → mount VHD on host → pull
   `C:\Windows\minidump\*.dmp` + test logs → `kd` triage here.
3. Fix, rebuild, push, reboot. Snapshot before anything spicy
   (`bridge_verified`, `pre_detonate`) — one revert undoes everything.

## Rules that earned themselves

- Verbose stage logging in loaders (a bare `failed` tells you nothing;
  every stage prints, successes included).
- Listeners must parse `Content-Length` exactly (empty POST bodies
  deadlock naive read-loops — found the hard way).
- Recursive copy, always (`Copy-Item *` skips subdirectory contents).
- The guest clock lies; monotonic ticks don't (NTP jumps break
  wall-clock boot comparisons — the one-map guard keys on ticks).
