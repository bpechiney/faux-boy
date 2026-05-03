# APU Review Checklist

Covers the NES APU (integrated into 2A03/2A07), the Game Boy APU (DMG/CGB), and the SNES SPC700 + DSP. Written with the NES APU as the primary case; `[GB]` and `[SNES]` deltas inline.

The SPC700 is a standalone CPU with its own clock and 64 KiB of private RAM — it's reviewed *as a CPU* via `cpu_review.md` for the SPC700 instruction set, plus the integration questions below for the communication ports and DSP. The system-level shape (independent clock domain, IPL boot handshake, communication ports, DSP register layout) is documented in [fullsnes — APU section](https://problemkaputt.de/fullsnes.htm).

Walk top-to-bottom. The APU has more silently-omitted hardware behaviour per line of code than any other component; rule 3 (read for missing behaviour) is the load-bearing rule here.

---

## 1. Frame counter / frame sequencer (methodology rules 2 & 5)

The single densest review item in the entire APU. Most "audio sounds wrong but I can't tell why" bugs trace to this section.

**`[NES]`** **$4017 mode-bit (bit 7) writes — the new mode takes effect on the next APU cycle, with a 3-or-4-cycle delay depending on whether the write occurred on an even or odd CPU cycle?**
Even-cycle write: 3 CPU cycle delay. Odd-cycle write: 4 CPU cycle delay. Hardcoding "instantly resets" is wrong and observable.

**`[NES]`** **Writing $4017 also clocks length counters and envelope counters immediately if mode bit 7 = 1?**
The 5-step sequence (mode 1) inserts a half-frame and quarter-frame clock on the write itself. This *generates* envelope/length state, not just changes timing.

**`[NES]`** **$4017 IRQ-inhibit bit (bit 6): when set, frame IRQ is *immediately cleared* in addition to being inhibited going forward?**
A common bug: only inhibiting future IRQs, leaving a pending IRQ asserted.

**`[NES]`** **In 4-step mode (default, bit 7 = 0): frame IRQ fires at the end of step 4 if not inhibited?**
The step-boundary CPU cycles are region-parameterized — NTSC and PAL each have their own sequence (canonical numbers at [nesdev wiki — APU frame counter](https://www.nesdev.org/wiki/APU_Frame_Counter)). Off by one cycle changes which instruction the IRQ interrupts, which changes whether software's `CLI` happens before or after — observable. A single hardcoded sequence silently breaks the other region.

**`[GB]`** **Frame sequencer is clocked off a falling edge of an internal-DIV bit (bit 5 single-speed, bit 6 double-speed)?**
The double-speed trap: every component's timing must be re-derived per cycle, not just "speed = 2; multiply everywhere." A naive "every 8192 CPU cycles" frame sequencer runs at 1024 Hz under double-speed (2× too fast); the correct shape is to re-derive the tap bit from the speed-mode flag. Reference: [pandocs — Reducing Power Consumption](https://gbdev.io/pandocs/Reducing_Power_Consumption.html) and [pandocs — Audio details](https://gbdev.io/pandocs/Audio_details.html). Verify the implementation re-derives the tap bit from the speed-mode flag rather than hardcoding a single-speed cycle count.

**`[GB]`** **DIV-write resets the internal counter, which can cause an unintended frame-sequencer step or skip one?**
*Pinball Deluxe*-class glitches.

**`[GB]`** **Length-counter extra-clock quirk: setting NRx4 bit 6 (length-enable) on the same cycle the next frame-sequencer step would clock length results in one extra length-clock?**
Mooneye `acceptance/timer/` and `acceptance/sound/` cover the family.

**`[SNES]`** SPC700 has no NES/GB-style frame counter; the DSP's envelope and gain processing is per-voice and clocked off the SPC700's own clock. Reviewer side: confirm the per-voice envelope state machine ticks at the documented rate, not at S-CPU cycle rate.

---

## 2. DMC sample-fetch CPU cycle stealing (methodology rule 2)

**Distinct from DMC IRQ — section 3 covers that.** This section is purely about the bus-arbitration cost.

**`[NES]`** **DMC fetch costs 4 CPU cycles by default, with documented variations: 3 cycles on certain RMW second-to-last cycles, 1–2 cycles when CPU is already halted by OAM DMA, +up to 4 additional cycles during write-cycle conflicts?**
The full per-alignment cycle table is at [nesdev wiki — DMA](https://www.nesdev.org/wiki/DMA). A flat "always 4 cycles" implementation is wrong but passes most games. Catches it: Blargg `apu_test/`, `dmc_dma_during_read4/`.

**`[NES]`** **DMC fetch can corrupt $4016/$4017 controller reads (the "DMC DMA controller glitch")?**
Reviewer side already covered in `checklists/bus_review.md` §2 — confirm the bus model exposes the conflict, then check that the APU drives the fetch through the bus path that participates.

**`[NES]`** **DMC sample address wraps from $FFFF to $8000 (not to $0000) when the address counter overflows?**
The DMC sample area is fixed at $C000–$FFFF; on overflow it wraps to $C000 (or $8000 depending on which doc you read — check fullsnes-equivalent for the 2A03 specifically). Cite the source.

**`[GB]`** No equivalent — GB APU samples are pulled from wave RAM ($FF30–$FF3F) for CH3 only, no CPU bus stealing.

**`[SNES]`** SPC700 + DSP run on independent silicon and don't steal S-CPU bus cycles. The review concern is the inverse: SPC700 communication-port reads/writes happen on the S-CPU bus at the documented per-access cost (S-CPU master-clock dividers per access region are at [fullsnes — Memory Map / CPU IO](https://problemkaputt.de/fullsnes.htm)).

---

## 3. Interrupts: DMC IRQ and frame IRQ (methodology rules 2 & 4)

**Distinct from §2 — DMC DMA stealing is bus-cycle-cost; DMC IRQ is interrupt assertion.**

**`[NES]`** **DMC IRQ asserts at sample end if $4010 bit 7 (IRQ-enable) is set, with loop ($4010 bit 6) cleared?**
Looped DMC samples should never fire IRQ. A common bug fires IRQ at every sample-end regardless of the loop bit.

**`[NES]`** **DMC IRQ is level-triggered: stays asserted until cleared by either writing $4015 with bit 4 = 0 (disable DMC) or writing $4010 with bit 7 = 0 (disable DMC IRQ)?**
**Reading $4015 does NOT clear the DMC IRQ flag.** Common bug: cargo-culting the frame-IRQ clear-on-read behaviour to the DMC IRQ.

**`[NES]`** **Frame IRQ ($4015 bit 6): cleared on read of $4015, AND cleared by writing $4017 with bit 6 = 1?**
Two separate clearing mechanisms.

**`[NES]`** **$4015 read returns: bit 0–4 length-counter active per channel, bit 6 frame IRQ, bit 7 DMC IRQ — and the read clears bit 6 only?**
Length-counter active bits are a queryable status, not an interrupt source.

**`[GB]`** No DMC equivalent. The only audio-related interrupt vector path is the joypad/serial/timer family — APU does not generate IRQs.

**`[SNES]`** SPC700 has its own internal interrupts; from the S-CPU side, the APU generates no IRQs.

---

## 4. Register-bit sharing (methodology rule 3)

The hardware reuses bits across logically-distinct functions. The implementation must too — splitting a shared bit into two flags is a missing-behaviour bug that breaks software toggling one and not the other.

**`[NES]`** **$4000/$4004 bit 5 is shared between length-counter halt AND envelope loop — same physical flip-flop?**
Square channels. Setting envelope loop accidentally halts length counter and vice versa. Games rely on this overlap intentionally to control multiple behaviours from one bit.

**`[NES]`** **$4008 bit 7 is shared between linear-counter control (don't reload linear counter from $400B writes) AND length-counter halt for triangle?**
Triangle channel. Same flip-flop, two semantic interpretations.

**`[NES]`** **$400C bit 5 shares the noise envelope loop and length-counter halt — same as $4000/$4004 for noise channel?**
Noise channel. Same family.

**`[GB]`** **NRx4 bit 6 (length enable): note the extra-clock quirk in §1. Bit is *not* shared with another function on GB, but its interaction with frame-sequencer phase is the analogous "two bits look like one" trap?**

**`[GB]`** **NR52 ($FF26) bit 7 is the master APU enable — clearing it resets every APU register to 0 except wave RAM?**
A common bug preserves register values across the off→on transition.

---

## 5. Channel-specific quirks (methodology rule 3)

### Square channels (CH1, CH2)

**`[NES]`** **Sweep silences (not just mutes) the channel when target period < 8 OR target period > 0x7FF?**
Two slightly different overflow rules: square 1 uses one's-complement of the shift result; square 2 uses two's-complement. Asymmetric behaviour — not a bug.

**`[NES]` / `[GB]`** **Duty cycle position is preserved across silence/length-counter-zero?**
The duty position counter does not reset on length-counter expiry. Restarting the channel resumes at the held position, not at 0.

### Triangle (NES) / Wave (GB CH3)

**`[NES]`** **Triangle 11-step sequencer holds phase across silence — when length counter or linear counter is 0, the channel freezes but does NOT reset phase?**
Output amplitude stops where it was; resumption is fade-in at the held phase.

**`[GB]`** **CH3 wave RAM ($FF30–$FF3F) access while CH3 is enabled returns garbage / corrupts wave RAM on DMG (the "wave RAM read corruption bug")?**
CGB allows access during the same M-cycle the APU is about to read, with deterministic results.

### Noise (CH4)

**`[NES]` / `[GB]`** **LFSR is initialised to a non-zero pattern on channel trigger (NES: 1; GB: 0xFFFF / 0x7FFF depending on width mode)?**
Initialising to 0 = silence forever. Common bug.

**`[NES]`** **LFSR width is 15 bits with feedback from bits 0 and 1 (or bits 0 and 6 in "short mode" set by $400E bit 7)?**
Two LFSR modes; the period table also depends on mode for some indices.

### DMC

**`[NES]`** **Output level can be written directly via $4011 (one-shot click), in addition to running sample playback?**
Many games do this for sound effects layered over music. A bug that requires DMC to be in playback to honour $4011 writes breaks them.

---

## 6. Sample resampling (methodology rules 2 & 4)

**Scope: rate correctness and determinism, not audio quality.** DSP filter design is out of scope (it's a design-review concern). The questions here are narrower.

**Question:** Does the APU run at the documented internal rate for the region — NES NTSC ~894886.5 Hz output rate (CPU rate / 2) before resampling, NTSC and PAL differ; GB 4194304 Hz internal divided down to per-channel rates per the period registers?

**Question:** Is resampling to host audio rate **deterministic**? Same input APU samples + same host rate must produce identical output buffers across runs. Non-deterministic resampling (e.g., a filter that uses uninitialised state, or a SRC library that uses non-deterministic threading) is a determinism bug under rule 4 even if it sounds fine.

**Question:** Per-channel mixer mix-down weights match the NES non-linear mix table (or GB / SNES equivalent), not a uniform "average them"?
NES has a documented non-linear mix function (square + triangle + noise + DMC each with their own weighting + cross-channel non-linearity). A linear mix sounds wrong on chord-heavy tracks.

**Question:** Sample boundary alignment — does the resampler buffer or interpolate across frame boundaries, and is the buffer state part of the save-state?
A save/load mid-frame that drops the resampler buffer produces an audible click.

---

## 7. Implicit state (methodology rule 5)

Same enumeration discipline as `mapper_review.md` §5: the failure mode is omission, so the only correct review activity is **walking every field on the canonical per-component reference** and verifying each round-trips through the serializer/deserializer. If a field on the canonical list isn't in the serializer, it's a finding.

**`[NES]`** Walk against [nesdev wiki — APU](https://www.nesdev.org/wiki/APU) (per-channel state) plus [APU frame counter](https://www.nesdev.org/wiki/APU_Frame_Counter) (cross-channel state — mode bit, step+sub-cycle position, IRQ-enable, IRQ-asserted, cycle index since last $4017 write for the 3/4 cycle delay window). The per-channel implicit fields the reference lists (length counter, envelope volume/start-flag/divider, sweep divider + reload flag, linear counter + reload, LFSR state, duty position, DMC sample buffer / bits-remaining / address counter / bytes-remaining / silence flag / IRQ flag / enable flag) are easy to omit because the names look like internal state rather than registers.

If a resampler is present, its filter taps, fractional sample-position accumulator, and output-buffer pointer must round-trip too — otherwise mid-frame save/load produces an audible click.

**`[GB]`** Walk against [pandocs — Audio details](https://gbdev.io/pandocs/Audio_details.html). Per-channel implicit fields (length counter, envelope volume / timer / direction, DAC enable, period timer, frequency timer, output DAC value) plus cross-channel (frame sequencer step, NR52 master enable, NR50 mixer levels, NR51 panning, the internal DIV-counter value the frame sequencer ticks off). Wave RAM round-trips iff mid-playback save semantics matter (otherwise it's regular IO state).

**`[SNES]` SPC700/DSP** Walk against [fullsnes — APU and DSP sections](https://problemkaputt.de/fullsnes.htm). The full SPC700 register file (A/X/Y/SP/PC/PSW), the entire 64 KiB APU RAM, all DSP registers (per fullsnes's exact count — 8 voices × per-voice registers + global registers), per-voice envelope/gain phase, per-voice BRR decode state (last 2 samples for filter, current block index), echo buffer write pointer, and the four communication-port bytes ($2140–$2143 from each side's perspective).

---

## 8. Determinism (methodology rule 4)

**Question:** APU power-on state is deterministic — channels disabled, length counters cleared, envelopes at 0, frame counter at step 0, mode 4-step, IRQ-inhibit clear?

**Question:** No host-time-based seeding for noise LFSR, envelope timing, or any other apparent randomness?

**Question:** SPC700 boot handshake is deterministic — even if the SPC and S-CPU clocks are in fixed integer ratio (which they aren't on real hardware), the emulator must pick a fixed ratio and stick to it for golden-frame regression tests.

---

## 9. Region / revision (methodology rule 7)

**`[NES]`** DMC rate table and noise period table are region-parameterized — NTSC and PAL have different sequences (canonical tables at [nesdev wiki — APU DMC](https://www.nesdev.org/wiki/APU_DMC) and [APU](https://www.nesdev.org/wiki/APU)). Hardcoding either region's table silently breaks releases on the other.

**`[NES]`** Frame-counter step boundaries are region-parameterized too — see §1; NTSC and PAL each have their own sequence at [nesdev wiki — APU frame counter](https://www.nesdev.org/wiki/APU_Frame_Counter). Verify region-threaded.

**`[GB]`** APU is region-independent (no PAL Game Boy).

**`[SNES]`** SPC700 + DSP are region-independent in the same sense — PAL vs NTSC SNES doesn't change SPC700 timing. SPC700 silicon revision varies the clock by ~5% (review-heuristic-flavoured fact); games sensitive to handshake timing (boot-up in particular) can fail when the emulator drives the SPC at a fixed-integer S-CPU ratio that real hardware doesn't have.

---

## 10. Citation hygiene (methodology rule 6)

The APU has the densest citation surface — almost every register has a quirk that nesdev or pandocs documents, and the cite-or-flag rule is strict here. When you encounter:

- **Unusual constant in a sweep, envelope, or LFSR initialisation:** demand a citation. APU magic numbers are *not* implementation choices.
- **Comment claiming a behaviour matches "real hardware" without naming the test ROM:** ask which one. Blargg, mooneye, and APU-specific test homebrew are the standard.
- **Code that toggles two bits "to be safe":** suspect rule 4 (register-bit sharing) — it may be that the two bits are actually the same flip-flop and the cite-or-remove rule applies.

---

## 11. Test-ROM correspondence (methodology rule 1)

This section maps **review triggers → ROMs to re-run**. For ROM identity follow the canonical-source pointers in `references/test_roms.md` to the upstream archive.

| Change touches... | Re-run at minimum (NES) |
|---|---|
| Frame counter / $4017 timing | Blargg `apu_test/`, `frame_irq_test/`, `apu_reset/` |
| DMC DMA cycle stealing | `dmc_dma_during_read4/`, `cpu_dummy_writes_oam` |
| DMC IRQ | `apu_test/`, `dmc_basics/` |
| Length counters | `length_counter/` (Blargg) |
| Envelope / sweep | `square_timer_div2/`, sweep test ROMs |
| LFSR | direct comparison with reference noise output |
| Mixer non-linearity | listening test against reference recordings |

| Change touches... | Re-run at minimum (Game Boy) |
|---|---|
| Frame sequencer / DIV interaction | mooneye `acceptance/timer/`, `acceptance/sound/` |
| Length counter extra-clock | mooneye `acceptance/sound/length_counter` |
| CH3 wave RAM corruption (DMG) | mooneye DMG-only suite |
| LFSR | listening tests; mooneye doesn't cover LFSR initialisation comprehensively |

| Change touches... | Re-run at minimum (SNES) |
|---|---|
| SPC700 instruction correctness | peter_lemon `SPCTest/` |
| DSP voice envelopes | blargg's snes_apu_tests if available; otherwise listening tests against bsnes-emu/bsnes |
| Communication-port handshake | game-level test (any game's boot succeeds = handshake works) |

An APU change with no associated test-ROM verification path is a yellow flag — note it explicitly and propose the minimal homebrew that would isolate the behaviour.
