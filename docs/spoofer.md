# sorana-spoofer — cave-mapped kernel driver

Target: **Windows 10 x64, build 19045** (22H2).

## What it does

Three jobs in one image, executed once at map time, then resident:

1. **HWID identity mangle** — SMBIOS Type 1/2 fields, UEFI firmware
   variables (BitLocker EK / UnlockID family), USBSTOR disk serials.
2. **Hook engine** — 13 syscall hooks (COW private pages + trampolines +
   shadow mappings), 14 EAT hooks on core kernel exports, exception hooks.
3. **Physical memory core** — CR3-aware physical read/write (DPM remap
   windows), page-table introspection, PFN-database access.

Plus the [astrum bridge](astrum-bridge.md) for usermode communication.

## How it loads (no service, no driver object)

The image never goes through the Service Control Manager. The
[mapper](mapper.md) writes it into a **PTE-split cave inside ntoskrnl's
`.data` padding** — executable bytes in a data region — applies relocations,
resolves imports, copies sections via physical writes, then calls its entry
point through an SSDT gate that is verified and force-restored afterwards.

## Identity surfaces

| Surface | Mechanism | Persistence |
|---|---|---|
| UEFI variables (`UnlockIDCopy`, `OfflineUniqueIDEKPubCRC`, `OfflineUniqueIDEKPub`) | `ExGet/SetFirmwareEnvironmentVariable` from kernel context | **Survives reboot** (flashed) |
| USBSTOR serials | `\Driver\USBSTOR` device walk, in-place serial rewrite | Session (memory-only) |
| SMBIOS (UUID, baseboard, CPU id) | Served through hooked `MmCopyMemory` / `MmMapIoSpace` path | Session (memory-only) |
| Volume serials | out of scope for this build | — |

> Note: on 19045 the in-memory SMBIOS pattern scan is a graceful no-op
> (patterns are 24H2-era); the live query path is served by the hooked
> copy/map routines. WMI-visible values depend on provider cache timing.

## Self-defense details

- **KVA-shadow aware PTE math** — the self-map base is resolved dynamically
  per boot (self-referencing PML4 entry), never hardcoded. The classic
  `0xFFFFF680…` base is deliberately unmapped on shadow systems and faulting
  it bugchecks.
- **PFN parity-bit hiding** (`hide_pages`) for cave regions.
- **No device object, no IOCTL surface, no callbacks** — nothing to enumerate.

## Statuses

The driver logs to `C:\sorana\sorana.log` (debug builds): cave/base/CR3
lines, per-surface mangle results, per-hook `hook state ready!` lines, and
the `astrum: HAL dispatch hook active` line when the bridge is up.
