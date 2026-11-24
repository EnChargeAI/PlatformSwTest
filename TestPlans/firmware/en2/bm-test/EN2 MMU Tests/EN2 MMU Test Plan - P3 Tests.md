# EN2 MMU: P3 Candidate Tests

Parent issue: #TBD (EN2 MMU Test Plan)

P3 = nice-to-have tests. Add these bare-metal tests for EN2 after the P1 (#TBD) and P2 (#TBD) sub-issues.

> **ID convention:** `MMU-DRV-T-nn` = **T**est exists. `MMU-DRV-C-nn` = **C**andidate test (to be written).

**Coverage after P2: 90.8%.**
**Coverage after this group (cumulative): 93.1% (+2.3%, 2 items).**

## Tests to add

| ID | Candidate test | Executable | Items | What it checks |
|---|---|---|---|---|
| MMU-DRV-C-12 | Enable and disable semantics | `test-nest-mmu-driver-bm.elf` (`TestMmuDriver.cpp`) | 1 | `enable()` leaves the RISC-V translation bit unchanged; `disable()` clears all bits. |
| MMU-DRV-C-13 | Soak | **New:** `test-nest-mmu-soak-bm.elf` (`TestMmuSoak.cpp`) | 1 | Randomized alloc, free, map and unmap sequence checked against an in-test model; page and record counts match at the end. |

## New ELF

Add to `tests/device/en2/nest/CMakeLists.txt` inside `if(ENABLE_EXTRA_METAL)`:

```cmake
add_test_suite(ELF test-nest-mmu-soak-bm.elf SOURCES "${TEST_NEST_BM_PATH}/src/TestMmuSoak.cpp" LIBS mmu-driver mmu-driver-test-interface)
```

The soak test is long running, so it stays out of the regular ELFs.

| Executable | Source file | Tests |
|---|---|---|
| `test-nest-mmu-driver-bm.elf` (existing) | `tests/nest/nest-unit-tests-bm/src/TestMmuDriver.cpp` | C-12, as a new `TEST_F` case. No CMake change. |
| `test-nest-mmu-soak-bm.elf` (new) | `tests/nest/nest-unit-tests-bm/src/TestMmuSoak.cpp` | C-13 |

## Tracking

- [ ] MMU-DRV-C-12 Enable and disable semantics
- [ ] MMU-DRV-C-13 Soak

## Definition of done

- Tests added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Every test frees what it allocates and restores MMU state.
- Coverage table in the parent issue updated.
