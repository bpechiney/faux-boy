# CPU Review Checklist

For 6502 / 2A03 (NES), LR35902 (Game Boy), and 65C816 (SNES) cores. Most items below are written for the 6502 — the most heavily exercised case — with `[GB]` and `[SNES]` deltas inline where they diverge.

Walk this checklist top-to-bottom. Don't skip sections because they "look fine in the diff" — rule 3 (missing behaviour) is the whole reason this exists.

---

## 1. Cycle accounting (methodology rule 2)

**Per-opcode cycle count matches the reference table?**
Compare against [nesdev wiki — Cycle reference chart](https://www.nesdev.org/wiki/Cycle_reference_chart) (or [pandocs](https://gbdev.io/pandocs/) for LR35902, [fullsnes](https://problemkaputt.de/fullsnes.htm) for 65C816). Off-by-one cycle errors silently desync the PPU/APU; symptoms appear far from the cause.
*Catches it:* nestest cycle column, Blargg `instr_timing`, mooneye-gb `instr_timing` for GB.

**Page-cross penalty applied to the right addressing modes?**
+1 cycle on `abs,X` / `abs,Y` / `(zp),Y` reads when the effective address crosses a page; **not** on writes (writes always pay the dummy cycle whether or not they cross). RMW instructions (`ASL abs,X`, `INC abs,X`, etc.) are a fixed 7 cycles regardless of cross.
*Catches it:* Blargg `cpu_dummy_reads`, `instr_misc`.

**Branch penalty: +1 if taken, +2 if taken AND crosses page?**
A non-taken branch is 2 cycles flat. A taken-but-no-cross branch is 3. A taken-and-crosses branch is 4. Easy to over-count.

**`[GB]`** Game Boy timings are quoted in M-cycles (4 T-states each). Confirm the core is consistent — mixing M and T silently breaks every interaction. Branch costs differ from 6502; check pandocs.

**`[SNES]`** 65C816 cycle costs depend on M/X flags and direct-page low byte. `MEMSEL` ($420D) selects 6 vs 8 master cycles per fast-ROM access. Hardcoded counts here are usually wrong.

---

## 2. Dummy reads & writes — the missing-behaviour core (methodology rule 3)

**RMW (Read-Modify-Write) writes the *original* value back before the modified value?**
RMW instructions do: read → write-back-original → write-modified. The intermediate write is observable when the target is a hardware register — most famously, RMW on `$2007` (PPUDATA) or `$4014` (OAM DMA register area) produces hardware-relevant glitches. "We just write the final value once" is a bug. Reference: [nesdev wiki — RMW behavior](https://www.nesdev.org/wiki/CPU_unofficial_opcodes) (and per-instruction pages off `wiki.nesdev.org`) for the canonical RMW opcode list (`ASL`/`LSR`/`ROL`/`ROR`/`INC`/`DEC` documented + the unofficial `SLO`/`SRE`/`RLA`/`RRA`/`ISB`/`DCP`).
*Catches it:* Blargg `cpu_dummy_writes_oam`, `cpu_dummy_writes_ppumem`.

**Page-crossing addressing modes perform a dummy read at the wrong (un-fixed) address?**
For `LDA abs,X` when X causes a page cross: the CPU reads from `(base & 0xFF00) | ((base + X) & 0xFF)` first, then re-reads from the corrected address. Skipping the dummy read breaks any mapper that side-effects on read (MMC3 IRQ counter, MMC5 scanline counter).
*Catches it:* Blargg `cpu_dummy_reads`.

**Indirect JMP `($xxFF)` page-wrap bug preserved?**
`JMP ($10FF)` reads the high byte from `$1000`, not `$1100`. This is a documented hardware bug; "fixing" it breaks any game that depends on it. Cite nesdev or this is rule 6 territory.

**`[GB]`** LR35902 has no documented dummy reads in the same shape. Instead check: `HALT` bug, `EI` delays IME by one instruction (not immediately), `RETI` enables IME *immediately*.

---

## 3. Flag-update ordering vs memory access (methodology rule 2)

**Flags updated in the order specified by reference, not "all at once at the end"?**
For RMW: N/Z come from the *modified* value; C comes from the original (for shifts). Order matters because an interrupt vectored mid-instruction (rare, mostly NMI on edge cases) may observe partial state. More commonly: `BIT` updates N from bit 7 *of the operand*, V from bit 6 *of the operand*, Z from `A AND operand`. Easy to get N/V from the AND result by accident.

**Status register pushes use the right B-flag bit pattern?**
`PHP` and `BRK` push with bit 4 (B) set; IRQ and NMI push with bit 4 clear. Bit 5 is always pushed as 1. Software detects BRK-vs-IRQ from the pushed B bit; getting this wrong breaks any cooperative IRQ/BRK handler.
*Catches it:* nestest, Blargg `instr_test-v5`.

---

## 4. Interrupt polling cycle (methodology rules 2 and 5)

**Interrupt poll happens on the penultimate cycle of each instruction?**
The 6502 samples NMI/IRQ during the cycle *before* the last cycle of the current instruction. Polling at instruction boundary is wrong and observable: a level-triggered IRQ that asserts during the last cycle delays one instruction; an NMI edge during the last cycle is captured for *next* instruction. NMI is edge-triggered and latched separately — check the latch is a real flip-flop, not a level read.
*Catches it:* Blargg `cpu_interrupts_v2`, `nmi_sync`.

**Interrupt-hijack of BRK / branch quirks handled?**
If an NMI asserts during the first two cycles of `BRK`, the BRK pushes the IRQ vector instead of $FFFE/F (or vice versa for the IRQ-during-NMI case). This is the "interrupt hijacking" quirk; most games don't hit it but the test ROMs do.

**Branch instructions poll interrupts at the right cycle?**
Taken branches without page cross poll on cycle 2; with page cross, poll on cycle 3. Subtle but `cpu_interrupts_v2` checks it.

**`[GB]`** IF and IE registers, IME flag, and the HALT bug interact: if IME=0 and `(IF & IE) != 0`, HALT skips advancing PC by one byte on the next fetch. Mooneye `halt_bug` confirms.

---

## 5. Implicit / mid-instruction state (methodology rule 5)

For save-states and rewind, ask explicitly:

- **In-progress instruction cycle counter** — if the CPU steps one cycle at a time, the partial cycle index of the current opcode must serialize.
- **Captured opcode and operand bytes** — already-fetched bytes that haven't been used yet.
- **NMI edge latch** — separate from the line state; an NMI that has been latched but not yet serviced must persist.
- **IRQ line state** — usually a wire from the mapper/APU, but the *acknowledged-but-not-cleared* status is per-source.
- **DMA in progress** — if OAM DMA / DMC fetch is mid-transfer, the remaining count and the source/dest pointers must serialize.
- **`[GB]`** EI-delayed-by-one-instruction state, the IME pending flip.
- **`[SNES]`** WAI / STP suspended state; native-vs-emulation mode (E flag); M and X flag widths; current direct-page register value (subtle quirk: changing DP changes future address calculations *immediately*).

If the save-state test only round-trips the named registers (A/X/Y/SP/PC/P), it's incomplete. "We don't save mid-instruction state" is acceptable only if every save point is *guaranteed* aligned to instruction boundary — and that needs to be enforced and commented.

---

## 6. Undocumented / unofficial opcodes (methodology rule 3)

**For NES (2A03): the stable unofficial opcode group implemented?**
The canonical list (`LAX`, `SAX`, `SLO`, `RLA`, `SRE`, `RRA`, `DCP`, `ISB`/`ISC`, `ANC`, `ALR`, `ARR`, `AXS`, `LAS`, plus the `JAM`/`KIL`/`HLT` halt-the-CPU group) is documented at [nesdev wiki — CPU unofficial opcodes](https://www.nesdev.org/wiki/CPU_unofficial_opcodes); `nestest` exercises every documented and most undocumented opcodes — coverage there is the bar. Several homebrew titles and a few commercial games (Disch's tables list them) use these.

**Unstable opcodes (`XAA`/`ANE`, `LAX #imm`, `TAS`, `AHX`/`SHA`, `SHX`, `SHY`, `LXA`) — at minimum return a deterministic value with a citation comment?**
The nesdev page above lists the per-opcode "magic constant" choices (typically 0xEE or 0xFF for `XAA`). "Returns 0" is a different bug from "returns the silicon value"; either way, *cite the choice*.

**`[GB]`** LR35902 has no unofficial opcodes — the unused opcode group documented in [pandocs — CPU instruction set](https://gbdev.io/pandocs/CPU_Instruction_Set.html) (the `0xD3` family) *truly* lock the CPU on real hardware. A panic on these is fine. A no-op is wrong (let bugs surface).

**`[SNES]`** 65C816 has no undocumented opcodes worth emulating; `WDM` (`0x42`) is reserved and effectively a 2-byte NOP.

---

## 7. Determinism (methodology rule 4)

**Power-on register state defined and reproducible?**
The reset state (canonical post-reset register values, SP-after-reset-push, P bit pattern) is documented at [nesdev wiki — CPU power-up state](https://www.nesdev.org/wiki/CPU_power_up_state). RAM contents are technically uninitialised on real hardware, but the emulator must pick a *deterministic* pattern — common choices: all 0xFF, or the canonical NES "powered-on" pattern (alternating runs). Random/`std.crypto.random` here breaks TAS replay and golden-frame regression tests.

**No host-endian-dependent reads of the program byte stream?**
`@bitCast` of a multi-byte fetch is a determinism trap on big-endian hosts. Use explicit little-endian reads.

**No `std.AutoHashMap` iteration influencing CPU-visible state?**
Map iteration order is unspecified. If opcode dispatch, watchpoint checks, or breakpoint matching iterate a hash map, two runs of the same ROM on the same host may diverge.

---

## 8. Region / revision (methodology rule 7)

**Decimal mode (`SED` / `CLD`, ADC/SBC with D=1) — for NES, explicitly disabled with a comment, not silently broken?**
The 2A03 (NES CPU) physically lacks decimal mode. ADC/SBC with D=1 should behave as if D=0. A comment citing nesdev is the correct shape; no comment is rule 6 territory.

**`[GB]`** LR35902 has no decimal mode but has `DAA` — confirm it implements the documented N/H/C dependence, not just the "ADC corrects to BCD" intuition.

**`[SNES]`** 65C816 *does* have working decimal mode; if D is ignored, that's a bug in native mode.

**For NES: region threaded as a parameter through the CPU master-clock divider, not hardcoded?**
NTSC, PAL, and Dendy each have their own CPU-master-clock ratio (the divider numbers are at [nesdev wiki — Cycle reference chart](https://www.nesdev.org/wiki/Cycle_reference_chart) and the per-region pages). A single hardcoded divider produces the wrong CPU rate on the other regions and silently desyncs every downstream APU/PPU timing relationship.

---

## 9. Citation hygiene (methodology rule 6)

When you encounter quirky CPU code:

- **Cited to nesdev / pandocs / fullsnes / a named test ROM?** ✅ Trust it. Don't suggest "simplifying" away the quirk.
- **Cited to a manufacturer datasheet?** ⚠️ Flag it. The MOS 6502 / Sharp LR35902 / WDC 65C816 datasheets describe the *intended* behaviour, not what software relies on.
- **Quirky and uncited?** ❌ This is the bug. Either it's wrong, or it's right but no future reader will know why. Either way, request a citation.

---

## 10. Test-ROM correspondence (methodology rule 1)

This section maps **review triggers → ROMs to re-run**. For ROM identity (what each ROM exercises, where it writes its result, how to interpret pass/fail codes) follow the canonical-source pointers in `references/test_roms.md` to the upstream archive (Blargg / mooneye-gb / peter_lemon).

When reviewing a CPU change, ask: **which test ROM does this affect?** If the answer is "I don't know", that's the first thing to find out. Suggested mapping for NES:

| Change touches... | Re-run at minimum |
|---|---|
| Opcode logic, flags | nestest, Blargg `instr_test-v5`, `cpu_timing_test6` |
| Addressing modes / dummy R-W | `cpu_dummy_reads`, `cpu_dummy_writes_oam`, `cpu_dummy_writes_ppumem` |
| Interrupt timing, NMI | `cpu_interrupts_v2`, `nmi_sync` |
| Branch / jump | `branch_timing_tests`, `cpu_timing_test6` |
| Reset / power-on | `cpu_reset` (Blargg) |

For Game Boy, the equivalent ROMs are Blargg's `cpu_instrs` and `instr_timing` plus mooneye-gb's `acceptance/` tree (see [`Gekkio/mooneye-gb`](https://github.com/Gekkio/mooneye-gb)). For SNES, [peter_lemon's CPUTest](https://github.com/PeterLemon/SNES).

A CPU change with no associated test-ROM verification path is a yellow flag — note it explicitly in the review output.
