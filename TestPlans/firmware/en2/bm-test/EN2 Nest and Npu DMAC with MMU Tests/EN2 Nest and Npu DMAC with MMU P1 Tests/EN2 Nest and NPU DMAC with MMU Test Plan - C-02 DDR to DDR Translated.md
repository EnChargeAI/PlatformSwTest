# Ticket: DMAC-MMU-C-02 DDR to DDR Translated

Parent issue: #TBD (EN2 Nest and NPU DMAC with MMU Test Plan). Group: P1 sub-issue #TBD.
Test description: `EN2 Nest and Npu DMAC with MMU Tests/EN2 Nest and Npu DMAC with MMU P1 Tests/EN2 Nest and NPU DMAC with MMU Test Plan - C-02 DDR to DDR Translated.md`

## Summary

Add a translated single-block DDR to DDR transfer on the Nest, with one client id on both the read and the write. Covers behaviour 14 (1 item).

## Driver GAP dependency

**None.** Uses the single-block path only. The linked-list variant is GAP-01 (DMAC-MMU-C-14) and is out of scope for this ticket.

## Dependencies

| Item | Type | Status |
|---|---|---|
| `stampClient`, `expectPage` fixture helpers | Shared, new | Added by whichever of C-01 to C-04 lands first |
| `physOf`, `readableByCore`, `freeClients`, `initMmu` | Existing in `TestDmacMmu.cpp` | Present |
| DDR to DDR without translation | Baseline | `TestDmacLife` T-01 and T-02 already pass |

## Work

- [ ] Add `stampClient` and `expectPage` to `DmacMmuFixture`, unless C-01 has already added them.
- [ ] Add a `TEST_F_CLEAN` case to `tests/common/test-bm/TestDmacMmu.cpp`, run for clients PF and VF7.
- [ ] Source page stamped `0xD0 + X`, destination page stamped `0xFF`.
- [ ] Translated `xfer` DDR to DDR, 256 bytes, id X. Translation off for the physical checks.
- [ ] Assert the destination holds `0xD0 + X`, the source is unchanged, and `IRPT_AXI` and `IRPT_LUT` are 0.
- [ ] Free every allocation on every exit path. Turn translation off in cleanup.
- [ ] Caution: a failing run writes 256 bytes at the physical address equal to the destination VA. Check the range is unused before running on hardware.

## Definition of done

- Test added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Existing `DmacMmu` tests still pass.
- Coverage table in the parent issue updated.
