# EN2 MMU driver gap: support 2048-entry page tables (GAP-01)

Parent issue: #TBD (EN2 MMU Test Plan)

> **ID convention:** `MMU-DRV-T-nn` = **T**est exists. `MMU-DRV-C-nn` = **C**andidate test (to be written).

## Problem

The EN2 MMU hardware has **2048 page-table entries per client**. The driver can only use **1024**, so entries 1024..2047 cannot be allocated or mapped. `MmuDriver::init` already clears all 2048 hardware entries, so the limit is in the driver's bookkeeping, not in initialization.

## Evidence in the code

| Location | Limit |
|---|---|
| `drivers/mmu/src/MmuDriver.cpp` | `PageTableResourceRecord::index : 10` (pages 0..1023) |
| `drivers/mmu/src/MmuDriver.cpp` | `static_assert(CONFIG_SYSTEM_MMU_MAX_PAGES_PER_TABLE <= 1024, ...)` |
| `drivers/mmu/src/MmuMemApi.cpp` | `MemAllocRecord::clientStartPageIndex_bits = 10` (start page 0..1023) |
| `drivers/mmu/src/MmuMemApi.cpp` | `MemAllocRecord::numPages_bits = 11` (at most 2047 pages; a full 2048-page allocation needs 12) |
| `drivers/mmu/src/MmuMemApi.cpp` | `static_assert`s that require the largest table to fit those fields |
| `drivers/mmu/include/PageTableDescriptor.hpp` | `frameNumber : 10`; unused legacy header |
| Hardware | `EaiNestMmuTbl::PAGE_TABLE_ENTRY_MMU_TABLE_RANGE_SIZE` = 2048 |

## Suggested driver changes (to be confirmed by the driver owner)

- [ ] Widen `PageTableResourceRecord::index` to 11 bits and relax the 1024 `static_assert` to the hardware depth (2048).
- [ ] Widen `MemAllocRecord::clientStartPageIndex_bits` to 11 and `numPages_bits` to 12. Re-check the packed record size and the related `static_assert`s.
- [ ] Update the `CONFIG_SYSTEM_MMU_PAGES_PER_TABLE` and `CONFIG_SYSTEM_MMU_MAX_PAGES_PER_TABLE` configuration so tables above 1024 pages are allowed.
- [ ] Review bounds checks that use `numPagesPerTable`, plus `ObjPool` sizes and 16-bit index types.
- [ ] Remove or update `PageTableDescriptor.hpp`.

## Interaction with GAP-02

With 32 MB pages, 2048 entries need 64 GB of DDR. Production EN2 has 32 GB, so the pool stays at 1024 pages. The upper entries are reached on production hardware only with 16 MB pages (GAP-02, #TBD) or a larger DDR. The client-side index range can still be tested earlier at `MmuDriver` level, because `commitPageTablesSpan` accepts any client page index and does not need DDR behind it.

## Tests blocked by this gap

These tests **depend on driver support** and cannot pass until the changes above land.

**Coverage after these tests (cumulative, after P1, P2, P3): 95.4% (+2.3%, 2 items).**

| ID | Candidate test | Executable | Items | What it checks |
|---|---|---|---|---|
| MMU-DRV-C-14 | Full 2048-entry table | `test-nest-mmu-driver-bm.elf` (`TestMmuDriver.cpp`) for client page indexes 1024..2047 through `commitPageTablesSpan` (no data access), and `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`) for allocation across all 2048 pages when the platform provides them | 1 | Allocate, commit, read back and free pages in the upper half of the table for PF, VF and RISC-V clients; every entry reads back valid with the right page number; free restores the counts. |
| MMU-DRV-C-15 | Index bounds at hardware depth | `test-nest-mmu-driver-bm.elf` (`TestMmuDriver.cpp`) | 1 | Client indexes 1023, 1024, 2047 (valid) and 2048 (rejected) on `commitPageTablesSpan`, `clearClientPages` and the `MmuMemApi` equivalents. Builds on MMU-DRV-C-02. |

## Where to add

Both tests are new `TEST_F` cases in existing ELFs. No CMake change is needed. Sources are in `tests/nest/nest-unit-tests-bm/src/`.

| Executable | Source file | Tests |
|---|---|---|
| `test-nest-mmu-driver-bm.elf` | `TestMmuDriver.cpp` | C-14 (`MmuDriver` part), C-15 |
| `test-nest-mmu-bm.elf` | `TestMmuEn2.cpp` | C-14 (`MmuMemApi` part; skip when the platform has fewer than 2048 pages) |

## Tracking

- [ ] Driver changes above
- [ ] MMU-DRV-C-14 Full 2048-entry table
- [ ] MMU-DRV-C-15 Index bounds at hardware depth

## Definition of done

- Driver supports client page indexes 0..2047 and the new limit is enforced.
- Both tests added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Existing tests (MMU-DRV-T-01 to T-25) still pass.
- Coverage table in the parent issue updated.
