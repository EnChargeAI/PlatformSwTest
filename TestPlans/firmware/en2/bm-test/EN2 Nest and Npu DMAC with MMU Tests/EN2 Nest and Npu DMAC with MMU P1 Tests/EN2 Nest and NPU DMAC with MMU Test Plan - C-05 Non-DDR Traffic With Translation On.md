# Ticket: DMAC-MMU-C-05 Non-DDR Traffic With Translation On

Parent issue: #TBD (EN2 Nest and NPU DMAC with MMU Test Plan). Group: P1 sub-issue #TBD.
Test description: `EN2 Nest and Npu DMAC with MMU Tests/EN2 Nest and Npu DMAC with MMU P1 Tests/EN2 Nest and NPU DMAC with MMU Test Plan - C-05 Non-DDR Traffic With Translation On.md`

## Summary

Run the existing `TestDmacLife` pairs that do not touch LPDDR with the MMU enabled. They must pass as they do without the MMU. Shows that translation does not disturb L3, DCCM or RVSRAM traffic, or the Nest's own access to its LPDDR image. Covers behaviour 21 (1 item).

## Driver GAP dependency

**None.** Uses `MmuDriver::init(0)`, `MmuMemApi::enable()` and `MmuMemApi::disable()`, all implemented. No MMU GAP-01 or GAP-02 dependency. No DMAC GAP.

## Dependencies

| Item | Type | Status |
|---|---|---|
| `TestDmacLife` pairs without translation (T-01, T-02) | Baseline | Passing |
| `enableNestDmacPrerequisites()` | Existing fixture call | Present |
| `MmuDriver::init`, `MmuMemApi::enable` / `disable` | Implemented driver API | Present |
| New compile definition `DMAC_LIFE_MMU_ENABLED` | New, in `TestDmacLife.cpp` | Add in this ticket |
| New CMake target `test-nest-dmac-life-mmu-bm.elf` | New, in `tests/device/en2/nest/CMakeLists.txt` | Add in this ticket |

The DDR pairs are excluded here. C-02 and C-04 cover DDR with translation on.

## Work

- [ ] In `tests/common/test-bm/TestDmacLife.cpp`, under `DMAC_LIFE_MMU_ENABLED`:
  - [ ] Include `MmuDriver.hpp`, `MmuMemApi.hpp` and `eai_nest_mmu_csr_nriscv.hpp`.
  - [ ] In `setUp`, after `enableNestDmacPrerequisites()`: assert `MmuDriver::init(0)`, call `MmuMemApi::enable()`, clear `IRPT_AXI` and `IRPT_LUT`. Enable no client.
  - [ ] In `tearDown`: call `MmuMemApi::disable()`.
  - [ ] At the end of each case, assert `IRPT_AXI == 0` and `IRPT_LUT == 0`.
  - [ ] Register only the non-DDR pairs.
- [ ] Add to `tests/device/en2/nest/CMakeLists.txt`, next to `test-nest-dmac-life-bm.elf`:

```cmake
add_test_suite(ELF test-nest-dmac-life-mmu-bm.elf SOURCES "${TEST_COMMON_PATH}/TestDmacLife.cpp" LDS nest_lpddr_test.lds LIBS mmu-driver mmu-driver-test-interface DEFINES -DDMAC_LIFE_MMU_ENABLED=1)
```

- [ ] Run and record the 18 cases on target.

**Pairs under test**

| Run (9 pairs, 18 cases, single block and linked list) | Skipped (4 pairs, 8 cases) |
|---|---|
| `L3_L3`, `DCCM_DCCM`, `DCCM_L3`, `L3_DCCM`, `RVSRAM_RVSRAM`, `RVSRAM_DCCM`, `DCCM_RVSRAM`, `RVSRAM_L3`, `L3_RVSRAM` | `DDR_L3`, `DDR_DCCM`, `DDR_DDR`, `DDR_RVSRAM` |

## Notes

- The Nest image runs from LPDDR, so the RISC-V client stays untranslated. `MmuMemApi::enable()` already does this.
- `test-nest-dmac-life-bm.elf` is unchanged. Only the new ELF has the MMU setup.
- A hang would mean an MMU setting broke the core's access to its LPDDR image.

## Definition of done

- The new ELF builds for EN2 with `ENABLE_EXTRA_METAL` and passes on target.
- The existing Life and HwSpeed ELFs still pass without translation.
- Coverage table in the parent issue updated.
