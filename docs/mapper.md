# mapper — self-sufficient kernel loader

`mapper.exe <driver.sys>`. No external dependencies, no `windx64.exe`,
no service installation of the payload.

## Pipeline

```
adjust privileges (SeDebug + SeSystemEnvironment)
  → resolve 8 PDB symbols (DIA, Microsoft symbol server)
  → install + start transient bootstrap driver (EBIoDispatch)
  → paging setup (CR3 discovery, physical RW)
  → SSDT gate setup (slot hijack + restore verification)
  → strip kernel residuals of the bootstrap
  → load PE (relocations, imports, delay-imports)
  → find PTE-split cave in ntoskrnl .data
  → copy sections via physical writes, verify mapping
  → call entry point through the SSDT gate
  → verify SSDT restored (force-restore if not)
  → FULL teardown
```

## Teardown (the part that matters)

| Stage | Target |
|---|---|
| K1 | `MmUnloadedDrivers` + `PiDDBCacheTable` entry removal |
| K2 | PiDDB lock-protected cache delete |
| K3 | Hash-bucket (`g_CiHashes`) unlink |
| K4 | WdFilter/Defender tracking unlink |
| U2–U4 | `NtUnloadDriver`, temp file wipe + delete, service key delete |

Every stage is logged. A_FAILED stage aborts before the map, never after.

## One-map-per-boot guard

The mapper appends a monotonic `map_tick:<GetTickCount64>` line to the
driver log on every **successful** map. On launch it scans the log tail:

- `logged_tick <= current_tick` → same boot → prints
  `refused: driver already mapped this boot` and exits **before touching
  anything** (no service, no driver load, no syscalls gated).
- `logged_tick > current_tick` → older boot → allowed.
- No tick line (pre-guard logs) → falls back to log-mtime vs boot-time.

Tick counts reset every boot and ignore wall-clock jumps, so NTP
corrections can't fool it. Rationale: two resident hook instances fight
over the same pages and produce delayed `SYSTEM_SERVICE_EXCEPTION`
bugchecks hours later — this guard makes that class impossible.

## Logging

Release builds log stage transitions to stdout (buffered — lost on
bugcheck, which is itself a signal: empty output + no `done` = the process
died mid-flight). Final line is always `done` or `failed`.
