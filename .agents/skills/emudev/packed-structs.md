# Packed structs — hardware register layouts

Modeling hardware registers (status flags, mode controls, channel parameters, etc.) with `packed struct` and `@bitCast` in Zig 0.16. Lean reference; not an exhaustive Zig packed-struct manual.

## The pattern

A hardware register is a fixed-width byte (or 16-bit word) where each bit (or bit range) has a named semantic. Model it as a `packed struct` with explicit bit-width fields, sized to match the underlying width:

```zig
// REF: <hardware doc anchor for this register>
pub const SomeReg = packed struct(u8) {
    flag_a: bool, // bit 0
    mode:   Mode, // bits 1-2
    // ... remaining bits ...
    enable: bool, // bit 7
};

comptime { std.debug.assert(@bitSizeOf(SomeReg) == 8); }
```

Note `packed struct(u8)` — the explicit backing type. This makes `@bitCast` to/from `u8` zero-cost and well-defined. Per-bit-width nested enums (`enum(u1)`, `enum(u2)`) are the right shape for multi-bit semantic fields.

## Reading and writing through the bus

The bus exposes register I/O as `u8` reads and writes. Convert at the boundary using `@bitCast`:

```zig
pub fn busRead(self: *Ppu, addr: u16) u8 {
    return switch (addr) {
        REG_A_ADDR => @bitCast(self.reg_a),
        // ...
    };
}

pub fn busWrite(self: *Ppu, addr: u16, val: u8) void {
    switch (addr) {
        REG_A_ADDR => self.writeRegA(@bitCast(val)),
        // ...
    }
}
```

Centralizing the `@bitCast` at the bus boundary keeps the rest of the code reading typed fields (`self.reg_a.enable` is clearer than `(reg >> 7) & 1`). Quirky write-side effects (e.g., resetting derived state when a flag transitions) live in the dedicated `writeXxx` function, not inline in the bus dispatch.

## Bit-order gotchas

Zig packs `packed struct` fields starting from the least significant bit. **Always declare fields bit-0 first**, even when the hardware doc lists them bit-7 first.

If you copy a hardware doc table top-down, you'll get the order reversed. Verify with a `comptime` assertion that a known-bit-pattern decodes correctly:

```zig
comptime {
    var x: SomeReg = @bitCast(@as(u8, 0b1000_0000));
    std.debug.assert(x.enable);   // bit 7 should be the enable flag
    std.debug.assert(!x.flag_a);  // bit 0 should be clear
}
```

Bake one of these per register so future-you can't misread the bit order silently.

## Read-only / write-only / read-with-side-effects

Some hardware registers have asymmetric read/write semantics, or reads with side effects (e.g., a status read that clears a latch). Model the *storage* with a packed struct; encode the asymmetry in the bus-boundary functions, **not** in the struct type:

```zig
pub fn busRead(self: *Ppu, addr: u16) u8 {
    return switch (addr) {
        STATUS_ADDR => blk: {
            // QUIRK: reading clears <bit> and resets <latch>.
            // REF: <doc anchor>
            const out: u8 = @bitCast(self.status);
            self.status.<bit> = false;
            self.<latch> = .reset;
            break :blk out;
        },
        // ...
    };
}
```

A register isn't "read-only at the type level" — it's read-only at the bus boundary, which is where the asymmetry is enforced.

## `@bitCast` alignment traps

`@bitCast` between equally-sized types is well-defined. Going through a pointer (`@ptrCast`) is **not** the same and carries alignment requirements. For register I/O, always `@bitCast` values, never `@ptrCast` register memory.

```zig
// GOOD:
const out: u8 = @bitCast(self.some_reg);

// BAD: alignment-fragile, can violate strict aliasing.
const ptr: *SomeReg = @ptrCast(&self.some_reg_byte);
```

## Non-byte-width registers

When a logical register isn't byte-sized (e.g. a 15-bit internal counter), declare the backing type to match the logical width — `packed struct(u15)` rather than `u16`. That documents the width in the type and prevents accidental sign-extension when the high bit is read.

## Sprite tables: array of packed structs

OAM / sprite tables are arrays of packed structs. The main per-sprite entry is typically 4 bytes; some systems carry auxiliary metadata in a separate, smaller table (e.g., SNES splits OAM into a primary table at 4 bytes per sprite plus a secondary table that packs upper-X / size bits across multiple sprites). Both shapes use the same pattern — a packed-struct array plus byte-indexed bus reads:

```zig
pub fn oamRead(self: *Ppu, addr: u8) u8 {
    const sprite_idx = addr / @sizeOf(Sprite);
    const field_idx  = addr % @sizeOf(Sprite);
    const bytes: [@sizeOf(Sprite)]u8 = @bitCast(self.oam.sprites[sprite_idx]);
    return bytes[field_idx];
}
```

Declare the sprite struct's fields in the order the hardware lays them out in memory; `@bitCast` then produces the correct byte order. Verify with a `comptime` assertion against a known sprite encoding from the system's spec. For systems with split OAM tables, repeat the same pattern for each table with its own packed-struct type.

## Cross-references

- [comments.md](./comments.md) — every register definition carries a `REF` to its hardware doc.
- [dispatch.md](./dispatch.md) — opcode handlers that read/write through the bus call into these typed accessors.
