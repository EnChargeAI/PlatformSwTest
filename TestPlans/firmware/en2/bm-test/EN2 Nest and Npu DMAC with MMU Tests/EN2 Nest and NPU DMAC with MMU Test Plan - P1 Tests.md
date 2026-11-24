# EN2 Nest and NPU DMAC with MMU: P1 Candidate Tests

Parent issue: #TBD (EN2 Nest and NPU DMAC with MMU Test Plan)

P1 = translated DMAC paths and driver functions that no test covers. These tests run on the **Nest**: the Nest owns the MMU, so Nest-side coverage comes first and the NPU pair tests (P2) build on it.

> **ID convention:** `DMAC-MMU-T-nn` = **T**est exists. `DMAC-MMU-C-nn` = **C**andidate test (to be written).

**Driver coverage with current tests: 57.6% (34 of 59 items).**
**Coverage after this group: 74.6% (+16.9%, 10 items).**

## Tests to add

| ID | Candidate test | Runs on | Executable | Items | What it checks |
|---|---|---|---|---|---|
| DMAC-MMU-C-01 | Multi-entry linked list, translated, one id for the whole list | Nest | `test-nest-dmac-mmu-bm.elf` (`TestDmacMmu.cpp`) | 4 | Build a list of four entries that read four 256 B slices of one client's page (VF3). The list carries one client id, set once before the first `addEntry`. Each entry must read its own slice. Also asserts `getCnt`, `getPoolCnt` and `getLLHead` against the list built. |
| DMAC-MMU-C-02 | DDR to DDR translated | Nest | `test-nest-dmac-mmu-bm.elf` (`TestDmacMmu.cpp`) | 1 | Source and destination both in LPDDR. The id must apply to the read and the write. Checked against the physical pages with translation off. |
| DMAC-MMU-C-03 | Page crossing and non-zero VA | Nest | `test-nest-dmac-mmu-bm.elf` (`TestDmacMmu.cpp`) | 2 | A transfer that starts inside a page at a non-zero offset, and one that spans two adjacent VA pages mapped to different physical pages. Each half must land in the right page. |
| DMAC-MMU-C-04 | Id rules | Nest | `test-nest-dmac-mmu-bm.elf` (`TestDmacMmu.cpp`) | 2 | (a) Translation on and no `setIid`: record which client table is used. (b) `setIid` on a transfer with no DDR end (L3 to DCCM): the id must have no effect, as `isIidProgrammingRequired` says. |
| DMAC-MMU-C-05 | Non-DDR traffic with translation on | Nest | **New:** `test-nest-dmac-life-mmu-bm.elf` (`TestDmacLife.cpp`, recompiled) | 1 | The L3, DCCM and RVSRAM pairs of `Dmac` (single block and linked list) run with translation enabled. They must pass exactly as without the MMU. |

## Notes

- **C-01 reads DDR, it does not write it.** A linked-list write to DDR fails on the Nest (GAP-01), so C-01 uses a DDR source. The translated write is C-14.
- **C-03** needs a client with more than one page. Allocate two pages for one client and check `physOf` for each before the transfer.
- **C-04(a)** is an observation first: write down which table the hardware uses, then turn it into an assertion.
- **C-05:** Life already supports being recompiled per target. Add a compile definition (for example `DMAC_LIFE_MMU_ENABLED`) so the fixture runs `MmuDriver::init(0)` and `MmuMemApi::enable()` in `setUp` and `disable()` in `tearDown`. Run the non-DDR pairs only; the DDR pairs are covered by C-02 and C-04. The image runs from LPDDR on the Nest, so keep the RISC-V client untranslated (`enable()` already does).
- **Existing test files:** `TestDmacMmu.cpp` is in `tests/common/test-bm` but only the EN2 Nest CMake file builds it today.

## New ELF

Add to `tests/device/en2/nest/CMakeLists.txt`, next to `test-nest-dmac-life-bm.elf`. Check the argument order against `add_test_suite`:

```cmake
add_test_suite(ELF test-nest-dmac-life-mmu-bm.elf SOURCES "${TEST_COMMON_PATH}/TestDmacLife.cpp" LDS nest_lpddr_test.lds LIBS mmu-driver mmu-driver-test-interface DEFINES -DDMAC_LIFE_MMU_ENABLED=1)
```

## Where to add

| Executable | Source file | Tests |
|---|---|---|
| `test-nest-dmac-mmu-bm.elf` (existing) | `tests/common/test-bm/TestDmacMmu.cpp` | C-01 to C-04, as new `TEST_F_CLEAN` cases in `DmacMmuFixture`. No CMake change. |
| `test-nest-dmac-life-mmu-bm.elf` (new) | `tests/common/test-bm/TestDmacLife.cpp` | C-05 |

## Tracking

- [ ] DMAC-MMU-C-01 Multi-entry linked list, translated, one id for the whole list
- [ ] DMAC-MMU-C-02 DDR to DDR translated
- [ ] DMAC-MMU-C-03 Page crossing and non-zero VA
- [ ] DMAC-MMU-C-04 Id rules
- [ ] DMAC-MMU-C-05 Non-DDR traffic with translation on

## Definition of done

- Tests added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Every test frees what it allocates, turns translation off and clears `IRPT_AXI` and `IRPT_LUT`.
- The existing Life and HwSpeed ELFs still pass without translation.
- Coverage table in the parent issue updated.
