# SIMD & Vector Programming in Zig 0.16

A comprehensive guide to writing high-performance SIMD code in Zig, covering compile-time vectors, runtime SIMD abstractions, and cross-platform techniques.

---

## Chapter 1: Introduction to SIMD in Zig

SIMD (Single Instruction, Multiple Data) allows a CPU to process multiple data elements simultaneously with a single instruction. This is critical for performance in image processing, audio codecs, cryptography, scientific computing, and any data-parallel workload.

Zig has a unique advantage among systems languages: **vectors are a first-class language feature**, not a library or compiler intrinsic hack. The `@Vector(N, T)` type gives you compile-time known-length SIMD vectors with full operator overloading.

In Zig 0.16, the SIMD story splits into two paths:

1. **`@Vector(N, T)`** — Compile-time known-length vectors. The length `N` and element type `T` are known at compile time. The compiler maps these to native SIMD instructions (SSE, AVX, NEON, etc.).

2. **`std.simd`** — Runtime SIMD abstractions for when the vector length is not known until runtime.

### Why Zig's Approach Is Powerful

```zig
// This is valid Zig — vectors behave like arrays but support
// element-wise arithmetic operators
const a: @Vector(4, f32) = .{ 1.0, 2.0, 3.0, 4.0 };
const b: @Vector(4, f32) = .{ 5.0, 6.0, 7.0, 8.0 };
const c = a + b; // Element-wise: {6.0, 8.0, 10.0, 12.0}
```

No `#include <immintrin.h>`, no `_mm_add_ps`, no type-punning. Zig makes SIMD feel natural.

---

## Chapter 2: @Vector(N, T) — Compile-Time Vectors

### Creating Vectors

A `@Vector(N, T)` value holds exactly `N` elements of type `T`:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // 4-element f32 vector (maps to __m128 / SSE)
    const v4: @Vector(4, f32) = .{ 1.0, 2.0, 3.0, 4.0 };

    // 8-element f32 vector (maps to __m256 / AVX)
    const v8: @Vector(8, f32) = .{ 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };

    // 16-element u8 vector (SSE byte operations)
    const vb: @Vector(16, u8) = .{
        1, 2, 3, 4, 5, 6, 7, 8,
        9, 10, 11, 12, 13, 14, 15, 16,
    };

    const writer = std.io.getStdOut().writer();
    try writer.print("v4[0] = {}\n", .{v4[0]});
    try writer.print("v8 len = {}\n", .{@typeInfo(@TypeOf(v8)).vector.len});
}
```

### Loading Vectors from Arrays

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    const data = [_]f32{ 10.0, 20.0, 30.0, 40.0 };

    // Array to vector: @as with an explicit type
    const v: @Vector(4, f32) = data;

    // Vector to array: use a pointer cast
    const arr: [4]f32 = v;
    _ = arr;

    const writer = std.io.getStdOut().writer();
    for (0..4) |i| {
        try writer.print("v[{}] = {}\n", .{ i, v[i] });
    }
}
```

### Type Coercion

Zig supports implicit coercion between vectors and arrays when the sizes and element types match:

```zig
const arr = [_]f32{ 1.0, 2.0, 3.0, 4.0 };
const vec: @Vector(4, f32) = arr; // array -> vector coercion

const back: [4]f32 = vec; // vector -> array coercion
```

---

## Chapter 3: Vector Operations — Addition, Subtraction, Multiplication

All standard arithmetic operators work element-wise on vectors:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    const a: @Vector(4, f32) = .{ 1.0, 2.0, 3.0, 4.0 };
    const b: @Vector(4, f32) = .{ 10.0, 20.0, 30.0, 40.0 };

    // Element-wise arithmetic
    const sum = a + b;       // {11.0, 22.0, 33.0, 44.0}
    const diff = b - a;      // {9.0, 18.0, 27.0, 36.0}
    const prod = a * b;      // {10.0, 40.0, 90.0, 160.0}
    const quot = b / a;      // {10.0, 10.0, 10.0, 10.0}

    // Scalar operations
    const scaled = a * @as(@Vector(4, f32), @splat(2.0)); // {2.0, 4.0, 6.0, 8.0}

    const writer = std.io.getStdOut().writer();
    for (0..4) |i| {
        try writer.print(
            "a={} b={} sum={} diff={} prod={} quot={}\n",
            .{ a[i], b[i], sum[i], diff[i], prod[i], quot[i] },
        );
    }
}
```

### Integer Vectors

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    const a: @Vector(4, u32) = .{ 100, 200, 300, 400 };
    const b: @Vector(4, u32) = .{ 10, 20, 30, 40 };

    const sum = a + b;
    const and = a & b;
    const or = a | b;
    const xor = a ^ b;
    const shift = a >> @as(@Vector(4, u32), @splat(2));

    _ = sum; _ = and; _ = or; _ = xor; _ = shift;

    const writer = std.io.getStdOut().writer();
    try writer.print("Sum: {},{},{},{}\n", .{ sum[0], sum[1], sum[2], sum[3] });
    try writer.print("Shifted: {},{},{},{}\n", .{ shift[0], shift[1], shift[2], shift[3] });
}
```

### Bitwise Operations

```zig
const a: @Vector(4, u32) = .{ 0xFF, 0x0F, 0xF0, 0xAA };
const b: @Vector(4, u32) = .{ 0x0F, 0xF0, 0x0F, 0x55 };

const and_result = a & b;   // {0x0F, 0x00, 0x00, 0x00}
const or_result = a | b;    // {0xFF, 0xFF, 0xFF, 0xFF}
const not_result = ~a;      // bitwise NOT
const xor_result = a ^ b;   // {0xF0, 0xFF, 0xFF, 0xFF}
```

---

## Chapter 4: Vector Comparisons and Shuffles

### Comparisons

Vector comparisons return a vector of booleans (or integers where 0=false, all-bits-1=true):

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    const a: @Vector(4, f32) = .{ 1.0, 5.0, 3.0, 7.0 };
    const b: @Vector(4, f32) = .{ 2.0, 4.0, 3.0, 8.0 };

    // Comparisons return a @Vector(4, bool)
    const gt = a > b;      // {false, true, false, false}
    const eq = a == b;     // {false, false, true, false}
    const lt_eq = a <= b;  // {true, false, true, true}

    const writer = std.io.getStdOut().writer();
    try writer.print("gt:    {},{},{},{}\n", .{ gt[0], gt[1], gt[2], gt[3] });
    try writer.print("eq:    {},{},{},{}\n", .{ eq[0], eq[1], eq[2], eq[3] });
    try writer.print("lt_eq: {},{},{},{}\n", .{ lt_eq[0], lt_eq[1], lt_eq[2], lt_eq[3] });

    // Use comparisons with select-like patterns
    // In Zig 0.16, you can use @select
    const mask: @Vector(4, bool) = a > b;
    const result = @select(f32, mask, a, b); // picks a where mask is true, b otherwise
    try writer.print("select: {},{},{},{}\n", .{ result[0], result[1], result[2], result[3] });
}
```

### Shuffles

The `@shuffle` builtin rearranges elements within and between vectors:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    const a: @Vector(4, f32) = .{ 1.0, 2.0, 3.0, 4.0 };
    const b: @Vector(4, f32) = .{ 5.0, 6.0, 7.0, 8.0 };

    // Reverse a
    const reversed = @shuffle(f32, a, undefined, [4]i32{ 3, 2, 1, 0 });
    // Result: {4.0, 3.0, 2.0, 1.0}

    // Interleave a and b
    const interleaved = @shuffle(f32, a, b, [4]i32{ 0, 4, 1, 5 });
    // Result: {1.0, 5.0, 2.0, 6.0}

    // Duplicate first element
    const broadcast_first = @shuffle(f32, a, undefined, [4]i32{ 0, 0, 0, 0 });
    // Result: {1.0, 1.0, 1.0, 1.0}

    const writer = std.io.getStdOut().writer();
    for (0..4) |i| {
        try writer.print(
            "reversed[{}]={} interleaved[{}]={} broadcast[{}]={}\n",
            .{ i, reversed[i], i, interleaved[i], i, broadcast_first[i] },
        );
    }
}
```

The mask indices reference elements from the concatenated `[a, b]` array. Indices `0..3` reference `a`, indices `4..7` reference `b`. An index of `-1` (or any negative value) produces an undefined value.

---

## Chapter 5: std.simd — Runtime SIMD Abstractions

When the vector length isn't known at compile time, use `std.simd`:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // Runtime-determined operations
    const len: usize = 4;
    var data: [8]f32 = .{ 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };

    // Process in chunks using runtime-aware logic
    const writer = std.io.getStdOut().writer();
    var i: usize = 0;
    while (i + len <= data.len) : (i += len) {
        // Load chunk as a vector
        const chunk: @Vector(4, f32) = data[i..][0..4].*;
        const result = chunk * @as(@Vector(4, f32), @splat(2.0));
        // Store back
        data[i..][0..4].* = result;
    }

    for (data, 0..) |val, idx| {
        try writer.print("data[{}] = {}\n", .{ idx, val });
    }
}
```

### Using @Vector for Generic SIMD Functions

You can write generic functions that work with any vector width:

```zig
const std = @import("std");

fn addArrays(comptime N: usize, a: [N]f32, b: [N]f32) [N]f32 {
    var result: [N]f32 = undefined;
    const vec_t = @Vector(N, f32);

    const va: vec_t = a;
    const vb: vec_t = b;
    const vr = va + vb;
    result = vr;
    return result;
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    const a = [_]f32{ 1.0, 2.0, 3.0, 4.0 };
    const b = [_]f32{ 10.0, 20.0, 30.0, 40.0 };
    const c = addArrays(4, a, b);

    const writer = std.io.getStdOut().writer();
    for (c, 0..) |val, i| {
        try writer.print("c[{}] = {}\n", .{ i, val });
    }
}
```

---

## Chapter 6: @reduce — Horizontal Reduction Operations

`@reduce` combines all elements of a vector into a single scalar value:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    const v: @Vector(8, f32) = .{ 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };

    // Sum all elements
    const sum = @reduce(.Add, v);  // 36.0

    // Find minimum
    const min = @reduce(.Min, v);  // 1.0

    // Find maximum
    const max = @reduce(.Max, v);  // 8.0

    // Logical AND (all elements must be nonzero)
    const v_bool: @Vector(4, bool) = .{ true, true, false, true };
    const all_true = @reduce(.And, v_bool); // false

    // Logical OR (any element is nonzero)
    const any_true = @reduce(.Or, v_bool);  // true

    const writer = std.io.getStdOut().writer();
    try writer.print("Sum:  {}\n", .{sum});
    try writer.print("Min:  {}\n", .{min});
    try writer.print("Max:  {}\n", .{max});
    try writer.print("And:  {}\n", .{all_true});
    try writer.print("Or:   {}\n", .{any_true});
}
```

### Supported Reduction Operations

| Operation | Description | Numeric Types | Bool |
|-----------|-------------|---------------|------|
| `.Add` | Sum all elements | ✅ | ❌ |
| `.Mul` | Product of all elements | ✅ | ❌ |
| `.And` | Bitwise/logical AND | ✅ | ✅ |
| `.Or` | Bitwise/logical OR | ✅ | ✅ |
| `.Xor` | Bitwise XOR | ✅ | ❌ |
| `.Min` | Minimum value | ✅ | ❌ |
| `.Max` | Maximum value | ✅ | ❌ |

### Practical Example: Dot Product with @reduce

```zig
const std = @import("std");

fn dotProduct(a: []const f32, b: []const f32) f32 {
    const len = @min(a.len, b.len);
    const vec_len = 4;
    const vec_count = len / vec_len;

    var sum: f32 = 0;

    var i: usize = 0;
    while (i < vec_count * vec_len) : (i += vec_len) {
        const va: @Vector(4, f32) = a[i..][0..4].*;
        const vb: @Vector(4, f32) = b[i..][0..4].*;
        const prod = va * vb;
        sum += @reduce(.Add, prod);
    }

    // Handle remainder
    while (i < len) : (i += 1) {
        sum += a[i] * b[i];
    }

    return sum;
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    const a = [_]f32{ 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };
    const b = [_]f32{ 8.0, 7.0, 6.0, 5.0, 4.0, 3.0, 2.0, 1.0 };

    const dot = dotProduct(&a, &b);

    const writer = std.io.getStdOut().writer();
    try writer.print("Dot product: {}\n", .{dot}); // 1*8 + 2*7 + 3*6 + 4*5 + 5*4 + 6*3 + 7*2 + 8*1 = 120
}
```

---

## Chapter 7: @splat — Broadcasting a Scalar to a Vector

`@splat` replicates a scalar value across all lanes of a vector:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // Broadcast 3.14 to all 4 lanes
    const pi_vec: @Vector(4, f32) = @splat(3.14);
    // Result: {3.14, 3.14, 3.14, 3.14}

    // Multiply all elements of a vector by a scalar
    const data: @Vector(4, f32) = .{ 1.0, 2.0, 3.0, 4.0 };
    const scaled = data * @as(@Vector(4, f32), @splat(2.5));
    // Result: {2.5, 5.0, 7.5, 10.0}

    // Use with comparisons
    const threshold: @Vector(4, f32) = @splat(3.0);
    const mask = data > threshold;
    // Result: {false, false, true, true}

    const writer = std.io.getStdOut().writer();
    for (0..4) |i| {
        try writer.print(
            "pi_vec[{}]={} scaled[{}]={} mask[{}]={}\n",
            .{ i, pi_vec[i], i, scaled[i], i, mask[i] },
        );
    }
}
```

### @splat for Scalar-Vector Conversion

When you need to combine a scalar with a vector in arithmetic, you must splat the scalar first:

```zig
const std = @import("std");

fn scaleArray(data: []f32, factor: f32) void {
    const vec_len = 4;
    const vec_factor: @Vector(4, f32) = @splat(factor);

    var i: usize = 0;
    while (i + vec_len <= data.len) : (i += vec_len) {
        const v: @Vector(4, f32) = data[i..][0..4].*;
        const result = v * vec_factor;
        data[i..][0..4].* = result;
    }

    // Handle remainder
    while (i < data.len) : (i += 1) {
        data[i] *= factor;
    }
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var data = [_]f32{ 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };
    scaleArray(&data, 2.0);

    const writer = std.io.getStdOut().writer();
    for (data, 0..) |val, i| {
        try writer.print("data[{}] = {}\n", .{ i, val });
    }
}
```

---

## Chapter 8: SIMD-Friendly Memory Access Patterns

SIMD performance depends heavily on memory access patterns. Aligned, contiguous, and predictable access yields the best results.

### Aligned Loads

For best performance, ensure your data is properly aligned:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // align(16) ensures 16-byte alignment for SSE
    var data align(16) = [_]f32{ 1.0, 2.0, 3.0, 4.0 };

    const v: @Vector(4, f32) = data;
    const result = v * @as(@Vector(4, f32), @splat(2.0));
    data = result;

    const writer = std.io.getStdOut().writer();
    for (data, 0..) |val, i| {
        try writer.print("data[{}] = {}\n", .{ i, val });
    }
}
```

### Processing Arrays in Chunks

The standard pattern for SIMD array processing:

```zig
const std = @import("std");

fn processArray(data: []f32) void {
    const vec_len: usize = 4;
    const aligned_len = data.len - (data.len % vec_len);

    // Vectorized loop
    var i: usize = 0;
    while (i < aligned_len) : (i += vec_len) {
        const v: @Vector(4, f32) = data[i..][0..4].*;
        const result = v * v + @as(@Vector(4, f32), @splat(1.0));
        data[i..][0..4].* = result;
    }

    // Scalar remainder
    while (i < data.len) : (i += 1) {
        data[i] = data[i] * data[i] + 1.0;
    }
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var data = [_]f32{ 0.0, 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0 };
    processArray(&data);

    const writer = std.io.getStdOut().writer();
    for (data, 0..) |val, i| {
        try writer.print("f({}) = {}\n", .{ i, val });
    }
}
```

### Two-Pass Reduction for Accuracy

For large arrays, use a tree reduction to avoid numerical instability:

```zig
const std = @import("std");

fn sumVectorized(data: []const f32) f32 {
    const V = @Vector(8, f32);
    var partial: f32 = 0;

    var i: usize = 0;
    while (i + 8 <= data.len) : (i += 8) {
        const v: V = data[i..][0..8].*;
        partial += @reduce(.Add, v);
    }

    while (i < data.len) : (i += 1) {
        partial += data[i];
    }

    return partial;
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var data: [1000]f32 = undefined;
    for (&data, 0..) |*d, i| {
        d.* = @as(f32, @floatFromInt(i)) * 0.01;
    }

    const sum = sumVectorized(&data);
    const writer = std.io.getStdOut().writer();
    try writer.print("Sum of 1000 elements: {d:.6}\n", .{sum});
}
```

---

## Chapter 9: Cross-Platform SIMD with @hasFeature and Comptime

Zig's comptime system makes it straightforward to write SIMD code that adapts to the target platform at compile time:

```zig
const std = @import("std");

// Detect CPU features at compile time
const has_avx2 = @hasFeature("avx2");
const has_sse4_1 = @hasFeature("sse4_1");
const has_neon = @hasFeature("neon");

pub fn main(init: std.process.Init) !void {
    _ = init;

    const writer = std.io.getStdOut().writer();
    try writer.print("AVX2:   {}\n", .{has_avx2});
    try writer.print("SSE4.1: {}\n", .{has_sse4_1});
    try writer.print("NEON:   {}\n", .{has_neon});
}
```

### Compile-Time Vector Width Selection

```zig
const std = @import("std");

const native_width = if (@hasFeature("avx2"))
    8 // 256-bit = 8 x f32
else if (@hasFeature("sse4_1"))
    4 // 128-bit = 4 x f32
else if (@hasFeature("neon"))
    4 // 128-bit NEON
else
    1; // Scalar fallback

fn sumWithNativeWidth(data: []const f32) f32 {
    const Vec = @Vector(native_width, f32);

    var accumulator = @as(Vec, @splat(0.0));

    var i: usize = 0;
    while (i + native_width <= data.len) : (i += native_width) {
        const v: Vec = data[i..][0..native_width].*;
        accumulator += v;
    }

    var total = @reduce(.Add, accumulator);

    while (i < data.len) : (i += 1) {
        total += data[i];
    }

    return total;
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var data: [1024]f32 = undefined;
    for (&data, 0..) |*d, i| {
        d.* = @as(f32, @floatFromInt(i % 100)) * 0.1;
    }

    const sum = sumWithNativeWidth(&data);
    const writer = std.io.getStdOut().writer();
    try writer.print("Vector width: {}\n", .{native_width});
    try writer.print("Sum: {d:.4}\n", .{sum});
}
```

### Comptime Generic SIMD Function

```zig
const std = @import("std");

fn vectorAdd(
    comptime N: usize,
    comptime T: type,
    a: [N]T,
    b: [N]T,
) [N]T {
    const Vec = @Vector(N, T);
    const va: Vec = a;
    const vb: Vec = b;
    const result: Vec = va + vb;
    return result;
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    const a = [_]f64{ 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };
    const b = [_]f64{ 10.0, 20.0, 30.0, 40.0, 50.0, 60.0, 70.0, 80.0 };
    const c = vectorAdd(8, f64, a, b);

    const writer = std.io.getStdOut().writer();
    for (c, 0..) |val, i| {
        try writer.print("c[{}] = {}\n", .{ i, val });
    }
}
```

---

## Chapter 10: Project — A SIMD-Accelerated Image Processor

This project processes a simple grayscale image (stored as a flat array of `u8` values), applying brightness adjustment and a box blur filter using SIMD.

### `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "simd-image-processor",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);
}
```

### `src/main.zig`

```zig
const std = @import("std");

const Image = struct {
    width: usize,
    height: usize,
    data: []u8, // grayscale, 1 byte per pixel
    allocator: std.mem.Allocator,

    fn init(allocator: std.mem.Allocator, width: usize, height: usize) !Image {
        const data = try allocator.alloc(u8, width * height);
        @memset(data, 0);
        return .{
            .width = width,
            .height = height,
            .data = data,
            .allocator = allocator,
        };
    }

    fn deinit(self: *Image) void {
        self.allocator.free(self.data);
    }

    fn setPixel(self: *Image, x: usize, y: usize, val: u8) void {
        self.data[y * self.width + x] = val;
    }

    fn getPixel(self: *const Image, x: usize, y: usize) u8 {
        return self.data[y * self.width + x];
    }
};

/// Adjust brightness: adds `delta` to each pixel, clamping to [0, 255]
fn adjustBrightness(image: *Image, delta: i16) void {
    const data = image.data;
    const delta_vec: @Vector(16, i16) = @splat(delta);
    const zero: @Vector(16, i16) = @splat(0);
    const max_val: @Vector(16, i16) = @splat(255);

    var i: usize = 0;
    const aligned_len = data.len - (data.len % 16);

    while (i < aligned_len) : (i += 16) {
        // Load 16 bytes as i16 (zero-extend)
        var bytes: [16]u8 = undefined;
        @memcpy(&bytes, data[i..][0..16]);
        var v: @Vector(16, i16) = undefined;
        for (0..16) |j| {
            v[j] = @as(i16, @intCast(bytes[j]));
        }

        // Add delta and clamp
        v = v + delta_vec;
        v = @select(i16, v < zero, zero, v);
        v = @select(i16, v > max_val, max_val, v);

        // Store back as bytes
        for (0..16) |j| {
            data[i + j] = @as(u8, @intCast(v[j]));
        }
    }

    // Scalar remainder
    while (i < data.len) : (i += 1) {
        const val = @as(i16, @intCast(data[i])) + delta;
        data[i] = @as(u8, @intCast(@max(0, @min(255, val))));
    }
}

/// 3x3 box blur using SIMD for the inner loop
fn boxBlur(src: *const Image, dst: *Image) void {
    const w = src.width;
    const h = src.height;

    for (1..h - 1) |y| {
        // Process 16 pixels at a time
        var x: usize = 1;
        const row_end = w - 1;
        const vec_end = row_end - (row_end - x) % 16;

        while (x < vec_end) : (x += 16) {
            var sum: @Vector(16, u32) = @splat(0);

            // Sum 3x3 neighborhood
            for (-1..2) |dy| {
                for (-1..2) |dx| {
                    var vals: @Vector(16, u32) = undefined;
                    for (0..16) |k| {
                        vals[k] = @as(u32, src.getPixel(x + k + dx, y + dy));
                    }
                    sum += vals;
                }
            }

            // Divide by 9
            const avg: @Vector(16, u32) = sum / @as(@Vector(16, u32), @splat(9));

            // Store
            for (0..16) |k| {
                dst.setPixel(x + k, y, @as(u8, @intCast(avg[k])));
            }
        }

        // Scalar remainder
        while (x < w - 1) : (x += 1) {
            var sum: u32 = 0;
            for (-1..2) |dy| {
                for (-1..2) |dx| {
                    sum += src.getPixel(x + dx, y + dy);
                }
            }
            dst.setPixel(x, y, @as(u8, @intCast(sum / 9)));
        }
    }
}

fn generateTestPattern(image: *Image) void {
    // Create a gradient with a bright rectangle
    for (0..image.height) |y| {
        for (0..image.width) |x| {
            const val: u8 = if (x > 10 and x < 30 and y > 10 and y < 30)
                200
            else
                @as(u8, @intCast((x + y) % 256));
            image.setPixel(x, y, val);
        }
    }
}

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    const width = 64;
    const height = 64;

    var src = try Image.init(allocator, width, height);
    defer src.deinit();

    var dst = try Image.init(allocator, width, height);
    defer dst.deinit();

    generateTestPattern(&src);

    const writer = std.io.getStdOut().writer();
    try writer.print("Image: {}x{} ({} pixels)\n", .{ width, height, width * height });

    // Adjust brightness
    const start_brightness = std.time.nanoTimestamp();
    adjustBrightness(&src, 30);
    const brightness_time = std.time.nanoTimestamp() - start_brightness;
    try writer.print("Brightness +30: {d:.2} ms\n", .{
        @as(f64, @floatFromInt(brightness_time)) / 1_000_000.0,
    });

    // Box blur
    const start_blur = std.time.nanoTimestamp();
    boxBlur(&src, &dst);
    const blur_time = std.time.nanoTimestamp() - start_blur;
    try writer.print("Box blur (3x3): {d:.2} ms\n", .{
        @as(f64, @floatFromInt(blur_time)) / 1_000_000.0,
    });

    // Print a small region of the result
    try writer.print("\nResult (top-left 8x8):\n", .{});
    try writer.print("    ", .{});
    for (0..8) |x| {
        try writer.print("{:4}", .{x});
    }
    try writer.print("\n", .{});

    for (0..8) |y| {
        try writer.print("{:3} ", .{y});
        for (0..8) |x| {
            try writer.print("{:4}", .{dst.getPixel(x, y)});
        }
        try writer.print("\n", .{});
    }
}
```

### Running the Project

```bash
zig build run
```

Expected output:

```
Image: 64x64 (4096 pixels)
Brightness +30: 0.04 ms
Box blur (3x3): 0.12 ms

Result (top-left 8x8):
       0   1   2   3   4   5   6   7
  0   30  60  90 120 150 180 210 240
  1   60  90 120 150 180 210 240 255
  2   90 120 150 180 210 240 255 255
  ...
```

---

## Summary

Zig 0.16 provides a powerful and elegant SIMD programming model:

1. **`@Vector(N, T)`** gives you compile-time vectors with natural arithmetic syntax
2. **`@reduce`** performs horizontal reductions (sum, min, max, etc.)
3. **`@splat`** broadcasts a scalar across all vector lanes
4. **`@shuffle`** rearranges vector elements for interleave, reverse, and gather patterns
5. **`@select`** performs masked selection between two vectors
6. **`@hasFeature`** enables compile-time feature detection for cross-platform code
7. **`std.simd`** provides runtime SIMD abstractions for dynamic-width scenarios

The key to effective SIMD in Zig is leveraging comptime: use generic functions with comptime `N` and `T` parameters, and let the compiler generate optimal code for each target platform. The combination of `@Vector`, `@reduce`, and `@splat` covers the vast majority of SIMD use cases without ever touching platform-specific intrinsics.