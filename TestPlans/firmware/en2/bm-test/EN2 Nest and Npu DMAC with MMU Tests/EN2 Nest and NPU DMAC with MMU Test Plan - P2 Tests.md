# EN2 Nest and NPU DMAC with MMU: P2 Candidate Tests

Parent issue: #TBD (EN2 Nest and NPU DMAC with MMU Test Plan)

P2 = NPU and CG translated transfers, and robustness tests. Do these after the P1 sub-issue (#TBD).

> **ID convention:** `DMAC-MMU-T-nn` = **T**est exists. `DMAC-MMU-C-nn` = **C**andidate test (to be written).

**Coverage after P1: 74.6%.**
**Coverage after this group (cumulative): 91.5% (+16.9%, 10 items).**

## Tests to add

| ID | Candidate test | Runs on | Executable | Items | What it checks |
|---|---|---|---|---|---|
| DMAC-MMU-C-06 | NPU linked list translated | Paired | `test-nest-dmac-mmu-npu-bm.elf` (`TestNestDmacMmuNpu.cpp`) + `test-npu-dmac-mmu-stub.elf` (`TestNpuDmacMmuStubMain.cpp`) | 1 | The stub runs a linked-list read from the DDR VA with the requested id. The Nest stamps the pages, checks the stub result, and checks the data matches the page of that client. DDR source only; the translated write is GAP-01. |
| DMAC-MMU-C-07 | NPU DDR to DDR translated | Paired | same pair | 1 | The stub copies from one client VA to another through DDR (id on read and write), then reads the destination back. The Nest checks both physical pages. |
| DMAC-MMU-C-08 | Every controller and engine, engines in parallel | Nest, and Paired for the NPU | `test-nest-dmac-mmu-bm.elf` (`TestDmacMmu.cpp`); NPU through the same pair | 2 | Repeat the id sweep on every controller and engine, not only controller 0 engine 0. Start both engines of one controller at once with different ids; each must hit its own page. The NPU request gains `controller` and `engine` fields. |
| DMAC-MMU-C-09 | Remap between transfers | Nest, and Paired for the NPU | `test-nest-dmac-mmu-bm.elf` (`TestDmacMmu.cpp`); NPU through the same pair | 1 | Transfer to a VA, free and allocate so the same VA maps to a different physical page, transfer again. The second transfer must land in the new page and leave the old one untouched. |
| DMAC-MMU-C-10 | Descriptor limits | Both, same source | **New:** `test-nest-dmac-desc-bm.elf` and `test-npu-dmac-desc-bm.elf` (`TestDmacDescriptor.cpp`) | 3 | Size below 32 B and above 16 MB; address and size not aligned to the region width; invalid region; full pool in `addEntry`; null or empty pool in `init`; the 64-bit `init` overload; `logDesc` on a built entry and on null. No MMU access, so the same source runs on both processors. |
| DMAC-MMU-C-11 | Controller reset and re-enable | Nest | `test-nest-dmac-mmu-bm.elf` (`TestDmacMmu.cpp`) | 1 | `disableDmac` then `enableDmac` called directly, then a translated transfer works. Today `enableDmac` is only reached through `init`. |
| DMAC-MMU-C-16 | CG translated write and read per client id | Paired | `test-nest-dmac-mmu-cg-bm.elf` (new, `TestNestDmacMmuNpu.cpp` with a CG define) + `test-cg-dmac-mmu-stub.elf` (new, `TestNpuDmacMmuStubMain.cpp` in the CG build) | 1 | CG programs the client id on its DMAC, and the stub writes then reads back through that id. The Nest checks that each id's physical page holds that client's pattern, which shows that CG's id selected the right table. Untranslated CG transfers are already covered by T-01. |

## New ELFs

Add to `tests/device/en2/nest/CMakeLists.txt` and `tests/device/en2/npu/CMakeLists.txt`, next to the existing DMAC lines:

```cmake
# Nest
add_test_suite(ELF test-nest-dmac-desc-bm.elf SOURCES "${TEST_COMMON_PATH}/TestDmacDescriptor.cpp")
# NPU
add_test_suite(ELF test-npu-dmac-desc-bm.elf SOURCES "${TEST_COMMON_PATH}/TestDmacDescriptor.cpp")

# CG peer stub for C-16 (add to tests/device/en2/cg/CMakeLists.txt; check the CG defines against the existing CG ELFs)
add_test_suite(ELF test-cg-dmac-mmu-stub.elf SOURCES "${TEST_COMMON_PATH}/TestNpuDmacMmuStubMain.cpp")
```

## Where to add

| Executable | Source file | Processor | Tests |
|---|---|---|---|
| `test-nest-dmac-mmu-npu-bm.elf` (existing) | `tests/nest/nest-unit-tests-bm/src/TestNestDmacMmuNpu.cpp` | Nest | Nest side of C-06 to C-09 (NPU part) |
| `test-nest-dmac-mmu-cg-bm.elf` (new) | `tests/nest/nest-unit-tests-bm/src/TestNestDmacMmuNpu.cpp` (CG define) | Nest | Nest side of C-16 |
| `test-cg-dmac-mmu-stub.elf` (new) | `tests/common/test-bm/TestNpuDmacMmuStubMain.cpp` | CG | CG side of C-16 |
| `test-npu-dmac-mmu-stub.elf` (existing) | `tests/common/test-bm/TestNpuDmacMmuStubMain.cpp` | NPU | NPU side of C-06 to C-09 |
| `test-nest-dmac-mmu-bm.elf` (existing) | `tests/common/test-bm/TestDmacMmu.cpp` | Nest | C-08, C-09 (Nest part), C-11 |
| `test-nest-dmac-desc-bm.elf`, `test-npu-dmac-desc-bm.elf` (new) | `tests/common/test-bm/TestDmacDescriptor.cpp` | Nest and NPU | C-10 |

The shared request and result are in `tests/common/test-bm/NpuDmacMmuTestInterface.hpp`.

## Notes

- **The NPU side runs what it is handed for translated tests.** The MMU is pre-programmed by the Nest, and the NPU never programs it. Every C-06 to C-09 check that needs the tables happens on the Nest.
- **NPU standalone, untranslated.** The NPU can address L3, DDR and DCCM, and `TestDmacLife` and `TestDmacHwSpeed` already run those pairs on the NPU. No new standalone NPU test is proposed here. A standalone translated NPU test is not possible, because the NPU cannot program the MMU.
- **Extend the request once.** C-06 to C-09 need new fields in `NpuDmacMmuRequest`: a `mode` (single block or linked list), a source address for DDR to DDR, `controller` and `engine`. Add them together, and keep the existing three tests working.
- **NPU linked-list placement.** The NPU RISC-V cannot write DDR, so C-06 keeps its descriptors in DCCM, and only the data is in DDR. The DMAC can reach NPU DCCM. A code comment in `DmacTestInterface.hpp` says the Nest's LLP fetch fails from DCCM and L3. No test result backs it up, so confirm the NPU fetch from DCCM on hardware. T-02 already runs NPU linked lists; if it uses DCCM, this is already proven.
- **NPU buffer placement.** The stub currently uses DCCM. The stub comment says the NPU "reaches no L3", but the NPU BSP maps L3. Verify this before moving the buffers to L3.
- **CG side.** CG has no MMU and cannot program it. The Nest pre-programs the tables, and CG programs the client id on its DMAC. C-16 checks that the id CG sends selects the right table. The CG copy of the control channel buffer must be addressed by the Nest, which needs its own view tag (see Open questions in the parent issue).
- **Run order:** peer stub (NPU or CG) first, then the Nest ELF. One pair at a time.
- **C-08** on the Nest should skip controllers or engines the platform does not provide, and report how many it found.

## Tracking

- [ ] DMAC-MMU-C-06 NPU linked list translated
- [ ] DMAC-MMU-C-07 NPU DDR to DDR translated
- [ ] DMAC-MMU-C-08 Every controller and engine, engines in parallel
- [ ] DMAC-MMU-C-09 Remap between transfers
- [ ] DMAC-MMU-C-10 Descriptor limits
- [ ] DMAC-MMU-C-11 Controller reset and re-enable
- [ ] DMAC-MMU-C-16 CG translated write and read per client id

## Definition of done

- Tests added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target. Paired tests are run as a pair, and both ELFs are listed in the result.
- Every test frees what it allocates, turns translation off and clears `IRPT_AXI` and `IRPT_LUT`.
- The existing `NpuTranslatedTransfer` tests still pass after the request is extended.
- Coverage table in the parent issue updated.
