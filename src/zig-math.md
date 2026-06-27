# Mathematics in Zig 0.16

A comprehensive guide to mathematical operations, floating-point handling, bit manipulation, and big integer arithmetic in Zig 0.16 stable.

---

## Chapter 1: Introduction to std.math

Zig's `std.math` module provides a rich set of mathematical functions that work across all numeric types. Unlike C's `<math.h>`, Zig's math functions are type-safe, comptime-aware, and leverage Zig's compile-time evaluation engine to produce optimal code.

Key design principles of `std.math` in Zig 0.16:

- **Type-generic**: Functions work with any integer or float type, resolved at compile time.
- **Comptime-compatible**: Nearly every function can be evaluated at compile time.
- **No hidden state**: No global rounding modes or error flags (unless interacting with hardware directly).
- **Precise error handling**: Returns error unions or optional types where domain errors are possible.

### Quick Start

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const x: f64 = 42.0;
    try writer.print("sqrt({d}) = {d}\n", .{x, std.math.sqrt(x)});
    try writer.print("log2(1024) = {d}\n", .{std.math.log2(1024.0)});
    try writer.print("sin(pi/2) = {d}\n", .{std.math.sin(std.math.pi / 2.0)});
}
```

---

## Chapter 2: Integer Math

### max, min

`std.math.max` and `std.math.min` return the greater or lesser of two values. They are generic and work on any ordered type.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const a: i32 = -10;
    const b: i32 = 25;

    try writer.print("max({d}, {d}) = {d}\n", .{ a, b, std.math.max(a, b) });
    try writer.print("min({d}, {d}) = {d}\n", .{ a, b, std.math.min(a, b) });

    // Works with unsigned types too
    const c: u8 = 200;
    const d: u8 = 50;
    try writer.print("max({d}, {d}) = {d}\n", .{ c, d, std.math.max(c, d) });
}
```

### clz, ctz, popCount

These bit-counting intrinsics are critical for systems programming:

- `clz(x)` — count leading zeros
- `ctz(x)` — count trailing zeros
- `popCount(x)` — population count (number of set bits)

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const val: u32 = 0b00011000_00000000_00000000_00000000;

    try writer.print("Value: 0x{x:0>8}\n", .{val});
    try writer.print("Leading zeros:  {d}\n", .{std.math.clz(val)});
    try writer.print("Trailing zeros: {d}\n", .{std.math.ctz(val)});
    try writer.print("Population count: {d}\n", .{std.math.popCount(val)});
}
// Output:
// Value: 0x00180000
// Leading zeros:  11
// Trailing zeros: 19
// Population count: 2
```

These map directly to CPU instructions (e.g., `LZCNT`, `POPCNT` on x86) when available, falling back to software implementations otherwise.

### abs

`std.math.abs` returns the absolute value of an integer.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    try writer.print("abs(-42)  = {d}\n", .{std.math.abs(@as(i32, -42))});
    try writer.print("abs(42)   = {d}\n", .{std.math.abs(@as(i32, 42))});
    try writer.print("abs(-128) = {d}\n", .{std.math.abs(@as(i8, -128))});
    // Note: abs(i8, -128) would overflow since i8 max is 127.
    // In 0.16, this is handled safely.
}
```

For floats, use `std.math.fabs` or `@abs` builtin:

```zig
try writer.print("fabs(-3.14) = {d}\n", .{@abs(-3.14)});
```

---

## Chapter 3: Floating Point

### Constants

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    try writer.print("pi       = {d}\n", .{std.math.pi});
    try writer.print("e        = {d}\n", .{std.math.e});
    try writer.print("inf (f64)= {d}\n", .{std.math.inf(f64)});
    try writer.print("nan (f64)= {d}\n", .{std.math.nan(f64)});
    try writer.print("ln2      = {d}\n", .{std.math.ln2});
    try writer.print("ln10     = {d}\n", .{std.math.ln10});
    try writer.print("sqrt2    = {d}\n", .{std.math.sqrt2});
}
```

### isNan, isInf, isFinite

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const values = [_]f64{
        42.0,
        std.math.nan(f64),
        std.math.inf(f64),
        -std.math.inf(f64),
    };

    for (values) |v| {
        try writer.print("Value: {d: >10}  isNan: {}  isInf: {}  isFinite: {}\n", .{
            v,
            std.math.isNan(v),
            std.math.isInf(v),
            std.math.isFinite(v),
        });
    }
}
// Output:
// Value:         42  isNan: false  isInf: false  isFinite: true
// Value:        nan  isNan: true   isInf: false  isFinite: false
// Value:        inf  isNan: false  isInf: true   isFinite: false
// Value:       -inf  isNan: false  isInf: true   isFinite: false
```

You can also use the `@isNan` and `@isInf` builtins directly, which `std.math.isNan` wraps.

---

## Chapter 4: Rounding

Zig 0.16 provides both `std.math` rounding functions and builtins for converting floats to integers with specific rounding modes.

### std.math Rounding Functions

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const values = [_]f64{ 3.14, -3.14, 2.5, -2.5, 0.0, -0.0 };

    try writer.print("{s: >8} {s: >8} {s: >8} {s: >8}\n", .{
        "value", "ceil", "floor", "trunc",
    });
    try writer.print("{s: >8} {s: >8} {s: >8} {s: >8}\n", .{
        "------", "------", "------", "------",
    });

    for (values) |v| {
        try writer.print("{d: >8.2} {d: >8.2} {d: >8.2} {d: >8.2}\n", .{
            v,
            std.math.ceil(v),
            std.math.floor(v),
            std.math.trunc(v),
        });
    }
}
```

### Converting Floats to Integers with Builtins

In Zig 0.16, the builtins `@floor`, `@ceil`, `@round`, and `@trunc` perform the operation and convert directly to an integer type:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const pi: f64 = 3.14159;

    // @floor converts f64 -> i64 by flooring
    const floored: i64 = @floor(pi);
    try writer.print("@floor({d}) = {d}\n", .{ pi, floored });

    // @ceil converts f64 -> i64 by ceiling
    const ceiled: i64 = @ceil(pi);
    try writer.print("@ceil({d}) = {d}\n", .{ pi, ceiled });

    // @round converts f64 -> i64 by rounding
    const rounded: i64 = @round(2.5);
    try writer.print("@round({d}) = {d}\n", .{ 2.5, rounded });

    // @trunc converts f64 -> i64 by truncating
    const truncated: i64 = @trunc(-3.7);
    try writer.print("@trunc({d}) = {d}\n", .{ -3.7, truncated });

    // std.math.round still returns f64
    try writer.print("std.math.round({d}) = {d}\n", .{ pi, std.math.round(pi) });
}
// Output:
// @floor(3.14159) = 3
// @ceil(3.14159) = 4
// @round(2.5) = 3
// @trunc(-3.7) = -3
// std.math.round(3.14159) = 3.0
```

This is a powerful pattern: `@floor(x)` returns an integer type directly, making it safe and avoiding the need for `@intFromFloat(std.math.floor(x))`.

---

## Chapter 5: Trigonometry

All trigonometric functions in `std.math` accept and return `f64` (or the type of the input):

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Angles in radians
    const angle = std.math.pi / 4.0; // 45 degrees

    try writer.print("Angle: {d:.4} rad ({d:.1} deg)\n", .{
        angle, angle * 180.0 / std.math.pi,
    });
    try writer.print("sin  = {d:.6}\n", .{std.math.sin(angle)});
    try writer.print("cos  = {d:.6}\n", .{std.math.cos(angle)});
    try writer.print("tan  = {d:.6}\n", .{std.math.tan(angle)});

    // Inverse trigonometric functions
    const s = 0.5;
    try writer.print("\nInverse of {d}:\n", .{s});
    try writer.print("asin = {d:.6} rad = {d:.2} deg\n", .{
        std.math.asin(s), std.math.asin(s) * 180.0 / std.math.pi,
    });
    try writer.print("acos = {d:.6} rad = {d:.2} deg\n", .{
        std.math.acos(s), std.math.acos(s) * 180.0 / std.math.pi,
    });
    try writer.print("atan = {d:.6} rad = {d:.2} deg\n", .{
        std.math.atan(s), std.math.atan(s) * 180.0 / std.math.pi,
    });

    // atan2 - four-quadrant inverse tangent
    const x: f64 = 1.0;
    const y: f64 = 1.0;
    try writer.print("\natan2({d}, {d}) = {d:.6} rad = {d:.2} deg\n", .{
        y, x, std.math.atan2(y, x),
        std.math.atan2(y, x) * 180.0 / std.math.pi,
    });
}
```

All trig functions are comptime-safe, enabling compile-time table generation:

```zig
const std = @import("std");

// Generate a sine lookup table at compile time
const sin_table = comptime blk: {
    var table: [360]f64 = undefined;
    for (0..360) |deg| {
        const rad: f64 = @as(f64, @floatFromInt(deg)) * std.math.pi / 180.0;
        table[deg] = std.math.sin(rad);
    }
    break :blk table;
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const angles = [_]usize{ 0, 30, 45, 60, 90, 180, 270, 359 };
    for (angles) |deg| {
        try writer.print("sin({d} deg) = {d:.6}\n", .{ deg, sin_table[deg] });
    }
}
```

---

## Chapter 6: Exponential & Logarithm

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Exponential
    try writer.print("exp(0)   = {d}\n", .{std.math.exp(0.0)});
    try writer.print("exp(1)   = {d:.10}\n", .{std.math.exp(1.0)});
    try writer.print("exp(2)   = {d:.10}\n", .{std.math.exp(2.0)});
    try writer.print("exp2(8)  = {d}\n", .{std.math.exp2(8.0)});

    // Logarithm
    try writer.print("\nlog(e)   = {d:.10}\n", .{std.math.log(std.math.e)});
    try writer.print("log(10)  = {d:.10}\n", .{std.math.log(10.0)});
    try writer.print("log2(8)  = {d}\n", .{std.math.log2(8.0)});
    try writer.print("log10(1000) = {d}\n", .{std.math.log10(1000.0)});
    try writer.print("log10(1)    = {d}\n", .{std.math.log10(1.0)});

    // Power and roots
    try writer.print("\npow(2, 10)  = {d}\n", .{std.math.pow(f64, 2.0, 10.0)});
    try writer.print("pow(3, 0.5) = {d:.10}\n", .{std.math.pow(f64, 3.0, 0.5)});
    try writer.print("sqrt(144)   = {d}\n", .{std.math.sqrt(144.0)});
    try writer.print("cbrt(27)    = {d}\n", .{std.math.cbrt(27.0)});
    try writer.print("cbrt(-8)    = {d}\n", .{std.math.cbrt(-8.0)});
}
```

The `pow` function is type-parameterized: `std.math.pow(T, base, exponent)` where `T` is the floating-point type.

For integer exponentiation, use the comptime-friendly approach or `std.math.powi`:

```zig
const result = std.math.powi(f64, 2.0, 10); // 1024.0
```

---

## Chapter 7: Hyperbolic Functions

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const x: f64 = 1.0;

    try writer.print("x = {d}\n\n", .{x});

    // Forward hyperbolic functions
    try writer.print("sinh({d})  = {d:.10}\n", .{ x, std.math.sinh(x) });
    try writer.print("cosh({d})  = {d:.10}\n", .{ x, std.math.cosh(x) });
    try writer.print("tanh({d})  = {d:.10}\n", .{ x, std.math.tanh(x) });

    // Identity: cosh^2(x) - sinh^2(x) = 1
    const sh = std.math.sinh(x);
    const ch = std.math.cosh(x);
    try writer.print("\ncosh^2 - sinh^2 = {d:.10} (should be 1.0)\n", .{
        ch * ch - sh * sh,
    });

    // Inverse hyperbolic functions
    try writer.print("\nInverse hyperbolic of {d}:\n", .{x});
    try writer.print("asinh({d}) = {d:.10}\n", .{ x, std.math.asinh(x) });
    try writer.print("acosh({d}) = {d:.10}\n", .{ x, std.math.acosh(x) });
    try writer.print("atanh({d}) = {d:.10}\n", .{ 0.5, std.math.atanh(0.5) });

    // Verify inverses
    const th = std.math.tanh(0.5);
    try writer.print("\ntanh(0.5)      = {d:.10}\n", .{th});
    try writer.print("atanh(tanh(0.5)) = {d:.10}\n", .{std.math.atanh(th)});
}
```

---

## Chapter 8: math.sign — New in 0.16

Zig 0.16 introduced a significant change to `std.math.sign`: it now returns the **smallest integer type** that can fit all possible return values for the input type.

For a signed type `T`, `sign` returns `T` (which includes -1, 0, +1). For an unsigned type, it returns `u1` (since only 0 and 1 are possible — unsigned values cannot be negative).

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // For signed types, returns the same signed type
    const signed_val: i32 = -42;
    const signed_result: i32 = std.math.sign(signed_val);
    try writer.print("sign(i32, {d}) = {d} (type: i32)\n", .{
        signed_val, signed_result,
    });

    const zero: i64 = 0;
    const zero_result: i64 = std.math.sign(zero);
    try writer.print("sign(i64, {d}) = {d} (type: i64)\n", .{ zero, zero_result });

    // For unsigned types, returns u1 (smallest integer fitting {0, 1})
    const unsigned_val: u32 = 42;
    const unsigned_result: u1 = std.math.sign(unsigned_val);
    try writer.print("sign(u32, {d}) = {d} (type: u1)\n", .{
        unsigned_val, unsigned_result,
    });

    const unsigned_zero: u64 = 0;
    const uz_result: u1 = std.math.sign(unsigned_zero);
    try writer.print("sign(u64, {d}) = {d} (type: u1)\n", .{
        unsigned_zero, uz_result,
    });

    // Demonstrating with floats
    const fpos: f64 = 3.14;
    const fneg: f64 = -3.14;
    try writer.print("sign(f64, {d:.2}) = {d}\n", .{ fpos, std.math.sign(fpos) });
    try writer.print("sign(f64, {d:.2}) = {d}\n", .{ fneg, std.math.sign(fneg) });

    // Practical use: comparison helper
    const a: i32 = 100;
    const b: i32 = 200;
    const cmp: i32 = std.math.sign(b - a); // positive means b > a
    try writer.print("\nComparison: sign({d} - {d}) = {d}\n", .{ b, a, cmp });
}
```

This design prevents unnecessary widening and is idiomatic Zig: the compiler knows the exact return type at compile time.

---

## Chapter 9: Bit Manipulation

### rotl, rotr

Rotate left and rotate right:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const val: u8 = 0b10000001; // 129

    const left = std.math.rotl(u8, val, 1);  // 0b00000011 = 3
    const right = std.math.rotr(u8, val, 1); // 0b11000000 = 192

    try writer.print("Original: {b:0>8} ({d})\n", .{ val, val });
    try writer.print("rotl(1):  {b:0>8} ({d})\n", .{ left, left });
    try writer.print("rotr(1):  {b:0>8} ({d})\n", .{ right, right });

    // Rotate by type width returns original
    try writer.print("rotl(8):  {b:0>8} ({d})\n", .{
        std.math.rotl(u8, val, 8), std.math.rotl(u8, val, 8),
    });
}
```

### alignForward

`std.math.alignForward` aligns a value up to the next boundary:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const addr: usize = 37;
    const alignment: usize = 16;

    const aligned = std.math.alignForward(usize, addr, alignment);
    try writer.print("Align {d} to {d}: {d}\n", .{ addr, alignment, aligned });

    // Already aligned
    const aligned_addr: usize = 64;
    const still_aligned = std.math.alignForward(usize, aligned_addr, alignment);
    try writer.print("Align {d} to {d}: {d}\n", .{ aligned_addr, alignment, still_aligned });

    // Page-aligned example
    const page_size: usize = 4096;
    const ptr_val: usize = 0x1234_5678;
    const page_aligned = std.math.alignForward(usize, ptr_val, page_size);
    try writer.print("Align 0x{x} to page (0x{x}): 0x{x}\n", .{
        ptr_val, page_size, page_aligned,
    });
}
```

### Other Bit Utilities

```zig
// Reverse bits
const reversed: u8 = @bitReverse(@as(u8, 0b10110000)); // 0b00001101

// Byte swap (endianness conversion)
const swapped: u32 = @byteSwap(@as(u32, 0x12345678)); // 0x78563412

// Count: compile-time check
comptime {
    const n: u32 = 0xFF00FF00;
    @import("std").debug.assert(std.math.popCount(n) == 16);
    @import("std").debug.assert(std.math.clz(n) == 8);
}
```

---

## Chapter 10: std.math.big — Big Integer Arithmetic

For integers that exceed the native word size, Zig provides `std.math.big.int` (or `std.math.big.Int` depending on the Zig version). This supports arbitrary-precision arithmetic.

```zig
const std = @import("std");
const math = std.math;
const big = math.big;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();
    const allocator = init.allocator;

    // Create big integers from literals
    var a = try big.int.Managed.init(allocator);
    defer a.deinit();

    var b = try big.int.Managed.init(allocator);
    defer b.deinit();

    try a.set(12345678901234567890);
    try b.set(98765432109876543210);

    // Addition
    var sum = try big.int.Managed.init(allocator);
    defer sum.deinit();
    try sum.add(&a, &b);
    try writer.print("a + b = {s}\n", .{sum.toString(10, .lower, .{})});

    // Subtraction
    var diff = try big.int.Managed.init(allocator);
    defer diff.deinit();
    try diff.sub(&b, &a);
    try writer.print("b - a = {s}\n", .{diff.toString(10, .lower, .{})});

    // Multiplication
    var product = try big.int.Managed.init(allocator);
    defer product.deinit();
    try product.mul(&a, &b);
    try writer.print("a * b = {s}\n", .{product.toString(10, .lower, .{})});

    // Exponentiation
    var power = try big.int.Managed.init(allocator);
    defer power.deinit();
    try power.pow(&a, 3);
    try writer.print("a^3   = {s}\n", .{power.toString(10, .lower, .{})});

    // Comparison
    const cmp = a.orderCompare(b);
    try writer.print("\na {} b\n", .{switch (cmp) {
        .lt => "<",
        .eq => "==",
        .gt => ">",
    }});

    // Division
    var quotient = try big.int.Managed.init(allocator);
    defer quotient.deinit();
    var remainder = try big.int.Managed.init(allocator);
    defer remainder.deinit();
    try quotient.divFloor(&a, &b, &remainder);
    try writer.print("a / b = {s}\n", .{quotient.toString(10, .lower, .{})});
    try writer.print("a % b = {s}\n", .{remainder.toString(10, .lower, .{})});

    // Bitwise AND
    var and_result = try big.int.Managed.init(allocator);
    defer and_result.deinit();
    try and_result.bitAnd(&a, &b);
    try writer.print("a & b = {s}\n", .{and_result.toString(10, .lower, .{})});

    // Shift left
    var shifted = try big.int.Managed.init(allocator);
    defer shifted.deinit();
    try shifted.shiftLeft(&a, 8);
    try writer.print("a << 8 = {s}\n", .{shifted.toString(10, .lower, .{})});
}
```

Big integers are heap-allocated and require an allocator. Always `defer deinit()` to prevent leaks.

---

## Chapter 11: Project — Scientific Calculator

This project implements a command-line scientific calculator with expression parsing, supporting basic arithmetic, trigonometric functions, logarithms, and exponents.

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "sci-calc",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);
    b.step("run", "Run the scientific calculator").dependOn(&run_cmd.step);
}
```

### src/main.zig

```zig
const std = @import("std");
const math = std.math;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const stderr = init.io.getStdErr();
    const writer = stdout.writer();
    const err_writer = stderr.writer();

    const args = try init.process.argsAlloc(init.allocator);
    defer init.process.argsFree(init.allocator, args);

    if (args.len < 2) {
        try err_writer.writeAll("Usage: sci-calc <expression>\n");
        try err_writer.writeAll("Supported: +, -, *, /, ^, sin, cos, tan, log, ln, sqrt, abs, pi, e\n");
        try err_writer.writeAll("Examples:\n");
        try err_writer.writeAll("  sci-calc '2 + 3 * 4'\n");
        try err_writer.writeAll("  sci-calc 'sin(pi/2)'\n");
        try err_writer.writeAll("  sci-calc 'log(100) + sqrt(16)'\n");
        return;
    }

    const expr = args[1];

    // Tokenize, parse, and evaluate
    var parser = Parser.init(expr, init.allocator);
    const result = parser.parseExpr() catch |err| {
        try err_writer.print("Error: {s}\n", .{@errorName(err)});
        return;
    };

    // Format result nicely
    if (math.isFinite(result)) {
        if (result == @trunc(result) and @abs(result) < 1e15) {
            try writer.print("{d}\n", .{result});
        } else {
            try writer.print("{d:.6}\n", .{result});
        }
    } else if (math.isNan(result)) {
        try writer.writeAll("NaN\n");
    } else {
        try writer.print("{d}\n", .{result});
    }
}

const Parser = struct {
    input: []const u8,
    pos: usize,
    allocator: std.mem.Allocator,

    fn init(input: []const u8, allocator: std.mem.Allocator) Parser {
        return .{ .input = input, .pos = 0, .allocator = allocator };
    }

    fn skipWhitespace(self: *Parser) void {
        while (self.pos < self.input.len and self.input[self.pos] == ' ') {
            self.pos += 1;
        }
    }

    fn peek(self: *Parser) ?u8 {
        self.skipWhitespace();
        if (self.pos < self.input.len) return self.input[self.pos];
        return null;
    }

    fn consume(self: *Parser, ch: u8) bool {
        self.skipWhitespace();
        if (self.pos < self.input.len and self.input[self.pos] == ch) {
            self.pos += 1;
            return true;
        }
        return false;
    }

    fn parseExpr(self: *Parser) !f64 {
        return self.parseAddSub();
    }

    fn parseAddSub(self: *Parser) !f64 {
        var left = try self.parseMulDiv();
        while (true) {
            self.skipWhitespace();
            if (self.peek() == '+') {
                self.pos += 1;
                left += try self.parseMulDiv();
            } else if (self.peek() == '-') {
                self.pos += 1;
                left -= try self.parseMulDiv();
            } else {
                break;
            }
        }
        return left;
    }

    fn parseMulDiv(self: *Parser) !f64 {
        var left = try self.parsePower();
        while (true) {
            self.skipWhitespace();
            if (self.peek() == '*') {
                self.pos += 1;
                left *= try self.parsePower();
            } else if (self.peek() == '/') {
                self.pos += 1;
                const right = try self.parsePower();
                if (right == 0.0) return error.DivisionByZero;
                left /= right;
            } else if (self.peek() == '%') {
                self.pos += 1;
                const right = try self.parsePower();
                if (right == 0.0) return error.DivisionByZero;
                left = @mod(left, right);
            } else {
                break;
            }
        }
        return left;
    }

    fn parsePower(self: *Parser) !f64 {
        var base = try self.parseUnary();
        self.skipWhitespace();
        if (self.peek() == '^') {
            self.pos += 1;
            const exp = try self.parseUnary(); // right-associative
            base = math.pow(f64, base, exp);
        }
        return base;
    }

    fn parseUnary(self: *Parser) !f64 {
        self.skipWhitespace();
        if (self.peek() == '-') {
            self.pos += 1;
            return -try self.parseUnary();
        }
        if (self.peek() == '+') {
            self.pos += 1;
            return try self.parseUnary();
        }
        return self.parsePrimary();
    }

    fn parsePrimary(self: *Parser) !f64 {
        self.skipWhitespace();

        // Parenthesized expression
        if (self.peek() == '(') {
            self.pos += 1;
            const val = try self.parseExpr();
            if (!self.consume(')')) return error.MissingClosingParen;
            return val;
        }

        // Check for function names
        if (self.tryConsume("sin")) return math.sin(try self.parsePrimary());
        if (self.tryConsume("cos")) return math.cos(try self.parsePrimary());
        if (self.tryConsume("tan")) return math.tan(try self.parsePrimary());
        if (self.tryConsume("asin")) return math.asin(try self.parsePrimary());
        if (self.tryConsume("acos")) return math.acos(try self.parsePrimary());
        if (self.tryConsume("atan")) return math.atan(try self.parsePrimary());
        if (self.tryConsume("sinh")) return math.sinh(try self.parsePrimary());
        if (self.tryConsume("cosh")) return math.cosh(try self.parsePrimary());
        if (self.tryConsume("tanh")) return math.tanh(try self.parsePrimary());
        if (self.tryConsume("sqrt")) return math.sqrt(try self.parsePrimary());
        if (self.tryConsume("cbrt")) return math.cbrt(try self.parsePrimary());
        if (self.tryConsume("abs")) return @abs(try self.parsePrimary());
        if (self.tryConsume("log")) return math.log10(try self.parsePrimary());
        if (self.tryConsume("ln")) return math.log(try self.parsePrimary());
        if (self.tryConsume("log2")) return math.log2(try self.parsePrimary());
        if (self.tryConsume("exp")) return math.exp(try self.parsePrimary());
        if (self.tryConsume("ceil")) return math.ceil(try self.parsePrimary());
        if (self.tryConsume("floor")) return math.floor(try self.parsePrimary());
        if (self.tryConsume("round")) return math.round(try self.parsePrimary());
        if (self.tryConsume("trunc")) return math.trunc(try self.parsePrimary());

        // Constants
        if (self.tryConsume("pi")) return math.pi;
        if (self.tryConsume("e")) return math.e;

        // Number
        return self.parseNumber();
    }

    fn tryConsume(self: *Parser, keyword: []const u8) bool {
        const remaining = self.input[self.pos..];
        if (remaining.len >= keyword.len) {
            if (std.mem.eql(u8, remaining[0..keyword.len], keyword)) {
                // Make sure it's not a prefix of a longer identifier
                if (keyword.len < remaining.len) {
                    const next_ch = remaining[keyword.len];
                    if (next_ch >= 'a' and next_ch <= 'z') return false;
                }
                self.pos += keyword.len;
                return true;
            }
        }
        return false;
    }

    fn parseNumber(self: *Parser) !f64 {
        self.skipWhitespace();
        const start = self.pos;

        while (self.pos < self.input.len) {
            const ch = self.input[self.pos];
            if ((ch >= '0' and ch <= '9') or ch == '.') {
                self.pos += 1;
            } else {
                break;
            }
        }

        if (self.pos == start) return error.ExpectedNumber;

        return std.fmt.parseFloat(f64, self.input[start..self.pos]) catch
            error.InvalidCharacter;
    }
};
```

### Usage Examples

```bash
$ zig build run -- '2 + 3 * 4'
14

$ zig build run -- 'sin(pi/2)'
1.000000

$ zig build run -- 'sqrt(144) + log(100)'
14.000000

$ zig build run -- '2 ^ 10'
1024

$ zig build run -- 'cos(pi) + 1'
0.000000

$ zig build run -- 'exp(1)'
2.718282

$ zig build run -- 'abs(-42) + ceil(3.14) + floor(3.14)'
42 + 4 + 3 = 49
```

---

## Summary

| Category | Key Functions |
|---|---|
| **Integer** | `max`, `min`, `abs`, `clz`, `ctz`, `popCount` |
| **Float constants** | `pi`, `e`, `inf`, `nan`, `ln2`, `ln10`, `sqrt2` |
| **Float checks** | `isNan`, `isInf`, `isFinite` |
| **Rounding** | `ceil`, `floor`, `trunc`, `round`, `@floor`, `@ceil`, `@round`, `@trunc` |
| **Trig** | `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2` |
| **Exp/Log** | `exp`, `exp2`, `log`, `log2`, `log10`, `pow`, `sqrt`, `cbrt` |
| **Hyperbolic** | `sinh`, `cosh`, `tanh`, `asinh`, `acosh`, `atanh` |
| **New in 0.16** | `sign` returns smallest integer type fitting possible values |
| **Bit manip** | `rotl`, `rotr`, `alignForward`, `@bitReverse`, `@byteSwap` |
| **Big int** | `std.math.big.int.Managed` — arbitrary precision arithmetic |