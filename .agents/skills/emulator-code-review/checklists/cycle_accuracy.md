# Cycle Accuracy — bugs to spot in code already written

This file is **not** a guide to choosing a cycle-accuracy architecture. The skill takes architecture as given (`SKILL.md` anti-features). This reference catalogues the bugs that recur in code written in each of the three common styles, plus the cross-architecture concerns of scheduler ordering and CPU/DMA races.

**Three architectures, three distinct bug catalogues.** Resist the temptation to write one unified bug list — the failure modes look nothing alike, and a unified list will train the reviewer to see all three through the lens of whichever architecture they happen to know best.

Cite **byuu's bsnes documentation** for catch-up scheduler patterns, **mooneye-gb source** for per-cycle state machines, and **fceux / Mesen source** for micro-op table approaches as exemplars.

---

## How to identify the architecture in 30 seconds

Look at the CPU's `step()` (or `tick()`) function:

| Signal | Architecture |
|---|---|
| Function returns after exactly one master/sub-cycle of work, with all component state mutated in place | **Per-cycle state machine** |
| Function reads the next entry from a per-opcode array, applies it, advances an index | **Micro-op table** |
| Function runs the CPU forward by N cycles, then *separately* invokes "PPU.catch_up_to(now)" / "APU.catch_up_to(now)" calls when state from those components is needed | **Catch-up scheduler** |

Hybrids exist (per-cycle CPU with catch-up audio is common). When in doubt, ask which component "owns" the current timestamp — if it's a global scheduler, you're in the catch-up family; if each component runs in lockstep on a shared tick, you're in the per-cycle family.

---

## 1. Per-cycle state machine — bugs to spot

The CPU/PPU/APU each advance one minimum-time-unit per `step()` call. State machines explicit; transitions visible in code.

**Forgetting to advance on a particular T-state / sub-cycle.**
Example: GB CPU has 4 T-states per M-cycle. A `LD A, (HL)` runs through T1 (fetch), T2 (memory address setup), T3 (memory read), T4 (commit to A). If T2 silently does nothing in the implementation but real hardware drives the address bus on T2, anything sniffing the bus at that T-state diverges. The bug is invisible to most software but breaks bus-snoop test ROMs.

**Edge-case state with no transition out.**
A state added for a rarely-reached condition (e.g., HALT bug entry, STP/WAI on 65C816) without a documented exit transition. Implementation reaches the state, never leaves it, locks up. The give-away in code: a state in the enum with no entry in the transition table.

**State enum growing past comprehensibility.**
When the enum has 100+ entries and the transition function is a labelled switch with conditional gotos, fields tracking "which sub-state am I in" multiply. The bug class: setting one tracking field but not the related one, producing impossible state combinations. Not a single bug — a *family* of bugs detectable by code structure.

**Skipping a state under specific flag conditions.**
Branch instructions with `if (taken && page_cross) { state = T5 } else { state = T4 }` — fine. Branch instructions with `if (taken) { state = T4 }` and a separate page-cross check inside T4 that *also* sets `state = T4 again` — the second-cycle-of-page-cross gets lost. Only catches with cycle-counting test ROMs.

**Mid-instruction state forgotten in save-states.**
Per-cycle architectures *always* have implicit state — the current opcode's T-state index, partial register reads, address-bus latch. Per-cycle is the architecture *most* prone to incomplete save-states because the implicit state is structurally everywhere. Cross-reference each component's `_review.md` §5.

**Performance bait: switch-on-state vs jump-table.**
A state machine that compiles to a tight labelled switch is correct *and* fast — do not push toward function-per-state for performance reasons.

---

## 2. Micro-op table — bugs to spot

Each opcode decoded once into a list of micro-operations (READ, WRITE, MODIFY, NOP, BRANCH, etc.); CPU advances by consuming one micro-op per cycle.

**Off-by-one on the last entry.**
The most common micro-op-table bug. Common shapes:
- Table length = 6, advance index 0 → 5 → reset to 0; the entry at index 5 fires and *also* the next opcode's index-0 fires on the same cycle. Symptoms: every instruction effectively executes one cycle fast.
- Table length = 6, advance index 0 → 5 → reset; entry at index 5 *doesn't* fire, only entries 0–4 do. Symptoms: every instruction runs one cycle slow.
Look at the loop bound (`< n` vs `<= n`) and the reset point (before vs after the last micro-op).

**Variable-length micro-op consuming the wrong number of cycles.**
Some micro-ops (e.g., a memory read from a slow region) inherently consume multiple master cycles. If the table represents "one entry = one master cycle" the slow-region micro-op needs to either expand into multiple entries or signal "stay on this entry for N more cycles." Implementations that conflate the two — entry consumes one cycle but adds extra cycles via a side counter — silently miss the side counter when the micro-op is at a position the dispatch loop doesn't check.

**Table indexing bug that survives common opcodes but breaks BRK / RTI / interrupt vectoring.**
BRK on the 6502 has a unique flow: stack pushes, vector fetches, status manipulation, partial flag clearing. RTI similarly. If the table is built by a code generator that treats `BRK` as "BRK = NOP with side effects" the side effects come out at the wrong cycle. Check that BRK and RTI have hand-written tables or that the generator special-cases them.

**Conditional micro-ops handled inline vs as table entries.**
"Page-crossed adds a cycle" can be implemented as (a) the table has a conditional entry whose effect is "skip / do an extra read depending on flag," or (b) the dispatch loop checks a per-opcode flag and inserts an extra cycle around the table walk. Both work. The bug appears when the implementation does *both* — extra cycle counted twice.

**Dummy reads and dummy writes as separate micro-ops vs implicit in addressing-mode resolution.**
6502 RMW: read → write-back-original → write-modified. Three distinct memory operations on consecutive cycles. A micro-op table that has only `READ` and `WRITE` (no `WRITE_BACK_ORIGINAL`) silently omits the dummy write. Catches: Blargg `cpu_dummy_writes_*`.

**Interrupt-poll micro-op location.**
The 6502 polls interrupts on the second-to-last cycle of every instruction (penultimate-cycle rule). In a micro-op table, this is either an explicit `POLL_INTERRUPTS` micro-op at position `n-2` or an implicit "always poll on penultimate" rule baked into the dispatch loop. The bug to spot: tables generated programmatically may put `POLL_INTERRUPTS` at the wrong position for non-standard-length opcodes (BRK, RTI, JSR — instruction lengths vary).

---

## 3. Catch-up scheduler — bugs to spot

CPU runs forward by N cycles; other components stay frozen at their last-known timestamp. When the CPU touches a component's register or the component needs to fire an interrupt, "catch-up" runs the component forward to the global timestamp.

**Component A catches up past an event that affects component B.**
Concrete example: PPU catches up to cycle 1000 to service a CPU read of $2002 at cycle 1000. During catch-up, PPU's vblank flag is supposed to assert at cycle 998. But the catch-up logic processes vblank assertion *after* the read returns, so the read sees vblank=0 instead of vblank=1. The fix is to interleave catch-up and event firing — a single "catch up to T, processing all events ≤ T in chronological order" function. The bug: catching up *then* firing events.

**Ordering when two components cross the same boundary on the same master cycle.**
Two PPU events at the same dot — say, vblank flag set and NMI line goes high — must fire in a documented order. If the scheduler processes them in registration order or pointer-comparison order, the order is non-deterministic across builds.

**Catch-up loop that re-reads the global time.**
```
while (component.time < scheduler.time) {
    component.step();
}
```
If `component.step()` *also* advances `scheduler.time` (e.g., by triggering a CPU bus access through the scheduler), the loop runs one iteration too far or one iteration short. The fix is to snapshot `scheduler.time` before the loop. Look for this exact pattern.

**"Last-event-time" tracking lossy when event happens between catch-up calls.**
If the PPU is caught up only when the CPU reads a PPU register, and the CPU never reads a PPU register for an entire frame, the PPU's last-known timestamp is hours behind. When something forces a catch-up (frame end, save state), the PPU must traverse all the events that happened in between, in order. A "skip ahead to T" optimisation that bypasses event processing in the gap = silently dropped IRQs, missed sprite-0 hits.

**DMA start mid-component-catch-up.**
DMA registration happens via a CPU-bus write. If the write triggers a catch-up of all components (say, to commit pending PPU state before the bus sees the write), and the catch-up *itself* causes a further catch-up of the just-registered DMA — recursion. Common shape: the catch-up function isn't re-entrant safe.

**Save-state captured mid-catch-up.**
Catch-up architectures are where this hazard most often surfaces, but it's broader than one architecture — see `checklists/save_state_review.md` ("Save-during-execution hazard") for the cross-cutting treatment and the architectures it affects.

---

## 4. Scheduler ordering when components share a clock (cross-architecture)

The single densest catch-up bug class. Per-cycle architectures sidestep most of it by construction; catch-up architectures live or die by it.

**Question: when component X catches up to time T, does it process events scheduled at time T-1 before events at time T?**
"Process all events with time ≤ T" — but in *what order* among events with the same timestamp? A heap keyed only on time has nondeterministic tie-break. The standard fix: secondary key on event-source priority, or a deterministic insertion order.

**Question: does the order of catch-up calls affect output?**
If `cpu.catch_up(); ppu.catch_up();` produces different state from `ppu.catch_up(); cpu.catch_up();`, the scheduler is missing events between them. Component independence at any given instant is the whole point of the catch-up architecture — losing it is the architecture-defeating bug.

**Question: do components share a clock with non-trivial divider?**
NES: PPU runs at 3× CPU rate (or 3.2× PAL); APU runs at CPU rate; mappers may tick on PPU dots or CPU cycles. SNES: master clock with per-region divider per access type. GB: PPU and APU both at CPU rate, but mode 3 length is variable per scanline.

A common bug: scheduler advances "by 1 CPU cycle" but PPU is owed 3 dots; if the implementation processes 3 PPU dots as a single batch instead of three discrete events, mid-batch interactions (CPU writes $2006 between dot 1 and dot 2) are lost.

**Question: are scheduler events deterministic across save/load?**
Save state captures the event queue *contents*, but what about pointer identity, event ID assignment, and per-source counters used for tie-breaks? A queue that compares events by pointer breaks on load (new pointers). A queue that compares by insertion-order ID breaks if IDs aren't part of the save format.

---

## 5. CPU/DMA race spotting (cross-architecture)

DMA-vs-CPU arbitration looks different per platform; the review questions are platform-specific even though the *category* (CPU suspension / re-entry / observable-mid-DMA-state) is shared.

**`[NES]`** OAM DMA: CPU suspended mid-instruction *between bus cycles*, not on instruction boundary. Resumes on the next CPU cycle after the 513 or 514 DMA cycles. Re-entry: PC unchanged, the CPU's mid-instruction state machine continues from where it was suspended. Bug to spot: implementations that conflate "CPU halted" with "CPU advanced to next instruction" effectively skip an instruction.

**`[NES]`** DMC fetch stealing: 1–4 cycles depending on alignment. A CPU instruction running through cycles N, N+1, N+2, N+3 may have a DMC fetch insert at cycle N+1 (extending it), at cycle N+2 (different cost), or not at all. The CPU's notion of "current cycle" must distinguish *instruction-internal cycle index* from *master-clock-cycle index*.

**`[GB]`** OAM DMA does NOT halt the CPU. It restricts the bus (non-HRAM reads return $FF). The bug: implementations that halt the CPU during OAM DMA pass mooneye `oam_dma/basic.gb` (which doesn't check) but fail any test ROM that runs CPU code from HRAM during DMA.

**`[GB]`** Interrupt service has a 5-M-cycle handshake (2 NOPs, 2 cycles of PC push, 1 cycle of vector load). DMA *starting* during this handshake is fine; DMA *active* when the handshake begins serializes correctly because of the HRAM-bus restriction. The race to spot: an interrupt firing on the first cycle of OAM DMA — does the handler push the correct PC?

**`[SNES]`** DMA halts CPU at byte boundaries but not necessarily at instruction boundaries (DMA can interrupt a multi-byte fetch). Re-entry: similar to NES OAM DMA but at master-cycle granularity, with MEMSEL-dependent re-entry timing.

**`[SNES]`** HDMA preempts in-progress DMA at HBlank — the in-progress DMA's state must save (which channel, how many bytes left, current source pointer) and resume after HDMA completes.

**`[SNES]`** WRAM-targeting DMA conflicts with CPU WRAM access. See `bus_review.md` §2.

---

## 6. Architectural anti-pattern: "fix the bug at the wrong level"

The skill is anti-architecture-redesign. But one anti-pattern is a code-level bug, not an architectural one, and worth flagging:

**Catching up a component to time T, finding state inconsistent, then *post-hoc adjusting* the timestamp.**

Code shape: `catch_up(t); if (component.something_inconsistent) { component.time = t - k; }`. The "fix" silently rewinds the component, hiding a real ordering bug. If the catch-up logic produces inconsistent state, the fix is in the catch-up logic, not in adjusting the timestamp afterward.

This is in scope because it's a code-level patch, not an architecture choice.

---

## 7. Zig-specific implementation notes

The skill is platform-aware and language-incidental, but two Zig 0.16+ patterns affect cycle-accuracy code review specifically:

**Labelled switch with `continue :dispatch .next` for opcode dispatch.**
A 256-case `switch` labelled `dispatch` where each opcode body ends with `continue :dispatch <next_opcode>` (when the next opcode is computable inline) lets LLVM inline the dispatched-to handler into the dispatching handler's tail. Function-pointer tables (e.g., `handlers[opcode]()`) cannot do this — the indirection blocks inlining. For cycle-accurate code, the labelled-switch shape is correct and produces measurably better code; do not suggest extracting opcode handlers into separate functions.

**`switch` exhaustiveness on packed enums representing CPU/PPU state.**
Zig's exhaustiveness check catches "added a new state, forgot to handle it in the transition function." Per-cycle state machines benefit massively. Code review should *expect* exhaustiveness — a `_ => unreachable` clause that exists "just in case" is suspicious; it papers over the exhaustiveness check that would otherwise catch missing transitions.

These are language-incidental notes, not the substance of cycle-accuracy review. Lead with the architecture and the bugs; treat Zig-specific patterns as the implementation language detail they are.

---

## 8. Common cited references

- byuu / Near, "Hello, world!" essay on bsnes scheduler architecture (the canonical catch-up exposition)
- Mesen / fceux source — micro-op table reference for 6502
- mooneye-gb source — per-cycle state machine reference for LR35902
- bsnes-emu/bsnes / higan source — catch-up scheduler reference for 65C816 + SPC700
- Andrew Kelley's Zig 0.16 release notes on labelled switch (for the dispatch pattern)

When code cites any of the above and matches, trust it. When code is structurally unusual and cites nothing, that's the bug.
