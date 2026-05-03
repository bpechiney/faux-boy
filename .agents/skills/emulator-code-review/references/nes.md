# NES Citations (RP2A03 / 2A07, RP2C02 / 2C07)

Cite **nesdev wiki** (`wiki.nesdev.org`) as canonical for every behaviour. Manufacturer datasheets are *not* authoritative.

---

## Common cited references

- [NES dev wiki — CPU](https://www.nesdev.org/wiki/CPU)
- [NES dev wiki — CPU unofficial opcodes](https://www.nesdev.org/wiki/CPU_unofficial_opcodes)
- [NES dev wiki — CPU power-up state](https://www.nesdev.org/wiki/CPU_power_up_state)
- [NES dev wiki — Cycle reference chart](https://www.nesdev.org/wiki/Cycle_reference_chart)
- [NES dev wiki — PPU](https://www.nesdev.org/wiki/PPU)
- [NES dev wiki — PPU registers](https://www.nesdev.org/wiki/PPU_registers)
- [NES dev wiki — PPU rendering](https://www.nesdev.org/wiki/PPU_rendering)
- [NES dev wiki — PPU sprite evaluation](https://www.nesdev.org/wiki/PPU_sprite_evaluation) — sprite-overflow hardware bug
- [NES dev wiki — PPU OAM](https://www.nesdev.org/wiki/PPU_OAM)
- [NES dev wiki — APU](https://www.nesdev.org/wiki/APU)
- [NES dev wiki — APU frame counter](https://www.nesdev.org/wiki/APU_Frame_Counter)
- [NES dev wiki — APU DMC](https://www.nesdev.org/wiki/APU_DMC)
- [NES dev wiki — DMA](https://www.nesdev.org/wiki/DMA) — OAM DMA + DMC fetch cycle stealing
- [NES dev wiki — Open bus behaviour](https://www.nesdev.org/wiki/Open_bus_behavior)
- [NES dev wiki — Standard controller](https://www.nesdev.org/wiki/Standard_controller) — DMC DMA glitch
- [NES dev wiki — MMC3](https://www.nesdev.org/wiki/MMC3)
- [NES dev wiki — Mappers](https://www.nesdev.org/wiki/Mapper)
- [NES dev wiki — NES 2.0](https://www.nesdev.org/wiki/NES_2.0)
- Disch's mapper documents (mirrored on nesdev)
- visual2A03, visual2C02 (Greg James et al.) — silicon-level reference for the truly contested behaviours

When a comment cites any of these and the code matches, trust it. When code is quirky and cites nothing, that's the bug.
