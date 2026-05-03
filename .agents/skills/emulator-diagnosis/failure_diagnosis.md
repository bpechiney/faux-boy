# Failure Diagnosis — symptom → likely causes

Symptom-first triage reference. The symptom phrasing is what a user (or playtester) would actually say — *not* what the fix looks like. ("Audio sounds wrong on some channels" is a symptom; "DMC rate table indexed wrong" is a fix. The triager doesn't know the fix yet — that's why they're triaging.)

Each symptom maps to **multiple causes ranked by likelihood**. Walk top-down. Causes are **platform-tagged** where the same symptom has different causes per platform.

This file is the most-likely-to-drift in the entire skill. New games get reverse-engineered, new symptoms get documented, new causes get attributed. **Contributors should add entries when they triage a real bug.**

---

## Boot / startup symptoms

### Game shows black screen indefinitely

- **`[NES]`** Reset vector ($FFFC/D) read returning 0 → check mapper bank-init mapping reset bank to $C000–$FFFF.
- **`[NES]`** PPU rendering disabled and never re-enabled by the game → check NMI delivery (PPUCTRL.7 + vblank flag must reach CPU).
- **`[GB]`** Boot ROM not provided + reset state wrong → registers must initialise to post-boot values (A=0x01 DMG / 0x11 CGB; PC=0x0100). See [pandocs — Hardware register list](https://gbdev.io/pandocs/Hardware_Reg_List.html).
- **`[GB]`** LCD not enabled (LCDC bit 7 = 0) and game waits on STAT IRQ that never fires.
- **`[SNES]`** S-CPU stuck waiting for SPC700 boot handshake → check SPC700 IPL ROM behaviour and $2140 echo. Most common SNES "doesn't boot" cause.
- **All:** generic CPU stuck in infinite loop at $0000 (or wherever) → check that initial PC matches reset vector and that vector is read with correct endianness.

### Game shows scrambled tiles on title screen

- **`[NES]`** CHR-ROM bank not mapped on power-on → check default CHR layout for the mapper.
- **`[NES]`** PPU palette unrolled wrong on first frame → check that $3F00–$3F1F write semantics (palette mirror $3F10/14/18/1C ↔ $3F00/04/08/0C) are correct.
- **`[GB]`** Tile data area (LCDC bit 4) not honoured — game uses signed-tile-index addressing ($8800–$97FF) but emulator uses unsigned.
- **`[GB]`** OAM corruption from an `inc/dec rr` near OAM during mode 2 (DMG only) → see [pandocs — OAM Corruption Bug](https://gbdev.io/pandocs/OAM_Corruption_Bug.html).
- **`[SNES]`** VRAM auto-increment direction or step wrong ($2115) → tiles populate in transposed order.
- **`[SNES]`** Mode-bits in $2105 not honoured for tile-size selection.

---

## Mid-game graphics symptoms

### Title screen correct, gameplay graphics garble

- **`[NES]`** MMC1 mode bit not honoured → game switches PRG/CHR mode for gameplay vs title.
- **`[NES]`** MMC3 IRQ misfiring on wrong scanline → status bar tears or splits at wrong row. A12 filter likely missing or wrong window.
- **`[NES]`** CHR-RAM writes silently dropped → game DMAs tile data from CPU RAM each frame; nothing changes.
- **`[GB]`** MBC1 mode bit (mode 0 vs mode 1) wrong with > 8 KiB cart RAM → wrong save-data layout.
- **`[GB]`** MBC3 bank 0 register not honoured (writing 0 should map as 1).
- **`[SNES]`** DMA transfer length wrong (0 = 65536 bytes, not 0 bytes) → graphics data truncated.
- **`[SNES]`** HDMA channel ordering wrong → mid-frame graphics changes happen on wrong line.
- **`[SNES]`** Coprocessor RAM uninitialised on power-on (SA-1 BWRAM, SuperFX cache) → first frame shows garbage that "fixes itself" by frame 2.

### One scanline of graphics is wrong (specific horizontal stripe)

- **`[NES]`** Sprite 0 hit firing at wrong dot → status bar split at off-by-one line.
- **`[NES]`** MMC3 IRQ off by one scanline → mid-screen split at wrong row. Check rev A vs rev B distinction.
- **`[GB]`** STAT IRQ blocking quirk missed → game expects IRQ on transition that's actually masked.
- **`[GB]`** LYC=LY compare timing off by one dot → split happens one line late.
- **`[SNES]`** HDMA writing to wrong PPU register one line early/late.

### Sprites flicker / disappear / move wrong

- **`[NES]`** Sprite-overflow flag "fixed" instead of emulating the hardware bug → games that rely on the false-positive behaviour glitch.
- **`[NES]`** OAMDATA writes during rendering not corrupting OAM correctly → most games are safe, some buggy ones expected the corruption.
- **`[GB]`** 10-sprite-per-line cap not enforced → too many sprites visible.
- **`[GB]`** OAM corruption bug not emulated on DMG → some sprites disappear "randomly" — but on DMG that's the *correct* behaviour.
- **`[SNES]`** Range-over / time-over flags not set → game's safety logic for sprite culling never triggers.
- **`[SNES]`** OAMADDR high-priority designation ($2102 bit 1) not honoured → priority order wrong on heavy-sprite scenes.

---

## Audio symptoms

### Audio wrong pitch on some channels

- **`[NES]`** Frame counter mode bit wrong → length / envelope clocked at wrong rate.
- **`[NES]`** DMC rate table indexed wrong, or NTSC table used for PAL release.
- **`[NES]`** Noise period table wrong region (NTSC vs PAL distinction missing).
- **`[GB]`** Frame sequencer clocked off wrong DIV bit (bit 5 / bit 6 confusion in double-speed).
- **`[SNES]`** SPC700/DSP clock ratio wrong → music tempo off; voice pitch follows DSP rate.

### Audio missing one or more channels

- **`[NES]`** Channel never enabled in $4015 by emulator (length counter cleared to 0 on power-on means no channel can fire).
- **`[NES]`** DAC enable bit (NRx2 bits 7–3) not honoured — silent envelope volume keeps the DAC off.
- **`[GB]`** NR52 master enable cleared and never re-set (writing 0 to NR52 resets *all* APU registers; if game writes 1 expecting state to persist, it's silent).
- **`[GB]`** LFSR initialised to 0 → noise channel silent forever.

### Audio "wrong tempo" (everything pitched correctly but timing off)

- **`[NES]`** APU run from wrong CPU rate (NTSC rate on PAL or vice versa).
- **`[GB]`** DIV-write resetting frame sequencer accumulator — game writes DIV often, sequencer skips/repeats steps.
- **`[SNES]`** SPC700 clock locked to S-CPU clock with fixed ratio → real silicon has independent clock; some games' music drifts.

### Audio clicks on save-load

- **All:** Resampler buffer not part of save-state → load resumes mid-sample with discontinuity.
- **All:** Output ring buffer not flushed before save → load plays stale buffer first.

---

## Hangs and lockups

### Game freezes after Start / on first input

- **`[NES]`** NMI not enabled when game expects it → game's main loop spins on a flag set by NMI handler.
- **`[NES]`** BRK / IRQ handler missing or vector wrong → if game uses BRK for debug logging that the emulator panics on.
- **`[NES]`** JAM / KIL undocumented opcode (e.g., $02, $12) treated as NOP instead of CPU-halt → corrupted code falls into one and emulator runs wrong code instead of stopping.
- **`[GB]`** HALT bug not emulated → `EI; HALT` followed by a multi-byte instruction executes wrong opcodes.
- **`[GB]`** STOP not handled → first STOP from game freezes emulator.
- **`[SNES]`** SA-1 / SuperFX waiting on a status flag the S-CPU is supposed to write → check coprocessor message-register routing.
- **`[SNES]`** Auto-joypad-read window not honoured → game polls $4218–$421F before read completes, gets garbage, infinite-loops on safety check.

### Sprite 0 hit hangs at boot

- **`[NES]`** Sprite 0 hit timing off by enough that it doesn't fire at all → game's status bar split waits forever. Most commonly: hit suppressed at x=255 incorrectly extended to x=254.
- **`[NES]`** Pre-render scanline counter not initialising → hit flag never clears between frames.
- Related: scanline counter init wrong in MMC3 → similar hang on games using scanline-IRQ-and-wait pattern.

### Game crashes / corrupts memory after long play

- **All:** Open-bus implementation returns 0 instead of last-bus-value → games that read open bus to "randomise" something get a deterministic-zero source.
- **All:** Save corruption from wrong cart-RAM-enable handling on poweroff.
- **`[NES]`** DMC IRQ never acknowledged → IRQ asserted permanently after first DMC end-of-sample, eventually game's BRK handler corrupts state.
- **`[GB]`** MBC3 RTC drift if sub-second accumulator not serialized → load resumes with stale time, game-internal time logic confused.

---

## Save / save-state symptoms

### Save file from one emulator doesn't load on another (same ROM)

- Endianness in save format.
- Pointer serialization — addresses serialized as integers don't survive across processes.
- Uninitialised RAM at save-creation pulled from host allocator → save replays produce different output.

### Save-state load → different output after running forward

- Implicit state missing from serialiser.
- Save captured mid-step (catch-up scheduler, mid-instruction CPU).
- Determinism leak: hash-map iteration, time-based seed, host-endian @bitCast.

### Save corruption mid-play (no save-state involved)

- **`[GB]`** RAM-enable not honoured on cart-RAM writes → writes during disabled-RAM corrupt random bytes.
- **`[NES]`** WRAM enable + write-protect bits ($A001 on MMC3 family) not honoured.
- **`[SNES]`** SRAM size mismatch with header → writes past actual cart RAM size wrap and corrupt.

---

## Region-specific symptoms

### Game runs on NTSC, fails on PAL (or vice versa)

- **`[NES]`** APU rate tables not region-parameterized.
- **`[NES]`** PPU scanline count hardcoded → PAL game runs with NTSC scanline count and timing falls apart.
- **`[GB]`** Not applicable — no PAL Game Boy.
- **`[SNES]`** PPU scanline count or HDMA per-line cost hardcoded NTSC.

### Game runs on DMG, fails on CGB (or vice versa)

- **`[GB]`** CGB-only IO ($FF4D, $FF51–$FF55, $FF68–$FF6B) not implemented or returns wrong default on DMG fallback.
- **`[GB]`** OAM corruption bug emulated on CGB (it's DMG-only) → CGB games glitch where they should be safe.
- **`[GB]`** Double-speed mode component matrix wrong → audio fast or video flickering depending on which component scaled wrong.

---

## Coprocessor-specific symptoms (SNES)

### SA-1 game runs but gameplay logic wrong

- SA-1 IRQ delivery to S-CPU → check message register handling.
- BWRAM bank mapping wrong between SA-1 view and S-CPU view.
- SA-1 timer not running → game-internal timing logic stalls.

### SuperFX game runs but graphics wrong

- GSU cache not flushed when game intends → stale plot operations.
- Plot register not honouring transparent-pixel handling.
- ROM/RAM bus arbitration wrong → GSU and S-CPU both writing to plot register.

### DSP-1 game (Super Mario Kart, Pilotwings) shows wrong perspective math

- DSP-1A vs DSP-1B revision difference not honoured.
- DSP firmware ROM blob version mismatch.

---

## Adding new entries

Format for new entries:

```
### Symptom phrasing (as user would say it)

- **`[platform]`** Cause (most likely first) → [optional pointer to reference section].
- **`[platform]`** Cause (next).
- **All:** Cross-platform cause if applicable.
```

Symptoms phrased like fixes ("DMC rate table is wrong") are not useful — the triager doesn't know the fix yet. Always phrase as observable behaviour ("audio at wrong pitch on some channels"). When in doubt: read the entry as if you didn't write it. If you can guess what the fix is from the symptom alone, the symptom is over-specified.

**Attributions must be falsifiable, not folkloric.** When naming a game in a cause, name the game *and* the specific observable failure mode it produces — "*Battletoads* level 2 bike: controls become unresponsive at specific points without the DMC controller-read glitch." Reputation-based attributions ("this game is known to be picky", "some games need X") are folklore and should not be added. The bar is: another contributor reading your entry can reproduce the failure on a stock emulator missing the cause and verify the dependency.
