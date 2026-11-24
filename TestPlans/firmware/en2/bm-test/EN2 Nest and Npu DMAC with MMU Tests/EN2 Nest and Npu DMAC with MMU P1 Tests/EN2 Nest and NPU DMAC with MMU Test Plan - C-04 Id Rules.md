# Ticket: DMAC-MMU-C-04 Id Rules

Parent issue: #TBD (EN2 Nest and NPU DMAC with MMU Test Plan). Group: P1 sub-issue #TBD.
Test description: `EN2 Nest and Npu DMAC with MMU Tests/EN2 Nest and Npu DMAC with MMU P1 Tests/EN2 Nest and NPU DMAC with MMU Test Plan - C-04 Id Rules.md`

## Summary

Pin down the two id rules on the Nest. With no id named, the transfer uses the default id 9 (the RISC-V client). The id is ignored when neither end of the transfer is LPDDR. Covers behaviours 19 and 20 (2 items).

## Driver GAP dependency

**None.** Descriptor-level checks plus translated hardware runs on existing DMAC paths. No MMU GAP-01 or GAP-02 dependency. No DMAC GAP.

## Dependencies

| Item | Type | Status |
|---|---|---|
| `ctlOf(const DmacLLHWDesc *)` | Shared, new | Added by the first of C-01 to C-03 to land |
| `stampClient` | Shared, new | As above |
| `dccmBuf_` (256 B, `.dccm_noinit`) | New fixture member | Add in this ticket |
| `processCtl`, `applyIid`, `setIid` | Existing in `drivers/dmac/src/Dmac.cpp` and `DmacDescriptor.hpp` | Present |
| `init` clears the id (T-04) | Existing test | Present |
| `MmuMemApi::disableClient` / `enableClient` | Implemented, not yet tested | Covered by MMU-DRV-C-05. This ticket relies on it. |

## Work

**NoIdNamed, descriptor level (no hardware):**

- [ ] Build a linked-list entry DDR to L3 with no `setIid`. Assert `__AR_CACHE == 9` and `__AW_CACHE == 9`.
- [ ] Call `setIid(5)`, then `init` again, build another entry. Assert both fields are 9.

**NoIdNamed, hardware (observation first, then assertion):**

- [ ] Enable VF3 and VF0. Allocate a VF3 filler page, then a VF0 page at VA 0. Stamp the VF0 page `0xD0`.
- [ ] Read the first 256 bytes at physical offset 0 with translation off and keep them.
- [ ] Translation on. Read DDR to L3 at VA 0 with no `setIid`.
- [ ] Compare against two candidates: the VF0 pattern, and the bytes from physical offset 0. The expected result from the code is physical offset 0. Log which candidate matched. If the hardware differs, record it and update the assertion and the plan.

**IdIgnoredForNonDdr, descriptor level:**

- [ ] With `setIid(5)`, build linked-list entries for each row and check the expected `AR_CACHE` / `AW_CACHE`:

| Source | Destination | `AR_CACHE` | `AW_CACHE` |
|---|---|---|---|
| L3 | DCCM | 9 | 9 |
| DCCM | L3 | 9 | 9 |
| DDR | L3 | 5 | 9 |
| L3 | DDR | 9 | 5 |
| DDR | DDR | 5 | 5 |

**IdIgnoredForNonDdr, hardware:**

- [ ] Disable one client with `MmuMemApi::disableClient`, then `setIid` with that client's id. Run L3 to DCCM single-block and linked-list with translation on. Assert both complete, the data matches, and `IRPT_AXI` and `IRPT_LUT` are 0.
- [ ] Re-enable the client before the test ends.

**General:**

- [ ] Free every allocation on every exit path. Turn translation off in cleanup.

## Notes

- `NoIdNamed` reads LPDDR offset 0 with the core. That is inside the DRAM window, so the read is safe.
- A controller-level DMAC error interrupt would hit the unimplemented `handleDmacControllerError` (DMAC GAP-02) and assert. A passing run does not raise one, so this is not a blocker.

## Definition of done

- Both test cases added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Frees what it allocates, turns translation off and clears `IRPT_AXI` and `IRPT_LUT`.
- Existing `DmacMmu` tests still pass.
- Coverage table in the parent issue updated.
