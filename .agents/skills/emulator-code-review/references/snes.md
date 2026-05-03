# SNES Citations (Super Famicom / SNES)

Cite **fullsnes** (`problemkaputt.de/fullsnes.htm`) as primary. Anomie's docs, byuu/Near's bsnes/higan technical notes, and the Super Famicom Development Wiki (`wiki.superfamicom.org`) are also authoritative. The Nintendo Developer Manual is **not** — it documents intent, not silicon.

---

## Common cited references

- [fullsnes (problemkaputt.de)](https://problemkaputt.de/fullsnes.htm) — primary; covers CPU/PPU/APU/DMA/HDMA/coprocessors/headers
- [Super Famicom Development Wiki](https://wiki.superfamicom.org/)
- anomie's docs — 65C816 mode boundaries, M/X corner cases, direct-page wrap; mirrored in many places
- [bsnes / higan source + comments](https://github.com/bsnes-emu/bsnes) — byuu/Near's bsnes is the de-facto behavioural reference (replaces the now-404 bsnes-mercury repo)
- [SNES Dev Manual](https://archive.org/details/SNESDevManual) — **for context only**, not authoritative
- nocash, neviksti, Overload (early SNES technical contributors)
- [peter_lemon SNES test suite](https://github.com/PeterLemon/SNES) — CPUTest, HDMA, Mode7, OAM, OpenBus, etc.

When a comment cites any of the above and the code matches, trust it. When code is quirky and cites nothing, that's the bug.
