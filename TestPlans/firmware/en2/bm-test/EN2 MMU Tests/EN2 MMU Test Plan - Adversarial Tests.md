# EN2 MMU: Adversarial Candidate Tests

Parent issue: #TBD (EN2 MMU Test Plan)
Related: [#4043](https://github.com/EnChargeAI/EAI_Firmware/issues/4043)

Adversarial = tests that make illegal accesses on purpose and check that the MMU reports the fault and the Nest stays responsive. Add these bare-metal tests for EN2 in their own ELF. They are independent of the P1 (#TBD), P2 (#TBD) and P3 (#TBD) sub-issues.

> **ID convention:** `MMU-ADV-C-nn` = **C**andidate adversarial test (to be written). No adversarial test exists today.

**Coverage: not counted.** These tests check hardware fault reporting. The driver does not read `IRPT_AXI` / `IRPT_LUT` and always sets `write_en`, so they add no items to the 87-item driver metric.

## Tests to add

| ID | Candidate test | Executable | What it checks |
|---|---|---|---|
| MMU-ADV-C-01 | `UnmappedVa` | **New:** `test-mmu-hw-adversarial-bm.elf` | With `xlat_en` on, access a VA whose PTE has `valid = 0`. Expect `buserr` set, and the IRQ if unmasked. Clear the error and IRQ. Check the Nest is still responsive. |
| MMU-ADV-C-02 | `Modify0Write` | same | Commit a PTE with `valid = 1` and `Modify` / `write_en = 0`. Write through that VA. Expect AXI `buserr`, and the IRQ if unmasked. Clear the error. Restore the PTE. |
| MMU-ADV-C-03 | `DisabledClientEn` | same | Use a spare IID (not the Nest RISC-V). Set `client_en` off and `xlat_en` on, then generate traffic for that IID. Expect AXI `buserr`, and the IRQ if unmasked. Clear the error. Restore `client_en`. If IID 9 is used, do not disable it after the test. |
| MMU-ADV-C-04 | `ApbPslverr` | same | Assert `IRPT_AXI.pslverr` is clear and unmask the `pslverr` IRQ. From the Nest RISC-V, issue an APB access in the MMU CSR/TBL range that asserts `PSLVERR` (for example, read an unimplemented CSR). Assert `IRPT_AXI.pslverr` is set. Clear the bit. Restore the initial mask state. |
| MMU-ADV-C-05 | `IrqMaskedVsUnmasked` | same | Generate an AXI fault with the `UnmappedVa` or `Modify0Write` method, once with the interrupt masked and once unmasked. Verify `buserr` is set in both cases and the IRQ fires only when unmasked. |

## Common structure

```mermaid
flowchart TDs
    save["Save mask, client_en and PTE state"] --> clear["Clear IRPT_AXI and IRPT_LUT"]
    clear --> arm["Mask or unmask the IRQ"]
    arm --> inject["Inject the fault"]
    inject --> check["Check status bit; check IRQ fired or not"]
    check --> clearFault["Clear status bit and IRQ"]
    clearFault --> alive["Check Nest is responsive: CSR read and DDR round trip"]
    alive --> restore["Restore PTE, client_en and mask"]
```

## Where to add

All five tests go in one new ELF. None go into the existing MMU ELFs.

| Executable | Source file | Tests |
|---|---|---|
| `test-mmu-bounds-adversarial-bm.elf` (new) | `tests/nest/nest-unit-tests-bm/src/TestMmuBoundsAdversarial.cpp` | C-01 to C-05 |

C-03 also needs NPU DMAC traffic. Reuse the NPU stub from `test-nest-dmac-mmu-npu-bm.elf` (`TestNestDmacMmuNpu.cpp`) and do not add the test there.

## New ELF

Add to `tests/device/en2/nest/CMakeLists.txt` inside `if(ENABLE_EXTRA_METAL)`:

```cmake
add_test_suite(ELF test-mmu-bounds-adversarial-bm.elf SOURCES "${TEST_NEST_BM_PATH}/src/TestMmuBoundsAdversarial.cpp" LIBS mmu-driver mmu-driver-test-interface)
```

The name comes from #4043. The other EN2 MMU ELFs use the `test-nest-*-bm.elf` form, so confirm whether to rename it. Keep these tests out of the regular MMU ELFs: a wedged bus would stop unrelated tests.

## Notes and open questions

- **Possible conflict with existing observations.** `TestMmuEn2.cpp` (`SegmentIndexSweep`, `TwoSegmentAperture`) and `TestNestDmacMmuNpu.cpp` say an unmapped access "returns no error and wedges the bus". C-01 expects a `buserr` and a responsive Nest. Either the hardware has changed or the earlier case was different (for example, a valid entry pointing beyond the DRAM window). Resolve this before writing C-01. If the bus still wedges, raise a hardware issue.
- **Protect the Nest.** The Nest RISC-V is IID 9. Keep test code and stack in RVSRAM, and fault only a VA window that holds neither. Run under a watchdog or timeout.
- **C-03 needs a second bus master.** A spare IID means NPU DMAC traffic, as in `test-nest-dmac-mmu-npu-bm.elf` with its NPU stub.
- **IRQ support.** The EN2 Nest interrupt table has `MMU_BUS_ERROR` and `MMU_PAGE_TABLE_ERROR` (`BspPlatformInterruptPriorityEn2Nest.cpp`). Confirm which register each one reflects (presumably `IRPT_AXI` and `IRPT_LUT`). `TestCgInterrupts.cpp` and `TestDummyInterrupt.cpp` show the handler pattern.
- **Register field names.** `buserr` and `pslverr` come from #4043. Confirm the field names and mask registers against the EN2 MMU CSR map.
- **Ordering.** Run C-01 first because it carries the highest risk. C-05 reuses the C-01 and C-02 injection code.
- **See also.** C-03 relates to `MMU-DRV-C-12` (P3, enable and disable semantics). C-12 checks the driver sets the bits. C-03 checks what the hardware does with them off.

## Tracking

- [ ] MMU-ADV-C-01 `UnmappedVa`
- [ ] MMU-ADV-C-02 `Modify0Write`
- [ ] MMU-ADV-C-03 `DisabledClientEn`
- [ ] MMU-ADV-C-04 `ApbPslverr`
- [ ] MMU-ADV-C-05 `IrqMaskedVsUnmasked`

## Definition of done

- Tests added, built for EN2 with `ENABLE_EXTRA_METAL`, and passing on target.
- Every test clears the faults it causes and restores PTE, `client_en` and mask state.
- The Nest is shown responsive after each fault.
- The conflict with the existing "wedges the bus" observation is resolved and recorded in the parent issue.
