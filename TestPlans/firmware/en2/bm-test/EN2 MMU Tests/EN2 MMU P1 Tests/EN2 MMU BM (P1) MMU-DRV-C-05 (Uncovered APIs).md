# EN2 Nest and NPU DMAC with MMU: DMAC-MMU-C-05 Non-DDR Traffic With Translation On

Parent issue: #TBD (EN2 Nest and NPU DMAC with MMU Test Plan). Group: P1 sub-issue #TBD.

> **ID convention:** `DMAC-MMU-C-nn` = **C**andidate test (to be written).

| | |
|---|---|
| Runs on | Nest |
| Executable | **New:** `test-nest-dmac-life-mmu-bm.elf`, built from `tests/common/test-bm/TestDmacLife.cpp` with a compile definition (for example `DMAC_LIFE_MMU_ENABLED`) |
| Source | `tests/common/test-bm/TestDmacLife.cpp` (recompiled) |
| Items closed | 1: behaviour 21 (non-DDR traffic unaffected by translation on) |
| Coverage | If P1 is done in order (after C-01 to C-04): 72.9% to **74.6%** (+1.7%). Final P1 figure is 74.6%. |

## Goal

The existing `TestDmacLife` pairs that do not touch LPDDR must pass exactly as before when the MMU is enabled. This shows that enabling translation does not disturb L3, DCCM or RVSRAM traffic, including the Nest's own access to its LPDDR image.

## Background

- `TestDmacLife` already runs these pairs without translation (T-01 and T-02). This test runs the same pairs with translation on.
- The skipped DDR pairs name no id, so they would run untranslated and prove nothing here. C-02 and C-04 cover DDR with translation on.
- A new ELF is needed because the MMU setup belongs in the fixture's `setUp`. The existing `test-nest-dmac-life-bm.elf` must keep running without translation.

## Steps

### Changes in `TestDmacLife.cpp`, only under the compile definition

1. Include `MmuDriver.hpp`, `MmuMemApi.hpp` and `eai_nest_mmu_csr_nriscv.hpp`.
2. In `setUp`, after `enableNestDmacPrerequisites()`: assert `MmuDriver::init(0)`, call `MmuMemApi::enable()`, and clear `IRPT_AXI` and `IRPT_LUT`. Enable no client.
3. In `tearDown`: call `MmuMemApi::disable()`.
4. At the end of each case, assert `IRPT_AXI == 0` and `IRPT_LUT == 0`.
5. Register only the pairs with no LPDDR end. Leave the DDR pairs out of this ELF.

### Pairs under test

| Run (9 pairs, single block and linked list, 18 cases) | Skipped (4 pairs, 8 cases) |
|---|---|
| `L3_L3`, `DCCM_DCCM`, `DCCM_L3`, `L3_DCCM`, `RVSRAM_RVSRAM`, `RVSRAM_DCCM`, `DCCM_RVSRAM`, `RVSRAM_L3`, `L3_RVSRAM` | `DDR_L3`, `DDR_DCCM`, `DDR_DDR`, `DDR_RVSRAM` |

### CMake

Add to `tests/device/en2/nest/CMakeLists.txt`, next to `test-nest-dmac-life-bm.elf`. Check the argument order against `add_test_suite`:

```cmake
add_test_suite(ELF test-nest-dmac-life-mmu-bm.elf SOURCES "${TEST_COMMON_PATH}/TestDmacLife.cpp" LDS nest_lpddr_test.lds LIBS mmu-driver mmu-driver-test-interface DEFINES -DDMAC_LIFE_MMU_ENABLED=1)
```

## Pass and failure

**Pass when** every case passes by the existing control-copy comparison, and no MMU status bit is set. **A failure means:** enabling the MMU disturbed non-DDR traffic. A hang means an MMU setting broke the core's own access to its LPDDR image, which the untranslated RISC-V client should prevent.

## Notes

- **Image location.** The Nest image runs from LPDDR, so keep the RISC-V client untranslated. `MmuMemApi::enable()` already does this.
- **Existing ELF.** `test-nest-dmac-life-bm.elf` is unchanged. Only the new ELF has the MMU setup.
- **Pair count.** The table counts 9 run pairs and 4 skipped, for 13 pairs total, matching the 13 Life pairs on the Nest.

## Tracking

- [ ] Compile definition and MMU setup in `TestDmacLife.cpp`, under `DMAC_LIFE_MMU_ENABLED`
- [ ] CMake line for `test-nest-dmac-life-mmu-bm.elf`
- [ ] Run and record the 18 cases on target

## Definition of done

- The new ELF builds for EN2 with `ENABLE_EXTRA_METAL` and passes on target.
- The existing Life and HwSpeed ELFs still pass without translation.
- Coverage table in the parent issue updated.
