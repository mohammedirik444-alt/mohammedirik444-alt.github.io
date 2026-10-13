# ExauXit — systems & security engineering

![status](https://img.shields.io/badge/status-active-brightgreen?style=flat-square)
![platform](https://img.shields.io/badge/platform-win10%20x64%2019045-blue?style=flat-square)
![license](https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square)

Low-level Windows engineering: cave-mapped kernel drivers, zero-footprint
kernel↔user channels, HWID identity research, and crash-forensics-driven
development. If it lives below the API, it's in scope.

Live dossier: **https://mohammedirik444-alt.github.io/**

---

## Projects

| Project | What it is | Docs |
|---|---|---|
| **sorana-spoofer** | Cave-mapped kernel driver: SMBIOS/UEFI/USBSTOR identity mangle, syscall + EAT hook engine, physical-memory RW core | [docs/spoofer.md](docs/spoofer.md) |
| **mapper** | Self-sufficient kernel loader: SSDT gate, PDB-driven symbol resolution, cave injection, full-residue teardown, one-map-per-boot guard | [docs/mapper.md](docs/mapper.md) |
| **astrum bridge** | Zero-footprint kernel channel over a HAL dispatch hook — no device, no thread, no handles. Query/read/write/batch/alloc | [docs/astrum-bridge.md](docs/astrum-bridge.md) |
| **test lab** | Snapshot-protected VirtualBox pipeline: offline image builds, guest command loop, result collection, zero host risk | [docs/lab.md](docs/lab.md) |

## Operating rules

1. **One map per boot.** Kernel work is single-shot with verified teardown.
   Stacked hook instances cause delayed bugchecks — the mapper refuses a
   second map in the same session.
2. **Isolate first.** Unknown binaries get static triage, then detonation in
   a snapshot-protected VM. Never the host.
3. **Prove it in the dump.** No fix ships on theory — faulting instruction,
   register state, mechanism, then patch.
4. **Leave no residue.** PiDDB scrubbed, services removed, files wiped,
   SSDT restored — or it didn't happen.

## Repository layout

```
├─ index.html          # live dossier (GitHub Pages)
├─ docs/
│  ├─ spoofer.md       # driver architecture + identity surfaces
│  ├─ mapper.md        # loader pipeline + teardown + guard
│  ├─ astrum-bridge.md # protocol spec + wire format
│  └─ lab.md           # isolated test methodology
└─ LICENSE
```

> Research artifacts. Built and tested in isolated labs against local
> machines. Nothing here phones home — there is no network code, period.
