# Ticket: DMAC-MMU-C-03 Page Crossing and Non-Zero VA

Parent issue: #TBD (EN2 Nest and NPU DMAC with MMU Test Plan). Group: P1 sub-issue #TBD.
Test description: `EN2 Nest and Npu DMAC with MMU Tests/EN2 Nest and Npu DMAC with MMU P1 Tests/EN2 Nest and NPU DMAC with MMU Test Plan - C-03 Page Crossing and Non-Zero VA.md`

## Summary

Add two translated Nest transfers: one that starts at a non-zero offset inside a page, and one that crosses from one VA page into the next, where the two pages are not physical neighbours. Covers behaviours 15 and 16 (2 items).

## Driver GAP dependency

**None.** Uses 32 MB pages, which the current driver supports. No MMU GAP-01 or GAP-02 dependency. No DMAC GAP.

## Dependencies

| Item | Type | Status |
|---|---|---|
| `stampClient`, `expectPage` fixture helpers | Shared, new | Added by whichever of C-01 to C-04 lands first |
| `MmuMemApi::allocate(bytes, client, va)` with non-zero VA | Implemented driver feature, not yet tested | Covered by MMU-DRV-C-04. This ticket relies on it working. |
| `physOf`, `readableByCore`, `freeClients`, `initMmu` | Existing in `TestDmacMmu.cpp` | Present |
| First-fit physical page allocation | Relied on by `PageCrossing` | Asserted in step 2 of the test |

## Work

- [ ] Add `stampClient` and `expectPage` to `DmacMmuFixture`, if not already added.
- [ ] `NonZeroVaOffset`: allocate a filler page for VF3, then one VF0 page at VA 0. Write 256 bytes at `VA + 0x12340` with id VF0. Check the bytes at `physOf(VA, VF0) + offset` match, the 32 bytes either side are unchanged, and offset 0 of the page does not hold the pattern. Read back and compare.
- [ ] `PageCrossing`: allocate VF0 page A at VA 0, a VF3 filler page, then VF0 page B at VA = page size. Assert `physOf(pageSize, VF0) != physOf(0, VF0) + pageStride_`. If they are equal, fail with a log rather than pass.
- [ ] `PageCrossing`: write 256 bytes at VA `pageSize - 128` with id VF0. Bytes 0 to 127 land at the end of page A, bytes 128 to 255 at the start of page B. Check the filler page's first 128 bytes are unchanged. Read back and compare.
- [ ] Free every allocation on every exit path. Turn translation off and clear `IRPT_AXI` and `IRPT_LUT`.

## Notes

- Page budget: `PageCrossing` uses three pages of 32 MB, `NonZeroVaOffset` uses two.
- A 32 MB boundary is also a 4 KB boundary, so bursts split there. A non-page-aligned crossing is not covered.

## Definition of done

- Both test cases added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Frees what it allocates, turns translation off and clears `IRPT_AXI` and `IRPT_LUT`.
- Existing `DmacMmu` tests still pass.
- Coverage table in the parent issue updated.
