# EN2 Nest and NPU DMAC with MMU: DMAC-MMU-C-02 DDR to DDR Translated

Parent issue: #TBD (EN2 Nest and NPU DMAC with MMU Test Plan). Group: P1 sub-issue #TBD.

> **ID convention:** `DMAC-MMU-C-nn` = **C**andidate test (to be written).

| | |
|---|---|
| Runs on | Nest |
| Executable | `test-nest-dmac-mmu-bm.elf` (existing) |
| Source | `tests/common/test-bm/TestDmacMmu.cpp`, as a new `TEST_F_CLEAN` case in `DmacMmuFixture`. No CMake change. |
| Items closed | 1: behaviour 14 (DDR to DDR translated, id on read and write) |
| Coverage | If P1 is done in order (after C-01): 64.4% to **66.1%** (+1.7%). Final P1 figure is 74.6%. |

## Goal

Run a single-block transfer with LPDDR at both ends. One id goes into **both** `AR_CACHE` and `AW_CACHE`, so both the read and the write must use that client's table.

## Background

- `applyIid` writes the id into `AR_CACHE` when the source is LPDDR and into `AW_CACHE` when the destination is LPDDR. With DDR at both ends, both fields carry the id.
- The default in either field is 9, the RISC-V client. `MmuMemApi::enable()` leaves it untranslated, so a field that missed the id would send that half of the transfer to the physical address equal to the VA.
- `TestDmacLife` already runs `DDR_DDR` without translation (T-01 and T-02). No test runs it with translation on.

## Fixture additions

| Addition | Purpose |
|---|---|
| `stampClient(client, va, pattern)` | Fills a client page by a translated L3 to DDR single-block write through `xfer`, as phase one of `UidSweep` does. |
| `expectPage(client, va, offset, expected, len)` | Turns translation off, finds the page with `physOf`, checks `readableByCore`, and compares `len` bytes. It wraps the read-back loop in `UidSweep`. |

These helpers are shared with C-01, C-03 and C-04. Whichever lands first adds them.

**Pattern:** `0xD0 + client`, so a page left behind by `UidSweep` (`0xC0 +`) or the disabled linked-list sweep (`0xB0 +`) cannot look like a pass.

## Steps

Repeat for PF and VF7.

1. Call `selectEngine()` and `initMmu()`. Enable client X. Allocate two pages for X: source at VA 0, destination at the next VA (two `allocate` calls). Assert the two VAs differ.
2. Stamp the source page with `0xD0 + X`. Stamp the destination page with `0xFF`, so an untouched destination is visible.
3. Clear `IRPT_AXI` and `IRPT_LUT`. Translation on. `xfer(src DDR lo, LPDDR hi, dst DDR lo, LPDDR hi, 256, X)`.
4. Turn translation off. Using `physOf` for each VA, assert:
    - The destination page holds 256 bytes of `0xD0 + X`.
    - The source page still holds `0xD0 + X`.
    - `IRPT_AXI` and `IRPT_LUT` are 0.
5. Free every allocation with `freeClients` on every exit path.

## Pass and failure

**Pass when** the destination page changed and the source did not. **A failure means:** if the id reached only the read side, the write would use id 9 and land at the physical address equal to the destination VA, leaving the destination page at `0xFF`. If it reached only the write side, the read would return the wrong data.

## Notes

- **Caution.** A failing run writes 256 bytes at the physical address equal to the destination VA. Check that range is not used by the image before running on hardware.
- **Page budget.** Each page is 32 MB, so two clients use four pages. Do not sweep all nine clients unless the page pool is large enough.
- **Linked-list variant.** A DDR to DDR linked-list write is GAP-01 (C-14), so this test uses the single-block path only.

## Tracking

- [ ] Fixture additions (`stampClient`, `expectPage`) in `DmacMmuFixture`, if C-01 has not added them
- [ ] DMAC-MMU-C-02 test case

## Definition of done

- Test added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Frees what it allocates, turns translation off and clears `IRPT_AXI` and `IRPT_LUT`.
- Existing `DmacMmu` tests still pass.
- Coverage table in the parent issue updated.
