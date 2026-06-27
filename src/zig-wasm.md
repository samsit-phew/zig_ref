# WebAssembly in Zig 0.16

A comprehensive guide to compiling Zig to WebAssembly, using WASM builtins, interacting with JavaScript, working with WASI, and building high-performance WASM modules.

---

## Chapter 1: Introduction to Zig + WebAssembly

Zig is one of the best languages for WebAssembly development. Its zero-overhead abstractions, explicit memory management, and lack of hidden control flow make it ideal for producing small, fast WASM binaries.

### Why Zig for WASM?

- **Small binaries**: No garbage collector, no runtime, minimal standard library.
- **C ABI compatible**: Zig can import and export functions with the C ABI, which is the standard WASM calling convention.
- **Comptime**: Generate lookup tables, unroll loops, and compute constants at compile time — all for free at runtime.
- **No hidden allocations**: You control every byte of memory.
- **Direct WASM support**: Zig treats WASM as a first-class target.

### The Two WASM Targets

| Target | Description | Use Case |
|---|---|---|
| `wasm32-freestanding` | Bare metal, no system interface | Pure computation, JS interop |
| `wasm32-wasi` | With WASI (WebAssembly System Interface) | File I/O, networking, CLI tools |

---

## Chapter 2: Building for WASM

### Freestanding WASM (no WASI)

```bash
zig build-exe src/main.zig -target wasm32-freestanding -O ReleaseSmall
```

This produces a `.wasm` file with no imports (unless you add `@extern` declarations).

### WASI Target

```bash
zig build-exe src/main.zig -target wasm32-wasi -O ReleaseSmall
```

WASI provides a standard system interface: file I/O, environment variables, clock, random, and more.

### Using build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const wasm_exe = b.addExecutable(.{
        .name = "wasm-module",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });

    b.installArtifact(wasm_exe);
}
```

Build with:

```bash
zig build -Dtarget=wasm32-freestanding -Doptimize=ReleaseSmall
```

### Size Optimization Tips

```bash
# Strip debug info
zig build-exe src/main.zig \
    -target wasm32-freestanding \
    -O ReleaseSmall \
    --strip

# Use single-threaded (disables atomics)
zig build-exe src/main.zig \
    -target wasm32-freestanding \
    -O ReleaseSmall \
    -fsingle-threaded
```

---

## Chapter 3: WASM Builtins — @wasm_memory_size and @wasm_memory_grow

Zig 0.16 introduces new builtins for WebAssembly memory management:

- **`@wasm_memory_size(index)`** — Returns the current size of the memory with the given index in units of WASM pages (64KB each).
- **`@wasm_memory_grow(index, delta)`** — Grows the specified memory by `delta` pages. Returns the old size in pages, or -1 on failure.

These replace the older `@wasmMemorySize()` and `@wasmMemoryGrow()` builtins (which operated on memory index 0 by default).

### Basic Memory Inspection

```zig
const std = @import("std");

// This module targets wasm32-freestanding
// No main() needed for library-style WASM

export fn getMemoryInfo() struct { size_pages: u32, size_bytes: u32, page_size: u32 } {
    const page_count = @wasm_memory_size(0);
    return .{
        .size_pages = page_count,
        .size_bytes = page_count * 65536,
        .page_size = 65536,
    };
}

export fn growMemory(pages: u32) i32 {
    const old_size = @wasm_memory_grow(0, pages);
    return old_size;
}

export fn memoryUsageDemo() u32 {
    const before = @wasm_memory_size(0);

    // Try to grow by 1 page (64KB)
    const result = @wasm_memory_grow(0, 1);

    if (result == @as(i32, -1)) {
        // Growth failed (OOM)
        return 0;
    }

    const after = @wasm_memory_size(0);
    return after;
}
```

### Custom Allocator Using WASM Memory

```zig
const std = @import("std");

var heap_ptr: [*]u8 = undefined;
var heap_len: usize = 0;

export fn initHeap() void {
    // Start after the WASM data segment
    heap_ptr = @ptrFromInt(@wasm_memory_size(0) * 65536);
    heap_len = 0;
}

export fn wasmAlloc(size: u32) [*]u8 {
    const aligned_size = std.mem.alignForward(usize, size, 8);
    const current_end = @intFromPtr(heap_ptr) + heap_len;
    const new_end = current_end + aligned_size;

    // Check if we need to grow memory
    const mem_end = @as(usize, @intCast(@wasm_memory_size(0))) * 65536;
    if (new_end > mem_end) {
        const pages_needed = (new_end - mem_end + 65535) / 65536;
        const result = @wasm_memory_grow(0, @intCast(pages_needed));
        if (result == @as(i32, -1)) {
            return @ptrFromInt(0); // null
        }
    }

    const ptr: [*]u8 = @ptrFromInt(current_end);
    heap_len += aligned_size;
    return ptr;
}

export fn getHeapStats() struct { used: u32, total_pages: u32 } {
    return .{
        .used = @intCast(heap_len),
        .total_pages = @wasm_memory_size(0),
    };
}
```

### Multi-Memory Support

WASM has a multi-memory proposal. The `index` parameter in `@wasm_memory_size` and `@wasm_memory_grow` allows operating on different memory spaces:

```zign
export fn sizeOfMemory0() u32 {
    return @wasm_memory_size(0);
}

// If multi-memory is enabled, you could do:
// export fn sizeOfMemory1() u32 {
//     return @wasm_memory_size(1);
// }
```

---

## Chapter 4: Exporting Functions to JS with @export

The `@export` builtin makes Zig functions callable from JavaScript (or any WASM host).

### Basic Exports

```zig
const std = @import("std");

// Simple add function
export fn add(a: i32, b: i32) i32 {
    return a + b;
}

// Export with pointer to a buffer
var result_buffer: [256]u8 = undefined;

export fn processString(ptr: [*]const u8, len: usize) [*]const u8 {
    const input = ptr[0..len];
    const output = std.ascii.upperString(&result_buffer, input);
    return output.ptr;
}

// Export a constant
export const version: [*:0]const u8 = "Zig WASM 0.16";

// Export with complex return type (passed via pointer)
export fn fibonacci(n: u32) u64 {
    if (n <= 1) return n;
    var a: u64 = 0;
    var b: u64 = 1;
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        const tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}
```

### Exporting Memory for JS Access

To let JavaScript read/write WASM memory, you typically export the memory:

```zig
const std = @import("std");

// Shared pixel buffer for image processing
var pixel_buffer: [1024 * 1024 * 4]u8 = undefined; // 1MP RGBA

export fn getPixelBufferPtr() [*]u8 {
    return &pixel_buffer;
}

export fn getPixelBufferSize() u32 {
    return @intCast(pixel_buffer.len);
}

export fn setPixel(x: u32, y: u32, r: u8, g: u8, b: u8, a: u8) void {
    if (x >= 1024 or y >= 1024) return;
    const offset = (y * 1024 + x) * 4;
    pixel_buffer[offset + 0] = r;
    pixel_buffer[offset + 1] = g;
    pixel_buffer[offset + 2] = b;
    pixel_buffer[offset + 3] = a;
}

export fn getPixel(x: u32, y: u32) u32 {
    if (x >= 1024 or y >= 1024) return 0;
    const offset = (y * 1024 + x) * 4;
    return @as(u32, pixel_buffer[offset + 3]) << 24 |
        @as(u32, pixel_buffer[offset + 2]) << 16 |
        @as(u32, pixel_buffer[offset + 1]) << 8 |
        @as(u32, pixel_buffer[offset + 0]);
}

export fn clearPixels() void {
    @memset(&pixel_buffer, 0);
}
```

### JavaScript Side

```javascript
// Load and use the WASM module
const fs = require('fs');
const wasmBuffer = fs.readFileSync('zig-out/bin/wasm-module.wasm');

WebAssembly.instantiate(wasmBuffer, {}).then(({ instance }) => {
    const { add, fibonacci, getPixelBufferPtr, setPixel } = instance.exports;

    console.log('add(2, 3) =', add(2, 3));
    console.log('fib(40) =', fibonacci(40).toString());

    // Access pixel buffer directly from JS
    const ptr = getPixelBufferPtr();
    const memory = new Uint8Array(instance.exports.memory.buffer);
    const pixels = new Uint32Array(memory.buffer, ptr, 1024 * 1024);

    setPixel(100, 100, 255, 0, 0, 255);
    console.log('Pixel at (100,100):', pixels[100 * 1024 + 100]);
});
```

---

## Chapter 5: Importing JS Functions with @extern

Use `@extern` to declare functions that the WASM module will import from the host environment.

```zig
const std = @import("std");

// Import console.log from JavaScript
extern fn consoleLog(ptr: [*]const u8, len: usize) void;

// Import a JS math function
extern fn jsRandom() f64;

// Import DOM manipulation
extern fn jsCreateElement(tag_ptr: [*]const u8, tag_len: usize) u32;
extern fn jsSetAttribute(elem_id: u32, name_ptr: [*]const u8, name_len: usize,
    value_ptr: [*]const u8, value_len: usize) void;
extern fn jsAppendChild(parent_id: u32, child_id: u32) void;
extern fn jsSetInnerText(elem_id: u32, text_ptr: [*]const u8, text_len: usize) void;

// Import timing
extern fn jsPerformanceNow() f64;

export fn logMessage(msg_ptr: [*]const u8, msg_len: usize) void {
    consoleLog(msg_ptr, msg_len);
}

export fn randomInRange(min: f64, max: f64) f64 {
    return min + jsRandom() * (max - min);
}

export fn measureComputation(iterations: u32) f64 {
    const start = jsPerformanceNow();
    var sum: f64 = 0;
    for (0..iterations) |i| {
        sum += @sqrt(@as(f64, @floatFromInt(i)));
    }
    const end = jsPerformanceNow();
    _ = sum;
    return end - start;
}
```

### JavaScript Import Object

```javascript
const importObject = {
    env: {
        consoleLog: (ptr, len) => {
            const bytes = new Uint8Array(wasmMemory.buffer, ptr, len);
            console.log(new TextDecoder().decode(bytes));
        },
        jsRandom: () => Math.random(),
        jsPerformanceNow: () => performance.now(),
        jsCreateElement: (tagPtr, tagLen) => {
            const tag = new TextDecoder().decode(
                new Uint8Array(wasmMemory.buffer, tagPtr, tagLen)
            );
            const elem = document.createElement(tag);
            return elemRegistry.register(elem);
        },
        jsSetAttribute: (elemId, namePtr, nameLen, valuePtr, valueLen) => {
            const elem = elemRegistry.get(elemId);
            const name = new TextDecoder().decode(
                new Uint8Array(wasmMemory.buffer, namePtr, nameLen)
            );
            const value = new TextDecoder().decode(
                new Uint8Array(wasmMemory.buffer, valuePtr, valueLen)
            );
            elem.setAttribute(name, value);
        },
        jsAppendChild: (parentId, childId) => {
            elemRegistry.get(parentId).appendChild(elemRegistry.get(childId));
        },
        jsSetInnerText: (elemId, textPtr, textLen) => {
            const text = new TextDecoder().decode(
                new Uint8Array(wasmMemory.buffer, textPtr, textLen)
            );
            elemRegistry.get(elemId).innerText = text;
        },
    },
};

const { instance } = await WebAssembly.instantiate(wasmBytes, importObject);
```

---

## Chapter 6: Memory Management in WASM

### Linear Memory Model

WASM has a linear memory model — a single contiguous byte array that can grow but never shrink. Understanding this is crucial for WASM development.

```zig
const std = @import("std");

// Arena allocator for WASM
const WasmArena = struct {
    data: []u8,
    offset: usize,

    fn init(buffer: []u8) WasmArena {
        return .{ .data = buffer, .offset = 0 };
    }

    fn alloc(self: *WasmArena, size: usize, alignment: usize) ?[]u8 {
        const current_addr = @intFromPtr(self.data.ptr) + self.offset;
        const aligned_addr = std.mem.alignForward(usize, current_addr, alignment);
        const padding = aligned_addr - current_addr;

        if (self.offset + padding + size > self.data.len) {
            // Try to grow memory
            const pages_needed = (self.offset + padding + size - self.data.len + 65535) / 65536;
            const old_size = @wasm_memory_grow(0, @intCast(pages_needed));
            if (old_size == @as(i32, -1)) return null;

            // After growing, we need to update our data slice
            // In practice, you'd store the base address instead
            self.data.len = @intCast(@as(usize, @intCast(old_size + @intCast(pages_needed))) * 65536);
        }

        const result_ptr: [*]u8 = @ptrFromInt(aligned_addr);
        self.offset += padding + size;
        return result_ptr[0..size];
    }

    fn reset(self: *WasmArena) void {
        self.offset = 0;
    }

    fn used(self: *const WasmArena) usize {
        return self.offset;
    }
};

var arena_buffer: [4 * 1024 * 1024]u8 = undefined; // 4MB static buffer

export fn arenaDemo() u32 {
    var arena = WasmArena.init(&arena_buffer);

    const block1 = arena.alloc(100, 8) orelse return 0;
    @memset(block1, 0xAA);

    const block2 = arena.alloc(200, 8) orelse return 0;
    @memset(block2, 0xBB);

    const block3 = arena.alloc(300, 8) orelse return 0;
    @memset(block3, 0xCC);

    // Verify blocks don't overlap
    return @intCast(arena.used());
}
```

### Using Zig's GeneralPurposeAllocator in WASM

```zig
const std = @import("std");

var gpa_state = std.heap.GeneralPurposeAllocator(.{}){};

export fn initAllocator() bool {
    return gpa_state.init() == .ok;
}

export fn allocate(size: u32) ?[*]u8 {
    const allocator = gpa_state.allocator();
    const mem = allocator.alloc(u8, size) catch return null;
    return mem.ptr;
}

export fn deallocate(ptr: [*]u8, size: u32) void {
    const allocator = gpa_state.allocator();
    allocator.free(ptr[0..size]);
}

export fn allocatorStats() struct { allocated: u32, freed: u32 } {
    return .{
        .allocated = @intCast(gpa_state.internal.getMetrics().allocated_bytes),
        .freed = @intCast(gpa_state.internal.getMetrics().freed_bytes),
    };
}
```

---

## Chapter 7: Working with WASI

WASI (WebAssembly System Interface) provides a standardized way for WASM modules to access system resources.

### Hello World with WASI

```zig
// Build: zig build-exe src/main.zig -target wasm32-wasi
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    try writer.writeAll("Hello from WASI!\n");

    // Access arguments
    const args = try init.process.argsAlloc(init.allocator);
    defer init.process.argsFree(init.allocator, args);

    try writer.print("Arguments: {d}\n", .{args.len});
    for (args, 0..) |arg, i| {
        try writer.print("  [{d}] {s}\n", .{i, arg});
    }

    // Read environment variables
    if (init.process.getEnv("MY_VAR")) |val| {
        try writer.print("MY_VAR = {s}\n", .{val});
    }
}
```

Run with Wasmtime:

```bash
zig build-exe src/main.zig -target wasm32-wasi -O ReleaseSmall --name wasi-demo
wasmtime wasi-demo.wasm arg1 arg2 -- MY_VAR=hello
```

### File I/O with WASI

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Open a directory (WASI requires preopened dirs)
    const dir = std.Io.Dir.cwd();

    // Create and write a file
    const file = try dir.createFile("wasi_output.txt", .{});
    defer file.close();

    try file.writeAll("Written by Zig WASI module!\n");
    try writer.writeAll("Created wasi_output.txt\n");

    // List directory
    try writer.writeAll("Directory contents:\n");
    var iter = dir.iterate();
    var count: usize = 0;
    while (try iter.next()) |entry| {
        try writer.print("  {s}\n", .{entry.name});
        count += 1;
        if (count >= 20) break; // limit output
    }
}
```

---

## Chapter 8: Debugging WASM in the Browser

### Enabling Debug Info

```bash
zig build-exe src/main.zig \
    -target wasm32-freestanding \
    -O Debug \
    -fdebug-dwarf \
    --name debug-module
```

### Source Maps

Zig can generate DWARF debug info that some tools can convert to source maps:

```bash
# Build with debug info
zig build-exe src/main.zig -target wasm32-freestanding -O Debug

# Use wasm-tools to generate source maps (if available)
wasm-tools objdump debug-module.wasm
```

### Debugging Techniques

```zig
const std = @import("std");

// Export a debug log function that JS can intercept
extern fn jsDebugLog(ptr: [*]const u8, len: usize) void;

fn debugLog(comptime fmt: []const u8, args: anytype) void {
    var buf: [512]u8 = undefined;
    const written = std.fmt.bufPrint(&buf, fmt, args) catch "<log error>";
    jsDebugLog(written.ptr, written.len);
}

export fn computeSomething(x: f64) f64 {
    debugLog("computeSomething called with x={d}", .{x});

    const result = std.math.sin(x) * std.math.cos(x);
    debugLog("result={d:.6}", .{result});

    return result;
}
```

### Chrome DevTools Integration

```javascript
// In your HTML/JS:
async function loadWasm() {
    const response = await fetch('zig-out/bin/wasm-module.wasm');
    const bytes = await response.arrayBuffer();
    const { instance } = await WebAssembly.instantiate(bytes, importObject);

    // Call exported functions from the console
    window.wasm = instance.exports;
    console.log('WASM loaded. Use window.wasm to call functions.');
}
```

---

## Chapter 9: Interacting with the DOM (Through JS Imports)

Zig cannot directly access the DOM. Instead, you declare JS functions as `@extern` imports and call them from Zig.

### DOM Helper Library

```zig
const std = @import("std");

// --- JS Import Declarations ---
extern fn jsQuerySelector(ptr: [*]const u8, len: usize) u32;
extern fn jsGetElementById(ptr: [*]const u8, len: usize) u32;
extern fn jsCreateElement(ptr: [*]const u8, len: usize) u32;
extern fn jsCreateTextNode(ptr: [*]const u8, len: usize) u32;
extern fn jsAppendChild(parent: u32, child: u32) void;
extern fn jsSetInnerHTML(elem: u32, ptr: [*]const u8, len: usize) void;
extern fn jsSetStyle(elem: u32, style_ptr: [*]const u8, style_len: usize,
    value_ptr: [*]const u8, value_len: usize) void;
extern fn jsAddEventListener(elem: u32, event_ptr: [*]const u8, event_len: usize) void;
extern fn jsGetInputValue(elem: u32, buf_ptr: [*]u8, buf_len: usize) usize;

// --- Zig DOM Wrapper ---
const Dom = struct {
    fn createElement(tag: []const u8) u32 {
        return jsCreateElement(tag.ptr, tag.len);
    }

    fn getElementById(id: []const u8) u32 {
        return jsGetElementById(id.ptr, id.len);
    }

    fn createTextNode(text: []const u8) u32 {
        return jsCreateTextNode(text.ptr, text.len);
    }

    fn appendChild(parent: u32, child: u32) void {
        jsAppendChild(parent, child);
    }

    fn setInnerHTML(elem: u32, html: []const u8) void {
        jsSetInnerHTML(elem, html.ptr, html.len);
    }
};

// Global element references
var app_div: u32 = 0;
var input_elem: u32 = 0;
var output_elem: u32 = 0;

export fn init() void {
    // Get the app container
    app_div = Dom.getElementById("app");

    // Create an input field
    input_elem = Dom.createElement("input");
    Dom.setInnerHTML(input_elem, "");
    Dom.appendChild(app_div, input_elem);

    // Create an output div
    output_elem = Dom.createElement("div");
    Dom.setInnerHTML(output_elem, "<p>Enter a number above</p>");
    Dom.appendChild(app_div, output_elem);
}

export fn processInput() void {
    var buf: [128]u8 = undefined;
    const len = jsGetInputValue(input_elem, &buf, buf.len);

    const input_str = buf[0..len];
    const value = std.fmt.parseInt(f64, input_str, 10) catch {
        Dom.setInnerHTML(output_elem, "<p style=\"color:red\">Invalid number</p>");
        return;
    };

    // Compute some results
    const sqrt_val = std.math.sqrt(@abs(value));
    const sin_val = std.math.sin(value);
    const cos_val = std.math.cos(value);
    const log_val = if (value > 0) std.math.log(value) else std.math.nan(f64);

    var result_buf: [512]u8 = undefined;
    const html = std.fmt.bufPrint(&result_buf,
        "<table border=\"1\" cellpadding=\"8\">" ++
        "<tr><td>Input</td><td>{d}</td></tr>" ++
        "<tr><td>sqrt(|x|)</td><td>{d:.4}</td></tr>" ++
        "<tr><td>sin(x)</td><td>{d:.6}</td></tr>" ++
        "<tr><td>cos(x)</td><td>{d:.6}</td></tr>" ++
        "<tr><td>log(x)</td><td>{d}</td></tr>" ++
        "</table>",
        .{ value, sqrt_val, sin_val, cos_val, log_val },
    ) catch "<p>Error formatting</p>";

    Dom.setInnerHTML(output_elem, html);
}
```

---

## Chapter 10: Performance Tips for WASM

### 1. Use ReleaseSmall or ReleaseFast

```bash
# Smallest binary (ideal for web delivery)
zig build-exe src/main.zig -target wasm32-freestanding -O ReleaseSmall --strip

# Fastest execution
zig build-exe src/main.zig -target wasm32-freestanding -O ReleaseFast --strip
```

### 2. Precompute at Comptime

```zig
const std = @import("std");

// Sine lookup table computed at compile time
const SIN_TABLE_SIZE = 360;
const sin_table = comptime blk: {
    var table: [SIN_TABLE_SIZE]f32 = undefined;
    for (0..SIN_TABLE_SIZE) |i| {
        const rad = @as(f32, @floatFromInt(i)) * std.math.pi / 180.0;
        table[i] = std.math.sin(rad);
    }
    break :blk table;
};

export fn fastSin(degrees: f32) f32 {
    const idx = @as(usize, @intFromFloat(@mod(degrees, 360.0 + 360.0)));
    return sin_table[@min(idx, SIN_TABLE_SIZE - 1)];
}
```

### 3. Avoid Unnecessary Allocations

```zig
const std = @import("std");

// BAD: allocates on every call
export fn processBad() f64 {
    var list = std.ArrayList(f64).init(std.heap.page_allocator);
    defer list.deinit();
    // ...
    return 0;
}

// GOOD: use stack buffers
export fn processGood() f64 {
    var buf: [1024]f64 = undefined;
    var list: std.ArrayListUnmanaged(f64) = .{};
    defer list.deinit(std.heap.page_allocator);
    // Use slice as backing store
    for (0..100) |i| {
        if (i < buf.len) {
            buf[i] = @as(f64, @floatFromInt(i)) * 1.5;
        }
    }
    return buf[99];
}
```

### 4. Use SIMD When Available

```zig
const std = @import("std");

// Vectorized addition using Zig's vector types
export fn vectorizedAdd(a_ptr: [*]const f32, b_ptr: [*]const f32,
    result_ptr: [*]f32, len: usize) void
{
    const Vec4 = @Vector(4, f32);
    const vec_len = len - (len % 4);

    var i: usize = 0;
    while (i < vec_len) : (i += 4) {
        const a: Vec4 = a_ptr[i..][0..4].*;
        const b: Vec4 = b_ptr[i..][0..4].*;
        result_ptr[i..][0..4].* = a + b;
    }

    // Handle remainder
    while (i < len) : (i += 1) {
        result_ptr[i] = a_ptr[i] + b_ptr[i];
    }
}
```

### 5. Profile in the Browser

```zig
const std = @import("std");

extern fn jsPerformanceNow() f64;

export fn benchmark(iterations: u32) f64 {
    var total: f64 = 0;

    const start = jsPerformanceNow();
    for (0..iterations) |i| {
        const x = @as(f64, @floatFromInt(i)) * 0.001;
        total += std.math.sin(x) * std.math.cos(x) + std.math.sqrt(@abs(x + 1.0));
    }
    const end = jsPerformanceNow();

    _ = total; // prevent dead code elimination
    return end - start;
}
```

---

## Chapter 11: Project — WASM Image Processing Module

This project creates a WASM module that performs grayscale conversion and box blur on RGBA image data, callable from JavaScript.

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const optimize = b.standardOptimizeOption(.{});

    const wasm_exe = b.addExecutable(.{
        .name = "image-proc",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = b.resolveTargetQuery(.{
                .cpu_arch = .wasm32,
                .os_tag = .freestanding,
            }),
            .optimize = optimize,
        }),
    });
    b.installArtifact(wasm_exe);
}
```

### src/main.zig

```zig
const std = @import("std");

// Image data buffer (shared with JS via WASM memory)
var image_data: [*]u8 = undefined;
var image_width: u32 = 0;
var image_height: u32 = 0;

var temp_data: [*]u8 = undefined;

// --- Exports for JS to set up the image buffer ---

export fn setImageBuffer(ptr: [*]u8, width: u32, height: u32) void {
    image_data = ptr;
    image_width = width;
    image_height = height;
}

export fn getImageWidth() u32 {
    return image_width;
}

export fn getImageHeight() u32 {
    return image_height;
}

// --- Helper functions ---

fn pixelOffset(x: u32, y: u32) usize {
    return @as(usize, y) * @as(usize, image_width) * 4 + @as(usize, x) * 4;
}

fn getPixelR(x: u32, y: u32) u8 {
    return image_data[pixelOffset(x, y) + 0];
}

fn getPixelG(x: u32, y: u32) u8 {
    return image_data[pixelOffset(x, y) + 1];
}

fn getPixelB(x: u32, y: u32) u8 {
    return image_data[pixelOffset(x, y) + 2];
}

fn getPixelA(x: u32, y: u32) u8 {
    return image_data[pixelOffset(x, y) + 3];
}

fn setPixelRGBA(x: u32, y: u32, r: u8, g: u8, b: u8, a: u8) void {
    if (x >= image_width or y >= image_height) return;
    const off = pixelOffset(x, y);
    image_data[off + 0] = r;
    image_data[off + 1] = g;
    image_data[off + 2] = b;
    image_data[off + 3] = a;
}

// --- Grayscale Conversion ---

/// Convert image to grayscale using luminance formula:
/// L = 0.299*R + 0.587*G + 0.114*B
export fn grayscale() void {
    const total_pixels = @as(usize, image_width) * @as(usize, image_height);
    for (0..total_pixels) |i| {
        const off = i * 4;
        const r = image_data[off + 0];
        const g = image_data[off + 1];
        const b = image_data[off + 2];
        const a = image_data[off + 3];

        // Fixed-point luminance: 0.299 = 77/256, 0.587 = 150/256, 0.114 = 29/256
        const gray: u8 = @intCast((@as(u32, r) * 77 +
            @as(u32, g) * 150 +
            @as(u32, b) * 29) >> 8);

        image_data[off + 0] = gray;
        image_data[off + 1] = gray;
        image_data[off + 2] = gray;
        // Keep alpha unchanged
        image_data[off + 3] = a;
    }
}

// --- Brightness Adjustment ---

export fn adjustBrightness(amount: i32) void {
    const total_pixels = @as(usize, image_width) * @as(usize, image_height);
    for (0..total_pixels) |i| {
        const off = i * 4;
        for (0..3) |c| {
            const val = @as(i32, image_data[off + c]) + amount;
            image_data[off + c] = @intCast(@max(0, @min(255, val)));
        }
        // Keep alpha
    }
}

// --- Invert Colors ---

export fn invertColors() void {
    const total_pixels = @as(usize, image_width) * @as(usize, image_height);
    for (0..total_pixels) |i| {
        const off = i * 4;
        image_data[off + 0] = 255 - image_data[off + 0];
        image_data[off + 1] = 255 - image_data[off + 1];
        image_data[off + 2] = 255 - image_data[off + 2];
        // Keep alpha
    }
}

// --- Box Blur ---

/// Apply box blur with the given kernel radius.
/// Uses a two-pass separable approach (horizontal then vertical)
/// for O(n) performance per pixel instead of O(r^2).
export fn boxBlur(radius: u32) void {
    if (radius == 0) return;

    const w = image_width;
    const h = image_height;
    const total = @as(usize, w) * @as(usize, h);

    // Allocate temp buffer (ask JS for memory via grow if needed)
    const buf_size = total * 4;
    const current_mem_end = @as(usize, @intCast(@wasm_memory_size(0))) * 65536;
    // temp_data starts after the image data
    const temp_start = @as(usize, @intFromPtr(image_data)) + buf_size;

    // Check we have enough memory, grow if needed
    if (temp_start + buf_size > current_mem_end) {
        const needed_pages = (temp_start + buf_size - current_mem_end + 65535) / 65536;
        const grow_result = @wasm_memory_grow(0, @intCast(needed_pages));
        if (grow_result == @as(i32, -1)) return; // OOM
    }

    temp_data = @ptrFromInt(temp_start);

    // Horizontal pass: image_data -> temp_data
    horizontalBlur(radius);
    // Vertical pass: temp_data -> image_data
    verticalBlur(radius);
}

fn horizontalBlur(radius: u32) void {
    const w = image_width;
    const h = image_height;

    var y: u32 = 0;
    while (y < h) : (y += 1) {
        for (0..4) |c| {
            var sum: u32 = 0;
            var count: u32 = 0;

            // Initialize window
            var x: u32 = 0;
            while (x < w) : (x += 1) {
                sum += @as(u32, image_data[pixelOffset(x, y) + c]);
                count += 1;

                if (x >= 2 * radius + 1) {
                    // Remove leftmost pixel
                    sum -= @as(u32, image_data[pixelOffset(x - 2 * radius - 1, y) + c]);
                    count -= 1;
                }

                // Write averaged value
                const x_write = if (x < radius) x else x - radius;
                const max_x = if (x + radius < w) x + radius else w - 1;
                const write_count = max_x - @as(u32, @max(x, radius)) + 1;

                // Simple sliding window average
                if (x >= radius) {
                    const actual_count = @min(count, 2 * radius + 1);
                    temp_data[pixelOffset(x - radius, y) + c] =
                        @intCast(sum / actual_count);
                }
            }

            // Handle the right edge
            if (w > radius) {
                const edge_count = @min(2 * radius + 1, w);
                const right_start = w - radius;
                if (right_start < w) {
                    for (right_start..w) |rx| {
                        var edge_sum: u32 = 0;
                        var edge_count: u32 = 0;
                        for (0..w) |kx| {
                            if (kx >= @as(i32, @intCast(rx)) - @as(i32, @intCast(radius)) and
                                kx <= rx + radius)
                            {
                                edge_sum += @as(u32, image_data[pixelOffset(@intCast(kx), y) + c]);
                                edge_count += 1;
                            }
                        }
                        if (edge_count > 0) {
                            temp_data[pixelOffset(@intCast(rx), y) + c] =
                                @intCast(edge_sum / edge_count);
                        }
                    }
                }
            }
        }
    }
}

fn verticalBlur(radius: u32) void {
    const w = image_width;
    const h = image_height;

    var x: u32 = 0;
    while (x < w) : (x += 1) {
        var y: u32 = 0;
        while (y < h) : (y += 1) {
            for (0..4) |c| {
                var sum: u32 = 0;
                var count: u32 = 0;

                const y_start = if (y >= radius) y - radius else 0;
                const y_end = @min(y + radius, h - 1);

                var ky = y_start;
                while (ky <= y_end) : (ky += 1) {
                    sum += @as(u32, temp_data[pixelOffset(x, ky) + c]);
                    count += 1;
                }

                if (count > 0) {
                    image_data[pixelOffset(x, y) + c] =
                        @intCast(sum / count);
                }
            }
        }
    }
}

// --- Threshold (Black & White) ---

export fn threshold(thresh: u8) void {
    const total_pixels = @as(usize, image_width) * @as(usize, image_height);
    for (0..total_pixels) |i| {
        const off = i * 4;
        // Use luminance
        const gray: u8 = @intCast((@as(u32, image_data[off + 0]) * 77 +
            @as(u32, image_data[off + 1]) * 150 +
            @as(u32, image_data[off + 2]) * 29) >> 8);

        const val: u8 = if (gray >= thresh) 255 else 0;
        image_data[off + 0] = val;
        image_data[off + 1] = val;
        image_data[off + 2] = val;
    }
}

// --- Get memory info for JS ---

export fn getMemoryInfo() struct {
    pages: u32,
    bytes: u32,
} {
    const pages = @wasm_memory_size(0);
    return .{
        .pages = pages,
        .bytes = pages * 65536,
    };
}
```

### HTML/JS Test Harness

```html
<!DOCTYPE html>
<html>
<head>
    <title>Zig WASM Image Processor</title>
    <style>
        body { font-family: monospace; padding: 20px; background: #1a1a2e; color: #eee; }
        canvas { border: 1px solid #444; margin: 10px; }
        .controls { margin: 10px 0; }
        button { padding: 8px 16px; margin: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>Zig WASM Image Processor</h1>
    <div class="controls">
        <canvas id="src" width="256" height="256"></canvas>
        <canvas id="dst" width="256" height="256"></canvas>
    </div>
    <div>
        <button onclick="doGrayscale()">Grayscale</button>
        <button onclick="doBlur()">Box Blur (r=3)</button>
        <button onclick="doInvert()">Invert</button>
        <button onclick="doThreshold()">Threshold</button>
        <button onclick="doBrighten()">Brighten +30</button>
        <button onclick="doReset()">Reset</button>
    </div>
    <script>
        const WIDTH = 256, HEIGHT = 256, SIZE = WIDTH * HEIGHT * 4;
        let wasm, memory, imageData, originalData;

        async function init() {
            const resp = await fetch('image-proc.wasm');
            const bytes = await resp.arrayBuffer();
            memory = new WebAssembly.Memory({ initial: 16 });
            const importObject = { env: { memory } };
            const { instance } = await WebAssembly.instantiate(bytes, importObject);
            wasm = instance.exports;

            generateTestImage();
            render();
        }

        function generateTestImage() {
            const canvas = document.getElementById('src');
            const ctx = canvas.getContext('2d');
            const img = ctx.createImageData(WIDTH, HEIGHT);
            for (let y = 0; y < HEIGHT; y++) {
                for (let x = 0; x < WIDTH; x++) {
                    const i = (y * WIDTH + x) * 4;
                    img.data[i]   = (x * 3) & 0xFF;
                    img.data[i+1] = (y * 3) & 0xFF;
                    img.data[i+2] = ((x + y) * 2) & 0xFF;
                    img.data[i+3] = 255;
                }
            }
            originalData = new Uint8Array(img.data);
            ctx.putImageData(img, 0, 0);
        }

        function processAndRender(fn) {
            const buf = new Uint8Array(memory.buffer, 0, SIZE);
            buf.set(originalData);
            wasm.setImageBuffer(0, WIDTH, HEIGHT);
            fn();
            const canvas = document.getElementById('dst');
            const ctx = canvas.getContext('2d');
            const img = new ImageData(new Uint8ClampedArray(buf.slice(0, SIZE)), WIDTH, HEIGHT);
            ctx.putImageData(img, 0, 0);
        }

        function doGrayscale() { processAndRender(() => wasm.grayscale()); }
        function doBlur() { processAndRender(() => wasm.boxBlur(3)); }
        function doInvert() { processAndRender(() => wasm.invertColors()); }
        function doThreshold() { processAndRender(() => wasm.threshold(128)); }
        function doBrighten() { processAndRender(() => wasm.adjustBrightness(30)); }
        function doReset() {
            const canvas = document.getElementById('dst');
            const ctx = canvas.getContext('2d');
            ctx.clearRect(0, 0, WIDTH, HEIGHT);
        }

        function render() {
            const info = wasm.getMemoryInfo();
            console.log(`WASM memory: ${info.pages} pages (${info.bytes} bytes)`);
        }

        init();
    </script>
</body>
</html>
```

### Building and Testing

```bash
# Build the WASM module
zig build -Doptimize=ReleaseFast

# Copy to your web server directory
cp zig-out/bin/image-proc.wasm /path/to/your/web/root/

# Or serve locally
python3 -m http.server 8000
# Open http://localhost:8000/index.html
```

---

## Summary

| Topic | Key Points |
|---|---|
| **Targets** | `wasm32-freestanding` (bare), `wasm32-wasi` (with system interface) |
| **Memory Builtins** | `@wasm_memory_size(index)`, `@wasm_memory_grow(index, delta)` — new in 0.16 |
| **Exporting** | `export fn name(...)` makes function callable from JS |
| **Importing** | `extern fn name(...)` declares JS functions callable from Zig |
| **Memory** | Linear model, grows in 64KB pages, never shrinks |
| **WASI** | File I/O, env vars, args via `std.Io` and `std.process` |
| **Debugging** | DWARF debug info, `jsDebugLog` pattern, Chrome DevTools |
| **Performance** | `ReleaseSmall`, comptime tables, stack buffers, SIMD vectors |
| **DOM** | Via `@extern` JS imports — no direct DOM access from WASM |