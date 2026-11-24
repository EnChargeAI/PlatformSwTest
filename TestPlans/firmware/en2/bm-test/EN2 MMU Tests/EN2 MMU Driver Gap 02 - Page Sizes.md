# EN2 MMU driver gap: support 8, 16, 32 and 64 MB pages (GAP-02)

Parent issue: #TBD (EN2 MMU Test Plan)

> **ID convention:** `MMU-DRV-T-nn` = **T**est exists. `MMU-DRV-C-nn` = **C**andidate test (to be written).

## Problem

The EN2 MMU hardware supports four segment sizes through `xlat_seg_size`: **8, 16, 32 and 64 MB**. The driver supports only **32 MB**. The page size is a compile-time constant, and the EN2 device layer refuses to build with any other value.

## Evidence in the code

| Location | Limit |
|---|---|
| `drivers/mmu/device/en2/MmuDeviceEn2.cpp` | `static_assert(CONFIG_SYSTEM_MMU_PAGE_SIZE_MB == 32)` in `configure()`; writes `xlat_seg_size` = 2 |
| `drivers/mmu/include/MmuDriver.hpp` | `pageSizeBytes()` is `constexpr` from `CONFIG_SYSTEM_MMU_PAGE_SIZE_MB` |
| `drivers/mmu/api/MmuMemApi.hpp` | `getPageSize()` is `constexpr` from the same constant |
| `cmake/fw-config/defaults/system.yaml` | `CONFIG_SYSTEM_MMU_PAGE_SIZE_MB: 32`; page counts derive from it |
| `drivers/mmu/device/en2/MmuDeviceEn2.cpp` | `configure()` leaves `xlat_seg_size` unchanged when the boot parameters say the MMU is already enabled |
| Hardware | `xlat_seg_size`: 0 = 8 MB, 1 = 16 MB, 2 = 32 MB, 3 = 64 MB |

`TestMmuEn2.cpp` already maps `CONFIG_SYSTEM_MMU_PAGE_SIZE_MB` to the `xlat_seg_size` value for 8, 16, 32 and 64 MB. The mapping exists in the test code but not in the driver.

## Suggested driver changes (to be confirmed by the driver owner)

- [ ] Replace the 32 MB `static_assert` in `MmuDeviceEn2::configure()` with a page-size to `xlat_seg_size` mapping.
- [ ] Decide compile-time (one build per size) or runtime page size. Runtime changes `pageSizeBytes()` and `getPageSize()` from `constexpr`.
- [ ] Re-derive pool sizes: pages = DDR size / page size, at most 2048 (hardware). See GAP-01 (#TBD) for the 1024-page driver limit.
- [ ] Recheck the RISC-V window. With `CONFIG_SYSTEM_MMU_MAX_PAGES_RV` = 64 it is 512 MB at 8 MB pages, 1 GB at 16 MB, 2 GB at 32 MB and 4 GB at 64 MB.
- [ ] Recheck the page-count alignment `static_assert`s (pages must be a multiple of 32) and any memory-map region sized from the page size.
- [ ] Define what `configure()` does when the boot parameters say the MMU is already enabled: adopt the configured size, or verify it matches the build.

## Interaction with GAP-01

At 8 MB pages, a 32 GB DDR needs 4096 entries, which is more than the hardware's 2048. Sizes below 16 MB cannot cover the production DDR. At 16 MB, 32 GB needs exactly 2048 entries, which requires GAP-01 (#TBD).

## Tests blocked by this gap

These tests **depend on driver support** and cannot pass until the changes above land.

**Coverage after these tests (cumulative, after P1, P2, P3 and GAP-01): 100% (+4.6%, 4 items).**

| ID | Candidate test | Executable | Items | What it checks |
|---|---|---|---|---|
| MMU-DRV-C-16 | Page-size sweep (8, 16, 32, 64 MB) | `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`): one build per size if the page size stays compile-time, or one run that sweeps all sizes if it becomes runtime | 3 | For each size: `getPageSize()` matches; allocate, map and free; a store through a translated address lands at `page_number x page size` (checked with translation off); free restores the counts. One item each for 8, 16 and 64 MB (32 MB is already covered). |
| MMU-DRV-C-17 | Per-size capacity and limits | `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`), same builds as C-16 | 1 | Total pages = DDR size / page size (capped at 2048); RISC-V window and `allocateRv` limits scale with the size; `getNumPages` rounding and `memMapRv` offsets are correct; with the MMU already enabled at boot, the driver adopts or verifies the segment size. |

If the page size stays compile-time, add the per-size variants to `tests/device/en2/nest/CMakeLists.txt` inside `if(ENABLE_EXTRA_METAL)`, for example by setting `CONFIG_SYSTEM_MMU_PAGE_SIZE_MB` per target.

## Where to add

Both tests are new `TEST_F` cases in `test-nest-mmu-bm.elf` (`tests/nest/nest-unit-tests-bm/src/TestMmuEn2.cpp`), which already maps the page size to `xlat_seg_size`. `test-nest-mmu-driver-bm.elf` needs no new test for this gap.

## Tracking

- [ ] Driver changes above
- [ ] MMU-DRV-C-16 Page-size sweep
- [ ] MMU-DRV-C-17 Per-size capacity and limits

## Definition of done

- Driver builds and runs on EN2 with 8, 16, 32 and 64 MB pages.
- Both tests added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target for every size.
- Existing tests (MMU-DRV-T-01 to T-25) still pass at 32 MB.
- Coverage table in the parent issue updated.
