# astrum bridge — zero-footprint kernel channel

Usemode ↔ kernel communication with **no device object, no thread, no
handles, no shared memory, no IOCTLs**. Nothing to enumerate, nothing to
open, nothing to hook-detect by object scans.

## Transport

```
usermode:  NtQueryAuxiliaryCounterFrequency(&out, request*)
kernel:    HalPrivateDispatchTable[116]  →  nt!HalpTimerQueryAuxiliaryCounterFrequency (hooked)
```

The NT wrapper clobbers RDX before the HAL call, so the hook recovers the
original user RDX (the request pointer) from the syscall trap frame
(`KTHREAD+0x90 → KTRAP_FRAME+0x40`). The first qword of the request must be
the magic, otherwise the call falls through to the legitimate counter
handler untouched (which returns `STATUS_NOT_SUPPORTED` on most hardware —
a clean, unsuspicious answer).

- **Magic:** `0xA5724DC0DEA11E` (first 8 bytes of every request)
- **Slot:** `HalPrivateDispatchTable[116]` (verified on 19041–19045;
  neighboring convert-counter slots are 114/115)
- **Hook type:** COW private page + trampoline + shadow mapping (same
  machinery as the syscall hooks)

## Commands

| ID | Name | Payload | Effect |
|---|---|---|---|
| 0 | null | — | no-op |
| 1 | read | `{pid, src_va, dst_va, size}` | VA→VA via per-request CR3 translation + physical copy |
| 2 | write | `{src_va, dst_va, size}` | caller buffer → target VA via physical copy |
| 3 | query | `{pid}` → `{peb, base, cr3}` | PEB + image base + directory-table base |
| 4 | alloc | `{pid, size}` → `{address}` | RWX allocation inside target process |
| `1 ^ 0xC0DEBA5E` | batch | N× `{pid, src, dst, size, status}` | up to 256 entries / 64 MB per call, per-entry status |

- Caps: 16 MB per single op. Every user pointer touched under SEH.
- Auth token field exists in the wire struct; `0` = check elided (dev posture).
- Reads never attach to the target and never call `MmCopyMemory` — pure
  translate → physical-copy through DPM remap windows.

## Wire structs (exact layout, little-endian, x64)

```c
struct request { uint64_t magic; void *data; uint64_t length;
                 uint32_t command; uint32_t token; };          // 0x28
struct mem_copy { uint32_t pid; uint32_t _pad; uint64_t src;
                  void *dst; uint64_t size; };                // 0x20
struct proc_info { uint32_t pid; uint32_t _pad; void *peb;
                   void *base; uint64_t cr3; };                // 0x20
struct mem_alloc { uint32_t pid; uint32_t _pad; uint64_t size;
                   void *address; };                          // 0x18
struct batch_entry { uint32_t pid; uint32_t _pad; uint64_t src; void *dst;
                     uint64_t size; int32_t status; uint32_t _pad2; }; // 0x28
```

## Client

Any process that can issue the syscall can use the bridge — reference
client implements `try_connect / attach / read / write / read_batch /
get_module_base / alloc` over this protocol with mutex + 3-attempt retry
(HAL traffic collisions can drop replies).
