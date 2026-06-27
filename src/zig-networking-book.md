# Network Programming in Zig 0.16

## A Practical Guide to `std.Io.net` and Building Real-World Networked Applications

---

# Table of Contents

1. [Introduction](#chapter-1-introduction)
2. [The `std.Io` Revolution in Zig 0.16](#chapter-2-the-stdio-revolution-in-zig-016)
3. [IP Addresses and Endpoints](#chapter-3-ip-addresses-and-endpoints)
4. [TCP Clients &mdash; Connecting to the World](#chapter-4-tcp-clients--connecting-to-the-world)
5. [TCP Servers &mdash; Accepting Connections](#chapter-5-tcp-servers--accepting-connections)
6. [Stream I/O &mdash; Reading and Writing Data](#chapter-6-stream-io--reading-and-writing-data)
7. [UDP Networking &mdash; Connectionless Communication](#chapter-7-udp-networking--connectionless-communication)
8. [HTTP Client &mdash; Making Web Requests](#chapter-8-http-client--making-web-requests)
9. [HTTP Server &mdash; Building Web Services](#chapter-9-http-server--building-web-services)
10. [Concurrency and I/O Backends](#chapter-10-concurrency-and-io-backends)
11. [Project 1: Static File Server](#project-1-static-file-server)
12. [Project 2: REST API Server](#project-2-rest-api-server)
13. [Project 3: TCP Chat Room](#project-3-tcp-chat-room)
14. [Project 4: Netcat Clone](#project-4-netcat-clone)

---

# Chapter 1: Introduction

## Why Zig for Networking?

Zig is a systems programming language that gives you low-level control over every byte of memory, every system call, and every network operation&mdash;without sacrificing the convenience of a modern standard library. When it comes to network programming, Zig sits in a unique sweet spot: it offers the raw power of C (direct socket access, zero-cost abstractions, no hidden allocations) while providing a growing, well-designed standard library that handles the tedious parts of networking for you.

Zig's approach to error handling through its error union type system (`!T`) means you are forced to handle every failure mode&mdash;a closed connection, a refused bind, a broken pipe&mdash;at compile time. There are no unchecked exceptions hiding in your call stack. Combined with Zig's `defer` for resource cleanup and its comptime for compile-time code generation, network programs written in Zig tend to be remarkably robust and efficient.

## What Changed in Zig 0.16?

Zig 0.16 brought a massive overhaul to all I/O, including networking. The headline change is that **all `net` APIs have been migrated into `std.Io`**. In previous versions, you might have reached for `std.net.Address.parseIp4()` or `std.net.Server`. In 0.16, the canonical path is `std.Io.net.IpAddress`, and every network operation now takes an `Io` instance as its first parameter.

This migration is not just a namespace reshuffle. It is the foundation for Zig's ambitious plan to make all I/O operations *portable across different concurrency backends*: threaded blocking I/O, evented I/O via epoll/kqueue/IOCP, and even io_uring on Linux. Your networking code can be written once and run on any of these backends without modification. The trade-off is that `Io.Evented` (the evented backend) does not yet support networking in 0.16, so you will use `Io.Threaded` for all the projects in this book. But the API you learn here will carry forward unchanged when evented networking lands.

Additionally, Zig 0.16 eliminated the `ws2_32.dll` dependency on Windows. All networking on Windows now uses direct AFD (Ancillary Function Driver) access, which fixes cancellation bugs, improves performance by avoiding unnecessary hash tables, and makes Zig programs leaner than those built with languages that still go through the Winsock layer.

## Who This Book Is For

This book assumes you have basic familiarity with Zig syntax&mdash;you understand `const`, `var`, `fn`, `!void`, `try`, `defer`, and the general idea of comptime. If you have written a "Hello, World!" in Zig and maybe a few functions, you are ready. You do not need prior networking experience; each concept is explained from the ground up, though readers who have used sockets in C, Go, or Python will find many familiar ideas.

## How to Follow Along

Every code sample in this book targets **Zig 0.16.0 stable**. Install it from [ziglang.org/download](https://ziglang.org/download/). Build and run any standalone example with:

```bash
zig build-exe example.zig
./example
```

Or use the simpler run-and-forget workflow:

```bash
zig run example.zig
```

The four projects at the end of the book each include a `build.zig` for proper dependency management and a recommended directory structure.

---

# Chapter 2: The `std.Io` Revolution in Zig 0.16

## The `Io` Instance

The central abstraction in Zig 0.16's I/O story is the `Io` instance. Every I/O operation&mdash;reading a file, opening a socket, writing to a stream&mdash;goes through an `Io` parameter. This is not bureaucracy; it is the mechanism that lets Zig swap between threaded, evented, and future io_uring backends without changing your code.

You obtain an `Io` instance from `std.process.Init`, the new "juicy main" parameter that Zig 0.16 introduced:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const gpa = init.gpa;
    // ...
}
```

The `std.process.Init` struct provides:

| Field | Type | Description |
|-------|------|-------------|
| `io` | `*std.Io` | The I/O backend for all I/O operations |
| `gpa` | `std.mem.Allocator` | General-purpose allocator (thread-safe) |
| `arena` | `std.mem.ArenaAllocator` | Arena allocator for the main function scope |
| `minimal.args` | argument iterator | Command-line arguments (non-owning) |

## The "Juicy Main"

In Zig 0.16, the canonical main function signature is no longer `pub fn main() !void`. Instead, it takes a `std.process.Init` parameter that bundles together everything your program needs: an allocator, an I/O backend, and access to command-line arguments. Zig calls this the "juicy main" because it gives your entry point access to the full runtime context in a single, explicit parameter.

This pattern means you rarely need global state. If a library function needs to perform I/O, it receives the `Io` instance as a parameter. If it needs memory, it receives an `Allocator`. Everything is explicit, everything is testable, and there is no hidden global state to worry about.

## I/O Backends

Zig 0.16 ships with two I/O backends:

- **`Io.Threaded`** &mdash; The default. Operations block OS threads. Works everywhere. This is what you get from `init.io` unless you configure otherwise. All examples in this book use this backend.
- **`Io.Evented`** &mdash; Uses epoll (Linux), kqueue (macOS/BSD), or IOCP (Windows) for non-blocking I/O. In 0.16, this does not yet support networking, so it cannot be used for the programs in this book. It is mentioned here so you understand the design.

If you find yourself in a context where you do not have an `Io` instance (for example, in a quick test or a utility function), you can create a single-threaded fallback:

```zig
var threaded: std.Io.Threaded = .init_single_threaded;
const io = threaded.io();
```

This works as long as you do not need task-level concurrency. It is the `Io` equivalent of reaching for `page_allocator`&mdash;functional, but not ideal for production code. The application's `main` function should be the one that creates and distributes the `Io` instance throughout the program.

## Testing with `std.testing.io`

For unit tests that perform I/O, Zig provides `std.testing.io` (analogous to `std.testing.allocator`). Use it in your test functions to keep I/O operations self-contained and predictable:

```zig
const testing = std.testing;

test "tcp echo round-trip" {
    const io = testing.io;
    // Your I/O-dependent test code here
}
```

---

# Chapter 3: IP Addresses and Endpoints

## `std.Io.net.IpAddress`

Every network connection starts with an address. In Zig 0.16, IP addresses with ports are represented by `std.Io.net.IpAddress` (aliased as `net.IpAddress` when you write `const net = std.Io.net`).

### Parsing IP Addresses

There are three ways to construct an `IpAddress` from a string:

```zig
const std = @import("std");
const net = std.Io.net;

// IPv4 only
const addr_v4 = try net.IpAddress.parseIp4("127.0.0.1", 8080);

// IPv6 only
const addr_v6 = try net.IpAddress.parseIp6("::1", 8080);

// Auto-detect IPv4 or IPv6
const addr_any = try net.IpAddress.parse("127.0.0.1", 8080);
const addr_any6 = try net.IpAddress.parse("::1", 8080);
```

The `parse` function inspects the string and determines whether it is IPv4 or IPv6 automatically. Use `parseIp4` or `parseIp6` when you need to enforce a specific protocol version at compile time.

### Binding to All Interfaces

To make your server accessible from any network interface (not just localhost), bind to `0.0.0.0` for IPv4 or `::` for IPv6:

```zig
// Listen on all IPv4 interfaces
const public_addr = try net.IpAddress.parseIp4("0.0.0.0", 8080);

// Listen on all IPv6 interfaces (also accepts IPv4 on dual-stack systems)
const public_addr_v6 = try net.IpAddress.parseIp6("::", 8080);
```

### Displaying Addresses

`IpAddress` implements the `std.fmt` format specifier, so you can print it directly:

```zig
std.log.info("Listening on {f}", .{addr});
// Output: Listening on 127.0.0.1:8080
```

### Reserved Addresses Reference

| Address | Meaning | Use Case |
|---------|---------|----------|
| `127.0.0.1` / `::1` | Loopback | Local-only development and testing |
| `0.0.0.0` / `::` | All interfaces | Public servers |
| `255.255.255.255` | Broadcast | UDP broadcast messages |

---

# Chapter 4: TCP Clients &mdash; Connecting to the World

TCP (Transmission Control Protocol) is the workhorse of the internet. It provides a reliable, ordered, error-checked stream of bytes between two endpoints. Before you can send or receive data over TCP, you must establish a connection.

## The `connect` Method

In Zig 0.16, connecting to a TCP server is a single method call on an `IpAddress`:

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const server = try net.IpAddress.parseIp4("127.0.0.1", 8080);

    // Connect to the server, returns a Stream
    const stream = try server.connect(io, .{ .mode = .stream });
    defer stream.close(io);

    std.log.info("Connected to {f}", .{server});
}
```

The `connect` method takes the `io` instance and a `ConnectOptions` struct. The `.mode = .stream` option specifies TCP (stream-oriented). The method returns a `net.Stream` on success or an error on failure (connection refused, timeout, DNS failure, etc.).

## A Complete TCP Client

Here is a full example that connects to a server, sends a message, and reads the response:

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    var args = init.minimal.args.iterate();
    _ = args.skip(); // skip program name

    const port_str = args.next() orelse {
        std.log.err("Usage: {s} <port>", .{"tcp_client"});
        return error.NoPort;
    };

    const port = try std.fmt.parseInt(u16, port_str, 10);
    const server = try net.IpAddress.parseIp4("127.0.0.1", port);

    // Connect
    const stream = try server.connect(io, .{ .mode = .stream });
    defer stream.close(io);
    std.log.info("Connected to {f}", .{server});

    // Send data
    var send_buf: [1024]u8 = undefined;
    var writer = stream.writer(io, &send_buf);
    try writer.interface.writeAll("Hello, Zig Server!");
    try writer.interface.flush();
    std.log.info("Sent message to server", .{});

    // Read response
    var recv_buf: [1024]u8 = undefined;
    var reader = stream.reader(io, &recv_buf);
    const bytes_read = try reader.interface.read(&recv_buf);
    std.log.info("Received {d} bytes: {s}", .{ bytes_read, recv_buf[0..bytes_read] });
}
```

## Connection Errors

The `connect` call can fail with various errors. Common ones include:

| Error | Meaning |
|-------|---------|
| `error.ConnectionRefused` | No server is listening on that address/port |
| `error.NetworkUnreachable` | The network cannot route to the destination |
| `error.TimedOut` | The connection attempt timed out |

Always handle these gracefully in production code. A simple pattern is to log the error and either retry or exit cleanly.

---

# Chapter 5: TCP Servers &mdash; Accepting Connections

A TCP server binds to an address, listens for incoming connections, and accepts them one at a time. Each accepted connection yields a new `Stream` that you can read from and write to independently.

## The `listen` Method

```zig
const addr = try net.IpAddress.parseIp4("127.0.0.1", 8080);

var server = try addr.listen(io, .{ .reuse_address = true });
defer server.deinit(io);
```

The `listen` method takes `io` and a `ListenOptions` struct. The `.reuse_address = true` option sets `SO_REUSEADDR` on the socket, allowing you to restart the server quickly without hitting "address already in use" errors during development. The method returns a server object that you can call `accept` on.

## The `accept` Loop

Once the server is listening, you enter a loop that accepts connections:

```zig
while (true) {
    const stream = server.accept(io) catch |err| {
        std.log.err("accept failed: {s}", .{@errorName(err)});
        continue;
    };
    // Handle the connection...
    stream.close(io);
}
```

Each call to `accept` blocks until a client connects, then returns a `net.Stream` representing that connection. If the accept fails (for example, if a file descriptor limit is hit), the error is caught and the loop continues.

## A Complete Echo Server

This server accepts connections and echoes back whatever the client sends:

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const addr = try net.IpAddress.parseIp4("127.0.0.1", 8080);
    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    std.log.info("Echo server listening on {f}", .{addr});

    while (true) {
        const stream = server.accept(io) catch |err| {
            std.log.err("accept failed: {s}", .{@errorName(err)});
            continue;
        };

        handleConnection(stream, io) catch |err| {
            std.log.err("connection error: {s}", .{@errorName(err)});
        };
    }
}

fn handleConnection(stream: net.Stream, io: *std.Io) !void {
    defer stream.close(io);

    var recv_buf: [1024]u8 = undefined;
    var send_buf: [1024]u8 = undefined;
    var reader = stream.reader(io, &recv_buf);
    var writer = stream.writer(io, &send_buf);

    const n = try reader.interface.read(&recv_buf);
    try writer.interface.writeAll(recv_buf[0..n]);
    try writer.interface.flush();
    std.log.info("Echoed {d} bytes", .{n});
}
```

Test it:

```bash
# Terminal 1
zig run echo_server.zig

# Terminal 2
zig run tcp_client.zig 8080
```

## The `reuse_address` Option

Without `.reuse_address = true`, restarting your server immediately after it exits will fail with "address already in use" because the OS keeps the socket in a `TIME_WAIT` state for a minute or so. Setting `SO_REUSEADDR` tells the OS to allow the address to be reused immediately. Always set this to `true` during development. In production, you may choose to set it to `false` if you want the OS to enforce that no other process grabs your port during the brief `TIME_WAIT` window.

---

# Chapter 6: Stream I/O &mdash; Reading and Writing Data

Once you have a `net.Stream` (either from `connect` on the client side or `accept` on the server side), you read and write data through **reader** and **writer** objects.

## Creating Readers and Writers

```zig
var recv_buf: [1024]u8 = undefined;
var send_buf: [1024]u8 = undefined;

var reader = stream.reader(io, &recv_buf);
var writer = stream.writer(io, &send_buf);
```

The `reader` and `writer` each take a stack-allocated buffer. This buffer is used internally for buffered I/O, which reduces the number of system calls. You control the buffer size based on your application's needs. For most networking code, 1024 to 8192 bytes is a reasonable range.

## Reading Data

```zig
// Read up to recv_buf.len bytes, returns actual count
const n = try reader.interface.read(&recv_buf);
const data = recv_buf[0..n];

// Read exactly N bytes (blocks until N bytes available or EOF)
try reader.interface.readAtLeast(&recv_buf, 256);
```

The `read` function returns the number of bytes actually read, which may be less than the buffer size. It returns `error.EndOfStream` when the remote end closes the connection. The `readAtLeast` function blocks until at least the specified number of bytes have been read.

## Writing Data

```zig
// Write all bytes (blocks until complete)
try writer.interface.writeAll("HTTP/1.1 200 OK\r\n\r\nHello!");

// Flush buffered data to the socket
try writer.interface.flush();
```

`writeAll` guarantees that all bytes are written (looping internally if necessary). `flush` pushes any buffered data to the underlying socket. Always call `flush` after `writeAll` if you need the data to be sent immediately.

## Buffered vs. Unbuffered I/O

The buffers you pass to `reader()` and `writer()` enable **buffered I/O**. Instead of making a system call for every single byte, Zig fills the buffer in one system call and then serves your reads from that buffer. This dramatically reduces overhead for protocols that do many small reads and writes (like line-by-line HTTP parsing).

If you truly need unbuffered I/O (for example, when implementing a protocol that must send each byte immediately), you can use a 1-byte buffer or access the underlying file descriptor directly. For 99% of networking code, buffered I/O is the right choice.

## A Line-Based Protocol Example

Many protocols (HTTP, SMTP, FTP, IRC) are line-based. Here is how you read lines from a stream:

```zig
const std = @import("std");

fn readLine(reader: anytype, buf: []u8) ![]const u8 {
    var i: usize = 0;
    while (i < buf.len) {
        const byte = reader.interface.readByte() catch |err| switch (err) {
            error.EndOfStream => if (i == 0) return error.EndOfStream else break,
            else => return err,
        };
        buf[i] = byte;
        i += 1;
        if (byte == '\n') break;
    }
    return buf[0..i];
}
```

This helper reads bytes one at a time until it encounters a newline (`\n`). It returns a slice of the buffer containing the line (including the newline). For production HTTP servers, you would use `std.http.Server` which handles this parsing for you, as we will see in Chapter 9.

---

# Chapter 7: UDP Networking &mdash; Connectionless Communication

UDP (User Datagram Protocol) is a connectionless protocol. Unlike TCP, there is no handshake, no stream, and no guarantee of delivery or ordering. Each operation sends or receives a single datagram (a discrete chunk of data) to or from a specific address.

UDP is useful for DNS queries, gaming, streaming media, and any scenario where low latency matters more than reliability.

## Binding a UDP Socket

```zig
const addr = try net.IpAddress.parse("127.0.0.1", 32100);
const sock = try addr.bind(io, .{ .mode = .dgram, .protocol = .udp });
defer sock.close(io);
```

The `bind` method creates a UDP socket bound to the specified address. The `.mode = .dgram` option specifies datagram (connectionless) mode, and `.protocol = .udp` selects the UDP protocol.

## Receiving Datagrams

```zig
var buf: [1024]u8 = undefined;
const msg = try sock.receive(io, &buf);

std.log.info(
    "received {d} bytes from {f}: {s}",
    .{ msg.data.len, msg.from, msg.data },
);
```

The `receive` method blocks until a datagram arrives. It returns a `Message` struct containing:
- `data`: a slice of the buffer with the received bytes
- `from`: the `IpAddress` of the sender

## Sending Datagrams

```zig
try sock.send(io, &msg.from, msg.data);
std.log.info("echoed {d} bytes back", .{msg.data.len});
```

The `send` method takes the destination address and the data to send. The data is sent as a single datagram.

## A Complete UDP Echo Server

```zig
const std = @import("std");
const net = std.Io.net;

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const addr = try net.IpAddress.parse("127.0.0.1", 32100);
    const sock = try addr.bind(io, .{ .mode = .dgram, .protocol = .udp });
    defer sock.close(io);

    std.log.info("UDP echo listening on {f}", .{addr});

    var buf: [1024]u8 = undefined;

    while (true) {
        const msg = try sock.receive(io, &buf);
        std.log.info("received {d} bytes from {f}: {s}", .{
            msg.data.len,
            msg.from,
            msg.data,
        });
        try sock.send(io, &msg.from, msg.data);
        std.log.info("echoed {d} bytes back", .{msg.data.len});
    }
}
```

Test it from another terminal:

```bash
echo "hello zig" | nc -u localhost 32100
```

## UDP Limitations

UDP has some important limitations to be aware of. First, there is a maximum datagram size. The IP layer limits UDP datagrams to roughly 65507 bytes of payload, and in practice, values above ~1400 bytes risk IP fragmentation, which reduces reliability. Second, `std.Io.net` currently lacks a way to do non-IP networking (Unix domain sockets, for example) as of Zig 0.16. Third, because UDP is connectionless, there is no built-in flow control or congestion control; you are responsible for implementing any reliability guarantees your application needs on top of UDP.

---

# Chapter 8: HTTP Client &mdash; Making Web Requests

Zig's standard library includes `std.http.Client`, a full-featured HTTP client that handles DNS resolution, TCP connections, TLS (when available), HTTP/1.1 and HTTP/2, redirects, chunked encoding, and more.

## A Basic HTTP GET Request

Here is the example from the Zig 0.16 release notes, which demonstrates making a HEAD request to any domain:

```zig
const std = @import("std");
const Io = std.Io;

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;
    const io = init.io;

    const args = try init.minimal.args.toSlice(init.arena.allocator());
    const host_name: Io.net.HostName = try .init(args[1]);

    var http_client: std.http.Client = .{ .allocator = gpa, .io = io };
    defer http_client.deinit();

    var request = try http_client.request(.HEAD, .{
        .scheme = "http",
        .host = .{ .percent_encoded = host_name.bytes },
        .port = 80,
        .path = .{ .percent_encoded = "/" },
    }, .{});
    defer request.deinit();

    try request.sendBodiless();

    var redirect_buffer: [1024]u8 = undefined;
    const response = try request.receiveHead(&redirect_buffer);
    std.log.info("received {d} {s}", .{ response.head.status, response.head.reason });
}
```

### What Happens Under the Hood

Thanks to the `std.Io` interface, this code:

1. Asynchronously sends DNS queries to each configured nameserver.
2. As each DNS response arrives, it immediately, asynchronously tries to TCP connect to the returned IP address.
3. Upon the first successful TCP connection, all other in-flight connection attempts (including outstanding DNS queries) are **canceled**.
4. The code also works when compiled with `-fsingle-threaded`, in which case the operations happen sequentially.
5. On Windows, this entire process happens without the `ws2_32.dll` dependency.

## The `std.http.Client` API

### Creating a Client

```zig
var client: std.http.Client = .{ .allocator = gpa, .io = io };
defer client.deinit();
```

The client needs an allocator (for response headers and body) and an `Io` instance (for all I/O operations).

### Making Requests

```zig
var request = try client.request(.GET, .{
    .scheme = "https",
    .host = .{ .percent_encoded = "example.com" },
    .port = 443,
    .path = .{ .percent_encoded = "/api/v1/data" },
}, .{});
defer request.deinit();

// For GET/HEAD/DELETE, send without a body
try request.sendBodiless();

// For POST/PUT, send with a body
// try request.send("request body here", .{});

// Read the response headers
var buffer: [4096]u8 = undefined;
const response = try request.receiveHead(&buffer);
```

### Reading the Response Body

```zig
// Read the entire body into an allocator-backed buffer
const body = try request.reader().readAllAlloc(gpa, 1024 * 1024); // 1MB max
defer gpa.free(body);

std.log.info("Status: {d}", .{response.head.status});
std.log.info("Body length: {d}", .{body.len});
std.log.info("Body: {s}", .{body});
```

### Setting Headers

```zig
var request = try client.request(.POST, .{
    .scheme = "https",
    .host = .{ .percent_encoded = "api.example.com" },
    .port = 443,
    .path = .{ .percent_encoded = "/items" },
}, .{
    .extra_headers = &.{
        .{ .name = "Content-Type", .value = "application/json" },
        .{ .name = "Authorization", .value = "Bearer token123" },
    },
});
```

### Methods Reference

| Method | Constant | Has Body? |
|--------|----------|-----------|
| GET | `.GET` | No |
| POST | `.POST` | Yes |
| PUT | `.PUT` | Yes |
| DELETE | `.DELETE` | No |
| HEAD | `.HEAD` | No |
| PATCH | `.PATCH` | Yes |
| OPTIONS | `.OPTIONS` | No |

---

# Chapter 9: HTTP Server &mdash; Building Web Services

Zig provides `std.http.Server` for parsing incoming HTTP requests and constructing responses. It handles the gritty details of HTTP/1.1 parsing: request lines, headers, chunked transfer encoding, keep-alive, and more.

## A Minimal HTTP Server

```zig
const std = @import("std");
const net = std.Io.net;
const Io = std.Io;

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const addr = try net.IpAddress.parse("127.0.0.1", 8080);
    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    std.log.info("HTTP server listening on {f}", .{addr});

    while (true) {
        const stream = server.accept(io) catch |err| {
            std.log.err("accept failed: {s}", .{@errorName(err)});
            continue;
        };

        // Spawn a thread per connection
        const thread = std.Thread.spawn(.{}, handleConnection, .{ stream, io }) catch |err| {
            std.log.err("unable to spawn thread: {s}", .{@errorName(err)});
            stream.close(io);
            continue;
        };
        thread.detach();
    }
}

fn handleConnection(stream: net.Stream, io: Io) !void {
    defer stream.close(io);

    var recv_buf: [8192]u8 = undefined;
    var send_buf: [4096]u8 = undefined;

    var stream_reader = stream.reader(io, &recv_buf);
    var stream_writer = stream.writer(io, &send_buf);

    var server = std.http.Server.init(
        &stream_reader.interface,
        &stream_writer.interface,
    );

    while (server.reader.state == .ready) {
        var request = server.receiveHead() catch |err| switch (err) {
            error.HttpConnectionClosing => return,
            else => return err,
        };

        try request.respond(
            "Hello from Zig!",
            .{
                .extra_headers = &.{
                    .{ .name = "Content-Type", .value = "text/plain" },
                },
            },
        );
    }
}
```

This server accepts connections, spawns a thread for each one, parses incoming HTTP requests, and responds with "Hello from Zig!" and a `Content-Type: text/plain` header. The `std.http.Server.init` takes a reader interface and a writer interface, and the `receiveHead` method blocks until a complete HTTP request header is received.

## The `request.respond` Method

The simplest way to send a response:

```zig
try request.respond(
    "<h1>Hello World</h1>",
    .{
        .status = .ok,
        .extra_headers = &.{
            .{ .name = "Content-Type", .value = "text/html" },
        },
    },
);
```

You can set the status code explicitly and add any number of custom headers.

## Accessing Request Properties

The `Request` struct exposes the parsed request data:

```zig
// HTTP method: GET, POST, etc.
const method = request.head.method;

// Request path: "/api/items/42"
const path = request.head.target;

// Request headers
// Iterate through request.head.headers to find specific headers
var it = request.head.iterateHeaders();
while (it.next()) |header| {
    std.log.info("Header: {s}: {s}", .{ header.name, header.value });
}
```

## Reading a Request Body (POST/PUT)

For requests that carry a body (POST, PUT, PATCH), you can read the body from the request's reader:

```zig
if (request.head.method == .POST) {
    var body_buf: [4096]u8 = undefined;
    const body_len = try request.reader().readAll(&body_buf);
    const body = body_buf[0..body_len];
    std.log.info("POST body ({d} bytes): {s}", .{ body_len, body });
}
```

---

# Chapter 10: Concurrency and I/O Backends

## Thread-Per-Connection

The simplest concurrency model for a server is to spawn a new OS thread for each accepted connection. This is what the HTTP server example in Chapter 9 does:

```zig
const thread = std.Thread.spawn(.{}, handleConnection, .{ stream, io }) catch |err| {
    std.log.err("unable to spawn thread: {s}", .{@errorName(err)});
    stream.close(io);
    continue;
};
thread.detach();
```

`std.Thread.spawn` creates a new OS thread that runs the given function with the provided arguments. Calling `detach()` tells Zig not to wait for the thread to finish; the thread cleans up after itself when the function returns.

This model is simple and works well for low-to-moderate connection counts (hundreds to low thousands). Each thread gets its own stack (default 8 MB on most systems), so memory usage grows linearly with the number of concurrent connections. For high-traffic servers with tens of thousands of connections, you would want a thread pool or an evented architecture.

## Thread Pools

Zig provides `std.Thread.Pool` for managing a fixed number of worker threads:

```zig
var pool: std.Thread.Pool = undefined;
try pool.init(.{ .allocator = gpa, .n_jobs = 4 });
defer pool.deinit();

// In your accept loop:
pool.spawn(handleConnection, .{ stream, io });
```

The pool reuses threads, avoiding the overhead of creating and destroying OS threads for every connection. The `.n_jobs` parameter sets the number of worker threads. A common pattern is to use the number of CPU cores available (`std.Thread.getCpuCount()`).

## The Future: `Io.Evented`

Zig 0.16's `Io.Evented` backend does not yet support networking. When it does, your code will be able to handle tens of thousands of concurrent connections on a single thread by using epoll (Linux), kqueue (macOS/BSD), or IOCP (Windows) under the hood. The key insight is that because all I/O operations go through the `Io` instance, switching from `Io.Threaded` to `Io.Evented` requires no code changes in your networking logic&mdash;only the initialization of the `Io` backend changes.

---

# Project 1: Static File Server

This project builds a fully functional static file server that serves files from a specified directory over HTTP. It supports correct MIME types, directory listings, and proper error handling for missing files.

## Directory Structure

```
static-file-server/
├── build.zig
└── src/
    └── main.zig
```

## `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "static-file-server",
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
    b.step("run", "Run the static file server").dependOn(&run_cmd.step);
}
```

## `src/main.zig`

```zig
const std = @import("std");
const net = std.Io.net;
const Io = std.Io;
const fs = std.fs;
const http = std.http;

const log = std.log;

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;
    const io = init.io;

    var args = init.minimal.args.iterate();
    _ = args.skip(); // skip program name

    const port_str = args.next() orelse "8080";
    const port = try std.fmt.parseInt(u16, port_str, 10);

    const dir_path = args.next() orelse ".";
    const dir = fs.cwd().openDir(dir_path, .{ .iterate = true }) catch |err| {
        log.err("failed to open directory '{s}': {s}", .{ dir_path, @errorName(err) });
        return err;
    };

    const addr = try net.IpAddress.parseIp4("0.0.0.0", port);
    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    log.info("Serving '{s}' on http://127.0.0.1:{d}", .{ dir_path, port });

    while (true) {
        const stream = server.accept(io) catch |err| {
            log.err("accept: {s}", .{@errorName(err)});
            continue;
        };

        const thread = std.Thread.spawn(.{}, serveConnection, .{
            stream, io, dir, gpa,
        }) catch |err| {
            log.err("spawn: {s}", .{@errorName(err)});
            stream.close(io);
            continue;
        };
        thread.detach();
    }
}

fn serveConnection(
    stream: net.Stream,
    io: Io,
    root_dir: fs.Dir,
    allocator: std.mem.Allocator,
) void {
    defer stream.close(io);

    var recv_buf: [8192]u8 = undefined;
    var send_buf: [16384]u8 = undefined;

    var stream_reader = stream.reader(io, &recv_buf);
    var stream_writer = stream.writer(io, &send_buf);

    var http_server = http.Server.init(
        &stream_reader.interface,
        &stream_writer.interface,
    );

    serveLoop(&http_server, root_dir, allocator) catch |err| {
        log.err("connection handler: {s}", .{@errorName(err)});
    };
}

fn serveLoop(
    http_server: *http.Server,
    root_dir: fs.Dir,
    allocator: std.mem.Allocator,
) !void {
    while (http_server.reader.state == .ready) {
        var request = http_server.receiveHead() catch |err| switch (err) {
            error.HttpConnectionClosing => return,
            else => return err,
        };

        const target = request.head.target;
        // Strip leading "/" to get relative path
        const rel_path = if (target.len > 0 and target[0] == '/')
            target[1..]
        else
            target;

        // Default to "index.html" for directory requests
        const file_path = if (std.mem.eql(u8, rel_path, "")) "index.html" else rel_path;

        // Security: reject paths with ".." to prevent directory traversal
        if (std.mem.indexOf(u8, file_path, "..") != null) {
            try request.respond("403 Forbidden", .{
                .status = .forbidden,
                .extra_headers = &.{
                    .{ .name = "Content-Type", .value = "text/plain" },
                },
            });
            continue;
        }

        // Try to open and read the file
        const file = root_dir.openFile(file_path, .{}) catch |err| switch (err) {
            error.FileNotFound => {
                try request.respond("404 Not Found", .{
                    .status = .not_found,
                    .extra_headers = &.{
                        .{ .name = "Content-Type", .value = "text/plain" },
                    },
                });
                continue;
            },
            error.IsDir => {
                // Try serving index.html inside the directory
                const index_path = try std.mem.concat(allocator, u8, &.{ file_path, "/index.html" });
                defer allocator.free(index_path);
                const index_file = root_dir.openFile(index_path, .{}) catch {
                    try serveDirectoryListing(&request, root_dir, file_path, allocator);
                    continue;
                };
                index_file.close();
                try serveFile(&request, root_dir, index_path, allocator);
                continue;
            },
            else => {
                try request.respond("500 Internal Server Error", .{
                    .status = .internal_server_error,
                    .extra_headers = &.{
                        .{ .name = "Content-Type", .value = "text/plain" },
                    },
                });
                continue;
            },
        };
        file.close();
        try serveFile(&request, root_dir, file_path, allocator);
    }
}

fn serveFile(
    request: *http.Server.Request,
    root_dir: fs.Dir,
    file_path: []const u8,
    allocator: std.mem.Allocator,
) !void {
    const file = try root_dir.openFile(file_path, .{});
    defer file.close();

    const stat = try file.stat();
    const file_size = stat.size;

    const mime = getMimeType(file_path);

    // Send response with Content-Length header
    const content_length = try std.fmt.allocPrint(allocator, "{d}", .{file_size});
    defer allocator.free(content_length);

    try request.respond("", .{
        .status = .ok,
        .extra_headers = &.{
            .{ .name = "Content-Type", .value = mime },
            .{ .name = "Content-Length", .value = content_length },
            .{ .name = "Connection", .value = "close" },
        },
    });

    // Stream the file contents
    var buf: [8192]u8 = undefined;
    var remaining: u64 = file_size;
    while (remaining > 0) {
        const to_read = @min(buf.len, remaining);
        const bytes_read = try file.read(buf[0..to_read]);
        if (bytes_read == 0) break;
        try request.connection.writer.writeAll(buf[0..bytes_read]);
        remaining -= bytes_read;
    }
}

fn serveDirectoryListing(
    request: *http.Server.Request,
    root_dir: fs.Dir,
    dir_path: []const u8,
    allocator: std.mem.Allocator,
) !void {
    var listing = std.ArrayList(u8).init(allocator);
    defer listing.deinit();

    const writer = listing.writer();

    try writer.writeAll(
        \\<!DOCTYPE html>
        \\<html>
        \\<head><title>Directory Listing: /</title>
        \\<style>
        \\body { font-family: monospace; padding: 2rem; background: #1a1a2e; color: #e0e0e0; }
        \\a { color: #64b5f6; text-decoration: none; }
        \\a:hover { text-decoration: underline; }
        \\.file { padding: 4px 0; }
        \\</style>
        \\</head>
        \\<body>
        \\<h1>Directory Listing: /
    );
    try writer.writeAll(dir_path);
    try writer.writeAll("</h1>\n");

    var iter = root_dir.iterate();
    while (try iter.next()) |entry| {
        try writer.writeAll("<div class=\"file\">");
        if (dir_path.len > 0) {
            try writer.writeAll("<a href=\"/");
            try writer.writeAll(dir_path);
            try writer.writeAll("/");
            try writer.writeAll(entry.name);
            try writer.writeAll("\">");
        } else {
            try writer.writeAll("<a href=\"/");
            try writer.writeAll(entry.name);
            try writer.writeAll("\">");
        }
        try writer.writeAll(entry.name);
        if (entry.kind == .directory) {
            try writer.writeAll("/");
        }
        try writer.writeAll("</a></div>\n");
    }

    try writer.writeAll("</body>\n</html>\n");

    const body = listing.items;
    const body_str = try std.fmt.allocPrint(allocator, "{d}", .{body.len});
    defer allocator.free(body_str);

    try request.respond("", .{
        .status = .ok,
        .extra_headers = &.{
            .{ .name = "Content-Type", .value = "text/html" },
            .{ .name = "Content-Length", .value = body_str },
            .{ .name = "Connection", .value = "close" },
        },
    });

    try request.connection.writer.writeAll(body);
}

fn getMimeType(path: []const u8) []const u8 {
    const ext_start = std.mem.lastIndexOfScalar(u8, path, '.') orelse return "application/octet-stream";
    const ext = path[ext_start..];

    const mime_map = std.ComptimeStringMap([]const u8, .{
        .{ ".html", "text/html; charset=utf-8" },
        .{ ".htm", "text/html; charset=utf-8" },
        .{ ".css", "text/css; charset=utf-8" },
        .{ ".js", "application/javascript; charset=utf-8" },
        .{ ".json", "application/json; charset=utf-8" },
        .{ ".png", "image/png" },
        .{ ".jpg", "image/jpeg" },
        .{ ".jpeg", "image/jpeg" },
        .{ ".gif", "image/gif" },
        .{ ".svg", "image/svg+xml" },
        .{ ".ico", "image/x-icon" },
        .{ ".webp", "image/webp" },
        .{ ".txt", "text/plain; charset=utf-8" },
        .{ ".md", "text/markdown; charset=utf-8" },
        .{ ".xml", "application/xml" },
        .{ ".pdf", "application/pdf" },
        .{ ".wasm", "application/wasm" },
        .{ ".mp4", "video/mp4" },
        .{ ".webm", "video/webm" },
        .{ ".mp3", "audio/mpeg" },
        .{ ".ogg", "audio/ogg" },
        .{ ".zip", "application/zip" },
        .{ ".gz", "application/gzip" },
        .{ ".tar", "application/x-tar" },
    });

    return mime_map.get(ext) orelse "application/octet-stream";
}
```

## Building and Running

```bash
# Build
zig build

# Run, serving the current directory on port 8080
zig build run

# Serve a specific directory on a specific port
zig build run -- 3000 /path/to/files

# Test it
curl http://localhost:8080/
curl http://localhost:8080/README.md
```

## Key Design Decisions

This server implements several important patterns for production-ready file serving. The **directory traversal prevention** (`..` check) is critical for security&mdash;without it, a client could request `/../../../../etc/passwd` and read arbitrary files. The **MIME type detection** uses a compile-time string map (`ComptimeStringMap`) which has zero runtime overhead for lookups. The **connection-per-thread model** keeps the implementation simple while still allowing concurrent requests; for a production deployment, you would swap this for a thread pool or an evented architecture.

---

# Project 2: REST API Server

This project builds a JSON-based REST API server that supports CRUD (Create, Read, Update, Delete) operations on an in-memory "todos" resource. It demonstrates route parsing, JSON serialization/deserialization, and proper HTTP status codes.

## Directory Structure

```
rest-api-server/
├── build.zig
└── src/
    └── main.zig
```

## `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "rest-api-server",
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
    b.step("run", "Run the REST API server").dependOn(&run_cmd.step);
}
```

## `src/main.zig`

```zig
const std = @import("std");
const net = std.Io.net;
const Io = std.Io;
const http = std.http;
const log = std.log;

const Todo = struct {
    id: u32,
    title: []const u8,
    completed: bool,
};

const MAX_TODOS = 1024;

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;
    const io = init.io;

    var args = init.minimal.args.iterate();
    _ = args.skip();

    const port_str = args.next() orelse "9090";
    const port = try std.fmt.parseInt(u16, port_str, 10);

    const addr = try net.IpAddress.parseIp4("0.0.0.0", port);
    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    log.info("REST API server listening on {f}", .{addr});

    var todos: [MAX_TODOS]Todo = undefined;
    var todo_count: u32 = 0;
    var next_id: u32 = 1;
    var mutex: std.Thread.Mutex = .{};

    while (true) {
        const stream = server.accept(io) catch |err| {
            log.err("accept: {s}", .{@errorName(err)});
            continue;
        };

        const thread = std.Thread.spawn(.{}, handleConnection, .{
            stream,
            io,
            gpa,
            &todos,
            &todo_count,
            &next_id,
            &mutex,
        }) catch |err| {
            log.err("spawn: {s}", .{@errorName(err)});
            stream.close(io);
            continue;
        };
        thread.detach();
    }
}

fn handleConnection(
    stream: net.Stream,
    io: Io,
    allocator: std.mem.Allocator,
    todos: *[MAX_TODOS]Todo,
    todo_count: *u32,
    next_id: *u32,
    mutex: *std.Io.Mutex,
) void {
    defer stream.close(io);

    var recv_buf: [8192]u8 = undefined;
    var send_buf: [16384]u8 = undefined;

    var stream_reader = stream.reader(io, &recv_buf);
    var stream_writer = stream.writer(io, &send_buf);

    var http_server = http.Server.init(
        &stream_reader.interface,
        &stream_writer.interface,
    );

    while (http_server.reader.state == .ready) {
        var request = http_server.receiveHead() catch |err| switch (err) {
            error.HttpConnectionClosing => return,
            else => {
                log.err("receiveHead: {s}", .{@errorName(err)});
                return;
            },
        };

        const method = request.head.method;
        const path = request.head.target;

        // Read body for POST/PUT
        var body_buf: [4096]u8 = undefined;
        var body: []const u8 = "";
        if (method == .POST or method == .PUT) {
            const n = request.reader().readAll(&body_buf) catch 0;
            body = body_buf[0..n];
        }

        // Route the request
        const response = routeRequest(.{
            .method = method,
            .path = path,
            .body = body,
            .todos = todos,
            .todo_count = todo_count,
            .next_id = next_id,
            .mutex = mutex,
            .allocator = allocator,
        });

        // Send response
        const resp_body = response.body orelse "";
        const resp_headers = response.headers orelse &.{};
        try request.respond(resp_body, .{
            .status = response.status,
            .extra_headers = resp_headers,
        });
    }
}

const RequestContext = struct {
    method: http.Method,
    path: []const u8,
    body: []const u8,
    todos: *[MAX_TODOS]Todo,
    todo_count: *u32,
    next_id: *u32,
    mutex: *std.Io.Mutex,
    allocator: std.mem.Allocator,
};

const Response = struct {
    status: http.Status,
    body: ?[]const u8,
    headers: ?[]const http.Server.Response.Field,
};

const json_header = http.Server.Response.Field{
    .name = "Content-Type",
    .value = "application/json",
};

const cors_header = http.Server.Response.Field{
    .name = "Access-Control-Allow-Origin",
    .value = "*",
};

fn routeRequest(ctx: RequestContext) Response {
    // GET /todos - list all todos
    if (ctx.method == .GET and std.mem.eql(u8, ctx.path, "/todos")) {
        ctx.mutex.lock();
        defer ctx.mutex.unlock();

        var list = std.ArrayList(u8).init(ctx.allocator);
        defer list.deinit();
        const w = list.writer();

        w.writeAll("[") catch return .{ .status = .internal_server_error, .body = null, .headers = null };
        for (ctx.todos[0..ctx.todo_count.*], 0..) |todo, i| {
            if (i > 0) w.writeAll(",") catch {};
            writeTodoJson(w, todo) catch {};
        }
        w.writeAll("]") catch {};

        return .{
            .status = .ok,
            .body = list.items,
            .headers = &.{ json_header, cors_header },
        };
    }

    // GET /todos/:id - get a specific todo
    if (ctx.method == .GET and startsWith(ctx.path, "/todos/")) {
        ctx.mutex.lock();
        defer ctx.mutex.unlock();

        const id_str = ctx.path["/todos/".len..];
        const id = std.fmt.parseInt(u32, id_str, 10) catch {
            return .{ .status = .bad_request, .body = "{\"error\":\"invalid id\"}", .headers = &.{json_header} };
        };

        for (ctx.todos[0..ctx.todo_count.*]) |todo| {
            if (todo.id == id) {
                var buf: [512]u8 = undefined;
                var fbs = std.io.fixedBufferStream(&buf);
                writeTodoJson(fbs.writer(), todo) catch {};
                const json_str = fbs.getWritten();
                const owned = ctx.allocator.dupe(u8, json_str) catch return .{
                    .status = .internal_server_error, .body = null, .headers = null,
                };
                return .{ .status = .ok, .body = owned, .headers = &.{json_header, cors_header} };
            }
        }

        return .{ .status = .not_found, .body = "{\"error\":\"todo not found\"}", .headers = &.{json_header} };
    }

    // POST /todos - create a new todo
    if (ctx.method == .POST and std.mem.eql(u8, ctx.path, "/todos")) {
        ctx.mutex.lock();
        defer ctx.mutex.unlock();

        if (ctx.todo_count.* >= MAX_TODOS) {
            return .{ .status = .service_unavailable, .body = "{\"error\":\"todo limit reached\"}", .headers = &.{json_header} };
        }

        // Simple JSON parsing: extract "title" field
        const title = extractJsonString(ctx.body, "title") orelse {
            return .{ .status = .bad_request, .body = "{\"error\":\"missing 'title' field\"}", .headers = &.{json_header} };
        };

        const id = ctx.next_id.*;
        ctx.next_id.* += 1;
        ctx.todos[ctx.todo_count.*] = .{
            .id = id,
            .title = ctx.allocator.dupe(u8, title) catch return .{
                .status = .internal_server_error, .body = null, .headers = null,
            },
            .completed = false,
        };
        ctx.todo_count.* += 1;

        var buf: [512]u8 = undefined;
        var fbs = std.io.fixedBufferStream(&buf);
        writeTodoJson(fbs.writer(), ctx.todos[ctx.todo_count.* - 1]) catch {};
        const json_str = fbs.getWritten();
        const owned = ctx.allocator.dupe(u8, json_str) catch return .{
            .status = .internal_server_error, .body = null, .headers = null,
        };

        return .{ .status = .created, .body = owned, .headers = &.{json_header, cors_header} };
    }

    // PUT /todos/:id - update a todo
    if (ctx.method == .PUT and startsWith(ctx.path, "/todos/")) {
        ctx.mutex.lock();
        defer ctx.mutex.unlock();

        const id_str = ctx.path["/todos/".len..];
        const id = std.fmt.parseInt(u32, id_str, 10) catch {
            return .{ .status = .bad_request, .body = "{\"error\":\"invalid id\"}", .headers = &.{json_header} };
        };

        for (ctx.todos[0..ctx.todo_count.*]) |*todo| {
            if (todo.id == id) {
                // Update title if provided
                if (extractJsonString(ctx.body, "title")) |title| {
                    if (ctx.allocator.free(todo.title)) {} // free old title
                    todo.title = ctx.allocator.dupe(u8, title) catch {};
                }
                // Update completed if provided
                if (extractJsonBool(ctx.body, "completed")) |completed| {
                    todo.completed = completed;
                }

                var buf: [512]u8 = undefined;
                var fbs = std.io.fixedBufferStream(&buf);
                writeTodoJson(fbs.writer(), todo.*) catch {};
                const json_str = fbs.getWritten();
                const owned = ctx.allocator.dupe(u8, json_str) catch return .{
                    .status = .internal_server_error, .body = null, .headers = null,
                };
                return .{ .status = .ok, .body = owned, .headers = &.{json_header, cors_header} };
            }
        }

        return .{ .status = .not_found, .body = "{\"error\":\"todo not found\"}", .headers = &.{json_header} };
    }

    // DELETE /todos/:id - delete a todo
    if (ctx.method == .DELETE and startsWith(ctx.path, "/todos/")) {
        ctx.mutex.lock();
        defer ctx.mutex.unlock();

        const id_str = ctx.path["/todos/".len..];
        const id = std.fmt.parseInt(u32, id_str, 10) catch {
            return .{ .status = .bad_request, .body = "{\"error\":\"invalid id\"}", .headers = &.{json_header} };
        };

        for (0..ctx.todo_count.*, 0..) |_, i| {
            if (ctx.todos[i].id == id) {
                // Shift remaining todos down
                const removed = ctx.todos[i];
                for (i..ctx.todo_count.* - 1) |j| {
                    ctx.todos[j] = ctx.todos[j + 1];
                }
                ctx.todo_count.* -= 1;

                ctx.allocator.free(removed.title) catch {};

                return .{
                    .status = .no_content,
                    .body = null,
                    .headers = &.{cors_header},
                };
            }
        }

        return .{ .status = .not_found, .body = "{\"error\":\"todo not found\"}", .headers = &.{json_header} };
    }

    return .{ .status = .not_found, .body = "{\"error\":\"not found\"}", .headers = &.{json_header} };
}

fn writeTodoJson(writer: anytype, todo: Todo) !void {
    try writer.print(
        \\"{{"id":{},"title":"{s}","completed":{}}}"
    , .{ todo.id, escapeJsonString(todo.title), todo.completed });
}

fn escapeJsonString(input: []const u8) [1024]u8 {
    var result: [1024]u8 = [_]u8{0} ** 1024;
    var j: usize = 0;
    for (input) |c| {
        if (j >= result.len - 2) break;
        switch (c) {
            '"' => {
                result[j] = '\\';
                result[j + 1] = '"';
                j += 2;
            },
            '\\' => {
                result[j] = '\\';
                result[j + 1] = '\\';
                j += 2;
            },
            '\n' => {
                result[j] = '\\';
                result[j + 1] = 'n';
                j += 2;
            },
            '\r' => {
                result[j] = '\\';
                result[j + 1] = 'r';
                j += 2;
            },
            '\t' => {
                result[j] = '\\';
                result[j + 1] = 't';
                j += 2;
            },
            else => {
                result[j] = c;
                j += 1;
            },
        }
    }
    return result;
}

fn startsWith(haystack: []const u8, prefix: []const u8) bool {
    if (haystack.len < prefix.len) return false;
    return std.mem.eql(u8, haystack[0..prefix.len], prefix);
}

/// Minimal JSON string extraction: finds "key":"value" in JSON body.
/// Returns a slice into the original body or null.
fn extractJsonString(body: []const u8, key: []const u8) ?[]const u8 {
    const key_pattern = tryFmt("\"{s}\":\"", .{key});
    const start = std.mem.indexOf(u8, body, key_pattern) orelse return null;
    const value_start = start + key_pattern.len;
    const end = std.mem.indexOfScalarPos(u8, body, value_start, '"') orelse return null;
    return body[value_start..end];
}

fn extractJsonBool(body: []const u8, key: []const u8) ?bool {
    const key_pattern = tryFmt("\"{s}\":", .{key});
    const start = std.mem.indexOf(u8, body, key_pattern) orelse return null;
    const after_key = std.mem.trimLeft(u8, body[start + key_pattern.len ..], " ");
    if (std.mem.startsWith(u8, after_key, "true")) return true;
    if (std.mem.startsWith(u8, after_key, "false")) return false;
    return null;
}

fn tryFmt(comptime fmt: []const u8, args: anytype) [256]u8 {
    var buf: [256]u8 = undefined;
    _ = std.fmt.bufPrint(&buf, fmt, args) catch {};
    return buf;
}
```

## Building and Running

```bash
zig build run

# In another terminal, test with curl:

# Create a todo
curl -X POST http://localhost:9090/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Zig networking"}'

# List all todos
curl http://localhost:9090/todos

# Get a specific todo
curl http://localhost:9090/todos/1

# Update a todo
curl -X PUT http://localhost:9090/todos/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Zig networking", "completed": true}'

# Delete a todo
curl -X DELETE http://localhost:9090/todos/1
```

## Key Design Decisions

This server uses a **mutex** (`std.Io.Mutex`) to protect the shared `todos` array from data races when multiple connection threads access it simultaneously. This is a straightforward approach that works well for moderate traffic. The **manual JSON parsing** (`extractJsonString`, `extractJsonBool`) avoids pulling in a JSON parser dependency while still being functional for the structured data our API handles. In a production server, you would use `std.json` for proper, robust JSON parsing and serialization. The **CORS header** (`Access-Control-Allow-Origin: *`) allows browser-based clients to make requests to the API directly.

---

# Project 3: TCP Chat Room

This project implements a multi-user chat room over TCP. It features a broadcast model where every message from any client is sent to all other connected clients. It uses a shared state protected by a mutex and a simple text-based protocol.

## Directory Structure

```
tcp-chat-room/
├── build.zig
└── src/
    └── main.zig
```

## `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "chat-server",
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
    b.step("run", "Run the chat server").dependOn(&run_cmd.step);
}
```

## `src/main.zig`

```zig
const std = @import("std");
const net = std.Io.net;
const Io = std.Io;
const log = std.log;

const MAX_CLIENTS = 64;
const MAX_USERNAME = 32;
const RECV_BUF_SIZE = 4096;
const BROADCAST_BUF_SIZE = 8192;

const Client = struct {
    stream: net.Stream,
    address: net.IpAddress,
    username: [MAX_USERNAME]u8,
    username_len: u8,
    active: bool,
};

const ChatState = struct {
    clients: [MAX_CLIENTS]Client,
    client_count: u32,
    mutex: std.Io.Mutex,
    allocator: std.mem.Allocator,

    fn init(allocator: std.mem.Allocator) ChatState {
        return .{
            .clients = undefined, // initialized on add
            .client_count = 0,
            .mutex = .{},
            .allocator = allocator,
        };
    }

    fn addClient(self: *ChatState, stream: net.Stream, address: net.IpAddress) !*Client {
        self.mutex.lock();
        defer self.mutex.unlock();

        if (self.client_count >= MAX_CLIENTS) return error.ServerFull;

        const client = &self.clients[self.client_count];
        client.* = .{
            .stream = stream,
            .address = address,
            .username = [_]u8{0} ** MAX_USERNAME,
            .username_len = 0,
            .active = true,
        };
        self.client_count += 1;
        return client;
    }

    fn removeClient(self: *ChatState, client: *Client) void {
        self.mutex.lock();
        defer self.mutex.unlock();

        // Find and remove by swapping with last
        for (0..self.client_count) |i| {
            if (&self.clients[i] == client) {
                self.clients[i] = self.clients[self.client_count - 1];
                self.client_count -= 1;
                return;
            }
        }
    }

    fn setUsername(self: *ChatState, client: *Client, name: []const u8) void {
        self.mutex.lock();
        defer self.mutex.unlock();

        const len = @min(name.len, MAX_USERNAME - 1);
        @memcpy(client.username[0..len], name[0..len]);
        client.username[len] = 0;
        client.username_len = @intCast(len);
    }

    fn getUsername(self: *ChatState, client: *Client) []const u8 {
        self.mutex.lock();
        defer self.mutex.unlock();
        return client.username[0..client.username_len];
    }

    fn broadcast(self: *ChatState, message: []const u8, exclude: ?*Client) void {
        self.mutex.lock();
        defer self.mutex.unlock();

        for (0..self.client_count) |i| {
            const c = &self.clients[i];
            if (!c.active) continue;
            if (exclude != null and &self.clients[i] == exclude.?) continue;

            // Best-effort send; ignore errors (client may have disconnected)
            var buf: [BROADCAST_BUF_SIZE]u8 = undefined;
            var writer = c.stream.writer(self.allocator, &buf);
            writer.interface.writeAll(message) catch continue;
            writer.interface.flush() catch continue;
        }
    }

    fn getClientCount(self: *ChatState) u32 {
        self.mutex.lock();
        defer self.mutex.unlock();
        return self.client_count;
    }
};

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;
    const io = init.io;

    var args = init.minimal.args.iterate();
    _ = args.skip();

    const port_str = args.next() orelse "6667";
    const port = try std.fmt.parseInt(u16, port_str, 10);

    const addr = try net.IpAddress.parseIp4("0.0.0.0", port);
    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    log.info("Chat server listening on {f} (max {d} clients)", .{ addr, MAX_CLIENTS });

    var state = ChatState.init(gpa);

    while (true) {
        const stream = server.accept(io) catch |err| {
            log.err("accept: {s}", .{@errorName(err)});
            continue;
        };

        const client_address: net.IpAddress = .parseIp4("0.0.0.0", 0) catch unreachable;
        _ = client_address;

        const thread = std.Thread.spawn(.{}, handleClient, .{
            stream, io, &state, gpa,
        }) catch |err| {
            log.err("spawn: {s}", .{@errorName(err)});
            stream.close(io);
            continue;
        };
        thread.detach();
    }
}

fn handleClient(
    stream: net.Stream,
    io: Io,
    state: *ChatState,
    allocator: std.mem.Allocator,
) void {
    // Get the peer address for logging
    var recv_buf: [RECV_BUF_SIZE]u8 = undefined;
    var send_buf: [BROADCAST_BUF_SIZE]u8 = undefined;

    var reader = stream.reader(io, &recv_buf);
    var writer = stream.writer(io, &send_buf);

    // Register client
    const client = state.addClient(stream, .parseIp4("0.0.0.0", 0) catch unreachable) catch {
        writer.interface.writeAll("ERROR: Server is full. Try again later.\r\n") catch {};
        writer.interface.flush() catch {};
        stream.close(io);
        return;
    };
    defer {
        state.removeClient(client);
        const username = state.getUsername(client);
        const msg = if (username.len > 0)
            tryFmtAlloc(allocator, "*** {s} has left the chat ***\r\n", .{username})
        else
            "*** A user has left the chat ***\r\n";
        defer allocator.free(msg);
        state.broadcast(msg, null);
        log.info("Client disconnected. {d} clients remaining.", .{state.getClientCount()});
    }

    // Send welcome and prompt for username
    writer.interface.writeAll(
        \\=====================================\r\n
        \\   Welcome to the Zig Chat Room!\r\n
        \\=====================================\r\n
        \\
    ) catch {};
    writer.interface.flush() catch {};

    // Read username
    writer.interface.writeAll("Enter your username: ") catch {};
    writer.interface.flush() catch {};

    const name_len = readLine(reader, &recv_buf) catch {
        return;
    };
    const username = recv_buf[0..name_len];
    // Strip \r\n
    const clean_name = stripCrLf(username);
    state.setUsername(client, clean_name);

    const join_msg = tryFmtAlloc(allocator, "*** {s} has joined the chat ***\r\n", .{clean_name});
    defer allocator.free(join_msg);
    state.broadcast(join_msg, null);

    log.info("{s} joined. {d} clients online.", .{ clean_name, state.getClientCount() });

    writer.interface.writeAll(tryFmtAlloc(allocator, "Hello, {s}! Type messages and press Enter.\r\n", .{clean_name})) catch {};
    writer.interface.flush() catch {};
    if (false) { _ = &tryFmtAlloc; }

    // Main message loop
    while (true) {
        const n = readLine(reader, &recv_buf) catch |err| switch (err) {
            error.EndOfStream => return,
            else => {
                log.err("read: {s}", .{@errorName(err)});
                return;
            },
        };
        if (n == 0) return;

        const line = recv_buf[0..n];
        const clean_line = stripCrLf(line);
        if (clean_line.len == 0) continue;

        // Commands
        if (std.mem.eql(u8, clean_line, "/quit")) {
            writer.interface.writeAll("Goodbye!\r\n") catch {};
            writer.interface.flush() catch {};
            return;
        }
        if (std.mem.eql(u8, clean_line, "/users")) {
            const count = state.getClientCount();
            const resp = tryFmtAlloc(allocator, "There are {d} users online.\r\n", .{count});
            defer allocator.free(resp);
            writer.interface.writeAll(resp) catch {};
            writer.interface.flush() catch {};
            continue;
        }
        if (std.mem.startsWith(u8, clean_line, "/name ")) {
            const new_name = clean_line["/name ".len..];
            if (new_name.len > 0 and new_name.len < MAX_USERNAME) {
                const old_name = state.getUsername(client);
                state.setUsername(client, new_name);
                const rename_msg = tryFmtAlloc(allocator, "*** {s} is now known as {s} ***\r\n", .{ old_name, new_name });
                defer allocator.free(rename_msg);
                state.broadcast(rename_msg, null);
                continue;
            }
        }

        // Broadcast message
        const username = state.getUsername(client);
        const chat_msg = tryFmtAlloc(allocator, "<{s}> {s}\r\n", .{ username, clean_line });
        defer allocator.free(chat_msg);
        state.broadcast(chat_msg, client);

        // Echo back to sender
        writer.interface.writeAll(chat_msg) catch return;
        writer.interface.flush() catch return;
    }
}

fn readLine(reader: anytype, buf: []u8) !usize {
    var i: usize = 0;
    while (i < buf.len) {
        const byte = reader.interface.readByte() catch |err| switch (err) {
            error.EndOfStream => if (i == 0) return error.EndOfStream else break,
            else => return err,
        };
        buf[i] = byte;
        i += 1;
        if (byte == '\n') break;
    }
    return i;
}

fn stripCrLf(input: []const u8) []const u8 {
    var end = input.len;
    if (end > 0 and input[end - 1] == '\n') end -= 1;
    if (end > 0 and input[end - 1] == '\r') end -= 1;
    return input[0..end];
}

fn tryFmtAlloc(allocator: std.mem.Allocator, comptime fmt: []const u8, args: anytype) []const u8 {
    return std.fmt.allocPrint(allocator, fmt, args) catch "???";
}
```

## Building and Running

```bash
# Start the server
zig build run

# Connect with multiple terminals using netcat:
nc localhost 6667
nc localhost 6667
nc localhost 6667

# Or use the netcat clone from Project 4:
zig run ../netcat-clone/src/main.zig localhost 6667
```

## Protocol

When a client connects, the server sends a welcome banner and prompts for a username. Once the username is entered, the client can type messages that are broadcast to all other connected clients.

| Command | Description |
|---------|-------------|
| `/quit` | Disconnect from the server |
| `/users` | Show the number of connected users |
| `/name <new_name>` | Change your username |

All other text is treated as a chat message and broadcast in the format `<username> message`.

## Key Design Decisions

The chat server uses a **shared `ChatState` struct** protected by a single mutex. While this is simple and correct, it means that broadcasting a message (which iterates over all clients and writes to each one) holds the mutex for the duration of all writes. In a production system, you would use finer-grained locking, per-client send queues, or an evented architecture. The **swap-remove pattern** for client removal (swapping the removed client with the last client in the array and decrementing the count) avoids the O(n) shift that would be required to maintain array order. The **best-effort broadcast** silently ignores write errors to individual clients, since a disconnected client should not crash the server or disrupt other clients' messages.

---

# Project 4: Netcat Clone

Netcat (`nc`) is the "Swiss army knife" of networking. It can create TCP connections, listen for incoming connections, and pipe data between the network and standard input/output. This project implements a subset of netcat's functionality in pure Zig.

## Directory Structure

```
netcat-clone/
├── build.zig
└── src/
    └── main.zig
```

## `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "znc",
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
    b.step("run", "Run znc (Zig netcat)").dependOn(&run_cmd.step);
}
```

## `src/main.zig`

```zig
const std = @import("std");
const net = std.Io.net;
const Io = std.Io;
const log = std.log;

const Mode = enum { connect, listen };

const Config = struct {
    mode: Mode,
    host: []const u8,
    port: u16,
    verbose: bool,
    udp: bool,
};

pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;
    const io = init.io;

    var args = init.minimal.args.iterate();
    _ = args.skip();

    const config = parseArgs(&args) catch {
        printUsage();
        return;
    };

    switch (config.mode) {
        .connect => try runClient(io, gpa, &config),
        .listen => try runServer(io, gpa, &config),
    }
}

fn parseArgs(args: *std.process.ArgIterator) !Config {
    var host: []const u8 = "127.0.0.1";
    var port_str: ?[]const u8 = null;
    var listen_mode = false;
    var verbose = false;
    var udp = false;

    while (args.next()) |arg| {
        if (std.mem.eql(u8, arg, "-h") or std.mem.eql(u8, arg, "--help")) {
            return error.HelpRequested;
        } else if (std.mem.eql(u8, arg, "-l") or std.mem.eql(u8, arg, "--listen")) {
            listen_mode = true;
        } else if (std.mem.eql(u8, arg, "-v") or std.mem.eql(u8, arg, "--verbose")) {
            verbose = true;
        } else if (std.mem.eql(u8, arg, "-u") or std.mem.eql(u8, arg, "--udp")) {
            udp = true;
        } else if (arg[0] != '-') {
            if (port_str == null) {
                port_str = arg;
            } else {
                host = port_str.?;
                port_str = arg;
            }
        }
    }

    const port = if (port_str) |p|
        std.fmt.parseInt(u16, p, 10) catch return error.InvalidPort
    else
        return error.MissingPort;

    return .{
        .mode = if (listen_mode) .listen else .connect,
        .host = host,
        .port = port,
        .verbose = verbose,
        .udp = udp,
    };
}

fn printUsage() void {
    std.debug.print(
        \\znc - Zig Netcat Clone
        \\
        \\Usage:
        \\  znc [options] <port>            Listen mode (with -l) or connect to localhost
        \\  znc [options] <host> <port>    Connect to host:port
        \\
        \\Options:
        \\  -l, --listen    Listen for incoming connections
        \\  -v, --verbose   Print connection info to stderr
        \\  -u, --udp       Use UDP instead of TCP
        \\  -h, --help      Show this help
        \\
        \\Examples:
        \\  znc -l 8080                  Listen on port 8080
        \\  znc example.com 80           Connect to example.com:80
        \\  znc -l -u 9000               Listen on UDP port 9000
        \\  echo "GET /" | znc 80        Pipe data to localhost:80
        \\
    , .{});
}

fn runClient(io: *Io, allocator: std.mem.Allocator, config: *const Config) !void {
    if (config.udp) {
        return runUdpClient(io, allocator, config);
    }

    const addr = net.IpAddress.parse(config.host, config.port) catch |err| {
        std.debug.print("Error: invalid address '{s}': {s}\n", .{ config.host, @errorName(err) });
        return err;
    };

    if (config.verbose) {
        std.debug.print("Connecting to {s}:{}...\n", .{ config.host, config.port });
    }

    const stream = addr.connect(io, .{ .mode = .stream }) catch |err| {
        std.debug.print("Error: connection failed: {s}\n", .{@errorName(err)});
        return err;
    };
    defer stream.close(io);

    if (config.verbose) {
        std.debug.print("Connected to {s}:{}\n", .{ config.host, config.port });
    }

    // Pipe stdin -> network (in a thread) and network -> stdout (in main)
    const pipe_ctx = PipeContext{ .stream = stream, .io = io };
    const net_to_stdout = std.Thread.spawn(.{}, pipeNetToStdout, .{pipe_ctx}) catch |err| {
        std.debug.print("Error: failed to spawn thread: {s}\n", .{@errorName(err)});
        return err;
    };
    defer net_to_stdout.join();

    // Pipe stdin -> network in the main thread
    try pipeStdinToNet(io, stream);

    // Give the other thread a moment to finish
    std.time.sleep(100 * std.time.ns_per_ms);
}

const PipeContext = struct {
    stream: net.Stream,
    io: *Io,
};

fn pipeStdinToNet(io: *Io, stream: net.Stream) !void {
    var send_buf: [4096]u8 = undefined;
    var writer = stream.writer(io, &send_buf);

    var read_buf: [4096]u8 = undefined;
    const stdin = std.io.getStdIn();
    while (true) {
        const n = stdin.read(&read_buf) catch |err| switch (err) {
            error.BrokenPipe => return, // stdin closed (pipe)
            error.ConnectionResetByPeer => return,
            else => return,
        };
        if (n == 0) return; // EOF

        writer.interface.writeAll(read_buf[0..n]) catch return;
        writer.interface.flush() catch return;
    }
}

fn pipeNetToStdout(ctx: PipeContext) void {
    var recv_buf: [4096]u8 = undefined;
    var reader = ctx.stream.reader(ctx.io, &recv_buf);
    const stdout = std.io.getStdOut();

    while (true) {
        const n = reader.interface.read(&recv_buf) catch return;
        if (n == 0) return;
        stdout.writeAll(recv_buf[0..n]) catch return;
    }
}

fn runUdpClient(io: *Io, allocator: std.mem.Allocator, config: *const Config) !void {
    const addr = net.IpAddress.parse(config.host, config.port) catch |err| {
        std.debug.print("Error: invalid address: {s}\n", .{@errorName(err)});
        return err;
    };

    // Bind to any available port
    const local = net.IpAddress.parseIp4("0.0.0.0", 0) catch unreachable;
    const sock = try local.bind(io, .{ .mode = .dgram, .protocol = .udp });
    defer sock.close(io);

    if (config.verbose) {
        std.debug.print("UDP mode: sending to {s}:{}\n", .{ config.host, config.port });
    }

    // Read from stdin and send as UDP datagrams
    var buf: [65507]u8 = undefined;
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();

    // Spawn a thread to receive
    const recv_ctx = UdpRecvContext{ .sock = sock, .io = io };
    const recv_thread = std.Thread.spawn(.{}, udpRecvToStdout, .{recv_ctx}) catch {
        std.debug.print("Warning: could not spawn receive thread\n", .{});
        // Fall through to send-only mode
        if (false) {
            _ = &udpRecvToStdout;
            _ = recv_ctx;
        }
    };
    defer if (@hasField(@TypeOf(recv_thread), "handle")) {
        recv_thread.join();
    };

    while (true) {
        const n = stdin.read(&buf) catch |err| switch (err) {
            error.BrokenPipe => return,
            else => return,
        };
        if (n == 0) return;

        sock.send(io, &addr, buf[0..n]) catch |err| {
            std.debug.print("Send error: {s}\n", .{@errorName(err)});
            return err;
        };
    }
}

const UdpRecvContext = struct {
    sock: net.DatagramSocket,
    io: *Io,
};

fn udpRecvToStdout(ctx: UdpRecvContext) void {
    var buf: [65507]u8 = undefined;
    const stdout = std.io.getStdOut();

    while (true) {
        const msg = ctx.sock.receive(ctx.io, &buf) catch return;
        stdout.writeAll(msg.data) catch return;
    }
}

fn runServer(io: *Io, allocator: std.mem.Allocator, config: *const Config) !void {
    if (config.udp) {
        return runUdpServer(io, allocator, config);
    }

    const addr = net.IpAddress.parseIp4(config.host, config.port) catch |err| {
        std.debug.print("Error: invalid address: {s}\n", .{@errorName(err)});
        return err;
    };

    var server = try addr.listen(io, .{ .reuse_address = true });
    defer server.deinit(io);

    if (config.verbose) {
        std.debug.print("Listening on {s}:{}\n", .{ config.host, config.port });
    }

    const stream = server.accept(io) catch |err| {
        std.debug.print("Error: accept failed: {s}\n", .{@errorName(err)});
        return err;
    };
    defer stream.close(io);

    if (config.verbose) {
        std.debug.print("Connection accepted.\n");
    }

    // Same as client: bidirectional pipe
    const pipe_ctx = PipeContext{ .stream = stream, .io = io };
    const net_to_stdout = std.Thread.spawn(.{}, pipeNetToStdout, .{pipe_ctx}) catch |err| {
        std.debug.print("Error: failed to spawn thread: {s}\n", .{@errorName(err)});
        return err;
    };
    defer net_to_stdout.join();

    try pipeStdinToNet(io, stream);
    std.time.sleep(100 * std.time.ns_per_ms);
}

fn runUdpServer(io: *Io, allocator: std.mem.Allocator, config: *const Config) !void {
    const addr = net.IpAddress.parse(config.host, config.port) catch |err| {
        std.debug.print("Error: invalid address: {s}\n", .{@errorName(err)});
        return err;
    };

    const sock = try addr.bind(io, .{ .mode = .dgram, .protocol = .udp });
    defer sock.close(io);

    if (config.verbose) {
        std.debug.print("UDP listening on {s}:{}\n", .{ config.host, config.port });
    }

    var buf: [65507]u8 = undefined;
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();
    var last_sender: ?net.IpAddress = null;

    // Main loop: receive datagrams and print them
    // Also read from stdin and send to last sender
    const recv_ctx = UdpServerRecvContext{
        .sock = sock,
        .io = io,
        .last_sender = &last_sender,
    };
    const recv_thread = std.Thread.spawn(.{}, udpServerRecv, .{recv_ctx}) catch {
        std.debug.print("Warning: could not spawn receive thread\n", .{});
        if (false) {
            _ = &udpServerRecv;
            _ = recv_ctx;
        }
    };
    defer if (@hasField(@TypeOf(recv_thread), "handle")) {
        recv_thread.join();
    };

    // Send from stdin
    while (true) {
        const n = stdin.read(&buf) catch |err| switch (err) {
            error.BrokenPipe => return,
            else => return,
        };
        if (n == 0) return;

        const sender = last_sender orelse continue;
        sock.send(io, &sender, buf[0..n]) catch continue;
    }
}

const UdpServerRecvContext = struct {
    sock: net.DatagramSocket,
    io: *Io,
    last_sender: *?net.IpAddress,
};

fn udpServerRecv(ctx: UdpServerRecvContext) void {
    var buf: [65507]u8 = undefined;
    const stdout = std.io.getStdOut();

    while (true) {
        const msg = ctx.sock.receive(ctx.io, &buf) catch return;
        ctx.last_sender.* = msg.from;
        stdout.writeAll(msg.data) catch return;
    }
}
```

## Building and Running

```bash
zig build

# Connect to an HTTP server (similar to nc example.com 80)
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | ./znc example.com 80

# Listen for a connection (similar to nc -l 8080)
./znc -l 8080

# Chat between two terminals
# Terminal 1:
./znc -l 7777
# Terminal 2:
./znc 127.0.0.1 7777

# Verbose mode
./znc -v -l 9000

# UDP mode
# Terminal 1:
./znc -l -u 5000
# Terminal 2:
echo "hello udp" | ./znc -u 127.0.0.1 5000

# Transfer files
# Terminal 1 (receiver):
./znc -l 9999 > received_file.txt
# Terminal 2 (sender):
./znc 127.0.0.1 9999 < file_to_send.txt
```

## Key Design Decisions

The netcat clone supports both **TCP and UDP** modes, controlled by the `-u` flag. In TCP mode, it creates a **bidirectional pipe** between stdin/stdout and the network socket using two threads: one that reads from stdin and writes to the socket, and another that reads from the socket and writes to stdout. This mirrors how real netcat works. In **listen mode**, the server accepts a single connection (unlike the chat server which accepts many), which is the default behavior of `nc -l`. The **UDP server mode** uses a "last sender" pattern where stdin data is sent to whichever peer last sent a datagram to the server, which is a reasonable default for interactive use. The **verbose flag** prints connection information to stderr (not stdout) so that it does not interfere with the data being piped through the connection.

---

# Appendix: Quick Reference

## `std.Io.net` API Summary (Zig 0.16)

### `IpAddress`

| Method | Description |
|--------|-------------|
| `IpAddress.parseIp4(host, port)` | Parse an IPv4 address from string |
| `IpAddress.parseIp6(host, port)` | Parse an IPv6 address from string |
| `IpAddress.parse(host, port)` | Auto-detect IPv4 or IPv6 |
| `addr.listen(io, options)` | Create a TCP server |
| `addr.connect(io, options)` | Create a TCP client connection |
| `addr.bind(io, options)` | Create a UDP socket |

### `net.Stream`

| Method | Description |
|--------|-------------|
| `stream.reader(io, buf)` | Create a buffered reader |
| `stream.writer(io, buf)` | Create a buffered writer |
| `stream.close(io)` | Close the connection |

### `net.DatagramSocket`

| Method | Description |
|--------|-------------|
| `sock.receive(io, buf)` | Receive a datagram |
| `sock.send(io, addr, data)` | Send a datagram |
| `sock.close(io)` | Close the socket |

### `std.http.Client`

| Method | Description |
|--------|-------------|
| `client.request(method, target, options)` | Create an HTTP request |
| `request.sendBodiless()` | Send a request without a body |
| `request.send(body, options)` | Send a request with a body |
| `request.receiveHead(buf)` | Read the response headers |
| `request.reader()` | Get a reader for the response body |

### `std.http.Server`

| Method | Description |
|--------|-------------|
| `http.Server.init(reader, writer)` | Initialize an HTTP server |
| `server.receiveHead()` | Parse the next HTTP request |
| `request.respond(body, options)` | Send an HTTP response |

## Common Error Codes

| Error | Meaning |
|-------|---------|
| `error.ConnectionRefused` | No server on the target port |
| `error.NetworkUnreachable` | Cannot route to destination |
| `error.TimedOut` | Operation timed out |
| `error.EndOfStream` | Remote end closed the connection |
| `error.BrokenPipe` | Write to a closed connection |
| `error.AddressInUse` | Port already bound |
| `error.FileNotFound` | Requested resource does not exist |
| `error.HttpConnectionClosing` | HTTP connection is closing |
| `error.ServerFull` | Server reached maximum clients |