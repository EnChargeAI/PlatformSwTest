# EN2 MMU: P2 Candidate Tests

Parent issue: #TBD (EN2 MMU Test Plan)

P2 = important robustness tests. Add these bare-metal tests for EN2. Do these after the P1 sub-issue (#TBD).

> **ID convention:** `MMU-DRV-T-nn` = **T**est exists. `MMU-DRV-C-nn` = **C**andidate test (to be written).

**Coverage after P1: 80.5%.**
**Coverage after this group (cumulative): 90.8% (+10.3%, 9 items).**

## Tests to add

| ID | Candidate test | Executable | Items | What it checks |
|---|---|---|---|---|
| MMU-DRV-C-06 | Size edges | `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`) | 2 | Zero bytes and `UINT64_MAX` are rejected. 16 and 17 pages (stack vs heap path) behave the same for `allocate` and `memMapRv`. |
| MMU-DRV-C-07 | `memMapRv` edges | `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`) | 3 | The (client, VA) overload; second map of one allocation; more than 16 pages; mapping-record exhaustion; mapping an RV allocation; double and invalid unmap; free with an active mapping; double free. |
| MMU-DRV-C-08 | Fragmentation | `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`) | 1 | Interleaved alloc and free; contiguous request fails although total free pages suffice; VA reuse; RV 64-page limit vs physical exhaustion. |
| MMU-DRV-C-09 | Shared allocation rollback | `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`) | 1 | A conflicting range in one client fails the shared allocation partway with nothing leaked. After success, RISC-V reads and writes through the shared VA work. |
| MMU-DRV-C-10 | `init` edges | **New:** `test-nest-mmu-init-bm.elf` (`TestMmuInit.cpp`) | 1 | Second call with a different `startPage` returns false; `startPage` above the table size; reserved RISC-V entries preserved when `startPage != 0`; `isMmuEnabled` boot-parameter path in `configure()`. |
| MMU-DRV-C-11 | Thread safety | **New:** `test-nest-mmu-mt-bm.elf` (`TestMmuMultiHart.cpp`) | 1 | Several harts allocate and free against one pool; no lost or duplicate pages, no corrupted records. |

## New ELFs

Add to `tests/device/en2/nest/CMakeLists.txt` inside `if(ENABLE_EXTRA_METAL)`, following the existing pattern:

```cmake
add_test_suite(ELF test-nest-mmu-init-bm.elf SOURCES "${TEST_NEST_BM_PATH}/src/TestMmuInit.cpp" LIBS mmu-driver mmu-driver-test-interface)
add_test_suite(ELF test-nest-mmu-mt-bm.elf SOURCES "${TEST_NEST_BM_PATH}/src/TestMmuMultiHart.cpp" LIBS mmu-driver mmu-driver-test-interface)
```

- **C-10:** `init` state is once per boot, so first-call behaviour needs a fresh ELF.
- **C-11:** needs a second hart started. See `TestHart1.cpp` for how EN2 starts one.

C-06 to C-09 go into the existing `test-nest-mmu-bm.elf` as new `TEST_F` cases in `tests/nest/nest-unit-tests-bm/src/TestMmuEn2.cpp` (`MmuMemApi` level, fixture expects a fully free pool). No CMake change is needed for them.

| Executable | Source file | Tests |
|---|---|---|
| `test-nest-mmu-bm.elf` (existing) | `TestMmuEn2.cpp` | C-06, C-07, C-08, C-09 |
| `test-nest-mmu-init-bm.elf` (new) | `TestMmuInit.cpp` | C-10 |
| `test-nest-mmu-mt-bm.elf` (new) | `TestMmuMultiHart.cpp` | C-11 |

## Tracking

- [ ] MMU-DRV-C-06 Size edges
- [ ] MMU-DRV-C-07 `memMapRv` edges
- [ ] MMU-DRV-C-08 Fragmentation
- [ ] MMU-DRV-C-09 Shared allocation rollback
- [ ] MMU-DRV-C-10 `init` edges
- [ ] MMU-DRV-C-11 Thread safety

## Definition of done

- Tests added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Every test frees what it allocates and restores MMU state.
- Coverage table in the parent issue updated.
