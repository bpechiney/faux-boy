# Bus Review Checklist

Covers the system bus and memory-map plumbing for NES, Game Boy, and SNES. The bus is the most **platform-divergent** component in the entire skill — NES has separate CPU and PPU buses interacting via OAM DMA / DMC steal; GB has a single bus with OAM DMA bus-lockout; SNES has S-CPU + B-bus (PPU/APU IO) + cart bus with multi-channel DMA/HDMA arbitration plus coprocessor contention.

---

## 1. Open bus (methodology rule 3)

Open bus needs **three distinct questions per platform**, not one. A generic "is open bus implemented?" check misses all three.

### `[NES]` Two latches and decay

- **Question:** Are CPU open-bus and PPU open-bus modelled as **separate** latches?
  - Reads from unmapped CPU regions ($4018–$401F, $4020–$4FFF when no mapper register) return the **CPU** latch (last value on the CPU data bus).
  - Bits 0–4 of $2002 come from the **PPU** latch (last value on the PPU data bus). Bits 5–7 are real status bits.
  - Returning CPU-open-bus for $2002 low bits is a common bug; test ROMs catch it.
- **Question:** Decay model — implementation acknowledges no decay, OR genuinely models the ~600 ms decay? Either is acceptable; what's *not* acceptable is a comment claiming decay is implemented while the latch value never changes.
- **Question:** Does the latch update on **every** bus access (including PPU rendering fetches that drive the address bus), or only on CPU memory operations? The former is correct; the latter desyncs after a few PPU cycles.
- See [nesdev wiki — Open bus behavior](https://www.nesdev.org/wiki/Open_bus_behavior) and [nesdev wiki — PPU registers](https://www.nesdev.org/wiki/PPU_registers) for the full open-bus specification.

### `[GB]` Region-specific deterministic fill vs true open bus

- **Question:** Distinguishes **deterministic fill** from **true open bus**? GB has both:
  - $FEA0–$FEFF (the unusable region): returns $00 on DMG, varies by PPU mode on CGB. Pandocs documents the per-mode value. **Not** open bus.
  - $FF03, $FF08–$FF0E, $FF15, $FF1F, $FF27–$FF2F, $FF4C, $FF4E, $FF50–$FF7F (most): unmapped IO. Returns $FF on DMG/MGB; some return alternate values on CGB. Mostly **not** open bus — fixed-fill.
  - Truly open-bus addresses are rare on GB; most "unmapped" reads have a deterministic answer.
- **Question:** During OAM DMA, **non-HRAM CPU reads return $FF (or last bus value, depending on implementation)** — see §2. This is a per-cycle effect, not a "permanently $FF" region.
- **Question:** Upper 3 bits of IF ($FF0F) read as 1 — flag as it's commonly forgotten.

### `[SNES]` Three sources

The framework — see [fullsnes — CPU and PPU registers / open-bus sections](https://problemkaputt.de/fullsnes.htm) — is that SNES open-bus has three distinct sources that must be modelled separately:

- **Question:** Does the implementation distinguish **CPU open-bus**, **PPU open-bus**, and **per-register last-value buffers**?
  - CPU open-bus: last value on the S-CPU data bus. Returned for unmapped reads in CPU bus regions.
  - PPU open-bus: last value on the B-bus. Returned for reads from write-only PPU registers ($2104, $2105, $2106, etc.).
  - Per-register buffers: $2138 (OAM read-back), $2139/$213A (VRAM read-back), $2134/$2135/$2136 (multiplication unit), $213C/$213D (H/V counter latch) all have register-specific behaviour with prefetch and stale-buffer semantics — not open bus, but easily confused with it.
- **Question:** Reading $2104 returns PPU open-bus (not CPU open-bus, not zero). *Super Mario World* depends on this in some palette-loading paths.
- **Question:** Are 1-CHIP-vs-original-PPU open-bus differences gated behind a revision parameter, or hardcoded? Most emulators hardcode; document the choice.

---

## 2. DMA and CPU interaction (methodology rules 2 & 3)

### `[NES]`

- **Question:** OAM DMA ($4014 write) stalls the CPU for the documented odd/even-cycle-aligned duration ([nesdev wiki — DMA](https://www.nesdev.org/wiki/DMA) gives the cycle counts)? The hazard the heuristic asks about is *parity* — is the cost different on even vs odd CPU cycles, or hardcoded to one value?
- **Question:** During OAM DMA, the bus performs 256× alternating read/write at the documented address pattern (read at `(value << 8) + i`, write to $2004)?
- **Question:** DMC sample fetch steals **1–4 CPU cycles** depending on alignment with the current instruction and any concurrent OAM DMA? Full table at [nesdev wiki — DMA](https://www.nesdev.org/wiki/DMA).
- **Question — bus model exposes the DMC controller-glitch hazard:** **Does the bus model expose DMA-vs-CPU read conflicts to the controller register ($4016/$4017)?** A DMC fetch that occurs during a `LDA $4016` causes a double-read on real hardware — the controller bit is duplicated or dropped. *Battletoads* is the canonical title that hits this. The standard mitigation in software is to read $4016 in a loop comparing two consecutive reads; the emulator must reproduce the hazard or that mitigation will look like a bug. Reference: [nesdev wiki — Standard controller](https://www.nesdev.org/wiki/Standard_controller).

### `[GB]`

- **Question:** OAM DMA ($FF46) is **160 M-cycles**, and during it the CPU bus is restricted as documented at [pandocs — OAM DMA](https://gbdev.io/pandocs/OAM_DMA_Transfer.html)? Specifically:
  - CPU reads from non-HRAM ($FF80–$FFFE) return $FF.
  - CPU writes to non-HRAM may or may not land (undefined; emulators typically drop them).
  - The CPU is **not** halted — it must keep executing. Verify the bus model doesn't accidentally halt-on-DMA.
- **Question:** GDMA (CGB, $FF55 bit 7 = 0): copies in 16-byte chunks at full bus speed, CPU halted. Per-chunk cost depends on speed mode — see [pandocs — CGB Registers](https://gbdev.io/pandocs/CGB_Registers.html) and [Reducing Power Consumption](https://gbdev.io/pandocs/Reducing_Power_Consumption.html).
- **Question:** HDMA (CGB, $FF55 bit 7 = 1): 16 bytes per HBlank; CPU runs between HBlanks. Disabling HDMA mid-transfer (writing $FF55 with bit 7 = 0) cancels the remaining transfer (and the cancel-mode write is itself a documented quirk: "remaining length register" semantics).

### `[SNES]`

- **Question:** General DMA cycle costs (per-byte master-clock cost regardless of MEMSEL — DMA always uses slow timing — plus per-channel startup) match the documented schedule? See [fullsnes — DMA / HDMA section](https://problemkaputt.de/fullsnes.htm) for the canonical numbers; the heuristic is "the per-byte cost should be the slow-bus number, not whatever MEMSEL would predict".
- **Question:** HDMA fires **at the start of each HBlank** during visible scanlines, channels processed in **0 → 7 order**? Out-of-order processing breaks per-line graphics effects.
- **Question:** HDMA can pause an in-progress DMA at HBlank, then resume after the HDMA chunk completes?
- **Question — load-bearing for SNES:** **WRAM read/write during a DMA whose source or destination is WRAM** produces undefined behaviour. Real silicon resolves the conflict in a specific (mostly-benign) way that some software depends on. The standard mitigation in software is to avoid this entirely; an emulator that panics or returns garbage when the conflict happens may break otherwise-working ROMs. Verify the bus model handles concurrent WRAM access from CPU + DMA without crashing, and document whichever resolution rule is used.

---

## 3. Bus conflicts (methodology rule 3)

### `[NES]` Discrete-logic mappers

- **Question:** For mappers with **no internal bank latch** (UxROM/iNES 2, AOROM/iNES 7, GxROM/iNES 66, BNROM/iNES 34, some CNROM variants): does a CPU write to ROM-mapped space see `cpu_value & rom_value_at_address` at the mapper input, not just `cpu_value`?
- **Question:** Real game dependency — *Cybernoid* is the canonical NES bus-conflict title. *RC Pro-Am* and several other UxROM games arrange ROM data such that the conflict resolves to the intended bank value (the write address contains the bank byte itself). An emulator that just latches `cpu_value` passes most games but breaks the dependent ones.
- Forward-reference: `checklists/mapper_review.md` §8 ("Bus conflicts on discrete-logic mappers") covers the per-mapper handling question. Bus-side asks "is the AND modelled at all?"; mapper-side asks "is the AND modelled correctly per mapper?".

### `[GB]`

- **Question:** No bus conflicts in the NES discrete-logic shape — MBC writes are latched in the MBC chip, not on the cart bus. If the implementation has a "bus conflict" code path for GB, that's a porting bug from a NES emulator and should be removed.

### `[SNES]`

- **Question:** Coprocessor-vs-S-CPU bus contention for shared memory regions (SA-1 BWRAM, SuperFX game-pak ROM): is the arbitration register's effect actually wired through the bus model?
  - SA-1: BWRAM access conflict between SA-1 and S-CPU is configurable via $2228; default is "S-CPU has priority" but games change it.
  - SuperFX: ROM/RAM bus arbitration via SCMR ($3037) — SuperFX vs S-CPU access; the GSU stalls when S-CPU has the bus and vice versa.
- These mostly affect retail games subtly; homebrew test code exercises them more aggressively.

---

## 4. Implicit bus state (methodology rule 5)

Same enumeration discipline as `mapper_review.md` §5: walk every implicit-state field on the canonical per-platform reference and verify each round-trips through the serializer/deserializer with no truncation. The failure mode is omission.

### Per-platform serialized state (walk against the reference)

- **`[NES]`** Walk against [nesdev wiki — Open bus behavior](https://www.nesdev.org/wiki/Open_bus_behavior) and [nesdev wiki — DMA](https://www.nesdev.org/wiki/DMA). The implicit fields the references list — CPU open-bus latch, PPU open-bus latch, DMC DMA in-progress (byte counter, address counter, sample buffer, fetch-pending flag), OAM DMA in-progress (source-page register, current byte index, alignment-cycle flag), OAMADDR's *current* value (not just "0 when forced during dots 257–320") — are easy to omit because the names look like internal state rather than registers.
- **`[GB]`** Walk against [pandocs — Memory Map](https://gbdev.io/pandocs/Memory_Map.html) and [pandocs — OAM DMA](https://gbdev.io/pandocs/OAM_DMA_Transfer.html). Bus-data-bus latch (for the few open-bus addresses), OAM DMA in-progress ($FF46 last value, bytes remaining, current target address), CGB GDMA/HDMA state ($FF51–$FF55, HDMA-vs-GDMA mode bit, mid-chunk flag).
- **`[SNES]`** Walk against [fullsnes — DMA / HDMA / CPU IO sections](https://problemkaputt.de/fullsnes.htm). CPU open-bus latch, PPU open-bus latch, per-register buffers (OAM read prefetch, VRAM read prefetch, multiplication-unit result bytes), per-channel DMA parameters + in-progress state, per-channel HDMA state (line counter, table pointer, "do-transfer" flag, per-line transfer state), WRAM access port + auto-incrementing address.

---

## 5. Determinism (methodology rule 4)

- **Question:** Power-on open-bus latch is initialised to a **deterministic** value (typically 0x00 or 0xFF), not whatever was in the host allocator?
- **Question:** When multiple HDMA channels need init at frame start, the order is deterministic (lowest channel first, per [fullsnes — DMA / HDMA section](https://problemkaputt.de/fullsnes.htm))?
- **Question:** When DMA + HDMA contend for the bus at the same wall-clock moment, the resolution is deterministic and matches the documented HDMA-pauses-DMA priority?

---

## 6. Citation hygiene (methodology rule 6)

- Bus quirks should cite **named test ROMs**, not just "real hardware does this":
  - NES: Blargg `cpu_dummy_reads` / `cpu_dummy_writes_*` / `oam_stress` / `dma_2007_write` / `ppu_open_bus` / `apu_test`.
  - GB: mooneye `acceptance/oam_dma/*`, `acceptance/intr_*` for bus-related interrupt tests.
  - SNES: peter_lemon `HDMA/`, anomie's bus tests, bsnes-emu/bsnes validation suite.
- Comments that describe a bus behaviour without naming the test ROM that proves it — flag for citation. The bus is harder to debug than other components because failures often manifest far from the cause; unverified bus behaviour is a long-tail compatibility liability.

---

## 7. Test-ROM correspondence (methodology rule 1)

This section maps **review triggers → ROMs to re-run**. For ROM identity follow the canonical-source pointers in `references/test_roms.md` to the upstream archive.

| Change touches... | Re-run at minimum (NES) |
|---|---|
| Open bus | Blargg `ppu_open_bus`, `apu_test` |
| OAM DMA | `oam_stress`, `dma_2007_write` |
| DMC DMA / controller glitch | *Battletoads* level-2 bike sequence (real-game test) |
| Memory mirroring | nestest implicitly covers most of it |
| Bus conflicts | *Cybernoid*, *RC Pro-Am* (real-game tests) |

| Change touches... | Re-run at minimum (Game Boy) |
|---|---|
| OAM DMA + bus lockout | mooneye `acceptance/oam_dma/` (full directory) |
| Echo RAM | mooneye `misc/bits/`, plus most test ROMs implicitly |
| Unmapped IO reads | mooneye `acceptance/bits/unused_hwio-*` |
| HDMA / GDMA (CGB) | mooneye `misc/`, dmg-acid2/cgb-acid2 for visible effects |

| Change touches... | Re-run at minimum (SNES) |
|---|---|
| Open bus | peter_lemon `OpenBus/` + bsnes-emu/bsnes bus tests |
| DMA byte timing | peter_lemon `DMA/` |
| HDMA channel ordering | peter_lemon `HDMA/` |
| WRAM-during-DMA hazard | no canonical test ROM; verify by reading bsnes-emu/bsnes source notes |
| Memory mirroring | basic boot succeeds = mirror works for the common path |

A bus change with no associated test-ROM verification path is a **yellow flag** — note it explicitly in the review output and propose the minimal homebrew test that would isolate the behaviour.
