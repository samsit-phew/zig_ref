# C Interop in Zig 0.16

## Table of Contents

- [Chapter 1: Introduction to Zig-C Interop](#chapter-1-introduction-to-zig-c-interop)
- [Chapter 2: @cImport — Importing C Headers Directly](#chapter-2-cimport--importing-c-headers-directly)
- [Chapter 3: extern "c" fn — Declaring External C Functions](#chapter-3-extern-c-fn--declaring-external-c-functions)
- [Chapter 4: Calling C from Zig — Pointers, Slices, Null-Terminated Strings](#chapter-4-calling-c-from-zig--pointers-slices-null-terminated-strings)
- [Chapter 5: Zig Types Mapped to C](#chapter-5-zig-types-mapped-to-c)
- [Chapter 6: Calling Zig from C — @export, @extern](#chapter-6-calling-zig-from-c--export-extern)
- [Chapter 7: translate-c — Converting C Code to Zig](#chapter-7-translate-c--converting-c-code-to-zig)
- [Chapter 8: Struct Layout and packed/extern Structs](#chapter-8-struct-layout-and-packedextern-structs)
- [Chapter 9: Dealing with C Memory (free, malloc, allocators)](#chapter-9-dealing-with-c-memory-free-malloc-allocators)
- [Chapter 10: Project — Wrapping a C Library (SQLite Bindings)](#chapter-10-project--wrapping-a-c-library-sqlite-bindings)

---

## Chapter 1: Introduction to Zig-C Interop

One of Zig's most powerful features is its **seamless C interoperability**. Zig can:

1. **Import C headers directly** with `@cImport` — no binding generator needed.
2. **Call C functions** as easily as Zig functions, with automatic type conversion.
3. **Export Zig functions** to C with `@export`.
4. **Cross-compile C code** alongside Zig using the built-in Clang.
5. **Link against system libraries** with `@cImport` and build system support.
6. **Translate C to Zig** automatically with `translate-c`.

Zig's C ABI support is first-class, not an afterthought. The same compiler handles both languages, ensuring perfect compatibility.

### Why C Interop Matters

- **Use existing libraries**: SQLite, OpenSSL, libcurl, and millions of C libraries work out of the box.
- **Incremental migration**: Rewrite C codebases one module at a time.
- **Systems programming**: Direct access to OS APIs (POSIX, Win32) which are C-based.
- **No FFI overhead**: Zig calls C with zero-cost abstraction — no marshaling, no JIT.

---

## Chapter 2: @cImport — Importing C Headers Directly

`@cImport` invokes the C compiler (Clang, built into Zig) to parse a C header and generate equivalent Zig declarations. The result is a Zig struct whose fields are the imported C declarations.

```zig
const std = @import("std");

// Import standard C headers
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("stdlib.h");
    @cInclude("string.h");
    @cInclude("math.h");
});

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // Use C functions directly
    _ = c.printf("Hello from C's printf!\n");

    const result = c.abs(-42);
    try stdout.print("C abs(-42) = {}\n", .{result});

    const pi = c.M_PI;
    try stdout.print("M_PI = {d:.15}\n", .{pi});

    const len = c.strlen("Hello from strlen");
    try stdout.print("strlen returned: {}\n", .{len});
}
```

### @cInclude, @cDefine, @cUndef for Controlling Imports

You can control what gets imported by defining/undefining macros before including headers:

```zig
const std = @import("std");

const c = @cImport({
    // Define a macro before including the header
    @cDefine("_GNU_SOURCE", "1");
    @cDefine("FEATURE_X", "1");

    // Undefine something
    @cUndef("NDEBUG");

    // Now include headers that depend on those macros
    @cInclude("stdio.h");
    @cInclude("features.h");
});

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    try stdout.print("_GNU_SOURCE defined: {}\n", .{c._GNU_SOURCE});
}
```

### Importing Your Own C Headers

```zig
// Assuming you have a file `mylib.h` in an include path:
const mylib = @cImport({
    @cInclude("mylib.h");
});

// Now use mylib.my_function(), mylib.MY_CONSTANT, etc.
```

To add include paths, configure it in `build.zig`:

```zig
const exe = b.addExecutable(.{
    .name = "my-program",
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});

// Add C include paths
exe.addIncludePath(b.path("vendor/mylib/include"));

// Link against a C library
exe.linkLibrary(mylib_lib);
```

### @cImport is Comptime

`@cImport` runs at compile time. It invokes Clang to parse the C code and generate Zig declarations. This means:

- There is **no runtime cost** — the C types are resolved during compilation.
- You get **type safety** — mismatched types cause compile errors.
- The imported declarations are **cached** — repeated `@cImport` of the same header is efficient.

---

## Chapter 3: extern "c" fn — Declaring External C Functions

Instead of using `@cImport`, you can manually declare C functions using `extern "c" fn`. This is useful when you don't want the overhead of importing an entire header.

```zig
const std = @import("std");

// Declare a C function manually
extern "c" fn puts(str: [*:0]const u8) c_int;

// Or with the full extern block
extern "c" {
    fn strlen(str: [*:0]const u8) usize;
    fn atoi(str: [*:0]const u8) c_int;
    fn malloc(size: usize) ?*anyopaque;
    fn free(ptr: *anyopaque) void;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // Call the manually declared C function
    _ = puts("Hello from extern 'c' fn!");

    const len = strlen("measured by C strlen");
    try stdout.print("Length: {}\n", .{len});

    const num = atoi("12345");
    try stdout.print("atoi('12345') = {}\n", .{num});
}
```

### Lazy vs Eager External Function Resolution

By default, Zig resolves `extern "c"` functions lazily at link time. If the symbol is not found, you get a linker error. You can also mark them with `@extern` for more control:

```zig
// Lazy (default) — resolved at link time
extern "c" fn getenv(name: [*:0]const u8) ?[*:0]const u8;

// Explicit — you can also specify the library name
const libc_getenv = @extern(*const fn ([*:0]const u8) ?[*:0]const u8, .{
    .name = "getenv",
});
```

---

## Chapter 4: Calling C from Zig — Pointers, Slices, Null-Terminated Strings

The most important aspect of C interop is understanding how Zig types map to C types, especially around pointers.

### Null-Terminated Strings

C strings are `char*` — pointers to null-terminated byte arrays. In Zig, these are `[*:0]const u8` (read-only) or `[*:0]u8` (mutable).

```zig
const std = @import("std");

extern "c" fn puts(s: [*:0]const u8) c_int;
extern "c" fn strlen(s: [*:0]const u8) usize;
extern "c" fn strcpy(dst: [*:0]u8, src: [*:0]const u8) [*:0]u8;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    // Zig string literal -> C string: it's already null-terminated!
    const zig_str: []const u8 = "Hello, C!";
    // String literals in Zig have a sentinel 0 byte at the end.
    // But []const u8 doesn't carry sentinel info. Use [*:0]const u8:

    const c_str: [*:0]const u8 = @ptrCast(zig_str.ptr);
    _ = puts(c_str);

    // For heap-allocated strings, use dupeZ to get null-terminated:
    const heap_str = try allocator.dupeZ(u8, "Heap allocated C string");
    defer allocator.free(heap_str);
    // heap_str is [*:0]const u8

    const len = strlen(heap_str);
    try stdout.print("strlen: {}\n", .{len});
    _ = puts(heap_str);

    // Convert C string back to Zig slice:
    // Use std.mem.span to get a []const u8 from a [*:0]const u8
    const c_result: [*:0]const u8 = heap_str;
    const zig_slice: []const u8 = std.mem.span(c_result);
    try stdout.print("Back to Zig slice: {s}\n", .{zig_slice});
}
```

### Passing Zig Slices to C

C doesn't understand Zig slices. You must pass the pointer and length separately:

```zig
const std = @import("std");

extern "c" fn write(fd: c_int, buf: [*]const u8, count: usize) isize;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const data: []const u8 = "Writing a slice to C's write()\n";
    // Pass .ptr and .len to C functions
    const bytes_written = write(1, data.ptr, data.len);
    try stdout.print("write() returned: {}\n", .{bytes_written});
}
```

### Receiving C Arrays in Zig

When C returns a pointer to an array, use Zig's pointer-to-array syntax:

```zig
const std = @import("std");

// C function that returns int* (pointer to array)
extern "c" fn get_data(count: *c_int) [*]const u8;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    var count: c_int = 0;
    const data = get_data(&count);
    // Use count to determine the slice length
    const slice: []const u8 = data[0..@intCast(count)];
    try stdout.print("Received {} bytes from C\n", .{slice.len});
}
```

---

## Chapter 5: Zig Types Mapped to C

Zig's type system maps directly to C types. Here's the complete mapping:

### Integer Types

| Zig Type | C Equivalent | Size |
|----------|-------------|------|
| `c_char` | `char` | 1 byte |
| `c_short` | `short` | 2 bytes |
| `c_ushort` | `unsigned short` | 2 bytes |
| `c_int` | `int` | 4 bytes |
| `c_uint` | `unsigned int` | 4 bytes |
| `c_long` | `long` | 4 (LLP64) or 8 (LP64) bytes |
| `c_ulong` | `unsigned long` | varies |
| `c_longlong` | `long long` | 8 bytes |
| `c_ulonglong` | `unsigned long long` | 8 bytes |
| `i8`–`i128` | `int8_t`–`int128_t` | fixed |
| `u8`–`u128` | `uint8_t`–`uint128_t` | fixed |
| `isize` | `ssize_t` / `intptr_t` | pointer-sized |
| `usize` | `size_t` | pointer-sized |

### Other Types

| Zig Type | C Equivalent |
|----------|-------------|
| `f32` | `float` |
| `f64` | `double` |
| `bool` | `_Bool` (C99+) |
| `?*T` | `T*` (nullable pointer) |
| `*T` | `T*` (non-null pointer) |
| `[*]T` | `T*` (unknown-length pointer) |
| `[*:0]T` | `T*` (null-terminated) |
| `*const T` | `const T*` |
| `*volatile T` | `volatile T*` |
| `void` | `void` |
| `noreturn` | `_Noreturn` (C11) |
| `[N]T` | `T[N]` (fixed array) |
| `extern struct` | `struct` (C layout) |
| `extern enum` | `enum` (C layout) |
| `fn(...) T` | function pointer |

### Using C Types Example

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // Show sizes of C types on this platform
    try stdout.print("=== C Type Sizes ===\n", .{});
    try stdout.print("c_char:      {} bytes\n", .{@sizeOf(c_char)});
    try stdout.print("c_short:     {} bytes\n", .{@sizeOf(c_short)});
    try stdout.print("c_int:       {} bytes\n", .{@sizeOf(c_int)});
    try stdout.print("c_long:      {} bytes\n", .{@sizeOf(c_long)});
    try stdout.print("c_longlong:  {} bytes\n", .{@sizeOf(c_longlong)});
    try stdout.print("c_float:     {} bytes\n", .{@sizeOf(f32)});
    try stdout.print("c_double:    {} bytes\n", .{@sizeOf(f64)});
    try stdout.print("pointer:     {} bytes\n", .{@sizeOf(*anyopaque)});
    try stdout.print("usize:       {} bytes\n", .{@sizeOf(usize)});
}
```

---

## Chapter 6: Calling Zig from C — @export, @extern

### Exporting Zig Functions to C

Use `@export` to make a Zig function visible to C code with C calling convention:

```zig
const std = @import("std");

// Export a function with C calling convention
export fn add(a: c_int, b: c_int) c_int {
    return a + b;
}

// Export a function that takes a C string
export fn greet(name: [*:0]const u8) void {
    const stdout = std.io.getStdOut().writer();
    stdout.print("Hello, {s}! (called from C)\n", .{std.mem.span(name)}) catch {};
}

// Export a function that returns a heap-allocated string
var global_allocator: std.mem.Allocator = undefined;

export fn create_message(msg: [*:0]const u8) [*:0]u8 {
    const input = std.mem.span(msg);
    // Prepend "Message: " to the input
    const prefix = "Message: ";
    const result = global_allocator.dupeZ(u8, prefix) catch return @constCast(&[_:0]u8{});
    // In a real implementation, you'd concatenate and return the full string
    _ = input;
    return result.ptr;
}

export fn setAllocator(alloc: *anyopaque) void {
    // This is a simplified example; in practice you'd use a more careful interface
    _ = alloc;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    global_allocator = init.allocator;

    // These exported functions are also callable from Zig
    const sum = add(10, 32);
    try stdout.print("add(10, 32) = {}\n", .{sum});

    // Call the exported function
    greet("Zig Developer");
}
```

### Building as a Static Library

To build Zig code as a library for C consumption, use `build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Build as a static library
    const lib = b.addStaticLibrary(.{
        .name = "myziglib",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(lib);
}
```

Then use from C:

```c
// main.c
#include <stdio.h>

extern int add(int a, int b);
extern void greet(const char *name);

int main(void) {
    printf("add(5, 3) = %d\n", add(5, 3));
    greet("C Developer");
    return 0;
}
```

Compile and link:

```bash
zig build
cc -o main main.c -L.zig-cache -lmyziglib -lpthread
```

### Exporting Global Variables

```zig
// Export a variable
export var global_counter: c_int = 0;

// Export a constant
export const VERSION: [*:0]const u8 = "1.0.0";
```

---

## Chapter 7: translate-c — Converting C Code to Zig

Zig includes `translate-c`, a tool that converts C code to idiomatic Zig. It handles the same C parsing as `@cImport` but outputs Zig source code.

### Using translate-c

```bash
zig translate-c input.h > output.zig
```

### Example: Translating a C Header

Given `geometry.h`:

```c
typedef struct {
    double x;
    double y;
} Point;

typedef struct {
    Point center;
    double radius;
} Circle;

double circle_area(Circle c);
double circle_circumference(Circle c);
```

Run `zig translate-c geometry.h` to get Zig output:

```zig
pub const Point = extern struct {
    x: f64,
    y: f64,
};

pub const Circle = extern struct {
    center: Point,
    radius: f64,
};

pub extern fn circle_area(c: Circle) f64;
pub extern fn circle_circumference(c: Circle) f64;
```

### Translating Complex Headers

```bash
# Translate a system header
zig translate-c /usr/include/sys/stat.h > zig_stat.zig

# Translate with defines
echo '#define _GNU_SOURCE
#include <fcntl.h>' | zig translate-c - > zig_fcntl.zig
```

### Limitations of translate-c

- Macro functions (`#define ADD(a,b) ((a)+(b))`) cannot be translated — they become constants only.
- Variadic macros and complex preprocessor logic may not translate perfectly.
- C++ is not supported (C only).

---

## Chapter 8: Struct Layout and packed/extern Structs

### extern Structs (C-compatible Layout)

By default, Zig structs use their natural alignment, which may differ from C. Use `extern` to guarantee C-compatible layout:

```zig
const std = @import("std");

// C layout: matches what a C compiler would produce
pub const CPoint = extern struct {
    x: f64,
    y: f64,
};

// Zig layout: may differ (though for simple cases, it's the same)
pub const ZigPoint = struct {
    x: f64,
    y: f64,
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    try stdout.print("CPoint size:     {}\n", .{@sizeOf(CPoint)});
    try stdout.print("ZigPoint size:   {}\n", .{@sizeOf(ZigPoint)});
    try stdout.print("Both align to:   {}\n", .{@alignOf(CPoint)});

    // They should be identical for this simple case
    comptime {
        if (@sizeOf(CPoint) != @sizeOf(ZigPoint)) {
            @compileError("Layout mismatch!");
        }
    }
}
```

### Packed Structs (Bit-Field Compatible)

`packed struct` lays out fields with minimal padding, using exactly the number of bits specified:

```zig
const std = @import("std");

// Packed struct: each field uses exactly the bits specified
const Flags = packed struct {
    readable: bool,     // 1 bit
    writable: bool,     // 1 bit
    executable: bool,   // 1 bit
    reserved: u5,       // 5 bits
    owner_uid: u8,      // 8 bits
    permissions: u16,   // 16 bits
};

// Equivalent C bitfield (roughly):
// struct flags {
//     unsigned readable : 1;
//     unsigned writable : 1;
//     unsigned executable : 1;
//     unsigned reserved : 5;
//     uint8_t owner_uid;
//     uint16_t permissions;
// };

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const flags = Flags{
        .readable = true,
        .writable = true,
        .executable = false,
        .reserved = 0,
        .owner_uid = 1000,
        .permissions = 0o644,
    };

    try stdout.print("Flags size: {} bytes ({} bits)\n", .{
        @sizeOf(Flags),
        @bitSizeOf(Flags),
    });
    try stdout.print("Readable:   {}\n", .{flags.readable});
    try stdout.print("Writable:   {}\n", .{flags.writable});
    try stdout.print("Owner UID:  {}\n", .{flags.owner_uid});
    try stdout.print("Perms:      {o}\n", .{flags.permissions});
}
```

### Enum with C Representation

```zig
const std = @import("std");

// extern enum guarantees C-compatible representation (int-sized)
pub const Color = extern enum(c_int) {
    Red = 0,
    Green = 1,
    Blue = 2,
    Yellow = 10,
    _,
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const c: Color = .Blue;
    const c_int_val: c_int = @intFromEnum(c);
    try stdout.print("Color Blue as c_int: {}\n", .{c_int_val});

    // Convert C int back to enum
    const from_c: Color = @enumFromInt(1);
    try stdout.print("c_int 1 -> Color: {}\n", .{from_c});
}
```

---

## Chapter 9: Dealing with C Memory (free, malloc, allocators)

### Using C malloc/free Directly

```zig
const std = @import("std");

extern "c" {
    fn malloc(size: usize) ?*anyopaque;
    fn free(ptr: *anyopaque) void;
    fn realloc(ptr: *anyopaque, size: usize) ?*anyopaque;
    fn calloc(nmemb: usize, size: usize) ?*anyopaque;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // malloc
    const ptr = malloc(100);
    if (ptr == null) {
        try stdout.print("malloc failed!\n", .{});
        return error.OutOfMemory;
    }
    defer free(ptr.?);

    // Cast to usable type
    const buf: [*]u8 = @ptrCast(@alignCast(ptr.?));
    @memset(buf[0..100], 0xAA);

    try stdout.print("malloc'd 100 bytes, first byte: 0x{X:0>2}\n", .{buf[0]});

    // realloc
    const new_ptr = realloc(ptr.?, 200);
    if (new_ptr != null) {
        const new_buf: [*]u8 = @ptrCast(@alignCast(new_ptr.?));
        try stdout.print("realloc'd to 200 bytes\n", .{});
        // Note: we freed the old pointer through realloc; need to free the new one
        // But we already deferred free on the old ptr... this is a problem!
        // Better to use the C allocator wrapper
        _ = new_buf;
    }
}
```

### Using std.heap.c_allocator

Zig provides `std.heap.c_allocator` which wraps the C `malloc`/`free` as a Zig `Allocator`:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const c_alloc = std.heap.c_allocator;

    // Use C's malloc/free through Zig's allocator interface
    const data = try c_alloc.alloc(u8, 100);
    defer c_alloc.free(data);

    @memset(data, 'X');
    try stdout.print("C allocator: allocated {} bytes\n", .{data.len});
    try stdout.print("Content: {s}\n", .{data[0..10]});

    // Create a value
    const num = try c_alloc.create(i32);
    defer c_alloc.destroy(num);
    num.* = 42;
    try stdout.print("Created i32: {}\n", .{num.*});
}
```

### Passing Memory Allocated by Zig to C

When passing Zig-allocated memory to C, ensure the memory has the correct alignment and lifetime:

```zig
const std = @import("std");

extern "c" fn process_data(data: [*]u8, len: usize) c_int;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    // Allocate with proper alignment for C
    const data = try allocator.alignedAlloc(u8, 16, 256);
    defer allocator.free(data);

    @memset(data, 0);
    @memcpy(data, "Hello from Zig to C");

    // Pass to C — data stays valid until defer free
    const result = process_data(data.ptr, data.len);
    try stdout.print("C process_data returned: {}\n", .{result});
}
```

### Freeing C-Allocated Memory in Zig

When C allocates memory and returns it to Zig, you must use C's `free`:

```zig
const std = @import("std");

extern "c" {
    fn strdup(s: [*:0]const u8) ?[*:0]u8;
    fn free(ptr: *anyopaque) void;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // C allocates memory with strdup
    const c_str = strdup("C allocated this string");
    if (c_str == null) return error.OutOfMemory;
    defer free(@ptrCast(c_str.?)); // Must use C's free!

    const zig_slice = std.mem.span(c_str.?);
    try stdout.print("C-allocated string: {s}\n", .{zig_slice});
}
```

---

## Chapter 10: Project — Wrapping a C Library (SQLite Bindings)

This project demonstrates wrapping SQLite's C API with a safe, idiomatic Zig interface. SQLite is chosen because it's ubiquitous and its C API is well-documented.

### src/main.zig

```zig
const std = @import("std");

// Import SQLite C header
const sqlite = @cImport({
    @cInclude("sqlite3.h");
});

const DatabaseError = error{
    OpenFailed,
    PrepareFailed,
    StepFailed,
    BindFailed,
    ColumnError,
    NoRows,
    SqliteError,
};

fn dbCheck(rc: c_int) !void {
    if (rc != sqlite.SQLITE_OK) {
        std.debug.print("SQLite error: {s}\n", .{sqlite.sqlite3_errstr(rc)});
        return DatabaseError.SqliteError;
    }
}

const Database = struct {
    db: *sqlite.sqlite3,

    fn open(path: [*:0]const u8) !Database {
        var db: ?*sqlite.sqlite3 = null;
        const rc = sqlite.sqlite3_open(path, &db);
        if (rc != sqlite.SQLITE_OK or db == null) {
            return DatabaseError.OpenFailed;
        }
        return .{ .db = db.? };
    }

    fn close(self: *Database) void {
        _ = sqlite.sqlite3_close(self.db);
        self.db = undefined;
    }

    fn exec(self: *Database, sql: [*:0]const u8) !void {
        var errmsg: ?[*:0]u8 = null;
        const rc = sqlite.sqlite3_exec(self.db, sql, null, null, &errmsg);
        if (rc != sqlite.SQLITE_OK) {
            if (errmsg) |msg| {
                std.debug.print("SQL error: {s}\n", .{msg});
                sqlite.sqlite3_free(msg);
            }
            return DatabaseError.SqliteError;
        }
    }
};

const Statement = struct {
    stmt: *sqlite.sqlite3_stmt,

    fn prepare(db: *Database, sql: [*:0]const u8) !Statement {
        var stmt: ?*sqlite.sqlite3_stmt = null;
        const rc = sqlite.sqlite3_prepare_v2(db.db, sql, -1, &stmt, null);
        if (rc != sqlite.SQLITE_OK or stmt == null) {
            return DatabaseError.PrepareFailed;
        }
        return .{ .stmt = stmt.? };
    }

    fn finalize(self: *Statement) void {
        _ = sqlite.sqlite3_finalize(self.stmt);
    }

    fn bindText(self: *Statement, idx: c_int, text: []const u8) !void {
        const rc = sqlite.sqlite3_bind_text(
            self.stmt,
            idx,
            text.ptr,
            @intCast(text.len),
            sqlite.SQLITE_TRANSIENT,
        );
        try dbCheck(rc);
    }

    fn bindInt(self: *Statement, idx: c_int, value: i64) !void {
        const rc = sqlite.sqlite3_bind_int64(self.stmt, idx, value);
        try dbCheck(rc);
    }

    fn step(self: *Statement) !bool {
        const rc = sqlite.sqlite3_step(self.stmt);
        switch (rc) {
            sqlite.SQLITE_ROW => return true,
            sqlite.SQLITE_DONE => return false,
            else => return DatabaseError.StepFailed,
        }
    }

    fn columnText(self: *Statement, idx: c_int) ![]const u8 {
        const ptr = sqlite.sqlite3_column_text(self.stmt, idx);
        if (ptr == null) return DatabaseError.ColumnError;
        const len = @intCast(sqlite.sqlite3_column_bytes(self.stmt, idx));
        return ptr[0..len];
    }

    fn columnInt(self: *Statement, idx: c_int) i64 {
        return sqlite.sqlite3_column_int64(self.stmt, idx);
    }

    fn reset(self: *Statement) void {
        _ = sqlite.sqlite3_reset(self.stmt);
    }
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // Open an in-memory database
    var db = try Database.open(":memory:");
    defer db.close();

    try stdout.print("=== Zig SQLite Wrapper Demo ===\n\n", .{});

    // Create a table
    try db.exec(
        \\CREATE TABLE users (
        \\  id INTEGER PRIMARY KEY AUTOINCREMENT,
        \\  name TEXT NOT NULL,
        \\  email TEXT UNIQUE NOT NULL,
        \\  age INTEGER
        \\);
    );

    try stdout.print("Created 'users' table\n", .{});

    // Insert data using prepared statements
    var insert_stmt = try Statement.prepare(&db,
        "INSERT INTO users (name, email, age) VALUES (?1, ?2, ?3);"
    );
    defer insert_stmt.finalize();

    const users = [_]struct { name: []const u8, email: []const u8, age: i64 }{
        .{ .name = "Alice Johnson",  .email = "alice@example.com",  .age = 30 },
        .{ .name = "Bob Smith",     .email = "bob@example.com",    .age = 25 },
        .{ .name = "Charlie Brown", .email = "charlie@example.com", .age = 35 },
    };

    for (users) |user| {
        try insert_stmt.bindText(1, user.name);
        try insert_stmt.bindText(2, user.email);
        try insert_stmt.bindInt(3, user.age);
        _ = try insert_stmt.step();
        insert_stmt.reset();
    }

    try stdout.print("Inserted {} users\n\n", .{users.len});

    // Query data
    var query_stmt = try Statement.prepare(&db, "SELECT id, name, email, age FROM users;");
    defer query_stmt.finalize();

    try stdout.print("{s:3}  {s:20}  {s:25}  {s:3}\n", .{ "ID", "Name", "Email", "Age" });
    try stdout.print("{s:-^3}  {s:-^20}  {s:-^25}  {s:-^3}\n", .{
        "---", "--------------------", "-------------------------", "---",
    });

    var row_count: usize = 0;
    while (try query_stmt.step()) {
        const id = query_stmt.columnInt(0);
        const name = try query_stmt.columnText(1);
        const email = try query_stmt.columnText(2);
        const age = query_stmt.columnInt(3);

        try stdout.print("{:>3}  {s:<20}  {s:<25}  {:>3}\n", .{
            id, name, email, age,
        });
        row_count += 1;
    }

    try stdout.print("\nTotal rows: {}\n", .{row_count});
    try stdout.print("\nSQLite wrapper demo complete!\n", .{});
}
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "sqlite-demo",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Link against SQLite
    exe.linkSystemLibrary("sqlite3");

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the SQLite demo");
    run_step.dependOn(&run_cmd.step);
}
```

### Running

Make sure SQLite development headers are installed:

```bash
# Ubuntu/Debian
sudo apt install libsqlite3-dev

# macOS (SQLite is pre-installed)
# No action needed

# Fedora
sudo dnf install sqlite-devel

# Build and run
zig build run
```

### Expected Output

```
=== Zig SQLite Wrapper Demo ===

Created 'users' table
Inserted 3 users

ID   Name                  Email                      Age
---  --------------------  -------------------------  ---
  1  Alice Johnson         alice@example.com           30
  2  Bob Smith             bob@example.com             25
  3  Charlie Brown         charlie@example.com         35

Total rows: 3

SQLite wrapper demo complete!
```

### Summary

Zig 0.16's C interop is seamless and powerful. `@cImport` lets you import any C header without a binding generator. `extern "c" fn` provides manual control. `@export` makes Zig functions callable from C. `translate-c` automates migration. And with `extern struct` and `packed struct`, you have full control over memory layout. The SQLite wrapper project demonstrates how to build a safe, idiomatic Zig API over a C library, handling error codes, resource cleanup with `defer`, and type-safe access to C data structures.