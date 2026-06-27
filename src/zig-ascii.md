# ASCII and Unicode in Zig 0.16

## Table of Contents

- [Chapter 1: Introduction to Text Processing in Zig](#chapter-1-introduction-to-text-processing-in-zig)
- [Chapter 2: std.ascii — Character Classification](#chapter-2-stdascii--character-classification)
- [Chapter 3: std.ascii — Case Conversion](#chapter-3-stdascii--case-conversion)
- [Chapter 4: std.ascii — Control Characters and Whitespace](#chapter-4-stdascii--control-characters-and-whitespace)
- [Chapter 5: std.unicode — UTF-8 Encoding/Decoding](#chapter-5-stdunicode--utf-8-encodingdecoding)
- [Chapter 6: std.unicode — Codepoint Iteration (Utf8View)](#chapter-6-stdunicode--codepoint-iteration-utf8view)
- [Chapter 7: std.unicode — Case Folding and Normalization](#chapter-7-stdunicode--case-folding-and-normalization)
- [Chapter 8: Practical — Parsing ASCII Protocols](#chapter-8-practical--parsing-ascii-protocols)
- [Chapter 9: Project — A Custom String Sanitizer / Slug Generator](#chapter-9-project--a-custom-string-sanitizer--slug-generator)

---

## Chapter 1: Introduction to Text Processing in Zig

Zig treats strings as arrays of bytes (`[]u8` or `[]const u8`). There is no dedicated `String` type. This design choice is deliberate: Zig gives you full control over text representation, encoding, and memory layout. Whether you're working with ASCII, UTF-8, or raw binary data, the same slice type handles it all.

Zig's standard library provides two key modules for text processing:

- **`std.ascii`** — Functions for 7-bit ASCII characters. Fast, simple, no allocation.
- **`std.unicode`** — Functions for Unicode (UTF-8) encoding, decoding, and codepoint operations.

The distinction matters: `std.ascii` functions are designed for single-byte ASCII (0x00–0x7F), while `std.unicode` handles the variable-width UTF-8 encoding that Zig uses natively for source code and string literals.

### Zig's String Model

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // A string literal in Zig is a []const u8 (a slice of bytes)
    const greeting: []const u8 = "Hello, 世界!";
    // This is valid UTF-8: 7 ASCII bytes + 6 UTF-8 bytes for the two CJK chars + '!' + null

    try stdout.print("String: {s}\n", .{greeting});
    try stdout.print("Byte length: {}\n", .{greeting.len}); // 14 (bytes, not codepoints)
    try stdout.print("Pointer: {*}\n", .{greeting.ptr});
}
```

Key insight: `greeting.len` is **14 bytes**, but only **10 codepoints**. This is the fundamental challenge of Unicode text processing, and Zig's `std.unicode` module handles it.

---

## Chapter 2: std.ascii — Character Classification

The `std.ascii` module provides compile-time evaluable functions to classify individual bytes as ASCII characters. All functions accept a `u8` and return a `bool`. They are designed to be fast — typically a single comparison or a lookup table.

### Character Classification Functions

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const test_chars = [_]u8{ 'A', 'z', '0', '9', ' ', '\n', '@', '_', 'a' };

    try stdout.print("=== ASCII Character Classification ===\n", .{});
    try stdout.print("{s:5} {s:8} {s:8} {s:8} {s:8} {s:8}\n", .{
        "Char", "isAlpha", "isDigit", "isHex", "isSpace", "isAlnum",
    });
    try stdout.print("{s:5} {s:8} {s:8} {s:8} {s:8} {s:8}\n", .{
        "----", "------", "------", "-----", "------", "------",
    });

    for (test_chars) |ch| {
        const display = if (ch < 0x20 or ch == 0x7F)
            std.fmt.formatIntValue(ch, 10, .lower, .{})
        else
            std.fmt.formatIntValue(ch, 10, .lower, .{});

        try stdout.print(" {c:4}  {:>8} {:>8} {:>8} {:>8} {:>8}\n", .{
            ch,
            std.ascii.isAlpha(ch),
            std.ascii.isDigit(ch),
            std.ascii.isHex(ch),
            std.ascii.isWhitespace(ch),
            std.ascii.isAlphanumeric(ch),
        });
    }
}
```

### Available Classification Functions

| Function | Description |
|----------|-------------|
| `isAlpha(c)` | A–Z or a–z |
| `isDigit(c)` | 0–9 |
| `isHex(c)` | 0–9, A–F, a–f |
| `isSpace(c)` / `isWhitespace(c)` | Space, tab, newline, carriage return, form feed, vertical tab |
| `isAlphanumeric(c)` | isAlpha or isDigit |
| `isPunct(c)` | Punctuation: ! " # $ % & ' ( ) * + , - . / : ; < = > ? @ [ \ ] ^ _ ` { \| } ~ |
| `isGraph(c)` | Printable and not space |
| `isPrint(c)` | Printable (including space) |
| `isControl(c)` | 0x00–0x1F, 0x7F |
| `isASCII(c)` | 0x00–0x7F |
| `isUpper(c)` | A–Z |
| `isLower(c)` | a–z |

All of these work at comptime, making them excellent for validation in type checks and compile-time logic.

### Comptime Usage Example

```zig
const std = @import("std");

fn isAllAsciiHex(comptime input: []const u8) bool {
    for (input) |ch| {
        if (!std.ascii.isHex(ch)) return false;
    }
    return true;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // This is evaluated at compile time
    comptime {
        if (!isAllAsciiHex("DEADBEEF")) {
            @compileError("expected hex string");
        }
    }
    try stdout.print("comptime hex validation passed!\n", .{});
}
```

---

## Chapter 3: std.ascii — Case Conversion

`std.ascii` provides `toUpper` and `toLower` for individual ASCII characters. These are fast single-byte operations.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const original = "Hello, World! 123";
    var upper_buf: [original.len]u8 = undefined;
    var lower_buf: [original.len]u8 = undefined;

    for (original, 0..) |ch, i| {
        upper_buf[i] = std.ascii.toUpper(ch);
        lower_buf[i] = std.ascii.toLower(ch);
    }

    try stdout.print("Original: {s}\n", .{original});
    try stdout.print("Upper:    {s}\n", .{&upper_buf});
    try stdout.print("Lower:    {s}\n", .{&lower_buf});
}
```

### EqlIgnoreCase

For comparing ASCII strings case-insensitively:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const a = "Content-Type";
    const b = "content-type";

    if (std.ascii.eqlIgnoreCase(a, b)) {
        try stdout.print("'{s}' equals '{s}' (case-insensitive)\n", .{a, b});
    }

    // There is also a version that takes explicit lengths
    const c = "HTTP/1.1";
    const d = "http/1.0";
    if (!std.ascii.eqlIgnoreCase(c, d)) {
        try stdout.print("'{s}' does NOT equal '{s}'\n", .{c, d});
    }
}
```

---

## Chapter 4: std.ascii — Control Characters and Whitespace

Understanding control characters is essential for parsing text protocols and processing raw data.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    try stdout.print("=== ASCII Control Characters ===\n", .{});

    // Common control characters
    const controls = [_]struct {
        byte: u8,
        name: []const u8,
        escape: []const u8,
    }{
        .{ .byte = 0x00, .name = "NUL",  .escape = "\\0" },
        .{ .byte = 0x09, .name = "TAB",  .escape = "\\t" },
        .{ .byte = 0x0A, .name = "LF",   .escape = "\\n" },
        .{ .byte = 0x0D, .name = "CR",   .escape = "\\r" },
        .{ .byte = 0x1B, .name = "ESC",  .escape = "\\x1b" },
        .{ .byte = 0x20, .name = "SPC",  .escape = " " },
        .{ .byte = 0x7F, .name = "DEL",  .escape = "\\x7f" },
    };

    for (controls) |c| {
        const is_ctrl = std.ascii.isControl(c.byte);
        const is_ws = std.ascii.isWhitespace(c.byte);
        try stdout.print("  0x{X:0>2} ({s:>3}) escape={s:<6} ctrl={} ws={}\n", .{
            c.byte, c.name, c.escape, is_ctrl, is_ws,
        });
    }

    // Strip whitespace from a string
    const raw = "   \t  hello world  \n  ";
    const stripped = std.mem.trim(u8, raw, " \t\n\r");
    try stdout.print("\nOriginal: '{s}'\n", .{raw});
    try stdout.print("Stripped: '{s}'\n", .{stripped});
}
```

### Whitespace Trimming

`std.mem.trim` works with any set of bytes to trim:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const input = "/// Hello ///";
    const trimmed = std.mem.trim(u8, input, "/ ");
    try stdout.print("Trimmed: '{s}'\n", .{trimmed}); // "Hello"
}
```

---

## Chapter 5: std.unicode — UTF-8 Encoding/Decoding

Zig string literals are UTF-8 encoded. The `std.unicode` module provides functions to safely encode and decode UTF-8 sequences.

### UTF-8 Encoding

Encode a Unicode codepoint (u21) into UTF-8 bytes:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    // Encode a codepoint to UTF-8
    const codepoint: u21 = 0x1F600; // 😀
    var buf: [4]u8 = undefined;
    const len = std.unicode.utf8Encode(codepoint, &buf) catch unreachable;

    try stdout.print("Codepoint U+{X} encodes to {} UTF-8 bytes:", .{codepoint, len});
    for (buf[0..len]) |byte| {
        try stdout.print(" 0x{X:0>2}", .{byte});
    }
    try stdout.print("\n");

    // Encode multiple codepoints into a buffer
    const codepoints = [_]u21{ 'H', 'e', 'l', 'l', 'o', 0x1F30D }; // 🌍
    const encoded = try allocator.alloc(u8, codepoints.len * 4); // max 4 bytes each
    defer allocator.free(encoded);

    var offset: usize = 0;
    for (codepoints) |cp| {
        const n = try std.unicode.utf8Encode(cp, encoded[offset..]);
        offset += n;
    }
    const final_str = encoded[0..offset];
    try stdout.print("Encoded string: {s} ({} bytes for {} codepoints)\n", .{
        final_str, offset, codepoints.len,
    });
}
```

### UTF-8 Decoding

Decode UTF-8 bytes back to codepoints:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const bytes = "Hello" ++ &[_]u8{ 0xE2, 0x9C, 0x93 }; // "Hello✓"

    var i: usize = 0;
    while (i < bytes.len) {
        const cp = std.unicode.utf8Decode(bytes[i..]) catch |err| {
            try stdout.print("Invalid UTF-8 at byte {}: {}\n", .{i, err});
            break;
        };
        try stdout.print("Byte {} -> U+{X:0>4} ({c})\n", .{i, cp, cp});
        i += std.unicode.utf8CodepointSequenceLength(cp) catch unreachable;
    }
}
```

### Validation

Validate that a byte sequence is well-formed UTF-8:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const valid = "Hello, 世界!";
    const invalid = &[_]u8{ 0xC3, 0x28 }; // Invalid continuation byte

    try stdout.print("Valid UTF-8:   {} ({s})\n", .{
        std.unicode.utf8ValidateSlice(valid),
        valid,
    });

    try stdout.print("Invalid UTF-8: {} ({any})\n", .{
        std.unicode.utf8ValidateSlice(invalid),
        invalid,
    });
}
```

---

## Chapter 6: std.unicode — Codepoint Iteration (Utf8View)

`std.unicode.Utf8View` provides an iterator over the codepoints in a UTF-8 encoded string. This is the safe, idiomatic way to iterate over Unicode text.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const text = "Héllo, 世界! 🌍";

    try stdout.print("Text: {s}\n", .{text});
    try stdout.print("Byte length: {}\n", .{text.len});
    try stdout.print("Codepoints:\n", .{});

    var view = std.unicode.Utf8View.init(text);
    var it = view.iterator();
    var count: usize = 0;
    while (it.nextCodepoint()) |cp| {
        count += 1;
        try stdout.print("  [{:>2}] U+{X:0>4}", .{count, cp});
        if (cp < 0x7F) {
            try stdout.print(" ('{c}')", .{cp});
        }
        try stdout.print("\n", .{});
    }

    try stdout.print("Total codepoints: {}\n", .{count});
}
```

### Counting Codepoints Without Iteration

For a quick count without full iteration, use `utf8CountCodepoints`:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const text = "abc世xyz界";
    const count = std.unicode.utf8CountCodepoints(text);
    try stdout.print("'{s}' has {} codepoints in {} bytes\n", .{text, count, text.len});
}
```

### Reverse Iteration

`Utf8View` also supports reverse iteration:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const text = "ABC世";
    try stdout.print("Reverse: ", .{});

    var view = std.unicode.Utf8View.init(text);
    var it = view.iterator();
    // Collect codepoints
    var codepoints: [10]u21 = undefined;
    var len: usize = 0;
    while (it.nextCodepoint()) |cp| {
        codepoints[len] = cp;
        len += 1;
    }

    // Print in reverse
    var i: usize = len;
    while (i > 0) {
        i -= 1;
        try stdout.print("{c}", .{codepoints[i]});
    }
    try stdout.print("\n");
}
```

---

## Chapter 7: std.unicode — Case Folding and Normalization

### Case Folding

Case folding converts text to a canonical form for case-insensitive comparison. It differs from lowercasing — for example, the German "ß" folds to "ss".

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // UTF-8 aware case folding for case-insensitive comparison
    const a = "HELLO";
    const b = "hello";

    // For ASCII-only, std.ascii.eqlIgnoreCase is faster
    try stdout.print("ASCII eqIgnoreCase: {}\n", .{std.ascii.eqlIgnoreCase(a, b)});

    // For Unicode, you need to iterate codepoints and compare folded forms
    const upper = "CAFÉ";
    const lower = "café";

    var view1 = std.unicode.Utf8View.init(upper);
    var view2 = std.unicode.Utf8View.init(lower);
    var it1 = view1.iterator();
    var it2 = view2.iterator();

    var equal = true;
    while (true) {
        const cp1 = it1.nextCodepoint();
        const cp2 = it2.nextCodepoint();

        if (cp1 == null and cp2 == null) break;
        if (cp1 == null or cp2 == null) {
            equal = false;
            break;
        }

        // Fold both codepoints and compare
        if (std.unicode.toLower(cp1.?) != std.unicode.toLower(cp2.?)) {
            equal = false;
            break;
        }
    }

    try stdout.print("Unicode case-insensitive '{s}' == '{s}': {}\n", .{upper, lower, equal});
}
```

### Codepoint Properties

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const codepoints = [_]u21{ 'A', 'z', '0', ' ', 0x1F600, 0x4E16 }; // 😀 世

    for (codepoints) |cp| {
        const width = std.unicode.utf8CodepointSequenceLength(cp) catch 1;
        try stdout.print("U+{X:0>4}  width={} bytes", .{cp, width});

        // Check if it's ASCII
        if (cp <= 0x7F) {
            try stdout.print("  ASCII", .{});
        }
        if (std.ascii.isAlpha(@intCast(cp))) {
            try stdout.print("  alpha", .{});
        }

        try stdout.print("\n", .{});
    }
}
```

---

## Chapter 8: Practical — Parsing ASCII Protocols

Many internet protocols are ASCII-based. Here's a minimal HTTP header parser using `std.ascii`:

```zig
const std = @import("std");

const Header = struct {
    name: []const u8,
    value: []const u8,
};

fn parseHeaders(allocator: std.mem.Allocator, raw: []const u8) !std.ArrayList(Header) {
    var headers = std.ArrayList(Header).init(allocator);
    errdefer headers.deinit();

    var lines = std.mem.splitSequence(u8, raw, "\r\n");
    while (lines.next()) |line| {
        if (line.len == 0) break; // End of headers

        // Skip status/request line (no colon on first line typically, or handle it)
        if (std.mem.indexOfScalar(u8, line, ':')) |colon_pos| {
            const name = std.mem.trim(u8, line[0..colon_pos], " ");
            const value = std.mem.trim(u8, line[colon_pos + 1 ..], " ");
            try headers.append(.{ .name = name, .value = value });
        }
    }

    return headers;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    const raw_response =
        "HTTP/1.1 200 OK\r\n" ++
        "Content-Type: text/html; charset=utf-8\r\n" ++
        "Content-Length: 42\r\n" ++
        "Server: zig/0.16\r\n" ++
        "\r\n";

    var headers = try parseHeaders(allocator, raw_response);
    defer headers.deinit();

    try stdout.print("=== Parsed HTTP Headers ===\n", .{});
    for (headers.items) |h| {
        // Case-insensitive lookup using std.ascii
        try stdout.print("  {s}: {s}\n", .{h.name, h.value});

        if (std.ascii.eqlIgnoreCase(h.name, "content-type")) {
            try stdout.print("    -> Detected content type: {s}\n", .{h.value});
        }
    }
}
```

### Parsing a Key=Value Config File

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const config =
        "host=localhost\n" ++
        "port=8080\n" ++
        "debug=true\n" ++
        "max_connections=100\n" ++
        "# This is a comment\n" ++
        "timeout=30s\n";

    try stdout.print("=== Config Parser ===\n", .{});
    var lines = std.mem.splitSequence(u8, config, "\n");
    while (lines.next()) |line| {
        const trimmed = std.mem.trim(u8, line, " \t");
        if (trimmed.len == 0) continue; // empty line
        if (trimmed[0] == '#') continue; // comment

        if (std.mem.indexOfScalar(u8, trimmed, '=')) |eq_pos| {
            const key = std.mem.trim(u8, trimmed[0..eq_pos], " ");
            const value = std.mem.trim(u8, trimmed[eq_pos + 1 ..], " ");

            // Validate key: must be alphanumeric + underscore
            var valid_key = true;
            for (key) |ch| {
                if (!std.ascii.isAlphanumeric(ch) and ch != '_' and ch != '-') {
                    valid_key = false;
                    break;
                }
            }

            if (valid_key) {
                try stdout.print("  {s} = {s}\n", .{key, value});
            } else {
                try stdout.print("  [INVALID KEY] {s} = {s}\n", .{key, value});
            }
        }
    }
}
```

---

## Chapter 9: Project — A Custom String Sanitizer / Slug Generator

This project builds a slug generator that converts arbitrary text (including Unicode) into URL-safe ASCII slugs. It demonstrates both `std.ascii` and `std.unicode` in a practical, real-world context.

### src/main.zig

```zig
const std = @import("std");

fn codepointToSlugChar(cp: u21, out: *std.ArrayList(u8)) !void {
    if (cp <= 0x7F) {
        // ASCII range — keep alphanumeric, convert space to dash
        if (std.ascii.isAlphanumeric(cp)) {
            try out.append(std.ascii.toLower(@intCast(cp)));
        } else if (cp == ' ' or cp == '-' or cp == '_') {
            try out.append('-');
        }
        // Skip other ASCII punctuation
    } else {
        // Unicode — try to transliterate common characters
        const translit = switch (cp) {
            'ä', 'á', 'à', 'â', 'ã' => "a",
            'ë', 'é', 'è', 'ê' => "e",
            'ï', 'í', 'ì', 'î' => "i",
            'ö', 'ó', 'ò', 'ô', 'õ' => "o",
            'ü', 'ú', 'ù', 'û' => "u",
            'ñ' => "n",
            'ç' => "c",
            'ß' => "ss",
            'æ' => "ae",
            'œ' => "oe",
            'ð' => "d",
            'þ' => "th",
            else => null,
        };

        if (translit) |t| {
            for (t) |ch| {
                try out.append(ch);
            }
        }
        // All other Unicode codepoints are dropped (not included in slug)
    }
}

fn generateSlug(allocator: std.mem.Allocator, input: []const u8, max_len: usize) ![]u8 {
    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    var view = std.unicode.Utf8View.init(input);
    var it = view.iterator();
    var last_was_dash = false;

    while (it.nextCodepoint()) |cp| {
        const prev_len = buf.items.len;

        // Check if we would add a dash
        if (cp <= 0x7F and (cp == ' ' or cp == '-' or cp == '_')) {
            if (!last_was_dash and buf.items.len > 0) {
                try buf.append('-');
                last_was_dash = true;
            }
        } else {
            try codepointToSlugChar(cp, &buf);
            if (buf.items.len > prev_len) {
                last_was_dash = false;
            }
        }
    }

    // Trim trailing dash
    while (buf.items.len > 0 and buf.items[buf.items.len - 1] == '-') {
        _ = buf.pop();
    }

    // Trim leading dash
    while (buf.items.len > 0 and buf.items[0] == '-') {
        _ = buf.orderedRemove(0);
    }

    // Collapse multiple dashes
    var result = std.ArrayList(u8).init(allocator);
    defer result.deinit();

    var in_dash = false;
    for (buf.items) |ch| {
        if (ch == '-') {
            if (!in_dash) {
                try result.append(ch);
                in_dash = true;
            }
        } else {
            try result.append(ch);
            in_dash = false;
        }
    }

    // Apply max length
    const final_slice = if (result.items.len > max_len)
        result.items[0..max_len]
    else
        result.items;

    return allocator.dupe(u8, final_slice);
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    const test_cases = [_][]const u8{
        "Hello World",
        "Zig Programming Language 0.16!",
        "Café & Crème Brûlée",
        "ßpecial Chäräctërs ñ ð þ",
        "  Multiple   Spaces   and  ---Dashes---  ",
        "日本語 Japanese Text",
        "¡Hola! ¿Qué tal?",
        "API Reference Guide (v2.0)",
    };

    try stdout.print("=== Slug Generator Demo ===\n\n", .{});

    for (test_cases) |input| {
        const slug = try generateSlug(allocator, input, 60);
        defer allocator.free(slug);
        try stdout.print("Input:  \"{s}\"\n", .{input});
        try stdout.print("Slug:   \"{s}\"\n", .{slug});
        try stdout.print("Bytes:  {}  Len:   {}\n\n", .{slug.len, slug.len});
    }
}
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "slug-generator",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the slug generator");
    run_step.dependOn(&run_cmd.step);
}
```

### Expected Output

```
=== Slug Generator Demo ===

Input:  "Hello World"
Slug:   "hello-world"

Input:  "Café & Crème Brûlée"
Slug:   "cafe-creme-brulee"

Input:  "ßpecial Chäräctërs ñ ð þ"
Slug:   "sspecial-characters-n-d-th"

Input:  "¡Hola! ¿Qué tal?"
Slug:   "hola-que-tal"
```

### Summary

This book covered Zig's text processing capabilities across `std.ascii` and `std.unicode`. The key patterns are:

1. Use `std.ascii` for fast, single-byte operations on known-ASCII data.
2. Use `std.unicode.Utf8View` for iterating over Unicode codepoints.
3. Use `utf8Encode`/`utf8Decode` for converting between codepoints and bytes.
4. Always validate external input with `utf8ValidateSlice` before decoding.
5. Remember that `[]const u8.len` is bytes, not characters — use `utf8CountCodepoints` or iterate for the true count.