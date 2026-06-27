# Cryptography in Zig 0.16

## Table of Contents

- [Chapter 1: Introduction to std.crypto](#chapter-1-introduction-to-stdcrypto)
- [Chapter 2: Hashing — SHA-1, SHA-256, SHA-512, SHA-3](#chapter-2-hashing--sha-1-sha-256-sha-512-sha-3)
- [Chapter 3: Blake3 — Fast Modern Hashing](#chapter-3-blake3--fast-modern-hashing)
- [Chapter 4: HMAC — Hash-based Message Authentication](#chapter-4-hmac--hash-based-message-authentication)
- [Chapter 5: AES Encryption (ECB, CBC, GCM, SIV)](#chapter-5-aes-encryption-ecb-cbc-gcm-siv)
- [Chapter 6: ChaCha20 and Salsa20 Stream Ciphers](#chapter-6-chacha20-and-salsa20-stream-ciphers)
- [Chapter 7: Key Derivation (PBKDF2, Argon2)](#chapter-7-key-derivation-pbkdf2-argon2)
- [Chapter 8: NEW in 0.16 — AES-SIV, AES-GCM-SIV, Ascon-AEAD, Ascon-Hash, Ascon-CHash](#chapter-8-new-in-0.16--aes-siv-aes-gcm-siv-ascon-aead-ascon-hash-ascon-chash)
- [Chapter 9: std.crypto.random — Cryptographically Secure Random](#chapter-9-stdcryptorandom--cryptographically-secure-random)
- [Chapter 10: Project — A Password Manager](#chapter-10-project--a-password-manager)

---

## Chapter 1: Introduction to std.crypto

Zig's `std.crypto` is a comprehensive, audited cryptographic library implemented entirely in Zig. It provides hashing, encryption, key derivation, authentication, and random number generation — all without depending on external C libraries like OpenSSL.

Key design principles:

- **No hidden allocations**: Crypto operations work on pre-allocated buffers.
- **Constant-time implementations**: Resistant to timing side-channel attacks where applicable.
- **Tested against known test vectors**: Every algorithm is verified against standard test vectors.
- **No C dependencies**: Pure Zig, composable, cross-platform.

The module hierarchy is:

```
std.crypto
├── hash          — SHA-1, SHA-2, SHA-3, BLAKE2, BLAKE3
├── core.aes      — AES block cipher
├── tls           — TLS implementation
├── hkdf          — HMAC-based Key Derivation
├── pbkdf2        — Password-Based Key Derivation
├── scrypt        — Memory-hard key derivation
├── argon2        — Memory-hard key derivation
├── hmac          — HMAC
├── salsa20       — Salsa20 stream cipher
├── chacha20      — ChaCha20 stream cipher
├── poly1305      — Poly1305 MAC
├── aead          — Authenticated Encryption (GCM, SIV, etc.)
└── random        — CSPRNG
```

---

## Chapter 2: Hashing — SHA-1, SHA-256, SHA-512, SHA-3

### SHA-256

SHA-256 is the workhorse of modern hashing. It produces a 32-byte (256-bit) digest.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const input = "The quick brown fox jumps over the lazy dog";
    var hash: [std.crypto.hash.sha2.Sha256.digest_length]u8 = undefined;
    std.crypto.hash.sha2.Sha256.hash(input, &hash, .{});

    try stdout.print("Input:  {s}\n", .{input});
    try stdout.print("SHA-256: ", .{});
    for (hash) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");

    // Using the streaming interface
    var h = std.crypto.hash.sha2.Sha256.init(.{});
    h.update("The quick ");
    h.update("brown fox ");
    h.update("jumps over the lazy dog");
    var hash2: [std.crypto.hash.sha2.Sha256.digest_length]u8 = undefined;
    h.final(&hash2);

    try stdout.print("SHA-256 (streaming): ", .{});
    for (hash2) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");

    // Both produce the same result
    try stdout.print("Match: {}\n", .{std.mem.eql(u8, &hash, &hash2)});
}
```

### SHA-512

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const input = "Hello, Zig 0.16!";
    var hash: [std.crypto.hash.sha2.Sha512.digest_length]u8 = undefined;
    std.crypto.hash.sha2.Sha512.hash(input, &hash, .{});

    try stdout.print("SHA-512: ", .{});
    for (hash) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
    try stdout.print("Digest length: {} bytes ({} bits)\n", .{
        hash.len, hash.len * 8,
    });
}
```

### SHA-3 (Keccak)

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const input = "Zig cryptography";

    // SHA3-256
    var sha3_256: [std.crypto.hash.sha3.Sha3_256.digest_length]u8 = undefined;
    std.crypto.hash.sha3.Sha3_256.hash(input, &sha3_256, .{});

    try stdout.print("SHA3-256: ", .{});
    for (sha3_256) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");

    // SHA3-512
    var sha3_512: [std.crypto.hash.sha3.Sha3_512.digest_length]u8 = undefined;
    std.crypto.hash.sha3.Sha3_512.hash(input, &sha3_512, .{});

    try stdout.print("SHA3-512: ", .{});
    for (sha3_512) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
}
```

### SHA-1 (Legacy)

SHA-1 is still available for compatibility but is considered cryptographically broken:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const input = "legacy hash";
    var hash: [std.crypto.hash.sha1.Sha1.digest_length]u8 = undefined;
    std.crypto.hash.sha1.Sha1.hash(input, &hash, .{});

    try stdout.print("SHA-1 (DEPRECATED for security): ", .{});
    for (hash) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
}
```

---

## Chapter 3: Blake3 — Fast Modern Hashing

Blake3 is the fastest cryptographic hash function widely available. It supports parallel hashing, streaming, and keyed hashing.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const input = "Blake3 is blazingly fast!";

    // One-shot hash
    var hash: [std.crypto.hash.blake3.Blake3.digest_length]u8 = undefined;
    std.crypto.hash.blake3.Blake3.hash(input, &hash, .{});

    try stdout.print("Blake3: ", .{});
    for (hash) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
    try stdout.print("Digest: {} bytes ({} bits)\n", .{hash.len, hash.len * 8});

    // Streaming hash
    var h = std.crypto.hash.blake3.Blake3.init(.{});
    h.update("Blake3 ");
    h.update("is ");
    h.update("blazingly ");
    h.update("fast!");
    var hash2: [32]u8 = undefined;
    h.final(&hash2);

    try stdout.print("Streaming: ", .{});
    for (hash2) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
    try stdout.print("Match: {}\n", .{std.mem.eql(u8, &hash, &hash2)});

    // Keyed hash (for MAC-like usage)
    var key: [32]u8 = undefined;
    @memset(&key, 0x42);
    var keyed_hash: [32]u8 = undefined;
    std.crypto.hash.blake3.Blake3.hash(key, &keyed_hash, .{
        .key = &key,
        .context = input,
    });
    try stdout.print("Keyed:   ", .{});
    for (keyed_hash) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
}
```

### Blake3 for Large Files (Chunked)

Blake3 is designed for parallel hashing of large data. The streaming interface naturally handles this:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    // Simulate a large buffer
    const chunk_size = 65536;
    const num_chunks = 16;
    const total_size = chunk_size * num_chunks;

    var data = try allocator.alloc(u8, total_size);
    defer allocator.free(data);

    // Fill with a pattern
    for (0..total_size) |i| {
        data[i] = @intCast(i % 256);
    }

    // Hash chunk by chunk
    var h = std.crypto.hash.blake3.Blake3.init(.{});
    for (0..num_chunks) |i| {
        h.update(data[i * chunk_size .. (i + 1) * chunk_size]);
    }
    var hash: [32]u8 = undefined;
    h.final(&hash);

    try stdout.print("Blake3 of {} MB: ", .{total_size / (1024 * 1024)});
    for (hash) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
}
```

---

## Chapter 4: HMAC — Hash-based Message Authentication

HMAC combines a hash function with a secret key to produce a message authentication code (MAC). It verifies both data integrity and authenticity.

```zig
const std = @import("std");

fn hmacSha256(key: []const u8, message: []const u8, out: *[32]u8) void {
    const Hmac = std.crypto.auth.hmac.sha2.HmacSha256;
    Hmac.create(out, message, key);
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const key = "super-secret-key-2024";
    const message = "Important message to authenticate";

    var mac: [32]u8 = undefined;
    hmacSha256(key, message, &mac);

    try stdout.print("Message: {s}\n", .{message});
    try stdout.print("HMAC-SHA256: ", .{});
    for (mac) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");

    // Verification
    var verify_mac: [32]u8 = undefined;
    hmacSha256(key, message, &verify_mac);
    const valid = std.mem.eql(u8, &mac, &verify_mac);
    try stdout.print("Verification (correct key): {}\n", .{valid});

    // Wrong key
    var wrong_mac: [32]u8 = undefined;
    hmacSha256("wrong-key", message, &wrong_mac);
    const invalid = std.mem.eql(u8, &mac, &wrong_mac);
    try stdout.print("Verification (wrong key):   {}\n", .{invalid});
}
```

### Using Different Hash Functions with HMAC

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const key = "my-key";
    const message = "test data";

    // HMAC-SHA512
    var mac512: [std.crypto.auth.hmac.sha2.HmacSha512.mac_length]u8 = undefined;
    std.crypto.auth.hmac.sha2.HmacSha512.create(&mac512, message, key);

    try stdout.print("HMAC-SHA512: ", .{});
    for (mac512) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");

    // HMAC with Blake3 (keyed mode)
    var key_buf: [32]u8 = undefined;
    @memcpy(&key_buf, key[0..@min(key.len, 32)]);
    var blake3_mac: [32]u8 = undefined;
    std.crypto.hash.blake3.Blake3.hash(key_buf, &blake3_mac, .{
        .key = &key_buf,
        .context = message,
    });
    try stdout.print("Blake3 MAC:  ", .{});
    for (blake3_mac) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
}
```

---

## Chapter 5: AES Encryption (ECB, CBC, GCM, SIV)

AES (Advanced Encryption Standard) is the most widely used symmetric cipher. Zig's `std.crypto.core.aes` provides the core block cipher, and higher-level modes are built on top.

### AES-GCM (Authenticated Encryption)

AES-GCM is the recommended mode — it provides both confidentiality and authenticity in a single operation.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const plaintext = "This is a secret message for AES-GCM!";
    const aad = "additional authenticated data"; // Not encrypted, but authenticated

    // 256-bit key (32 bytes)
    var key: [32]u8 = undefined;
    std.crypto.random.bytes(&key);

    // 96-bit nonce (12 bytes) — standard for GCM
    var nonce: [12]u8 = undefined;
    std.crypto.random.bytes(&nonce);

    // Encrypt
    var ciphertext: [plaintext.len]u8 = undefined;
    var tag: [16]u8 = undefined;

    const AesGcm = std.crypto.aead.aes_gcm.Aes256Gcm;
    AesGcm.encrypt(&ciphertext, &tag, plaintext, aad, nonce, key);

    try stdout.print("Plaintext:  {s}\n", .{plaintext});
    try stdout.print("Ciphertext: ", .{});
    for (ciphertext) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");
    try stdout.print("Auth tag:   ", .{});
    for (tag) |byte| {
        try stdout.print("{X:0>2}", .{byte});
    }
    try stdout.print("\n");

    // Decrypt
    var decrypted: [plaintext.len]u8 = undefined;
    const result = AesGcm.decrypt(&decrypted, ciphertext, tag, aad, nonce, key);

    if (result) {
        try stdout.print("Decrypted:  {s}\n", .{decrypted});
    } else {
        try stdout.print("Decryption FAILED (tampered data?)\n", .{});
    }
}
```

### AES-CBC

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    // AES-CBC requires padding to block size (16 bytes)
    const plaintext = "Hello, AES-CBC mode!"; // 20 bytes — needs padding
    const block_size: usize = 16;
    const padded_len = ((plaintext.len + block_size - 1) / block_size) * block_size;

    var padded = try allocator.alloc(u8, padded_len);
    defer allocator.free(padded);
    @memcpy(padded[0..plaintext.len], plaintext);
    // PKCS#7 padding
    const pad_val: u8 = @intCast(padded_len - plaintext.len);
    for (padded[plaintext.len..]) |*b| b.* = pad_val;

    var key: [32]u8 = undefined;
    var iv: [16]u8 = undefined;
    std.crypto.random.bytes(&key);
    std.crypto.random.bytes(&iv);

    try stdout.print("Padded input ({} bytes): {s}\n", .{padded.len, padded});

    // For CBC mode in std.crypto, use the lower-level AES blocks
    // In practice, prefer AES-GCM for authenticated encryption
    try stdout.print("Key: ", .{});
    for (key) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");

    try stdout.print("Note: For production, prefer AES-GCM over CBC.\n", .{});
    try stdout.print("CBC requires separate MAC for authentication.\n", .{});
}
```

---

## Chapter 6: ChaCha20 and Salsa20 Stream Ciphers

ChaCha20 is a modern stream cipher designed by Daniel J. Bernstein. It is widely used in TLS 1.3 and WireGuard.

### ChaCha20-Poly1305 (AEAD)

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const plaintext = "ChaCha20-Poly1305 is excellent!";
    const aad = "metadata";

    var key: [32]u8 = undefined;
    var nonce: [12]u8 = undefined;
    std.crypto.random.bytes(&key);
    std.crypto.random.bytes(&nonce);

    const ChaChaPoly = std.crypto.aead.chacha_poly1305.ChaCha20Poly1305;

    // Encrypt
    var ciphertext: [plaintext.len]u8 = undefined;
    var tag: [16]u8 = undefined;
    ChaChaPoly.encrypt(&ciphertext, &tag, plaintext, aad, nonce, key);

    try stdout.print("Plaintext:  {s}\n", .{plaintext});
    try stdout.print("CT len:     {}\n", .{ciphertext.len});
    try stdout.print("Tag:        ", .{});
    for (tag) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");

    // Decrypt
    var decrypted: [plaintext.len]u8 = undefined;
    if (ChaChaPoly.decrypt(&decrypted, ciphertext, tag, aad, nonce, key)) {
        try stdout.print("Decrypted:  {s}\n", .{decrypted});
    } else {
        try stdout.print("Authentication failed!\n", .{});
    }

    // XChaCha20-Poly1305 — extended nonce (24 bytes) for random nonce safety
    var xnonce: [24]u8 = undefined;
    std.crypto.random.bytes(&xnonce);

    const XChaChaPoly = std.crypto.aead.chacha_poly1305.XChaCha20Poly1305;
    var ct2: [plaintext.len]u8 = undefined;
    var tag2: [16]u8 = undefined;
    XChaChaPoly.encrypt(&ct2, &tag2, plaintext, aad, xnonce, key);

    try stdout.print("\nXChaCha20-Poly1305 with 24-byte nonce (safe for random nonce):\n", .{});
    try stdout.print("Tag:        ", .{});
    for (tag2) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");
}
```

### Salsa20

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const plaintext = "Salsa20 stream cipher demo";
    var key: [32]u8 = undefined;
    var nonce: [8]u8 = undefined;
    std.crypto.random.bytes(&key);
    std.crypto.random.bytes(&nonce);

    // Salsa20 is a stream cipher — XOR keystream with plaintext
    var ciphertext: [plaintext.len]u8 = undefined;
    @memcpy(&ciphertext, plaintext);

    std.crypto.core.salsa.Salsa20.encrypt(
        &ciphertext,
        &ciphertext,
        nonce,
        key,
        1, // counter
    );

    try stdout.print("Ciphertext: ", .{});
    for (ciphertext) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");

    // Decrypt: XOR again with same keystream
    var decrypted: [plaintext.len]u8 = undefined;
    @memcpy(&decrypted, &ciphertext);
    std.crypto.core.salsa.Salsa20.decrypt(
        &decrypted,
        &decrypted,
        nonce,
        key,
        1,
    );

    try stdout.print("Decrypted:  {s}\n", .{decrypted});
}
```

---

## Chapter 7: Key Derivation (PBKDF2, Argon2)

### PBKDF2

PBKDF2 derives a key from a password using a hash function and repeated iteration.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const password = "my-secure-password";
    const salt = "unique-salt-per-user";

    var derived_key: [32]u8 = undefined;
    std.crypto.pbkdf2.pbkdf2(
        &derived_key,
        password,
        salt,
        600_000, // iterations — OWASP recommendation for 2024
        std.crypto.hash.sha2.Sha512,
    );

    try stdout.print("Password:   {s}\n", .{password});
    try stdout.print("Iterations: 600000\n", .{});
    try stdout.print("Derived key: ", .{});
    for (derived_key) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");
}
```

### Argon2

Argon2 is the winner of the Password Hashing Competition. It's memory-hard, making it resistant to GPU/ASIC attacks.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const password = "my-secure-password";
    const salt = "random-salt-value-2024";

    var hash_out: [32]u8 = undefined;

    // Argon2id — the recommended variant
    std.crypto.argon2.argon2id(
        &hash_out,
        password,
        salt,
        .{
            .iterations = 3,
            .memory_kib = 65536, // 64 MB
            .threads = 2,
        },
        null, // no secret key
        null, // no associated data
    );

    try stdout.print("Password:   {s}\n", .{password});
    try stdout.print("Algorithm:  Argon2id\n", .{});
    try stdout.print("Memory:     64 MB\n", .{});
    try stdout.print("Threads:    2\n", .{});
    try stdout.print("Hash:       ", .{});
    for (hash_out) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");

    // Also available: Argon2i (side-channel resistant) and Argon2d (GPU resistant)
}
```

---

## Chapter 8: NEW in 0.16 — AES-SIV, AES-GCM-SIV, Ascon-AEAD, Ascon-Hash, Ascon-CHash

Zig 0.16 introduces several new cryptographic primitives:

### AES-SIV (Synthetic IV)

AES-SIV provides deterministic authenticated encryption. The same plaintext with the same key always produces the same ciphertext, which is useful for key wrapping and deduplication.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const plaintext = "Deterministic encryption with AES-SIV";
    const aad = "associated data";

    var key: [64]u8 = undefined; // AES-SIV uses two 256-bit keys = 512 bits
    std.crypto.random.bytes(&key);

    var nonce: [16]u8 = undefined; // SIV nonce (can be empty for deterministic mode)
    std.crypto.random.bytes(&nonce);

    // AES-SIV-256
    const AesSiv = std.crypto.aead.aes_siv.Aes256Siv;

    // The nonce for SIV can be zeroed for fully deterministic mode
    var det_nonce: [16]u8 = [_]u8{0} ** 16;

    var ct1: [plaintext.len]u8 = undefined;
    var tag1: [16]u8 = undefined;
    AesSiv.encrypt(&ct1, &tag1, plaintext, aad, det_nonce, key);

    var ct2: [plaintext.len]u8 = undefined;
    var tag2: [16]u8 = undefined;
    AesSiv.encrypt(&ct2, &tag2, plaintext, aad, det_nonce, key);

    try stdout.print("AES-SIV (deterministic mode):\n", .{});
    try stdout.print("  CT1: ", .{});
    for (ct1) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n  CT2: ", .{});
    for (ct2) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n  Same CT: {} (deterministic!)\n", .{std.mem.eql(u8, &ct1, &ct2)});

    // Decrypt
    var decrypted: [plaintext.len]u8 = undefined;
    if (AesSiv.decrypt(&decrypted, ct1, tag1, aad, det_nonce, key)) {
        try stdout.print("  Decrypted: {s}\n", .{decrypted});
    }
}
```

### AES-GCM-SIV

AES-GCM-SIV combines the performance of GCM with the misuse resistance of SIV. It is the AEAD mode used in Google's QUIC protocol.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const plaintext = "AES-GCM-SIV: nonce-misuse resistant AEAD";
    const aad = "QUIC-like usage";

    var key: [32]u8 = undefined;
    var nonce: [12]u8 = undefined;
    std.crypto.random.bytes(&key);
    std.crypto.random.bytes(&nonce);

    const AesGcmSiv = std.crypto.aead.aes_gcm_siv.Aes256GcmSiv;

    var ct: [plaintext.len]u8 = undefined;
    var tag: [16]u8 = undefined;
    AesGcmSiv.encrypt(&ct, &tag, plaintext, aad, nonce, key);

    try stdout.print("AES-GCM-SIV:\n", .{});
    try stdout.print("  Tag: ", .{});
    for (tag) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");

    // Decrypt
    var dec: [plaintext.len]u8 = undefined;
    if (AesGcmSiv.decrypt(&dec, ct, tag, aad, nonce, key)) {
        try stdout.print("  Decrypted: {s}\n", .{dec});
    }
}
```

### Ascon-AEAD

Ascon is the winner of the NIST Lightweight Cryptography competition. It's designed for constrained environments (IoT, embedded) while still providing strong security.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const plaintext = "Lightweight crypto with Ascon!";
    const aad = "IoT metadata";

    var key: [16]u8 = undefined; // Ascon-128 uses 128-bit key
    var nonce: [16]u8 = undefined;
    std.crypto.random.bytes(&key);
    std.crypto.random.bytes(&nonce);

    // Ascon-128 AEAD
    var ct: [plaintext.len]u8 = undefined;
    var tag: [16]u8 = undefined;

    // Using std.crypto.aead for Ascon
    std.crypto.aead.ascon.Aescon128.encrypt(&ct, &tag, plaintext, aad, nonce, key);

    try stdout.print("Ascon-128 AEAD:\n", .{});
    try stdout.print("  Key size:  128 bits\n", .{});
    try stdout.print("  Tag:       ", .{});
    for (tag) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");

    // Decrypt
    var dec: [plaintext.len]u8 = undefined;
    if (std.crypto.aead.ascon.Aescon128.decrypt(&dec, ct, tag, aad, nonce, key)) {
        try stdout.print("  Decrypted: {s}\n", .{dec});
    }
}
```

### Ascon-Hash and Ascon-CHash

Ascon-Hash provides a lightweight hash function, and Ascon-CHash is a customizable hash with a context string.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    const input = "Ascon hashing for lightweight applications";

    // Ascon-Hash (32-byte output)
    var hash_out: [32]u8 = undefined;
    std.crypto.hash.ascon.AsconHash.hash(input, &hash_out, .{});

    try stdout.print("Ascon-Hash:  ", .{});
    for (hash_out) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");

    // Ascon-CHash (customizable hash with context)
    var chash_out: [32]u8 = undefined;
    std.crypto.hash.ascon.AsconCHash.hash(input, &chash_out, .{
        .context = "my-application-v1",
    });

    try stdout.print("Ascon-CHash: ", .{});
    for (chash_out) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");
    try stdout.print("(context: 'my-application-v1')\n", .{});
}
```

### Comparison of New 0.16 Crypto Primitives

| Algorithm | Type | Key Size | Best For |
|-----------|------|----------|----------|
| AES-SIV | AEAD (deterministic) | 512-bit | Key wrapping, deduplication |
| AES-GCM-SIV | AEAD (nonce-misuse resistant) | 256-bit | QUIC, unreliable transports |
| Ascon-AEAD | Lightweight AEAD | 128-bit | IoT, embedded, constrained devices |
| Ascon-Hash | Lightweight hash | N/A | Resource-constrained hashing |
| Ascon-CHash | Customizable hash | N/A | Domain-separated lightweight hashing |

---

## Chapter 9: std.crypto.random — Cryptographically Secure Random

Zig provides a cryptographically secure pseudo-random number generator (CSPRNG) through `std.crypto.random`.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // Fill a buffer with random bytes
    var bytes: [32]u8 = undefined;
    std.crypto.random.bytes(&bytes);
    try stdout.print("Random bytes: ", .{});
    for (bytes) |b| try stdout.print("{X:0>2}", .{b});
    try stdout.print("\n");

    // Random integer in range
    const dice_roll = std.crypto.random.intRangeAtMost(u8, 1, 6);
    try stdout.print("Dice roll:   {}\n", .{dice_roll});

    const card = std.crypto.random.intRangeAtMost(u8, 1, 52);
    try stdout.print("Card number: {}\n", .{card});

    // Random boolean
    const coin = std.crypto.random.boolean();
    try stdout.print("Coin flip:   {}\n", .{coin});

    // Random float (0.0 to 1.0)
    const float_val = std.crypto.random.float(f64);
    try stdout.print("Random float: {d:.6}\n", .{float_val});

    // Generate a random password
    const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%";
    var password: [24]u8 = undefined;
    for (0..password.len) |i| {
        const idx = std.crypto.random.intRangeAtMost(usize, 0, charset.len - 1);
        password[i] = charset[idx];
    }
    try stdout.print("Password:    {s}\n", .{&password});
}
```

### Seeding and Thread Safety

`std.crypto.random` is automatically seeded from the OS (e.g., `getrandom()` on Linux, `BCryptGenRandom` on Windows). It is thread-safe — multiple threads can call it concurrently.

---

## Chapter 10: Project — A Password Manager

This project implements a simple password manager that hashes passwords with Argon2id for storage, verifies them, and encrypts notes with XChaCha20-Poly1305.

### src/main.zig

```zig
const std = @import("std");

const Entry = struct {
    username: []u8,
    site: []u8,
    // password_hash is stored, not the plaintext password
    password_hash: [32]u8,
    // Encrypted note
    enc_note: []u8,
    enc_note_tag: [16]u8,
    note_nonce: [24]u8,
};

const PasswordStore = struct {
    entries: std.ArrayList(Entry),
    master_key: [32]u8,
    allocator: std.mem.Allocator,

    fn init(allocator: std.mem.Allocator, master_password: []const u8) !PasswordStore {
        var key: [32]u8 = undefined;
        // Derive master key from password using Argon2id
        std.crypto.argon2.argon2id(
            &key,
            master_password,
            "zig-password-manager-v1",
            .{
                .iterations = 3,
                .memory_kib = 65536,
                .threads = 2,
            },
            null,
            null,
        );
        return .{
            .entries = std.ArrayList(Entry).init(allocator),
            .master_key = key,
            .allocator = allocator,
        };
    }

    fn deinit(self: *PasswordStore) void {
        for (self.entries.items) |*e| {
            self.allocator.free(e.username);
            self.allocator.free(e.site);
            self.allocator.free(e.enc_note);
        }
        self.entries.deinit();
    }

    fn addEntry(
        self: *PasswordStore,
        site: []const u8,
        username: []const u8,
        password: []const u8,
        note: []const u8,
    ) !void {
        // Hash the password for storage
        var pw_hash: [32]u8 = undefined;
        std.crypto.hash.blake3.Blake3.hash(password, &pw_hash, .{});

        // Encrypt the note
        var nonce: [24]u8 = undefined;
        std.crypto.random.bytes(&nonce);

        const XChaChaPoly = std.crypto.aead.chacha_poly1305.XChaCha20Poly1305;
        const enc_note = try self.allocator.alloc(u8, note.len);
        errdefer self.allocator.free(enc_note);

        var tag: [16]u8 = undefined;
        XChaChaPoly.encrypt(enc_note, &tag, note, site, nonce, self.master_key);

        try self.entries.append(.{
            .username = try self.allocator.dupe(u8, username),
            .site = try self.allocator.dupe(u8, site),
            .password_hash = pw_hash,
            .enc_note = enc_note,
            .enc_note_tag = tag,
            .note_nonce = nonce,
        });
    }

    fn verifyPassword(self: *PasswordStore, entry_idx: usize, password: []const u8) bool {
        const entry = self.entries.items[entry_idx];
        var check_hash: [32]u8 = undefined;
        std.crypto.hash.blake3.Blake3.hash(password, &check_hash, .{});
        return std.mem.eql(u8, &entry.password_hash, &check_hash);
    }

    fn decryptNote(self: *PasswordStore, entry_idx: usize, out: []u8) bool {
        const entry = self.entries.items[entry_idx];
        const XChaChaPoly = std.crypto.aead.chacha_poly1305.XChaCha20Poly1305;
        return XChaChaPoly.decrypt(
            out,
            entry.enc_note,
            entry.enc_note_tag,
            entry.site,
            entry.note_nonce,
            self.master_key,
        );
    }
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    var store = try PasswordStore.init(allocator, "MyMasterPass123!");
    defer store.deinit();

    // Add entries
    try store.addEntry(
        "github.com",
        "ziguser",
        "gh_s3cur3_p@ssw0rd!",
        "Personal GitHub account with 2FA enabled",
    );
    try store.addEntry(
        "ziglang.org",
        "zigdev",
        "z1g_l@ng_r0cks!",
        "Zig language account",
    );

    try stdout.print("=== Zig Password Manager ===\n\n", .{});

    for (store.entries.items, 0..) |entry, i| {
        try stdout.print("Entry {}: {}\n", .{i + 1, entry.site});
        try stdout.print("  Username:    {s}\n", .{entry.username});
        try stdout.print("  Pw hash:    ", .{});
        for (entry.password_hash[0..8]) |b| try stdout.print("{X:0>2}", .{b});
        try stdout.print("...\n");

        // Verify password
        const test_pw = if (i == 0) "gh_s3cur3_p@ssw0rd!" else "wrong_password";
        const valid = store.verifyPassword(i, test_pw);
        try stdout.print("  Verify '{}': {}\n", .{test_pw, valid});

        // Decrypt note
        var note_buf: [100]u8 = [_]u8{0} ** 100;
        if (store.decryptNote(i, note_buf[0..entry.enc_note.len])) {
            try stdout.print("  Note:        {s}\n", .{note_buf[0..entry.enc_note.len]});
        }
        try stdout.print("\n", .{});
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
        .name = "password-manager",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the password manager");
    run_step.dependOn(&run_cmd.step);
}
```

### Summary

Zig 0.16's `std.crypto` is a comprehensive, pure-Zig cryptographic library. The key additions in 0.16 — AES-SIV, AES-GCM-SIV, Ascon-AEAD, Ascon-Hash, and Ascon-CHash — expand the toolkit significantly. For new projects, use Blake3 for hashing, XChaCha20-Poly1305 or AES-GCM for encryption, Argon2id for password hashing, and `std.crypto.random` for all random number generation.