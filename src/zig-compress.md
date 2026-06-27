# Compression in Zig 0.16

A comprehensive guide to data compression and decompression using Zig's standard library, covering Deflate, Zlib, Gzip, LZMA, LZMA2, and XZ formats with the new `std.Io.Reader`/`std.Io.Writer` interfaces.

---

## Chapter 1: Introduction to std.compress

Zig 0.16 ships a robust `std.compress` module that provides compression and decompression for several widely used formats. The major change in 0.16 is the migration of all compress submodules to use the `std.Io.Reader` and `std.Io.Writer` interfaces, making streaming I/O uniform across the entire standard library.

### Formats Available in std.compress

| Format | Module | Direction | Notes |
|--------|--------|-----------|-------|
| Deflate | `std.compress.deflate` | Compress + Decompress | **New in 0.16** |
| Zlib | `std.compress.zlib` | Compress + Decompress | Wrapper around Deflate |
| Gzip | `std.compress.gzip` | Compress + Decompress | File format with headers |
| LZMA | `std.compress.lzma` | Compress + Decompress | High ratio, single-stream |
| LZMA2 | `std.compress.lzma2` | Compress + Decompress | Improved LZMA |
| XZ | `std.compress.xz` | Decompress (streaming) | Container for LZMA2 |

### The 0.16 I/O Revolution

In Zig 0.16, all compression operations accept or return `std.Io.Reader` and `std.Io.Writer` types. This means:

- **Compressors** implement `std.Io.Writer` — you write uncompressed data, they produce compressed output.
- **Decompressors** implement `std.Io.Reader` — you read from them, they decompress on the fly.
- All I/O is parameterized, meaning you pass an `io` parameter (from `std.process.Init`) to get access to stdout, stderr, and file handles.

```zig
pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    // stdout is a std.Io.Writer — write compressed data here
}
```

This pattern eliminates the previous patchwork of `Reader`, `Writer`, and ad-hoc buffer interfaces. Every compression format now speaks the same I/O language.

---

## Chapter 2: Deflate Compression (NEW in 0.16)

Zig 0.16 introduces first-class Deflate support. Deflate is the raw compression algorithm that underpins both Zlib and Gzip. Having it exposed directly lets you work with the bare algorithm when you don't need Zlib headers or Gzip framing.

### Compressing with Deflate

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    const data = "Hello, Deflate in Zig 0.16! This is a test string for compression.";

    // Compress using Deflate
    var compressed = std.ArrayList(u8).init(allocator);
    defer compressed.deinit();

    var comp_writer = std.compress.deflate.compressor(compressed.writer());
    try comp_writer.writeAll(data);
    try comp_writer.finish();

    const stdout_writer = stdout.writer();
    try stdout_writer.print("Original: {d} bytes\n", .{data.len});
    try stdout_writer.print("Compressed: {d} bytes\n", .{compressed.items.len});
    try stdout_writer.print("Ratio: {d:.2}%\n", .{
        @as(f64, @floatFromInt(compressed.items.len)) /
            @as(f64, @floatFromInt(data.len)) * 100.0,
    });
}
```

### Decompressing with Deflate

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    const compressed_data = [_]u8{
        // Pre-compressed deflate bytes (example)
        0xf3, 0x48, 0xcd, 0xc9, 0xc9, 0x57, 0x08, 0xcf,
        0x2f, 0xca, 0x49, 0x51, 0xe4, 0x02, 0x00, 0x14,
        0x56, 0x07, 0xab,
    };

    var decompressed = std.ArrayList(u8).init(allocator);
    defer decompressed.deinit();

    var comp_reader = std.compress.deflate.decompressor(
        std.io.fixedBufferStream(&compressed_data).reader(),
    );
    const bytes_read = comp_reader.readAll(decompressed.writer(), .{}) catch |err| switch (err) {
        error.EndOfStream => return, // normal termination
        else => return err,
    };
    _ = bytes_read;

    const stdout_writer = stdout.writer();
    try stdout_writer.print("Decompressed: {s}\n", .{decompressed.items});
}
```

### Compression Levels

Deflate supports different compression levels. Level 0 is no compression (fastest), while higher levels trade CPU time for better compression ratios:

```zig
// Fast compression (low CPU, larger output)
var fast_writer = std.compress.deflate.compressor(
    output.writer(),
    .{ .level = 1 },
);

// Best compression (high CPU, smaller output)
var best_writer = std.compress.deflate.compressor(
    output.writer(),
    .{ .level = 9 },
);
```

---

## Chapter 3: Zlib Compression/Decompression

Zlib wraps Deflate with a 2-byte header (CMF + FLG) and a 4-byte Adler-32 checksum trailer. It is the most common format for in-memory compression.

### Compressing with Zlib

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    const input = "Zlib compression is built on top of Deflate with headers and checksums.";

    // Compress
    var compressed = std.ArrayList(u8).init(allocator);
    defer compressed.deinit();

    var zlib_comp = std.compress.zlib.compressor(compressed.writer());
    try zlib_comp.writeAll(input);
    try zlib_comp.finish();

    // Decompress
    var decompressed = std.ArrayList(u8).init(allocator);
    defer decompressed.deinit();

    var zlib_decomp = std.compress.zlib.decompressor(
        std.io.fixedBufferStream(compressed.items).reader(),
        allocator,
    );
    _ = zlib_decomp.readAll(decompressed.writer(), .{}) catch |err| switch (err) {
        error.EndOfStream => {},
        else => return err,
    };

    const w = stdout.writer();
    try w.print("Input:     {d} bytes\n", .{input.len});
    try w.print("Compressed: {d} bytes\n", .{compressed.items.len});
    try w.print("Output:    {d} bytes\n", .{decompressed.items.len});
    try w.print("Match:     {}\n", .{std.mem.eql(u8, input, decompressed.items)});
}
```

### Round-Trip Verification

A common pattern is to compress and then verify the round-trip:

```zig
fn verifyRoundTrip(
    allocator: std.mem.Allocator,
    data: []const u8,
) !bool {
    // Compress
    var comp_buf = std.ArrayList(u8).init(allocator);
    defer comp_buf.deinit();

    var comp = std.compress.zlib.compressor(comp_buf.writer());
    try comp.writeAll(data);
    try comp.finish();

    // Decompress
    var decomp_buf = std.ArrayList(u8).init(allocator);
    defer decomp_buf.deinit();

    var decomp = std.compress.zlib.decompressor(
        std.io.fixedBufferStream(comp_buf.items).reader(),
        allocator,
    );
    _ = decomp.readAll(decomp_buf.writer(), .{}) catch |err| switch (err) {
        error.EndOfStream => {},
        else => return err,
    };

    return std.mem.eql(u8, data, decomp_buf.items);
}
```

---

## Chapter 4: Gzip Compression/Decompression

Gzip adds its own header (magic number, flags, modification time, OS byte) and trailer (CRC-32 + size) around Deflate-compressed data. It is the standard format for compressed files (`.gz`).

### Basic Gzip Compression

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    const data =
        "The Gzip format uses a 10-byte header containing a magic number " ++
        "(0x1f8b), compression method, flags, and optional fields, followed " ++
        "by Deflate-compressed data and a CRC-32 checksum with the original size.";

    // Compress with Gzip
    var compressed = std.ArrayList(u8).init(allocator);
    defer compressed.deinit();

    var gzip_comp = std.compress.gzip.compressor(compressed.writer());
    try gzip_comp.writeAll(data);
    try gzip_comp.finish();

    // Decompress
    var decompressed = std.ArrayList(u8).init(allocator);
    defer decompressed.deinit();

    var gzip_decomp = std.compress.gzip.decompressor(
        std.io.fixedBufferStream(compressed.items).reader(),
        allocator,
    );
    _ = gzip_decomp.readAll(decompressed.writer(), .{}) catch |err| switch (err) {
        error.EndOfStream => {},
        else => return err,
    };

    const w = stdout.writer();
    try w.print("Gzip compression demo\n", .{});
    try w.print("  Original:   {d} bytes\n", .{data.len});
    try w.print("  Compressed: {d} bytes\n", .{compressed.items.len});
    try w.print("  Round-trip: {}\n", .{
        std.mem.eql(u8, data, decompressed.items),
    });
}
```

### Gzip with Filename in Header

```zig
var gzip_comp = std.compress.gzip.compressor(output.writer(), .{
    .filename = "example.txt",
});
```

### Reading a .gz File

```zig
fn readGzipFile(
    allocator: std.mem.Allocator,
    io: *std.Io,
    path: []const u8,
) ![]const u8 {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();

    var decomp = std.compress.gzip.decompressor(file.reader(), allocator);
    var buf = std.ArrayList(u8).init(allocator);
    _ = decomp.readAll(buf.writer(), .{}) catch |err| switch (err) {
        error.EndOfStream => {},
        else => return err,
    };
    return buf.toOwnedSlice();
}
```

---

## Chapter 5: LZMA and LZMA2

LZMA (Lempel-Ziv-Markov chain Algorithm) provides very high compression ratios, making it ideal for software distribution. LZMA2 improves on LZMA by allowing better handling of incompressible data and more efficient reset operations.

### LZMA Compression

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    const data =
        "LZMA provides excellent compression ratios by using a " ++
        "dictionary-based approach combined with range coding. " ++
        "It is the algorithm behind 7z and XZ formats. " ++
        "This text is repeated to show compression: " ++
        "LZMA provides excellent compression ratios by using a " ++
        "dictionary-based approach combined with range coding.";

    // Compress with LZMA
    var compressed = std.ArrayList(u8).init(allocator);
    defer compressed.deinit();

    var lzma_comp = std.compress.lzma.compressor(compressed.writer());
    try lzma_comp.writeAll(data);
    try lzma_comp.finish();

    // Decompress with LZMA
    var decompressed = std.ArrayList(u8).init(allocator);
    defer decompressed.deinit();

    var lzma_decomp = std.compress.lzma.decompressor(
        std.io.fixedBufferStream(compressed.items).reader(),
        allocator,
    );
    _ = lzma_decomp.readAll(decompressed.writer(), .{}) catch |err| switch (err) {
        error.EndOfStream => {},
        else => return err,
    };

    const w = stdout.writer();
    try w.print("LZMA Compression\n", .{});
    try w.print("  Original:   {d} bytes\n", .{data.len});
    try w.print("  Compressed: {d} bytes\n", .{compressed.items.len});
    try w.print("  Round-trip: {}\n", .{
        std.mem.eql(u8, data, decompressed.items),
    });
}
```

### LZMA2 Compression

LZMA2 is a container that wraps LZMA chunks, providing better handling of mixed compressible and incompressible data:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    const data = "LZMA2 is an improvement over raw LZMA for mixed data streams.";

    var compressed = std.ArrayList(u8).init(allocator);
    defer compressed.deinit();

    var lzma2_comp = std.compress.lzma2.compressor(compressed.writer());
    try lzma2_comp.writeAll(data);
    try lzma2_comp.finish();

    var decompressed = std.ArrayList(u8).init(allocator);
    defer decompressed.deinit();

    var lzma2_decomp = std.compress.lzma2.decompressor(
        std.io.fixedBufferStream(compressed.items).reader(),
        allocator,
    );
    _ = lzma2_decomp.readAll(decompressed.writer(), .{}) catch |err| switch (err) {
        error.EndOfStream => {},
        else => return err,
    };

    const w = stdout.writer();
    try w.print("LZMA2 Compression\n", .{});
    try w.print("  Original:   {d} bytes\n", .{data.len});
    try w.print("  Compressed: {d} bytes\n", .{compressed.items.len});
    try w.print("  Round-trip: {}\n", .{
        std.mem.eql(u8, data, decompressed.items),
    });
}
```

---

## Chapter 6: XZ Format

XZ is a file format that wraps LZMA2 compression with a stream header, block headers, and integrity checks (CRC-32 or CRC-64). It is widely used in Linux package distribution.

### Decompressing XZ Streams

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    // Open an .xz file for decompression
    const file = try std.fs.cwd().openFile("data.xz", .{});
    defer file.close();

    var decompressed = std.ArrayList(u8).init(allocator);
    defer decompressed.deinit();

    var xz_decomp = std.compress.xz.decompressor(file.reader(), allocator);
    _ = xz_decomp.readAll(decompressed.writer(), .{}) catch |err| switch (err) {
        error.EndOfStream => {},
        else => return err,
    };

    const w = stdout.writer();
    try w.print("XZ decompressed: {d} bytes\n", .{decompressed.items.len});
}
```

### XZ Compression

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    const data = "This data will be compressed into the XZ format using LZMA2.";

    var compressed = std.ArrayList(u8).init(allocator);
    defer compressed.deinit();

    var xz_comp = std.compress.xz.compressor(compressed.writer());
    try xz_comp.writeAll(data);
    try xz_comp.finish();

    // Write compressed data to a file
    const out_file = try std.fs.cwd().createFile("output.xz", .{});
    defer out_file.close();

    try out_file.writeAll(compressed.items);

    const w = stdout.writer();
    try w.print("XZ compressed {d} bytes -> {d} bytes\n", .{
        data.len,
        compressed.items.len,
    });
}
```

---

## Chapter 7: Using Compression with Io.Reader/Io.Writer (0.16 Pattern)

The key design principle in Zig 0.16 is that every I/O object implements `std.Io.Reader` or `std.Io.Writer`. Compression fits naturally into this model.

### The Writer Chain Pattern

When compressing, you chain writers: your code writes to the compressor, which writes to the underlying transport:

```
Your Code → Compressor (Writer) → File/Buffer (Writer)
```

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    // Create the transport writer (writes to an ArrayList)
    var output_buf = std.ArrayList(u8).init(allocator);
    defer output_buf.deinit();

    // Wrap with a compressor — the compressor is also a Writer
    var compressor = std.compress.gzip.compressor(output_buf.writer());

    // Write data through the chain
    try compressor.writeAll("First chunk of data. ");
    try compressor.writeAll("Second chunk of data. ");
    try compressor.writeAll("Third chunk of data.");
    try compressor.finish();

    const w = stdout.writer();
    try w.print("Compressed {d} bytes into {d} bytes\n", .{
        3 * 20, // approximate original
        output_buf.items.len,
    });
}
```

### The Reader Chain Pattern

When decompressing, you chain readers: you read from the decompressor, which reads from the underlying transport:

```
Your Code ← Decompressor (Reader) ← File/Buffer (Reader)
```

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();

    // Simulate a compressed byte stream
    var comp_buf = std.ArrayList(u8).init(allocator);
    defer comp_buf.deinit();

    {
        var comp = std.compress.zlib.compressor(comp_buf.writer());
        try comp.writeAll("Hello from the reader chain!");
        try comp.finish();
    }

    // Chain: read from decompressor, which reads from the buffer
    var decomp = std.compress.zlib.decompressor(
        std.io.fixedBufferStream(comp_buf.items).reader(),
        allocator,
    );

    var result_buf: [128]u8 = undefined;
    const n = decomp.readAll(&result_buf, .{}) catch |err| switch (err) {
        error.EndOfStream => 0,
        else => return err,
    };

    const w = stdout.writer();
    try w.print("Decompressed: {s}\n", .{result_buf[0..n]});
}
```

### Type-Erased I/O

Since both compressors and decompressors implement the standard interfaces, you can pass them to any function expecting a `std.Io.Reader` or `std.Io.Writer`:

```zig
fn writeDataTo(writer: anytype, data: []const u8) !void {
    try writer.writeAll(data);
}

fn readDataFrom(reader: anytype, buf: []u8) !usize {
    return reader.read(buf) catch |err| switch (err) {
        error.EndOfStream => 0,
        else => return err,
    };
}
```

---

## Chapter 8: Streaming Compression for Large Files

For large files that don't fit in memory, you must stream: read a chunk, compress it, write it out, repeat.

### Streaming File Compression

```zig
const std = @import("std");

const BUFFER_SIZE = 64 * 1024; // 64 KB chunks

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    if (init.process.args.len < 3) {
        try w.print("Usage: {s} <input> <output.gz>\n", .{init.process.args[0]});
        return;
    }

    const input_path = init.process.args[1];
    const output_path = init.process.args[2];

    const in_file = try std.fs.cwd().openFile(input_path, .{});
    defer in_file.close();

    const out_file = try std.fs.cwd().createFile(output_path, .{});
    defer out_file.close();

    var compressor = std.compress.gzip.compressor(out_file.writer());
    var buf: [BUFFER_SIZE]u8 = undefined;

    while (true) {
        const n = in_file.read(&buf) catch |err| switch (err) {
            error.EndOfStream => break,
            else => return err,
        };
        if (n == 0) break;
        try compressor.writeAll(buf[0..n]);
    }
    try compressor.finish();

    try w.print("Compressed {s} -> {s}\n", .{ input_path, output_path });
}
```

### Streaming File Decompression

```zig
const std = @import("std");

const BUFFER_SIZE = 64 * 1024;

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    if (init.process.args.len < 3) {
        try w.print("Usage: {s} <input.gz> <output>\n", .{init.process.args[0]});
        return;
    }

    const input_path = init.process.args[1];
    const output_path = init.process.args[2];

    const in_file = try std.fs.cwd().openFile(input_path, .{});
    defer in_file.close();

    const out_file = try std.fs.cwd().createFile(output_path, .{});
    defer out_file.close();

    var decompressor = std.compress.gzip.decompressor(in_file.reader(), allocator);
    var buf: [BUFFER_SIZE]u8 = undefined;

    while (true) {
        const n = decompressor.read(&buf) catch |err| switch (err) {
            error.EndOfStream => break,
            else => return err,
        };
        if (n == 0) break;
        try out_file.writeAll(buf[0..n]);
    }

    try w.print("Decompressed {s} -> {s}\n", .{ input_path, output_path });
}
```

### Progress Reporting During Streaming

```zig
var total_read: u64 = 0;
var total_written: u64 = 0;

while (true) {
    const n = in_file.read(&buf) catch |err| switch (err) {
        error.EndOfStream => break,
        else => return err,
    };
    if (n == 0) break;
    total_read += n;
    try compressor.writeAll(buf[0..n]);
}

try compressor.finish();
total_written = out_file.getPos() catch 0;

try w.print("  Read:    {d} bytes\n", .{total_read});
try w.print("  Written: {d} bytes\n", .{total_written});
try w.print("  Ratio:   {d:.1}%\n", .{
    @as(f64, @floatFromInt(total_written)) /
        @as(f64, @floatFromInt(total_read)) * 100.0,
});
```

---

## Chapter 9: Choosing the Right Compression Format

### Comparison Table

| Format | Header Size | Checksum | Compression | Best For |
|--------|------------|----------|-------------|----------|
| Deflate | 0 bytes | None | Deflate | Custom protocols, embedded systems |
| Zlib | 2 bytes | Adler-32 | Deflate | In-memory data, network protocols |
| Gzip | 10+ bytes | CRC-32 | Deflate | File storage, HTTP, .gz files |
| LZMA | 13 bytes | CRC-32/64 | LZMA | Software distribution, 7z |
| LZMA2 | Variable | Per-chunk | LZMA | Mixed data, better resets |
| XZ | 12+ bytes | CRC-32/64 | LZMA2 | Linux packages, long-term archives |

### Decision Helper

```zig
const std = @import("std");

const Format = enum { deflate, zlib, gzip, lzma, lzma2, xz };

fn recommendFormat(
    use_case: enum {
        memory,
        file_storage,
        network,
        max_ratio,
        streaming,
    },
) Format {
    return switch (use_case) {
        .memory => .zlib,      // Minimal overhead, checksum
        .file_storage => .gzip, // Standard, widely supported
        .network => .zlib,      // Used in TLS, HTTP
        .max_ratio => .lzma,    // Highest compression ratio
        .streaming => .gzip,    // Good for large files
    };
}
```

### Benchmarking Compression Formats

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Generate test data with some redundancy
    const base = "The quick brown fox jumps over the lazy dog. ";
    var data = std.ArrayList(u8).init(allocator);
    defer data.deinit();
    for (0..100) |_| {
        try data.appendSlice(base);
    }

    try w.print("Benchmark: {d} bytes of test data\n\n", .{data.items.len});

    // Benchmark each format
    const formats = [_]struct {
        name: []const u8,
        comptime_fn: *const fn (std.Io.Writer, []const u8) anyerror!void,
    }{
        // In practice, you'd benchmark each format here
        .{ .name = "Deflate", .comptime_fn = undefined },
        .{ .name = "Zlib", .comptime_fn = undefined },
        .{ .name = "Gzip", .comptime_fn = undefined },
    };

    for (formats) |fmt| {
        var comp_buf = std.ArrayList(u8).init(allocator);
        defer comp_buf.deinit();

        var timer = try std.time.Timer.start();

        // Measure compression time
        // ... (actual compression call here)

        const elapsed = timer.read();
        try w.print("{s}: {d} bytes, {d} ns\n", .{
            fmt.name,
            comp_buf.items.len,
            elapsed,
        });
    }
}
```

---

## Chapter 10: Project — A File Compressor CLI Tool

This project builds a full CLI tool that supports gzip, zlib, and deflate compression and decompression, with automatic format detection.

### Project Structure

```
file-compressor/
├── build.zig
├── src/
│   └── main.zig
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "file-compressor",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }
    const run_step = b.step("run", "Run the file compressor");
    run_step.dependOn(&run_cmd.step);
}
```

### src/main.zig

```zig
const std = @import("std");

const Format = enum {
    gzip,
    zlib,
    deflate,
    auto,

    fn fromString(s: []const u8) ?Format {
        if (std.mem.eql(u8, s, "gzip")) return .gzip;
        if (std.mem.eql(u8, s, "zlib")) return .zlib;
        if (std.mem.eql(u8, s, "deflate")) return .deflate;
        if (std.mem.eql(u8, s, "auto")) return .auto;
        return null;
    }

    fn detect(data: []const u8) ?Format {
        // Gzip magic: 0x1f 0x8b
        if (data.len >= 2 and data[0] == 0x1f and data[1] == 0x8b) return .gzip;
        // Zlib: first byte low nibble = 8 (deflate), CMF*256+FLG % 31 == 0
        if (data.len >= 2) {
            const cmf: u16 = data[0];
            const flg: u16 = data[1];
            if ((cmf & 0x0f) == 8 and (cmf * 256 + flg) % 31 == 0) return .zlib;
        }
        // Assume raw deflate
        return .deflate;
    }

    fn extension(self: Format) []const u8 {
        return switch (self) {
            .gzip => ".gz",
            .zlib => ".zlib",
            .deflate => ".deflate",
            .auto => ".compressed",
        };
    }
};

fn compressFile(
    allocator: std.mem.Allocator,
    input_path: []const u8,
    output_path: []const u8,
    format: Format,
) !void {
    const in_file = try std.fs.cwd().openFile(input_path, .{});
    defer in_file.close();
    const stat = try in_file.stat();
    const input_size = stat.size;

    const out_file = try std.fs.cwd().createFile(output_path, .{});
    defer out_file.close();

    var buf: [64 * 1024]u8 = undefined;

    switch (format) {
        .gzip, .auto => {
            var comp = std.compress.gzip.compressor(out_file.writer());
            try streamCompress(&buf, in_file, &comp);
            try comp.finish();
        },
        .zlib => {
            var comp = std.compress.zlib.compressor(out_file.writer());
            try streamCompress(&buf, in_file, &comp);
            try comp.finish();
        },
        .deflate => {
            var comp = std.compress.deflate.compressor(out_file.writer());
            try streamCompress(&buf, in_file, &comp);
            try comp.finish();
        },
    }

    const out_stat = try out_file.stat();
    const stdout = std.io.getStdOut().writer();
    try stdout.print(
        "Compressed {s} ({d} bytes) -> {s} ({d} bytes) [{s}]\n",
        .{ input_path, input_size, output_path, out_stat.size, @tagName(format) },
    );
}

fn streamCompress(
    buf: *[64 * 1024]u8,
    in_file: std.fs.File,
    comp: anytype,
) !void {
    while (true) {
        const n = in_file.read(buf) catch |err| switch (err) {
            error.EndOfStream => break,
            else => return err,
        };
        if (n == 0) break;
        try comp.writeAll(buf[0..n]);
    }
}

fn decompressFile(
    allocator: std.mem.Allocator,
    input_path: []const u8,
    output_path: []const u8,
    format: Format,
) !void {
    const in_file = try std.fs.cwd().openFile(input_path, .{});
    defer in_file.close();

    // Read header for auto-detection
    var header_buf: [16]u8 = undefined;
    const header_len = in_file.readAll(&header_buf) catch |err| switch (err) {
        error.EndOfStream => 0,
        else => return err,
    };
    _ = header_len;

    const detected_format: Format = if (format == .auto)
        Format.detect(&header_buf) orelse return error.UnknownFormat
    else
        format;

    // Seek back to start
    try in_file.seekTo(0);

    const out_file = try std.fs.cwd().createFile(output_path, .{});
    defer out_file.close();

    var buf: [64 * 1024]u8 = undefined;

    switch (detected_format) {
        .gzip => {
            var decomp = std.compress.gzip.decompressor(in_file.reader(), allocator);
            try streamDecompress(&buf, out_file, &decomp);
        },
        .zlib => {
            var decomp = std.compress.zlib.decompressor(in_file.reader(), allocator);
            try streamDecompress(&buf, out_file, &decomp);
        },
        .deflate => {
            var decomp = std.compress.deflate.decompressor(in_file.reader());
            try streamDecompress(&buf, out_file, &decomp);
        },
        .auto => unreachable,
    }

    const stdout = std.io.getStdOut().writer();
    try stdout.print(
        "Decompressed {s} -> {s} [{s}]\n",
        .{ input_path, output_path, @tagName(detected_format) },
    );
}

fn streamDecompress(
    buf: *[64 * 1024]u8,
    out_file: std.fs.File,
    decomp: anytype,
) !void {
    while (true) {
        const n = decomp.read(buf) catch |err| switch (err) {
            error.EndOfStream => break,
            else => return err,
        };
        if (n == 0) break;
        try out_file.writeAll(buf[0..n]);
    }
}

fn printUsage(program: []const u8) !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print(
        \\Usage: {s} <command> [options] <input> <output>
        \\
        \\Commands:
        \\  compress   Compress a file
        \\  decompress Decompress a file
        \\
        \\Formats: gzip, zlib, deflate, auto
        \\
        \\Examples:
        \\  {s} compress gzip data.txt data.txt.gz
        \\  {s} decompress auto data.txt.gz data.txt
        \\
    , .{ program, program, program });
}

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();
    const args = init.process.args;

    if (args.len < 4) {
        try printUsage(args[0]);
        return;
    }

    const command = args[1];
    const format_str = if (args.len >= 5) args[2] else "auto";
    const input_path = if (args.len >= 5) args[3] else args[2];
    const output_path = if (args.len >= 5) args[4] else args[3];

    const format = Format.fromString(format_str) orelse {
        try w.print("Unknown format: {s}\n", .{format_str});
        try printUsage(args[0]);
        return;
    };

    if (std.mem.eql(u8, command, "compress")) {
        try compressFile(allocator, input_path, output_path, format);
    } else if (std.mem.eql(u8, command, "decompress")) {
        try decompressFile(allocator, input_path, output_path, format);
    } else {
        try w.print("Unknown command: {s}\n", .{command});
        try printUsage(args[0]);
    }
}
```

### Usage

```bash
# Compress with auto-format detection on decompression
zig build run -- compress gzip README.md README.md.gz
zig build run -- compress zlib data.bin data.bin.zlib
zig build run -- decompress auto README.md.gz README.md

# Build the optimized binary
zig build -Doptimize=ReleaseFast
./zig-out/bin/file-compressor compress gzip large.log large.log.gz
```

---

## Summary

Zig 0.16's compression story is unified and elegant:

- **Deflate** is the new first-class raw compression algorithm.
- **Zlib** and **Gzip** wrap Deflate with headers and checksums.
- **LZMA/LZMA2** provide high-ratio compression for archival needs.
- **XZ** wraps LZMA2 in a robust container format.
- All formats use **`std.Io.Reader`/`std.Io.Writer`** for consistent streaming.
- The **writer chain** (compress) and **reader chain** (decompress) patterns make it easy to compose compression with any I/O backend.

The `std.compress` module in Zig 0.16 is production-ready for building tools like archive managers, network protocol implementations, and data pipelines.