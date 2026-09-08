# Hashing — A Complete Reference

## Table of Contents
1. [What is Hashing?](#1-what-is-hashing)
2. [Key Characteristics of Hashing](#2-key-characteristics-of-hashing)
3. [Use Cases of Hashing](#3-use-cases-of-hashing)
4. [Commonly Used Hashing Algorithms](#4-commonly-used-hashing-algorithms)
   - [4.1 Non-Cryptographic (General-Purpose) Hash Functions](#41-non-cryptographic-general-purpose-hash-functions)
   - [4.2 Cryptographic Hash Functions](#42-cryptographic-hash-functions)
   - [4.3 Password-Hashing Algorithms](#43-password-hashing-algorithms)
5. [Choosing the Right Algorithm — Quick Decision Guide](#5-choosing-the-right-algorithm--quick-decision-guide)

---

## 1. What is Hashing?

**Hashing** is the process of transforming an input (of arbitrary size — a string, file, object, or block of data) into a fixed-size output, called a **hash value**, **hash code**, or **digest**, using a mathematical function known as a **hash function**.

```
hash_function(input) → fixed-size output
```

Formally: `h: {0,1}* → {0,1}^n`

The input can be any length, but the output (`n` bits) is always the same size for a given algorithm — e.g., SHA-256 always produces a 256-bit digest, whether you hash one character or a 10 GB file.

Hashing is distinct from **encryption**: encryption is reversible (you can decrypt with a key), hashing is intentionally **one-way** — you cannot reconstruct the original input from the hash alone.

---

## 2. Key Characteristics of Hashing

| Characteristic | Description |
|---|---|
| **Deterministic** | The same input always produces the same hash output, every time, on every machine. |
| **Fixed output size** | Regardless of input length, the digest size is constant (e.g., 128, 160, 256, 512 bits). |
| **Fast computation** | A good hash function computes quickly for any input size — O(n) in input length. |
| **Pre-image resistance** | Given a hash `h`, it should be computationally infeasible to find any input `m` such that `hash(m) = h` (one-way property). *Cryptographic hashes only.* |
| **Second pre-image resistance** | Given input `m1`, it should be infeasible to find a different input `m2` such that `hash(m1) = hash(m2)`. *Cryptographic hashes only.* |
| **Collision resistance** | It should be infeasible to find *any two* distinct inputs that produce the same hash. *Cryptographic hashes only — general-purpose hashes accept some collisions.* |
| **Avalanche effect** | A tiny change in input (even one bit) should produce a drastically different, unrelated-looking output. |
| **Uniform distribution** | Hash values should be spread evenly across the output space to minimize clustering/collisions in hash tables. |
| **Non-reversibility** | Unlike encryption, there is no "unhashing" — the original data cannot be derived from the digest. |

> **Note:** Not every hash function needs *all* of these properties. Hash functions used in hash tables (e.g., Java's `hashCode()`, MurmurHash) prioritize **speed and distribution**, not collision resistance against a malicious attacker. Cryptographic hash functions (SHA-256, BLAKE3) must satisfy the security properties above.

---

## 3. Use Cases of Hashing

| Category | Use Case | Example |
|---|---|---|
| **Data structures** | Hash tables / hash maps for O(1) average lookup, insert, delete | `HashMap`, `dict`, `unordered_map` |
| **Data integrity** | Verifying a file/message hasn't been tampered with or corrupted in transit | Checksums, file download verification (SHA-256 checksums) |
| **Password storage** | Storing a hash of a password instead of the plaintext | bcrypt/Argon2 hash in a user database |
| **Digital signatures & certificates** | Hashing a message before signing it (signing the digest, not the full message) | TLS/SSL certificates, code signing |
| **Data deduplication** | Identifying duplicate files/blocks by comparing hashes instead of full content | Git object storage, backup systems (dedup by content hash) |
| **Caching** | Generating cache keys from request parameters | CDN cache keys, memoization keys |
| **Load balancing** | Consistent hashing to distribute requests/data across servers with minimal reshuffling on scale-up/down | Distributed caches (Memcached), sharded databases, CDNs |
| **Blockchain** | Linking blocks via hash pointers; proof-of-work mining | Bitcoin (SHA-256), Ethereum (Keccak-256) |
| **Digital forensics** | Verifying evidence integrity, identifying known files | MD5/SHA-1 hash matching against known-file databases |
| **Bloom filters / set membership** | Space-efficient probabilistic membership testing using multiple hash functions | Databases (avoid disk lookup for missing keys), spell checkers |
| **Version control** | Content-addressable storage — identifying commits/objects by hash | Git (SHA-1, moving to SHA-256) |
| **Rate limiting / sharding keys** | Deterministically mapping a user/IP to a bucket or partition | API rate limiters, database sharding |

---

## 4. Commonly Used Hashing Algorithms

### 4.1 Non-Cryptographic (General-Purpose) Hash Functions

Optimized for **speed and good distribution** — used in hash tables, checksums for accidental (not adversarial) corruption, and data structures. **Not safe against intentional collision attacks.**

| Algorithm | Variants | Hash Size | Pros | Cons | Typical Use Case |
|---|---|---|---|---|---|
| **Division/Modulo Hashing** | — | Depends on table size | Extremely simple; fast | Poor distribution if table size shares factors with data pattern; not a "real" hash algorithm, more a technique | Basic hash table indexing (`key % tableSize`) |
| **Multiplicative Hashing (Knuth's method)** | — | Word-sized (32/64-bit) | Good distribution with a well-chosen constant; simple | Sensitive to constant choice; not collision-resistant | Hash table indexing |
| **FNV (Fowler–Noll–Vo)** | FNV-1, FNV-1a | 32, 64, 128, 256, 512, 1024-bit | Very fast; simple to implement; decent distribution | Not adversarially collision-resistant; weaker avalanche than Murmur/xxHash | In-memory hash tables, checksums, DNS/networking code |
| **MurmurHash** | MurmurHash1, MurmurHash2, MurmurHash3 | 32, 64, 128-bit | Excellent speed/distribution trade-off; widely battle-tested | Not cryptographically secure; MurmurHash2 has known weaknesses vs. crafted input | Hash tables, Bloom filters, distributed systems (Cassandra, Elasticsearch) |
| **xxHash** | XXH32, XXH64, XXH3 | 32, 64, 128-bit | One of the fastest hash functions available (SIMD-optimized); excellent distribution | Not cryptographic | High-throughput checksums, LZ4 compression, data pipelines |
| **CityHash** | CityHash64, CityHash128 | 64, 128-bit | Very fast on short strings; good distribution | Superseded by FarmHash/xxHash by its own authors (Google) | Hash tables, in-memory indexing (legacy Google infra) |
| **Jenkins Hash (lookup3)** | one-at-a-time, lookup2, lookup3 | 32-bit (and 32+32 for two outputs) | Good avalanche behavior; simple, public domain | Slower than Murmur/xxHash for large inputs | Hash tables, Linux kernel data structures |
| **CRC32 (Cyclic Redundancy Check)** | CRC-32, CRC-32C | 32-bit | Extremely fast (often hardware-accelerated); great at detecting accidental bit errors | Trivially reversible/forgeable — **not** for security; higher collision rate than modern non-crypto hashes | Network packet checksums (Ethernet, ZIP, PNG), storage integrity checks |

### 4.2 Cryptographic Hash Functions

Designed to satisfy **pre-image, second pre-image, and collision resistance** — safe against an adversary deliberately trying to forge or find collisions (unless noted as broken below).

| Algorithm | Variants | Hash Size | Pros | Cons | Typical Use Case |
|---|---|---|---|---|---|
| **MD5** | — | 128-bit | Very fast; still fine for non-security checksums | **Cryptographically broken** — collisions can be generated in seconds; must not be used for security purposes | Legacy checksums, non-security file-integrity checks only |
| **SHA-1** | — | 160-bit | Faster than SHA-2; historically ubiquitous | **Broken** — practical collision attacks demonstrated (SHAttered, 2017); deprecated for security use | Legacy systems, Git object hashing (being phased out) |
| **SHA-2** | SHA-224, SHA-256, SHA-384, SHA-512, SHA-512/224, SHA-512/256 | 224–512-bit | Strong security margin; hardware-accelerated on most modern CPUs; industry standard | Somewhat slower than SHA-1/MD5; vulnerable to length-extension attacks (SHA-256/512, not the truncated variants) | TLS/SSL, digital signatures, Bitcoin proof-of-work, code signing, checksums |
| **SHA-3 (Keccak)** | SHA3-224, SHA3-256, SHA3-384, SHA3-512, SHAKE128, SHAKE256 (XOFs) | 224–512-bit (SHAKE = variable) | Different internal structure (sponge construction) than SHA-2 — immune to length-extension attacks; strong security margin | Slower than SHA-2 in pure software (faster in hardware); less widely adopted yet | Next-gen protocols, Ethereum (Keccak-256 variant), post-quantum readiness |
| **BLAKE2** | BLAKE2b (64-bit optimized), BLAKE2s (32-bit optimized) | up to 512-bit (BLAKE2b), up to 256-bit (BLAKE2s) | Faster than SHA-2/SHA-3 and MD5 in software; as secure as SHA-3 | Less battle-tested at massive internet scale than SHA-2 | File integrity tools, WireGuard VPN, password hashing building block |
| **BLAKE3** | — | 256-bit (extendable output) | Extremely fast (parallelizable, SIMD + tree hashing); modern design | Newer — less time in the field, smaller history of cryptanalysis | High-performance checksumming, content-addressed storage |
| **RIPEMD** | RIPEMD-128, RIPEMD-160, RIPEMD-256, RIPEMD-320 | 128–320-bit | Independent design from SHA family (diversity of trust) | Less widely used/analyzed than SHA-2 | Bitcoin address generation (RIPEMD-160 combined with SHA-256) |

### 4.3 Password-Hashing Algorithms

General-purpose cryptographic hashes (SHA-256, etc.) are **too fast** for password storage — that speed lets attackers brute-force offline at billions of guesses/second. Password-hashing algorithms are deliberately **slow and memory-hard**.

| Algorithm | Variants | Hash Size | Pros | Cons | Typical Use Case |
|---|---|---|---|---|---|
| **bcrypt** | — (cost factor tunable) | 184-bit (192-bit encoded output) | Adjustable work factor; battle-tested since 1999; resistant to GPU brute-force to a degree | Fixed max input length (72 bytes); not memory-hard, so still somewhat vulnerable to ASIC/FPGA attacks | Web application password storage |
| **scrypt** | — (N, r, p tunable) | Configurable (commonly 256-bit) | Memory-hard — expensive to brute-force with GPUs/ASICs | More complex tuning; can be misconfigured to be too weak or too resource-heavy | Password storage, cryptocurrency (Litecoin mining) |
| **Argon2** | Argon2d, Argon2i, Argon2id | Configurable (commonly 256-bit) | Winner of the 2015 Password Hashing Competition; tunable time/memory/parallelism cost; Argon2id balances side-channel resistance and GPU resistance | Newer than bcrypt (less legacy tooling); requires careful parameter tuning | **Current best-practice** for password storage (OWASP-recommended) |
| **PBKDF2** | PBKDF2-HMAC-SHA1, PBKDF2-HMAC-SHA256 | Configurable | Simple, NIST-approved, widely implemented (FIPS compliance contexts) | Not memory-hard — more vulnerable to GPU/ASIC cracking than bcrypt/Argon2 | Legacy/FIPS-compliant systems, WPA2 Wi-Fi key derivation |

---

## 5. Choosing the Right Algorithm — Quick Decision Guide

| If you need to... | Use |
|---|---|
| Build a hash table / hash map | MurmurHash3, xxHash, or your language's built-in `hashCode()` |
| Detect accidental data corruption (network/disk) | CRC32, xxHash |
| Verify file integrity / checksums for downloads | SHA-256 |
| Store user passwords securely | Argon2id (preferred) or bcrypt |
| Sign a document / build a digital signature | SHA-256 or SHA-3 |
| Build a blockchain or content-addressable store | SHA-256, Keccak-256, or BLAKE3 |
| Maximize raw hashing speed for large data (non-security) | BLAKE3 or xxHash |
| Avoid a known-broken algorithm | Do **not** use MD5 or SHA-1 for anything security-relevant |

---

*Reference document — hashing fundamentals, characteristics, use cases, and algorithm comparison.*
