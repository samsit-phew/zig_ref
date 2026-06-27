# POSIX in Zig 0.16

A comprehensive guide to POSIX system programming in Zig 0.16, covering the major migration from mid-level wrappers to the new two-tier architecture: `std.posix.system` (raw syscalls) and `std.Io` (high-level abstractions).

---

## Chapter 1: Introduction to std.posix in 0.16

Zig 0.16 represents a fundamental restructuring of how POSIX interfaces are exposed. The standard library now follows a clear two-tier model:

1. **`std.Io`** — High-level, cross-platform I/O abstractions (files, directories, networking, child processes). This is what most applications should use.
2. **`std.posix.system`** — Raw syscall wrappers, one-to-one with the kernel syscall interface. Use this when you need maximum control or when `std.Io` doesn't expose the feature you need.

`std.posix` itself still exists as a namespace, but its mid-level convenience functions (the old `std.posix.open`, `std.posix.read`, `std.posix.write`, etc.) have been **removed**. You must now choose: go high with `std.Io` or go low with `std.posix.system`.

This book focuses on the low-level POSIX path via `std.posix.system`, which is essential for systems programming, network servers, and understanding how Zig interfaces with the operating system.

---

## Chapter 2: The 0.16 Migration — What Was Removed and Why

### What Changed

In Zig 0.15 and earlier, `std.posix` contained a large collection of mid-level wrappers:

```zig
// OLD (0.15 and earlier) - NO LONGER EXISTS
const fd = try std.posix.open("file.txt", .{ .mode = .read_only }, 0);
const bytes_read = try std.posix.read(fd, buffer);
try std.posix.write(fd, data);
try std.posix.close(fd);
```

In Zig 0.16, these mid-level functions were removed. The rationale:

- **Reduce surface area**: The mid-level layer was a leaky abstraction that duplicated functionality with `std.Io`.
- **Eliminate ambiguity**: Developers shouldn't have to choose between `std.os`, `std.posix`, and `std.Io` for the same operation.
- **Simplify maintenance**: Two well-defined layers (high and low) are easier to maintain than three overlapping ones.

### The New Paths

```zig
// HIGH-LEVEL (recommended for most code):
// Use std.Io for files, directories, sockets, etc.
const file = try std.Io.Dir.openFile(.cwd(), "file.txt", .{});
defer file.close();
const bytes_read = try file.read(&buffer);

// LOW-LEVEL (for raw syscall access):
// Use std.posix.system for direct kernel calls
const fd = std.posix.system.open("/path/to/file", std.posix.O.RDONLY, 0);
const n = std.posix.system.read(fd, &buffer);
_ = std.posix.system.close(fd);
```

### What Remains in std.posix

`std.posix` still contains:
- **Types**: `fd_t`, `mode_t`, `pid_t`, `sigset_t`, etc.
- **Constants**: `O.RDONLY`, `O.CREAT`, `O.WRONLY`, `PROT.READ`, `MAP.SHARED`, etc.
- **Structs**: `stat`, `sigaction`, `pollfd`, etc.
- **`std.posix.system`**: The actual syscall wrappers
- **Error mapping**: POSIX errno to Zig error set conversion

---

## Chapter 3: std.posix.system — Direct Syscall Access

`std.posix.system` provides functions that map directly to kernel syscalls. These are the lowest-level portable POSIX wrappers Zig offers.

### Open, Read, Write, Close

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Open a file using raw syscall
    const path = "/tmp/posix_test.txt";

    // Write some data
    const fd_write = std.posix.system.open(
        path,
        std.posix.O.WRONLY | std.posix.O.CREAT | std.posix.O.TRUNC,
        0o644,
    );
    if (fd_write < 0) {
        const err = std.posix.errno(fd_write);
        try writer.print("open failed: {s}\n", .{@tagName(err)});
        return;
    }
    defer _ = std.posix.system.close(fd_write);

    const msg = "Hello from std.posix.system!\n";
    const written = std.posix.system.write(fd_write, msg);
    if (written < 0) {
        const err = std.posix.errno(written);
        try writer.print("write failed: {s}\n", .{@tagName(err)});
        return;
    }
    try writer.print("Wrote {d} bytes\n", .{written});

    // Read it back
    const fd_read = std.posix.system.open(path, std.posix.O.RDONLY, 0);
    if (fd_read < 0) return error.OpenFailed;
    defer _ = std.posix.system.close(fd_read);

    var buf: [256]u8 = undefined;
    const n = std.posix.system.read(fd_read, &buf);
    if (n < 0) return error.ReadFailed;
    try writer.print("Read back: {s}", .{buf[0..@intCast(n)]});
}
```

**Important**: In `std.posix.system`, errors are returned as negative values (following the C/POSIX convention), not as Zig error unions. Use `std.posix.errno(ret)` to convert a negative return value into a Zig errno enum.

### lseek

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const path = "/tmp/seek_test.txt";

    // Create file with content
    const fd = std.posix.system.open(
        path,
        std.posix.O.WRONLY | std.posix.O.CREAT | std.posix.O.TRUNC,
        0o644,
    );
    if (fd < 0) return error.OpenFailed;
    defer _ = std.posix.system.close(fd);

    _ = std.posix.system.write(fd, "ABCDEFGHIJ");

    // Seek to offset 3
    const off = std.posix.system.lseek(fd, 3, std.posix.SEEK.SET);
    if (off < 0) return error.SeekFailed;
    try writer.print("Seeked to offset: {d}\n", .{off});

    // Overwrite from offset 3
    _ = std.posix.system.write(fd, "XYZ");

    // Seek to beginning and read
    _ = std.posix.system.lseek(fd, 0, std.posix.SEEK.SET);
    var buf: [16]u8 = undefined;
    const n = std.posix.system.read(fd, &buf);
    if (n < 0) return error.ReadFailed;

    try writer.print("Result: {s}\n", .{buf[0..@intCast(n)]});
    // Output: Result: ABCXYZGHIJ
}
```

---

## Chapter 4: Using std.Io Instead

For most applications, the high-level `std.Io` API is preferred. It provides Zig-native error handling, automatic resource management, and a cleaner interface.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Open a file using the high-level std.Io API
    const dir = std.Io.Dir.cwd();
    const file = try dir.openFile("test_io.txt", .{ .mode = .read_write });
    defer file.close();

    // Write
    const msg = "Written via std.Io\n";
    const written = try file.writeAll(msg);
    try writer.print("Wrote {d} bytes via std.Io\n", .{written});

    // Seek and read
    try file.seekTo(0);
    var buf: [256]u8 = undefined;
    const n = try file.read(&buf);
    try writer.print("Read: {s}", .{buf[0..n]});
}
```

The `std.Io` layer handles:
- Proper error sets (no manual errno conversion)
- Resource cleanup via `defer`
- Cross-platform abstractions
- Buffered I/O where appropriate

---

## Chapter 5: File Descriptors — open, close, read, write, lseek

This chapter provides a deeper reference for the core file descriptor operations via `std.posix.system`.

### Error Handling Pattern

```zig
const std = @import("std");

fn posixOpen(path: []const u8, flags: u32, mode: std.posix.mode_t) !std.posix.fd_t {
    const fd = std.posix.system.open(
        @ptrCast(path.ptr),
        flags,
        @intCast(mode),
    );
    if (fd < 0) {
        return std.posix.errno(fd);
    }
    return @intCast(fd);
}

fn posixWrite(fd: std.posix.fd_t, buf: []const u8) !usize {
    const n = std.posix.system.write(fd, buf);
    if (n < 0) return std.posix.errno(n);
    return @intCast(n);
}

fn posixRead(fd: std.posix.fd_t, buf: []u8) !usize {
    const n = std.posix.system.read(fd, buf);
    if (n < 0) return std.posix.errno(n);
    return @intCast(n);
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const fd = try posixOpen("/tmp/fd_demo.txt", std.posix.O.WRONLY | std.posix.O.CREAT | std.posix.O.TRUNC, 0o644);
    defer _ = std.posix.system.close(fd);

    _ = try posixWrite(fd, "Line 1\n");
    _ = try posixWrite(fd, "Line 2\n");
    _ = try posixWrite(fd, "Line 3\n");
    try writer.print("Wrote 3 lines to /tmp/fd_demo.txt\n", .{});
}
```

### File Descriptor Flags

```zig
// Open flags (combinable with |)
std.posix.O.RDONLY     // Read only
std.posix.O.WRONLY     // Write only
std.posix.O.RDWR       // Read and write
std.posix.O.CREAT      // Create if doesn't exist
std.posix.O.TRUNC      // Truncate to zero length
std.posix.O.APPEND     // Append mode
std.posix.O.NONBLOCK   // Non-blocking I/O
std.posix.O.DIRECT     // Direct I/O (bypass page cache)
std.posix.O.SYNC       // Synchronous writes

// Seek constants
std.posix.SEEK.SET     // Seek from beginning
std.posix.SEEK.CUR     // Seek from current position
std.posix.SEEK.END     // Seek from end
```

---

## Chapter 6: Signals — sigaction, kill, Signal Handling

```zig
const std = @import("std");

var signal_count: usize = 0;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const pid = std.posix.system.getpid();
    try writer.print("PID: {d}\n", .{pid});
    try writer.print("Send SIGINT (Ctrl+C) or SIGUSR1 to this process.\n", .{});
    try writer.print("Run: kill -USR1 {d}\n", .{pid});
    try writer.print("Press Ctrl+\\ (SIGQUIT) to exit.\n", .{});

    // Set up SIGINT handler
    var sa_int = std.mem.zeroInit(std.posix.Sigaction, .{});
    sa_int.handler.sigaction = handleSigint;
    sa_int.flags = std.posix.SA.RESTART;
    _ = std.posix.system.sigaction(std.posix.SIG.INT, &sa_int, null);

    // Set up SIGUSR1 handler
    var sa_usr = std.mem.zeroInit(std.posix.Sigaction, .{});
    sa_usr.handler.sigaction = handleSigusr1;
    sa_usr.flags = std.posix.SA.RESTART;
    _ = std.posix.system.sigaction(std.posix.SIG.USR1, &sa_usr, null);

    // Ignore SIGQUIT for clean exit demo — or handle it
    var sa_quit = std.mem.zeroInit(std.posix.Sigaction, .{});
    sa_quit.handler.sigaction = handleSigquit;
    _ = std.posix.system.sigaction(std.posix.SIG.QUIT, &sa_quit, null);

    // Loop until SIGQUIT
    while (true) {
        _ = std.posix.system.pause();
    }
}

fn handleSigint(sig: c_int) callconv(.c) void {
    _ = sig;
    const stdout = std.io.getStdOut().writer();
    stdout.print("\nCaught SIGINT! Count so far: {d}\n", .{signal_count}) catch {};
    signal_count += 1;
}

fn handleSigusr1(sig: c_int) callconv(.c) void {
    _ = sig;
    const stdout = std.io.getStdOut().writer();
    stdout.print("Caught SIGUSR1!\n", .{}) catch {};
    signal_count += 1;
}

fn handleSigquit(sig: c_int) callconv(.c) void {
    _ = sig;
    const stdout = std.io.getStdOut().writer();
    stdout.print("\nCaught SIGQUIT. Total signals handled: {d}. Exiting.\n", .{signal_count}) catch {};
    _ = std.posix.system.exit(0);
}
```

### Sending Signals

```zig
// Send signal to another process
_ = std.posix.system.kill(target_pid, std.posix.SIG.USR1);

// Send signal to process group
_ = std.posix.system.kill(0, std.posix.SIG.TERM); // send to own process group
```

---

## Chapter 7: Pipes and FIFOs

### Anonymous Pipes

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    var pipe_fds: [2]std.posix.fd_t = undefined;

    const result = std.posix.system.pipe(&pipe_fds);
    if (result < 0) {
        const err = std.posix.errno(result);
        try writer.print("pipe failed: {s}\n", .{@tagName(err)});
        return;
    }

    const read_fd = pipe_fds[0];
    const write_fd = pipe_fds[1];

    try writer.print("pipe created: read_fd={d}, write_fd={d}\n", .{ read_fd, write_fd });

    // Write to pipe
    const msg = "Hello through the pipe!";
    const written = std.posix.system.write(write_fd, msg);
    if (written < 0) return error.WriteFailed;
    try writer.print("Wrote {d} bytes to pipe\n", .{written});

    // Close write end
    _ = std.posix.system.close(write_fd);

    // Read from pipe
    var buf: [256]u8 = undefined;
    const n = std.posix.system.read(read_fd, &buf);
    if (n < 0) return error.ReadFailed;

    try writer.print("Read from pipe: {s}\n", .{buf[0..@intCast(n)]});
    _ = std.posix.system.close(read_fd);
}
```

### Named Pipes (FIFOs)

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const fifo_path = "/tmp/my_fifo";

    // Create named pipe (FIFO)
    const result = std.posix.system.mkfifo(fifo_path, 0o666);
    if (result < 0) {
        const err = std.posix.errno(result);
        if (err != .EXIST) {
            try writer.print("mkfifo failed: {s}\n", .{@tagName(err)});
            return;
        }
        try writer.print("FIFO already exists, reusing\n", .{});
    } else {
        try writer.print("Created FIFO at {s}\n", .{fifo_path});
    }

    try writer.print("Writer: Open the FIFO for writing (blocks until reader connects)...\n", .{});
    const fd = std.posix.system.open(fifo_path, std.posix.O.WRONLY);
    if (fd < 0) {
        const err = std.posix.errno(fd);
        try writer.print("open fifo failed: {s}\n", .{@tagName(err)});
        return;
    }
    defer _ = std.posix.system.close(fd);

    _ = std.posix.system.write(fd, "Message via FIFO\n");
    try writer.print("Sent message via FIFO\n", .{});
}
```

---

## Chapter 8: Memory Mapping — mmap, munmap

```zig
const std = @import("std");
const posix = std.posix;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const page_size = std.mem.writableAlignedSlice(@as([]u8, undefined), std.heap.page_size_min).len;
    // Use the system page size
    const size: usize = 4096;

    // Anonymous mmap (shared between related processes)
    const ptr = std.posix.system.mmap(
        null, // let kernel choose address
        size,
        posix.PROT.READ | posix.PROT.WRITE,
        posix.MAP.ANONYMOUS | posix.MAP.SHARED,
        -1,  // no file descriptor for anonymous mapping
        0,
    );

    if (ptr == std.posix.system.MAP_FAILED) return error.MmapFailed;

    const mapped = @as([*]u8, @ptrCast(@alignCast(ptr)))[0..size];

    // Write to mapped memory
    const msg = "Data in shared memory region";
    @memcpy(mapped, msg.*);

    try writer.print("Wrote {d} bytes to mmap region at {*}\n", .{ msg.len, ptr });
    try writer.print("Content: {s}\n", .{mapped[0..msg.len]});

    // Unmap
    const unmap_result = std.posix.system.munmap(ptr, size);
    if (unmap_result != 0) return error.MunmapFailed;
    try writer.print("Unmapped {d} bytes\n", .{size});
}

// Helper for aligned allocation using mmap
fn mmapAlloc(size: usize) ![]u8 {
    const aligned_size = std.mem.alignForward(usize, size, std.heap.page_size_min);
    const ptr = std.posix.system.mmap(
        null,
        aligned_size,
        std.posix.PROT.READ | std.posix.PROT.WRITE,
        std.posix.MAP.ANONYMOUS | std.posix.MAP.PRIVATE,
        -1,
        0,
    );
    if (ptr == std.posix.system.MAP_FAILED) return error.OutOfMemory;
    return @as([*]u8, @ptrCast(@alignCast(ptr)))[0..aligned_size];
}
```

### File-Backed mmap

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const path = "/tmp/mmap_file.txt";

    // Create and write file
    const fd = std.posix.system.open(
        path,
        std.posix.O.WRONLY | std.posix.O.CREAT | std.posix.O.TRUNC,
        0o644,
    );
    if (fd < 0) return error.OpenFailed;
    const data = "Hello, memory-mapped world!";
    _ = std.posix.system.write(fd, data);
    _ = std.posix.system.close(fd);

    // Open for reading and mmap
    const fd_read = std.posix.system.open(path, std.posix.O.RDONLY, 0);
    if (fd_read < 0) return error.OpenFailed;
    defer _ = std.posix.system.close(fd_read);

    const size = @as(usize, @intCast(data.len));
    const ptr = std.posix.system.mmap(
        null,
        size,
        std.posix.PROT.READ,
        std.posix.MAP.PRIVATE,
        fd_read,
        0,
    );
    if (ptr == std.posix.system.MAP_FAILED) return error.MmapFailed;
    defer _ = std.posix.system.munmap(ptr, size);

    const mapped = @as([*]const u8, @ptrCast(@alignCast(ptr)))[0..size];
    try writer.print("File contents via mmap: {s}\n", .{mapped});
}
```

---

## Chapter 9: Select/Poll for I/O Multiplexing

### Using poll

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const stdin_fd = std.posix.STDIN_FILENO;

    try writer.print("Waiting for input on stdin (5 second timeout)...\n", .{});

    var pfds: [1]std.posix.pollfd = .{
        .{
            .fd = stdin_fd,
            .events = std.posix.POLL.IN,
            .revents = 0,
        },
    };

    // Poll with 5 second timeout
    const timeout_ms: c_int = 5000;
    const ready = std.posix.system.poll(&pfds, timeout_ms);

    if (ready < 0) {
        const err = std.posix.errno(ready);
        try writer.print("poll error: {s}\n", .{@tagName(err)});
        return;
    } else if (ready == 0) {
        try writer.print("Timeout! No input within 5 seconds.\n", .{});
    } else {
        if (pfds[0].revents & std.posix.POLL.IN != 0) {
            var buf: [256]u8 = undefined;
            const n = std.posix.system.read(stdin_fd, &buf);
            if (n > 0) {
                try writer.print("Read: {s}", .{buf[0..@intCast(n)]});
            }
        }
    }
}
```

---

## Chapter 10: Sockets (Low-Level)

In Zig 0.16, all networking has been migrated to `std.Io.net`. However, for raw POSIX socket access, you can still use `std.posix.system` directly:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Create a TCP socket using raw POSIX
    const sockfd = std.posix.system.socket(
        std.posix.AF.INET,
        std.posix.SOCK.STREAM,
        0,
    );
    if (sockfd < 0) {
        try writer.print("socket failed: {s}\n", .{@tagName(std.posix.errno(sockfd))});
        return;
    }
    defer _ = std.posix.system.close(sockfd);

    try writer.print("Created socket fd={d}\n", .{sockfd});

    // Set SO_REUSEADDR
    var optval: u32 = 1;
    _ = std.posix.system.setsockopt(
        sockfd,
        std.posix.SOL.SOCKET,
        std.posix.SO.REUSEADDR,
        std.mem.asBytes(&optval),
        @intCast(@sizeOf(u32)),
    );

    // Bind to port 8080
    const addr = std.posix.sockaddr.in{
        .family = std.posix.AF.INET,
        .port = std.mem.nativeToBig(u16, 8080),
        .addr = std.mem.nativeToBig(u32, 0), // INADDR_ANY
    };

    const bind_result = std.posix.system.bind(
        sockfd,
        @ptrCast(&addr),
        @intCast(@sizeOf(std.posix.sockaddr.in)),
    );
    if (bind_result < 0) {
        try writer.print("bind failed: {s}\n", .{@tagName(std.posix.errno(bind_result))});
        return;
    }
    try writer.print("Bound to port 8080\n", .{});

    // Listen
    const listen_result = std.posix.system.listen(sockfd, 128);
    if (listen_result < 0) return error.ListenFailed;
    try writer.print("Listening for connections...\n", .{});

    // Accept one connection
    var client_addr: std.posix.sockaddr.in = undefined;
    var addr_len: std.posix.socklen_t = @intCast(@sizeOf(std.posix.sockaddr.in));
    const client_fd = std.posix.system.accept(
        sockfd,
        @ptrCast(&client_addr),
        &addr_len,
        0,
    );
    if (client_fd < 0) return error.AcceptFailed;

    const client_port = std.mem.bigToNative(u16, client_addr.port);
    try writer.print("Accepted connection from port {d}\n", .{client_port});
    defer _ = std.posix.system.close(client_fd);

    // Echo loop
    var buf: [1024]u8 = undefined;
    while (true) {
        const n = std.posix.system.read(client_fd, &buf);
        if (n <= 0) break;
        _ = std.posix.system.write(client_fd, buf[0..@intCast(n)]);
        try writer.print("Echoed {d} bytes\n", .{n});
    }

    try writer.print("Connection closed\n", .{});
}
```

> **Note**: For production networking code in Zig 0.16, prefer `std.Io.net` which provides a safer, higher-level API with proper Zig error handling.

---

## Chapter 11: Project — Multi-Client Echo Server Using Raw POSIX Sockets

This project builds a multi-client echo server using `poll` for I/O multiplexing and raw POSIX socket calls.

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "echo-server",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    b.step("run", "Run the echo server").dependOn(&run_cmd.step);
}
```

### src/main.zig

```zig
const std = @import("std");
const posix = std.posix;
const system = posix.system;

const MAX_CLIENTS = 64;
const BUFFER_SIZE = 4096;

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Create TCP socket
    const server_fd = system.socket(posix.AF.INET, posix.SOCK.STREAM, 0);
    if (server_fd < 0) return error.SocketFailed;
    defer _ = system.close(server_fd);

    // Set SO_REUSEADDR
    var reuse: u32 = 1;
    _ = system.setsockopt(server_fd, posix.SOL.SOCKET, posix.SO.REUSEADDR,
        std.mem.asBytes(&reuse), @intCast(@sizeOf(u32)));

    // Bind
    const addr = posix.sockaddr.in{
        .family = posix.AF.INET,
        .port = std.mem.nativeToBig(u16, 8080),
        .addr = std.mem.nativeToBig(u32, 0),
    };
    if (system.bind(server_fd, @ptrCast(&addr),
        @intCast(@sizeOf(posix.sockaddr.in))) < 0)
        return error.BindFailed;

    // Listen
    if (system.listen(server_fd, 128) < 0) return error.ListenFailed;
    try writer.print("Echo server listening on port 8080\n", .{});
    try writer.print("Connect with: nc localhost 8080\n", .{});

    // Poll array: index 0 = server socket, rest = clients
    var poll_fds: [MAX_CLIENTS + 1]posix.pollfd = undefined;

    // Server socket
    poll_fds[0] = .{
        .fd = server_fd,
        .events = posix.POLL.IN,
        .revents = 0,
    };

    var client_count: usize = 0;

    // Client info for logging
    var client_addrs: [MAX_CLIENTS]posix.sockaddr.in = undefined;

    // Read buffers per client
    var client_buffers: [MAX_CLIENTS][BUFFER_SIZE]u8 = undefined;

    while (true) {
        const ready = system.poll(
            poll_fds[0 .. client_count + 1],
            -1, // block indefinitely
        );
        if (ready < 0) {
            const err = posix.errno(ready);
            if (err == .INTR) continue;
            try writer.print("poll error: {s}\n", .{@tagName(err)});
            continue;
        }

        // Check server socket for new connections
        if (poll_fds[0].revents & posix.POLL.IN != 0) {
            if (client_count >= MAX_CLIENTS) {
                try writer.print("Too many clients, rejecting\n", .{});
            } else {
                var client_addr: posix.sockaddr.in = undefined;
                var addr_len: posix.socklen_t = @intCast(@sizeOf(posix.sockaddr.in));
                const client_fd = system.accept(
                    server_fd,
                    @ptrCast(&client_addr),
                    &addr_len,
                    0,
                );
                if (client_fd >= 0) {
                    client_count += 1;
                    poll_fds[client_count] = .{
                        .fd = @intCast(client_fd),
                        .events = posix.POLL.IN,
                        .revents = 0,
                    };
                    client_addrs[client_count - 1] = client_addr;
                    const port = std.mem.bigToNative(u16, client_addr.port);
                    try writer.print("Client {d} connected from port {d} (fd={d})\n", .{
                        client_count, port, client_fd,
                    });
                }
            }
        }

        // Check all clients
        var i: usize = 1;
        while (i <= client_count) {
            if (poll_fds[i].revents & posix.POLL.IN != 0) {
                const n = system.read(poll_fds[i].fd, &client_buffers[i - 1]);
                if (n <= 0) {
                    // Client disconnected
                    const fd = poll_fds[i].fd;
                    try writer.print("Client disconnected (fd={d})\n", .{fd});
                    _ = system.close(fd);

                    // Remove from poll array by shifting
                    for (i..client_count) |j| {
                        poll_fds[j] = poll_fds[j + 1];
                        client_addrs[j - 1] = client_addrs[j];
                    }
                    client_count -= 1;
                    continue; // don't increment i since we shifted
                } else {
                    // Echo data back
                    const data = client_buffers[i - 1][0..@intCast(n)];
                    _ = system.write(poll_fds[i].fd, data);
                }
            } else if (poll_fds[i].revents & (posix.POLL.HUP | posix.POLL.ERR) != 0) {
                const fd = poll_fds[i].fd;
                try writer.print("Client error/hangup (fd={d})\n", .{fd});
                _ = system.close(fd);
                for (i..client_count) |j| {
                    poll_fds[j] = poll_fds[j + 1];
                    client_addrs[j - 1] = client_addrs[j];
                }
                client_count -= 1;
                continue;
            }
            i += 1;
        }
    }
}
```

### Testing the Server

```bash
# Terminal 1: Start server
$ zig build run
Echo server listening on port 8080
Connect with: nc localhost 8080

# Terminal 2: Connect client
$ nc localhost 8080
Hello, echo server!
Hello, echo server!

# Terminal 3: Another client
$ nc localhost 8080
Second client here
Second client here
```

---

## Summary

| Layer | Module | Use Case |
|---|---|---|
| **High-level** | `std.Io` | Files, directories, networking, child processes |
| **Low-level** | `std.posix.system` | Raw syscalls, maximum control |
| **Types/Constants** | `std.posix` | `fd_t`, `O.*`, `SIG.*`, `sockaddr.*`, etc. |
| **Error mapping** | `std.posix.errno()` | Convert negative syscall returns to Zig errors |

The key takeaway for Zig 0.16: **use `std.Io` for applications, `std.posix.system` for systems programming**. The removed mid-level layer forced a cleaner separation of concerns.