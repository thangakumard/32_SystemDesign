# Encryption: A Complete Guide

## Table of Contents
1. [What is Encryption?](#1-what-is-encryption)
2. [How Encryption Works](#2-how-encryption-works)
3. [Types of Encryption](#3-types-of-encryption)
   - [3.1 Symmetric Encryption](#31-symmetric-encryption)
   - [3.2 Asymmetric Encryption](#32-asymmetric-encryption)
   - [3.3 Hybrid Encryption](#33-hybrid-encryption)
4. [Encryption vs. Hashing vs. Encoding](#4-encryption-vs-hashing-vs-encoding)
5. [Commonly Used Encryption Algorithms](#5-commonly-used-encryption-algorithms)
6. [Choosing the Right Approach](#6-choosing-the-right-approach)
7. [Summary](#7-summary)

---

## 1. What is Encryption?

**Encryption** is the process of transforming readable data (**plaintext**) into an unreadable form (**ciphertext**) using a mathematical algorithm (**cipher**) and a **key**, such that only someone holding the correct key can reverse the process (**decryption**) and recover the original data.

Core terminology:

| Term | Meaning |
|---|---|
| Plaintext | The original, readable data |
| Ciphertext | The scrambled, unreadable output |
| Key | A secret value that controls the cipher's transformation |
| Cipher / Algorithm | The mathematical procedure used to transform data |
| Key size | Bit-length of the key; generally, larger = harder to brute-force |

Encryption provides **confidentiality** — it does not, by itself, guarantee integrity (that data wasn't altered) or authenticity (that it came from who it claims to). Those are handled by complementary mechanisms like MACs, digital signatures, and AEAD modes (e.g., AES-GCM combines confidentiality + integrity).

---

## 2. How Encryption Works

At a high level, every encryption scheme follows the same pattern:

```
Plaintext + Key  --[ Encryption Algorithm ]-->  Ciphertext
Ciphertext + Key --[ Decryption Algorithm ]-->  Plaintext
```

1. **Key generation** — A key (or key pair) is generated using a cryptographically secure random process.
2. **Encryption** — The algorithm applies mathematical transformations (substitution, permutation, modular exponentiation, elliptic curve operations, etc.) to the plaintext, guided by the key, producing ciphertext.
3. **Transmission/Storage** — The ciphertext is sent over a network or stored on disk. Intercepting it without the key should reveal nothing useful.
4. **Decryption** — The receiving party, holding the correct key, runs the inverse transformation to recover the plaintext.

The strength of an encryption scheme rests on:
- **Algorithm design** — resistance to known cryptanalytic attacks.
- **Key length and randomness** — larger, truly random keys resist brute force.
- **Key management** — how keys are generated, distributed, rotated, and protected (often the weakest link in practice, not the math).

---

## 3. Types of Encryption

### 3.1 Symmetric Encryption

**Definition:** The **same secret key** is used for both encryption and decryption.

**How it works:** Sender and receiver must both possess the identical shared key beforehand. The plaintext is run through a block or stream cipher (e.g., AES) using that key to produce ciphertext; the same key run through the inverse operation recovers the plaintext.

**Use case:** Bulk data encryption where speed matters — encrypting files at rest, database columns, full-disk encryption, and the "data channel" portion of TLS after the handshake completes.

**Sample (Java — AES-256-GCM):**

```java
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import java.security.SecureRandom;
import java.util.Base64;

public class AesExample {
    public static void main(String[] args) throws Exception {
        // 1. Generate a 256-bit AES key
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256);
        SecretKey secretKey = keyGen.generateKey();

        // 2. Generate a random 96-bit IV/nonce (required for GCM)
        byte[] iv = new byte[12];
        new SecureRandom().nextBytes(iv);
        GCMParameterSpec gcmSpec = new GCMParameterSpec(128, iv);

        // 3. Encrypt
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, gcmSpec);
        byte[] cipherText = cipher.doFinal("Sensitive data".getBytes());

        // 4. Decrypt (same key, same IV)
        cipher.init(Cipher.DECRYPT_MODE, secretKey, gcmSpec);
        byte[] plainText = cipher.doFinal(cipherText);

        System.out.println("Ciphertext: " + Base64.getEncoder().encodeToString(cipherText));
        System.out.println("Decrypted:  " + new String(plainText));
    }
}
```

> **Note:** Never reuse an IV/nonce with the same key in GCM mode — doing so catastrophically breaks confidentiality and integrity.

---

### 3.2 Asymmetric Encryption

**Definition:** Uses a **mathematically linked key pair** — a **public key** (freely shared) and a **private key** (kept secret). Data encrypted with the public key can only be decrypted with the corresponding private key (and vice versa, for signing).

**How it works:** The public/private key relationship relies on "trapdoor" math problems that are easy to compute in one direction but computationally infeasible to reverse without the private key — e.g., factoring large primes (RSA) or the elliptic curve discrete logarithm problem (ECC).

**Use case:** Solving the key-distribution problem — securely exchanging a symmetric session key over an untrusted network (TLS handshake), digital signatures, code signing, and email encryption (PGP/S-MIME).

**Sample (Java — RSA-2048 with OAEP padding):**

```java
import java.security.*;
import javax.crypto.Cipher;
import java.util.Base64;

public class RsaExample {
    public static void main(String[] args) throws Exception {
        // 1. Generate an RSA key pair (2048-bit)
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048);
        KeyPair pair = keyGen.generateKeyPair();

        // 2. Encrypt with the PUBLIC key
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.ENCRYPT_MODE, pair.getPublic());
        byte[] cipherText = cipher.doFinal("Symmetric session key".getBytes());

        // 3. Decrypt with the PRIVATE key
        cipher.init(Cipher.DECRYPT_MODE, pair.getPrivate());
        byte[] plainText = cipher.doFinal(cipherText);

        System.out.println("Ciphertext: " + Base64.getEncoder().encodeToString(cipherText));
        System.out.println("Decrypted:  " + new String(plainText));
    }
}
```

> **Note:** RSA/ECC are computationally expensive relative to AES, so they're used to encrypt small payloads (like a symmetric key), not large data streams.

---

### 3.3 Hybrid Encryption

**Definition:** Combines asymmetric and symmetric encryption to get the best of both — asymmetric encryption's easy key exchange plus symmetric encryption's speed.

**How it works:** A random symmetric session key is generated, encrypted with the recipient's public key (asymmetric step), and sent alongside the bulk data — which is itself encrypted with the fast symmetric key. The recipient decrypts the small session key with their private key, then uses it to decrypt the bulk payload.

**Use case:** This is exactly how **TLS/HTTPS** works: the handshake uses asymmetric crypto (RSA/ECDHE) to agree on a shared secret, then the actual page/API traffic is encrypted with a symmetric cipher (AES-GCM or ChaCha20-Poly1305) for speed. PGP/GPG email encryption follows the same pattern.

```
Client                                   Server
  |--- "Hello, here's what I support" -->|
  |<-- Server certificate (public key) --|
  | Generate random session key          |
  | Encrypt session key w/ server's      |
  | public key ------------------------->|
  |                    Server decrypts session key with its private key
  |<==== All further traffic encrypted with AES using session key ====>|
```

---

## 4. Encryption vs. Hashing vs. Encoding

These three are frequently confused but serve different purposes:

| | Encryption | Hashing | Encoding |
|---|---|---|---|
| Reversible? | Yes, with the key | No (one-way) | Yes, no key needed |
| Purpose | Confidentiality | Integrity verification, password storage | Data format transformation (e.g., transmission-safe) |
| Example | AES, RSA | SHA-256, bcrypt | Base64, URL encoding |

Hashing is *not* a type of encryption — it's included here only to prevent a common mix-up.

---

## 5. Commonly Used Encryption Algorithms

| Algorithm | Type | Typical Key Size | Pros | Cons | Common Use Case |
|---|---|---|---|---|---|
| **AES** (Advanced Encryption Standard) | Symmetric (block) | 128 / 192 / 256-bit | Fast, hardware-accelerated (AES-NI), extensively vetted, current industry standard | Requires secure key distribution/exchange | TLS bulk data, disk/file encryption, database column encryption |
| **ChaCha20-Poly1305** | Symmetric (stream, AEAD) | 256-bit | Very fast in pure software (no special hardware needed), resistant to timing side-channels — great for mobile/IoT | Newer than AES, slightly less universal hardware support | TLS 1.3 on mobile devices, WireGuard VPN |
| **RSA** | Asymmetric | 2048 / 3072 / 4096-bit | Mature, extremely widely supported, enables both encryption and digital signatures | Slow on large data, needs large keys for adequate security, theoretically breakable by future quantum computers | TLS handshakes, digital certificates (X.509), code/document signing |
| **ECC** (e.g., ECDH, ECDSA) | Asymmetric | 256 / 384-bit (equiv. to much larger RSA keys) | Smaller keys for equivalent security → faster, less compute/battery use | More complex to implement correctly; curve choice matters (some curves have raised trust concerns) | Mobile/embedded TLS, cryptocurrency wallets, modern CAs |
| **Diffie-Hellman / ECDHE** | Asymmetric key *exchange* | 2048-bit (DH) / 256-bit (ECDHE) | Enables two parties to derive a shared secret over an insecure channel; ephemeral variants give forward secrecy | Vulnerable to man-in-the-middle without additional authentication | TLS key exchange, VPN protocols (IPsec, WireGuard) |
| **3DES** (Triple DES) | Symmetric (block) | 112-bit effective | Backward-compatible with legacy systems | Slow, deprecated, vulnerable to the Sweet32 birthday attack | Legacy financial/payment systems (being retired) |
| **DES** | Symmetric (block) | 56-bit | Historically important, simple design | Trivially brute-forced with modern hardware; deprecated | None recommended — legacy/historical only |
| **Blowfish / Twofish** | Symmetric (block) | Up to 448-bit (Blowfish) | Free of patents, flexible key length | Blowfish's 64-bit block size is vulnerable to birthday attacks on large data volumes | bcrypt password hashing (uses Blowfish internals), some legacy embedded systems |
| **RC4** | Symmetric (stream) | 40–2048-bit | Simple, historically fast | Cryptographically broken (biased keystream); banned in modern TLS | None recommended — historical only |

---

## 6. Choosing the Right Approach

| If you need to... | Use |
|---|---|
| Encrypt large files, disks, or database fields | **AES-256-GCM** (symmetric) |
| Securely exchange a key over the internet | **ECDHE** or **RSA** (asymmetric key exchange) |
| Sign a document/code to prove authenticity | **RSA** or **ECDSA** (asymmetric signatures) |
| Secure a mobile app's network traffic efficiently | **ChaCha20-Poly1305** (symmetric) |
| Build a secure protocol from scratch | **Hybrid**: asymmetric for key exchange + symmetric for data (this is what TLS does) |
| Store a password | Neither — use a **password hashing** function (bcrypt, scrypt, Argon2), not encryption |

---

## 7. Summary

- **Encryption** protects confidentiality by transforming plaintext into ciphertext using a key and algorithm.
- **Symmetric encryption** (AES, ChaCha20) is fast and ideal for bulk data but requires a pre-shared secret key.
- **Asymmetric encryption** (RSA, ECC) solves key distribution using public/private key pairs, at the cost of speed.
- **Hybrid encryption** — used by TLS/HTTPS and PGP — combines both: asymmetric crypto to exchange a key, symmetric crypto to encrypt the actual data.
- When picking an algorithm, prefer modern, actively-maintained standards (AES-256-GCM, ChaCha20-Poly1305, RSA-2048+/ECC-256+) and avoid deprecated ones (DES, 3DES, RC4).
- Security in practice usually fails at **key management** (generation, storage, rotation) far more often than at the underlying math.
