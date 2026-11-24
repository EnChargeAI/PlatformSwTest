# EN2 MMU: P1 Candidate Tests

Parent issue: #TBD (EN2 MMU Test Plan)

P1 = driver capability not covered by any test, or a suspected defect. Add these bare-metal tests for EN2.

> **ID convention:** `MMU-DRV-T-nn` = **T**est exists. `MMU-DRV-C-nn` = **C**andidate test (to be written).

**Driver coverage with current tests: 60.9% (53 of 87 items).**
**Coverage after this group: 80.5% (+19.5%, 17 items).**

## Tests to add

| ID | Candidate test | Executable | Items | What it checks |
|---|---|---|---|---|
| MMU-DRV-C-01 | Invalid client IDs | `test-nest-mmu-driver-bm.elf` (`TestMmuDriver.cpp`) for `MmuDriver` APIs, and `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`) for `MmuMemApi` APIs | 1 | Ids 10 and 255 on every public API return false or do nothing. |
| MMU-DRV-C-02 | Index bounds | `test-nest-mmu-driver-bm.elf` (`TestMmuDriver.cpp`) | 1 | Pages `total-1` (valid), `total` and `total+1` (invalid) on `isAllocated`, `isCommitted`, `getPageOwner`, `freePhysicalPage`, `commitPageTablesSpan`. |
| MMU-DRV-C-03 | Page owner | `test-nest-mmu-driver-bm.elf` (`TestMmuDriver.cpp`) | 2 | Commit pages for VF1; `getPageOwner` returns VF1 for each. Verifies the function and the behaviour. |
| MMU-DRV-C-04 | Fixed-VA allocate | `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`) | 1 | `allocate(bytes, client, va)`: aligned VA succeeds; unaligned, beyond pool, overlapping, and pool end are handled. Accounting unchanged on failure. |
| MMU-DRV-C-05 | Uncovered APIs | `test-nest-mmu-bm.elf` (`TestMmuEn2.cpp`) for the 10 `MmuMemApi` functions, and `test-nest-mmu-driver-bm.elf` (`TestMmuDriver.cpp`) for `MmuDriver::disable` and `getNumPages` | 12 | Direct tests for `freeClient`, `disableClient`, `clearClientPages`, `isClientEnabled`, `isClientTranslationEnabled`, `isClientTranslationCacheEnabled`, `getNumPages`, `getNumFreeRvMemMapRecords`, `getClientNumAllocatedPages`, `getClientAllocatedPages` (including a too-small span), plus the two `MmuDriver` functions. |

## Notes (suspected from code reading, to confirm)

- **C-01:** `commitPageTablesSpan` rejects only `clientId > numTables` (10), and `allocate` compares against the size of the 10-entry configuration table, so id 10 passes both. The `MmuDriver` client helpers reject 10.
- **C-02:** The checks use `> numPagesPerTable`, which accepts index `numPagesPerTable` itself.
- **C-03:** `ownerId` is declared in `PageTableResourceRecord` but is not assigned in the driver, so `getPageOwner` appears to always return 0 (PF). `MMU-DRV-T-14` depends on it and would pass for any owner.

If a test confirms a defect, fix the driver or file a separate bug.

## Where to add

All tests are added to existing ELFs, as new `TEST_F` cases in the existing fixtures. Sources are in `tests/nest/nest-unit-tests-bm/src/`.

| Executable | Source file | Fixture | Tests |
|---|---|---|---|
| `test-nest-mmu-driver-bm.elf` | `TestMmuDriver.cpp` | Runs `MmuDriver::init(0)` | C-01 (`MmuDriver` APIs), C-02, C-03, C-05 (`MmuDriver::disable`, `getNumPages`) |
| `test-nest-mmu-bm.elf` | `TestMmuEn2.cpp` | Expects a fully free pool | C-01 (`MmuMemApi` APIs), C-04, C-05 (10 `MmuMemApi` functions) |

Both ELFs are registered in `tests/device/en2/nest/CMakeLists.txt` inside `if(ENABLE_EXTRA_METAL)`. No CMake change is needed.

## Tracking

- [ ] MMU-DRV-C-01 Invalid client IDs
- [ ] MMU-DRV-C-02 Index bounds
- [ ] MMU-DRV-C-03 Page owner
- [ ] MMU-DRV-C-04 Fixed-VA allocate
- [ ] MMU-DRV-C-05 Uncovered APIs

## Definition of done

- Tests added to the ELFs above, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Every test frees what it allocates and restores MMU state.
- Coverage table in the parent issue updated.
