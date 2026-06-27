# Random Number Generation in Zig 0.16

A comprehensive guide to random number generation in Zig, covering the `DefaultPrng` (Xoshiro256++), cryptographically secure randomness, distribution patterns, and practical projects.

---

## Chapter 1: Introduction to std.Random

Zig's `std.Random` module provides a clean, composable interface for random number generation. In 0.16, the default pseudo-random number generator is `Xoshiro256++`, a fast, high-quality PRNG with a 256-bit state space and a period of 2^256 - 1.

### Core Design Principles

1. **No global state by default**: PRNGs are explicit values you create and pass around.
2. **Composable**: Any PRNG that implements the `Random` interface can be used interchangeably.
3. **Testable**: Seed a PRNG deterministically for reproducible results.
4. **Secure when needed**: `std.crypto.random` provides CSPRNG for security-sensitive use cases.

### The Random Interface

Every PRNG in Zig 0.16 provides a `.random()` method that returns a `std.Random` value. This value has methods like `int()`, `float()`, `boolean()`, and more:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    // Create a PRNG with a fixed seed
    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    const w = stdout.writer();
    try w.print("Random u64:  {}\n", .{random.int(u64)});
    try w.print("Random f64:  {d:.6}\n", .{random.float(f64)});
    try w.print("Random bool: {}\n", .{random.boolean()});
}
```

---

## Chapter 2: DefaultPrng — Xoshiro256++ Pseudo-Random Generator

Zig 0.16 uses `Xoshiro256++` as the default PRNG. This generator was chosen for its excellent statistical properties, speed, and large state space.

### Why Xoshiro256++?

| Property | Value |
|----------|-------|
| State size | 256 bits (four u64 values) |
| Period | 2^256 - 1 |
| Speed | Very fast (simple bit operations) |
| Quality | Passes all major statistical tests (TestU01 Big Crush) |
| Memory | 32 bytes |

### Creating a DefaultPrng

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Create with a 64-bit seed
    var prng = std.Random.DefaultPrng.init(12345);
    const random = prng.random();

    // Generate some values
    for (0..5) |i| {
        try w.print("Xoshiro256++ #{}: {}\n", .{ i + 1, random.int(u64) });
    }
}
```

### Seeding with More Entropy

For non-security applications, you can seed with the current timestamp:

```zig
const seed = @intCast(std.time.milliTimestamp());
var prng = std.Random.DefaultPrng.init(seed);
```

Or combine multiple entropy sources:

```zig
const seed: u64 = @bitCast([8]u8{
    @truncate(std.time.nanoTimestamp()),
    @truncate(std.time.milliTimestamp()),
    0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF,
});
var prng = std.Random.DefaultPrng.init(seed);
```

### Accessing the State

You can inspect or serialize the PRNG state for save/resume functionality:

```zig
var prng = std.Random.DefaultPrng.init(42);
const state = prng.state; // [4]u64 — the four state words

// Save the state
const saved = state;

// ... generate some values ...
_ = prng.random().int(u64);
_ = prng.random().int(u64);

// Restore the state (resume from the saved point)
prng.state = saved;
const same_value = prng.random().int(u64); // same as first generation
```

---

## Chapter 3: Seeding and Reproducibility

Deterministic seeding is crucial for testing, simulations, and procedural generation. In Zig, seeding is explicit and simple.

### Reproducible Sequences

The same seed always produces the same sequence:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Run 1
    var prng1 = std.Random.DefaultPrng.init(999);
    const r1 = prng1.random();
    try w.print("Run 1: {}, {}, {}\n", .{
        r1.int(u32),
        r1.int(u32),
        r1.int(u32),
    });

    // Run 2 — identical seed, identical output
    var prng2 = std.Random.DefaultPrng.init(999);
    const r2 = prng2.random();
    try w.print("Run 2: {}, {}, {}\n", .{
        r2.int(u32),
        r2.int(u32),
        r2.int(u32),
    });
}
```

### Seeding from a String

Convert a string seed to a numeric seed using a hash function:

```zig
fn seedFromString(s: []const u8) u64 {
    var hasher = std.hash.Fnv1a_64.init();
    hasher.update(s);
    return hasher.final();
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const seed = seedFromString("my-world-seed-v1");
    var prng = std.Random.DefaultPrng.init(seed);
    const random = prng.random();

    // This will always produce the same world layout
    try w.print("World seed hash: {}\n", .{seed});
    try w.print("First terrain: {}\n", .{random.intRangeAtMost(u8, 0, 3)});
    try w.print("Biome type:   {}\n", .{random.intRangeAtMost(u8, 0, 7)});
}
```

### Split PRNGs for Parallel Work

Use `std.Random.split()` to derive independent child PRNGs from a parent:

```zig
fn splitExample() void {
    var parent = std.Random.DefaultPrng.init(42);

    // Create two independent child PRNGs
    const child1_seed = parent.random().int(u64);
    const child2_seed = parent.random().int(u64);

    var child1 = std.Random.DefaultPrng.init(child1_seed);
    var child2 = std.Random.DefaultPrng.init(child2_seed);

    // child1 and child2 produce independent streams
    _ = child1.random().int(u64);
    _ = child2.random().int(u64);
}
```

---

## Chapter 4: Generating Random Integers

### Bounded Integers with `uintLessThan`

Use `uintLessThan` to generate random integers in a half-open range `[0, bound)`:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    // Random u8 in [0, 10)
    const dice_roll = random.uintLessThan(u8, 6) + 1; // 1-6
    try w.print("Dice roll: {}\n", .{dice_roll});

    // Random u32 in [0, 100)
    const percentile = random.uintLessThan(u32, 100);
    try w.print("Percentile: {}\n", .{percentile});

    // Random usize in [0, items.len)
    const items = [_][]const u8{ "apple", "banana", "cherry", "date" };
    const index = random.uintLessThan(usize, items.len);
    try w.print("Random fruit: {s}\n", .{items[index]});
}
```

### Full Range with `int`

```zig
// Full range random
const a: u8 = random.int(u8);    // 0..255
const b: i32 = random.int(i32);  // -2147483648..2147483647
const c: u64 = random.int(u64);  // 0..18446744073709551615
```

### Range with `intRangeAtMost` and `intRangeLessThan`

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    // intRangeAtMost: [min, max] (inclusive both ends)
    const damage = random.intRangeAtMost(u8, 10, 25);
    try w.print("Damage dealt: {}\n", .{damage});

    // intRangeLessThan: [min, max) (inclusive min, exclusive max)
    const hour = random.intRangeLessThan(u8, 0, 24);
    try w.print("Random hour: {}\n", .{hour});

    // Works with signed types too
    const temperature = random.intRangeAtMost(i16, -10, 40);
    try w.print("Temperature: {}°C\n", .{temperature});
}
```

### Weighted Random Selection

```zig
fn weightedSelect(random: std.Random, items: []const u32, weights: []const u64) usize {
    var total: u64 = 0;
    for (weights) |w| total += w;

    var pick = random.uintLessThan(u64, total);
    for (0..items.len) |i| {
        if (pick < weights[i]) return i;
        pick -= weights[i];
    }
    return items.len - 1;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    const items = [_]u32{ 1, 2, 3 };
    const weights = [_]u64{ 50, 30, 20 }; // 50%, 30%, 20%

    var counts = [_]u32{ 0, 0, 0 };
    const trials: u32 = 10000;

    for (0..trials) |_| {
        const idx = weightedSelect(random, &items, &weights);
        counts[idx] += 1;
    }

    for (0..items.len) |i| {
        try w.print(
            "Item {} (weight {d}): {d} ({d:.1}%)\n",
            .{ items[i], weights[i], counts[i], @as(f64, @floatFromInt(counts[i])) / @as(f64, @floatFromInt(trials)) * 100.0 },
        );
    }
}
```

---

## Chapter 5: Generating Random Floats

### Basic Float Generation

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    // Random float in [0.0, 1.0)
    const f1 = random.float(f32);
    try w.print("Random f32: {d:.6}\n", .{f1});

    // Random float in [0.0, 1.0)
    const f2 = random.float(f64);
    try w.print("Random f64: {d:.15}\n", .{f2});

    // Random float in a custom range [min, max)
    const min_val: f64 = -100.0;
    const max_val: f64 = 100.0;
    const custom = min_val + (max_val - min_val) * random.float(f64);
    try w.print("Custom range [{d}, {d}): {d:.4}\n", .{
        min_val, max_val, custom,
    });
}
```

### Boolean from Float

```zig
// 70% chance of true
const likely = random.float(f64) < 0.7;
```

### Random Angle

```zig
// Random angle in radians [0, 2π)
const angle = 2.0 * std.math.pi * random.float(f64);
const degrees = angle * 180.0 / std.math.pi;
try w.print("Random angle: {d:.2}°\n", .{degrees});
```

---

## Chapter 6: Shuffling Arrays

Zig provides `std.Random.shuffle` for in-place Fisher-Yates shuffling:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    // Shuffle a slice
    var deck = std.ArrayList(u8).init(allocator);
    defer deck.deinit();

    for (1..=52) |card| {
        try deck.append(@intCast(card));
    }

    // In-place Fisher-Yates shuffle
    random.shuffle(u8, deck.items);

    try w.print("Shuffled deck (first 10): ", .{});
    for (deck.items[0..10]) |card| {
        try w.print("{} ", .{card});
    }
    try w.print("\n");

    // Shuffle a string array
    var names = [_][]const u8{
        "Alice", "Bob", "Charlie", "Diana", "Eve",
    };
    random.shuffle([]const u8, &names);

    try w.print("Random order: ", .{});
    for (names) |name| {
        try w.print("{s} ", .{name});
    }
    try w.print("\n");
}
```

### Partial Shuffle (Select K Random Elements)

```zig
fn selectKRandom(random: std.Random, items: []u32, k: usize) []u32 {
    // Partial Fisher-Yates: only shuffle the first k positions
    const n = items.len;
    const select_count = @min(k, n);

    var i: usize = 0;
    while (i < select_count) : (i += 1) {
        const j = random.intRangeLessThan(usize, i, n);
        std.mem.swap(u32, &items[i], &items[j]);
    }

    return items[0..select_count];
}
```

---

## Chapter 7: Random Bytes and Buffer Filling

### Filling a Buffer with Random Bytes

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    // Fill a buffer with random bytes
    var key: [32]u8 = undefined;
    random.bytes(&key);

    try w.print("Random 32-byte key: ", .{});
    for (key) |byte| {
        try w.print("{x:0>2}", .{byte});
    }
    try w.print("\n");

    // Generate a random nonce
    var nonce: [12]u8 = undefined;
    random.bytes(&nonce);

    try w.print("Random 12-byte nonce: ", .{});
    for (nonce) |byte| {
        try w.print("{x:0>2}", .{byte});
    }
    try w.print("\n");
}
```

### Random UUID Generation

```zig
fn generateUuid(random: std.Random) [16]u8 {
    var uuid: [16]u8 = undefined;
    random.bytes(&uuid);

    // Version 4 (random)
    uuid[6] = (uuid[6] & 0x0f) | 0x40;
    // Variant 1 (RFC 4122)
    uuid[8] = (uuid[8] & 0x3f) | 0x80;

    return uuid;
}

fn formatUuid(uuid: [16]u8, buf: *[36]u8) void {
    const hex = "0123456789abcdef";
    var i: usize = 0;
    for (uuid, 0..) |byte, j| {
        if (j == 4 or j == 6 or j == 8 or j == 10) {
            buf[i] = '-';
            i += 1;
        }
        buf[i] = hex[byte >> 4];
        buf[i + 1] = hex[byte & 0x0f];
        i += 2;
    }
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    const uuid = generateUuid(random);
    var buf: [36]u8 = undefined;
    formatUuid(uuid, &buf);

    try w.print("UUID v4: {s}\n", .{&buf});
}
```

---

## Chapter 8: std.crypto.random — Cryptographically Secure Random

For security-sensitive applications (password generation, token creation, key generation), use `std.crypto.random`. This is backed by the operating system's CSPRNG (e.g., `getrandom()` on Linux, `BCryptGenRandom` on Windows).

### Basic Cryptographic Random

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Cryptographically secure random u64
    const secure_int = std.crypto.random.int(u64);
    try w.print("Secure random: {}\n", .{secure_int});

    // Cryptographically secure float
    const secure_float = std.crypto.random.float(f64);
    try w.print("Secure float:  {d:.15}\n", .{secure_float});

    // Secure random bytes
    var token: [32]u8 = undefined;
    std.crypto.random.bytes(&token);

    try w.print("Secure token: ", .{});
    for (token) |byte| {
        try w.print("{x:0>2}", .{byte});
    }
    try w.print("\n");
}
```

### Password Generation

```zig
const std = @import("std");

const CHARSET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*";

fn generatePassword(length: usize) [64]u8 {
    var password: [64]u8 = [_]u8{0} ** 64;
    const actual_len = @min(length, 63);
    for (0..actual_len) |i| {
        password[i] = CHARSET[std.crypto.random.intRangeLessThan(
            usize,
            0,
            CHARSET.len,
        )];
    }
    password[actual_len] = 0;
    return password;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Generate 5 passwords of length 20
    for (0..5) |i| {
        const password = generatePassword(20);
        try w.print("Password #{}: {s}\n", .{ i + 1, std.mem.sliceTo(&password, 0) });
    }
}
```

### Secure Token for Web APIs

```zig
const std = @import("std");

fn generateApiToken(buf: *[48]u8) void {
    const alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    for (0..48) |i| {
        buf[i] = alphabet[std.crypto.random.intRangeLessThan(usize, 0, alphabet.len)];
    }
    buf[48] = 0; // null terminator
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var token: [49]u8 = undefined;
    generateApiToken(&token);

    try w.print("API Token: {s}\n", .{&token});
}
```

---

## Chapter 9: Custom PRNGs

Zig includes several PRNGs in addition to the default. You can also implement your own by satisfying the `std.Random` interface.

### Available PRNGs in std.Random

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Pcg (Permuted Congruential Generator)
    var pcg = std.Random.Pcg.init(42);
    try w.print("PCg:        {}\n", .{pcg.random().int(u64)});

    // Xoroshiro128
    var xoro = std.Random.Xoroshiro128.init(.{ 1, 2 });
    try w.print("Xoroshiro128: {}\n", .{xoro.random().int(u64)});

    // DefaultPrng (Xoshiro256++)
    var default_prng = std.Random.DefaultPrng.init(42);
    try w.print("Xoshiro256++: {}\n", .{default_prng.random().int(u64)});
}
```

### Implementing a Custom PRNG

A PRNG needs to implement a `next()` method that returns a `u64`:

```zig
const std = @import("std");

/// A simple Linear Congruential Generator (LCG) for educational purposes.
/// NOT suitable for production — use Xoshiro256++ instead.
const Lcg = struct {
    state: u64,

    const mul: u64 = 6364136223846793005;
    const inc: u64 = 1442695040888963407;

    pub fn init(seed: u64) Lcg {
        return .{ .state = seed };
    }

    pub fn random(self: *Lcg) std.Random {
        return std.Random.init(self, next);
    }

    fn next(r: *std.Random) u64 {
        const self: *Lcg = @ptrCast(@alignCast(r.ptr));
        self.state = self.state *% mul +% inc;
        return self.state;
    }
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var lcg = Lcg.init(12345);
    const random = lcg.random();

    for (0..5) |i| {
        try w.print("LCG #{}: {}\n", .{ i + 1, random.int(u32) });
    }
}
```

---

## Chapter 10: Distribution Patterns

### Uniform Distribution

The default `int()` and `float()` methods provide uniform distribution — every value in the range is equally likely.

### Gaussian (Normal) Approximation — Box-Muller Transform

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    // Generate Gaussian samples using Box-Muller transform
    const NUM_SAMPLES: usize = 1000;
    const BINS: usize = 20;
    var histogram = [_]u32{0} ** BINS;

    for (0..NUM_SAMPLES) |_| {
        const u1 = random.float(f64);
        const u2 = random.float(f64);
        const z0 = std.math.sqrt(-2.0 * @log(u1)) * @cos(2.0 * std.math.pi * u2);
        // Map z0 (roughly -4 to 4) into histogram bins
        const bin = @as(usize, @intFromFloat(@round((z0 + 4.0) / 8.0 * @as(f64, @floatFromInt(BINS - 1)))));
        const clamped = @min(@max(bin, 0), BINS - 1);
        histogram[clamped] += 1;
    }

    try w.print("Gaussian Distribution Histogram (μ=0, σ=1)\n", .{});
    try w.print("{s}\n", .{"─" ** 50});
    for (0..BINS) |i| {
        const value = -4.0 + @as(f64, @floatFromInt(i)) * (8.0 / @as(f64, @floatFromInt(BINS - 1)));
        const bar_len = histogram[i] / 3;
        try w.print("{d:+5.1f} | {s}\n", .{
            value,
            "█" ** bar_len,
        });
    }
    _ = allocator;
}
```

### Exponential Distribution

```zig
fn exponentialRandom(random: std.Random, lambda: f64) f64 {
    return -@log(1.0 - random.float(f64)) / lambda;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    try w.print("Exponential distribution (λ=0.5, mean=2.0):\n", .{});
    for (0..10) |i| {
        const val = exponentialRandom(random, 0.5);
        try w.print("  Sample {}: {d:.4}\n", .{ i + 1, val });
    }
}
```

### Poisson Distribution

```zig
fn poissonRandom(random: std.Random, lambda: f64) u32 {
    const L = @exp(-lambda);
    var k: u32 = 0;
    var p: f64 = 1.0;

    while (p > L) {
        k += 1;
        p *= random.float(f64);
    }

    return k - 1;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    try w.print("Poisson distribution (λ=5.0):\n", .{});
    var counts = [_]u32{0} ** 20;
    for (0..10000) |_| {
        const val = poissonRandom(random, 5.0);
        if (val < 20) counts[val] += 1;
    }

    for (0..15) |i| {
        const bar_len = counts[i] / 20;
        try w.print("  k={d:2}: {s}\n", .{ i, "█" ** bar_len });
    }
}
```

---

## Chapter 11: Project — Monte Carlo Pi Estimator & Dice Simulator

This project implements two tools: a Monte Carlo Pi estimator and a multi-dice simulator with statistics.

### Project Structure

```
random-tools/
├── build.zig
└── src/
    └── main.zig
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "random-tools",
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

    const run_step = b.step("run", "Run random tools");
    run_step.dependOn(&run_cmd.step);
}
```

### src/main.zig

```zig
const std = @import("std");

// ============================================================
// Monte Carlo Pi Estimation
// ============================================================

fn estimatePi(random: std.Random, num_samples: u64) struct {
    pi: f64,
    inside: u64,
    total: u64,
} {
    var inside: u64 = 0;

    for (0..num_samples) |_| {
        const x = random.float(f64); // [0, 1)
        const y = random.float(f64); // [0, 1)

        // Check if point falls inside the quarter circle
        // (x-0.5)^2 + (y-0.5)^2 <= 0.25
        const dx = x - 0.5;
        const dy = y - 0.5;
        if (dx * dx + dy * dy <= 0.25) {
            inside += 1;
        }
    }

    const pi_estimate = 4.0 * @as(f64, @floatFromInt(inside)) /
        @as(f64, @floatFromInt(num_samples));

    return .{
        .pi = pi_estimate,
        .inside = inside,
        .total = num_samples,
    };
}

fn runMonteCarlo(allocator: std.mem.Allocator, stdout: anytype, args: []const []const u8) !void {
    const num_samples = if (args.len >= 2)
        std.fmt.parseInt(u64, args[1], 10) catch 1_000_000
    else
        1_000_000;

    var prng = std.Random.DefaultPrng.init(std.time.milliTimestamp());
    const random = prng.random();

    try stdout.print("Monte Carlo Pi Estimation\n", .{});
    try stdout.print("{s}\n", .{"=" ** 40});
    try stdout.print("Samples: {d}\n\n", .{num_samples});

    // Run in batches for progress reporting
    const batch_size = num_samples / 10;
    var total_inside: u64 = 0;
    var total_samples: u64 = 0;

    for (0..10) |batch| {
        const this_batch = if (batch < 9) batch_size else num_samples - total_samples;
        const result = estimatePi(random, this_batch);
        total_inside += result.inside;
        total_samples += result.total;

        const current_pi = 4.0 * @as(f64, @floatFromInt(total_inside)) /
            @as(f64, @floatFromInt(total_samples));
        const error_val = @abs(current_pi - std.math.pi) / std.math.pi * 100.0;

        try stdout.print(
            "  Batch {d:2}/10: π ≈ {d:.10} (error: {d:.4}%)\n",
            .{ batch + 1, current_pi, error_val },
        );
    }

    const final_pi = 4.0 * @as(f64, @floatFromInt(total_inside)) /
        @as(f64, @floatFromInt(total_samples));
    try stdout.print("\n  Final:   π ≈ {d:.10}\n", .{final_pi});
    try stdout.print("  Actual:  π = {d:.10}\n", .{std.math.pi});
    try stdout.print("  Error:   {d:.6}%\n", .{
        @abs(final_pi - std.math.pi) / std.math.pi * 100.0,
    });

    _ = allocator;
}

// ============================================================
// Dice Simulator
// ============================================================

const Die = struct {
    sides: u8,

    fn roll(self: Die, random: std.Random) u8 {
        return random.intRangeAtMost(u8, 1, self.sides);
    }
};

fn runDiceSimulator(allocator: std.mem.Allocator, stdout: anytype, args: []const []const u8) !void {
    const stdout_writer = stdout;

    // Parse dice specification: "3d6" means roll 3 six-sided dice
    const spec = if (args.len >= 2) args[1] else "2d6";
    const num_dice = if (args.len >= 3)
        std.fmt.parseInt(u8, args[2], 10) catch 10000
    else
        10000;

    // Parse "XdY" format
    const d_pos = std.mem.indexOfScalar(u8, spec, 'd') orelse {
        try stdout_writer.print("Invalid dice spec. Use format: XdY (e.g., 2d6)\n", .{});
        return;
    };

    const count = std.fmt.parseInt(u8, spec[0..d_pos], 10) catch {
        try stdout_writer.print("Invalid dice count\n", .{});
        return;
    };
    const sides = std.fmt.parseInt(u8, spec[d_pos + 1 ..], 10) catch {
        try stdout_writer.print("Invalid dice sides\n", .{});
        return;
    };

    var prng = std.Random.DefaultPrng.init(std.time.milliTimestamp());
    const random = prng.random();

    const die = Die{ .sides = sides };
    const max_total = count * sides;
    var histogram = try allocator.alloc(u32, max_total + 1);
    defer allocator.free(histogram);
    @memset(histogram, 0);

    var total_sum: u64 = 0;
    var min_roll: u32 = max_total;
    var max_roll: u32 = 1;

    try stdout_writer.print("Dice Simulator: {d}d{d} × {d} rolls\n", .{
        count, sides, num_dice,
    });
    try stdout_writer.print("{s}\n", .{"=" ** 40});

    // Roll dice and collect statistics
    for (0..num_dice) |_| {
        var sum: u32 = 0;
        for (0..count) |_| {
            sum += die.roll(random);
        }
        histogram[sum] += 1;
        total_sum += sum;
        if (sum < min_roll) min_roll = sum;
        if (sum > max_roll) max_roll = sum;
    }

    const mean = @as(f64, @floatFromInt(total_sum)) / @as(f64, @floatFromInt(num_dice));

    // Calculate standard deviation
    var variance: f64 = 0.0;
    for (min_roll..max_roll + 1) |val| {
        const freq = @as(f64, @floatFromInt(histogram[val]));
        const diff = @as(f64, @floatFromInt(val)) - mean;
        variance += freq * diff * diff;
    }
    variance /= @as(f64, @floatFromInt(num_dice));
    const std_dev = std.math.sqrt(variance);

    // Find mode
    var mode_val: u32 = min_roll;
    var mode_count: u32 = 0;
    for (min_roll..max_roll + 1) |val| {
        if (histogram[val] > mode_count) {
            mode_count = histogram[val];
            mode_val = @intCast(val);
        }
    }

    try stdout_writer.print("\nStatistics:\n", .{});
    try stdout_writer.print("  Mean:   {d:.4}\n", .{mean});
    try stdout_writer.print("  Median: (approx) {d:.1}\n", .{mean});
    try stdout_writer.print("  StdDev: {d:.4}\n", .{std_dev});
    try stdout_writer.print("  Min:    {d}\n", .{min_roll});
    try stdout_writer.print("  Max:    {d}\n", .{max_roll});
    try stdout_writer.print("  Mode:   {d} ({d} times)\n\n", .{ mode_val, mode_count });

    // Print histogram
    try stdout_writer.print("Distribution:\n", .{});
    for (min_roll..max_roll + 1) |val| {
        const bar_len = histogram[val] / (@divTrunc(num_dice, 50) + 1);
        try stdout_writer.print("  {d:3} | {s}\n", .{
            val,
            "█" ** @min(bar_len, 60),
        });
    }
}

// ============================================================
// Main Entry Point
// ============================================================

fn printUsage(program: []const u8, writer: anytype) !void {
    try writer.print(
        \\Random Number Tools for Zig 0.16
        \\
        \\Usage: {s} <command> [args...]
        \\
        \\Commands:
        \\  pi [samples]     Monte Carlo Pi estimation (default: 1,000,000)
        \\  dice <XdY> [n]   Dice simulator (default: 2d6 × 10,000)
        \\
        \\Examples:
        \\  {s} pi 10000000
        \\  {s} dice 3d6 50000
        \\  {s} dice 1d20 100000
        \\
    , .{ program, program, program, program });
}

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();
    const args = init.process.args;

    if (args.len < 2) {
        try printUsage(args[0], w);
        return;
    }

    const command = args[1];

    if (std.mem.eql(u8, command, "pi")) {
        try runMonteCarlo(allocator, w, args[1..]);
    } else if (std.mem.eql(u8, command, "dice")) {
        try runDiceSimulator(allocator, w, args[1..]);
    } else if (std.mem.eql(u8, command, "--help") or std.mem.eql(u8, command, "-h")) {
        try printUsage(args[0], w);
    } else {
        try w.print("Unknown command: {s}\n\n", .{command});
        try printUsage(args[0], w);
    }
}
```

### Usage

```bash
# Monte Carlo Pi estimation with 10 million samples
zig build run -- pi 10000000

# Simulate rolling 3d6 ten thousand times
zig build run -- dice 3d6 10000

# Simulate rolling a d20 fifty thousand times
zig build run -- dice 1d20 50000
```

### Expected Output (Pi Estimation)

```
Monte Carlo Pi Estimation
========================================
Samples: 10000000

  Batch  1/10: π ≈ 3.1412345600 (error: 0.0113%)
  Batch  2/10: π ≈ 3.1418762300 (error: 0.0091%)
  ...
  Final:   π ≈ 3.1415926500
  Actual:  π = 3.1415926536
  Error:   0.000115%
```

---

## Summary

Zig 0.16's `std.Random` module provides:

- **`DefaultPrng` (Xoshiro256++)**: Fast, high-quality PRNG with 256-bit state.
- **Deterministic seeding**: Same seed → same sequence, perfect for testing.
- **Rich integer APIs**: `int()`, `uintLessThan()`, `intRangeAtMost()`, `intRangeLessThan()`.
- **Float generation**: `float(f32)`, `float(f64)` in [0, 1).
- **`shuffle()`**: In-place Fisher-Yates shuffling.
- **`bytes()`**: Fill buffers with random bytes.
- **`std.crypto.random`**: CSPRNG backed by OS entropy for security.
- **Custom PRNGs**: Easy to implement via the `Random` interface.
- **Distribution patterns**: Build Gaussian, exponential, Poisson, and weighted distributions on top of the uniform generator.