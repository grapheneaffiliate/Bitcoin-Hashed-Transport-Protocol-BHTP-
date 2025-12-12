# Bitcoin-Hashed Transport Protocol (BHTP)

**A Censorship-Resistant Communication Protocol Using Blockchain-Derived Ephemeral Keys**

## Overview

The Bitcoin-Hashed Transport Protocol (BHTP) is a novel time-based obfuscation layer that renders encrypted network traffic statistically indistinguishable from random noise. By deriving ephemeral encryption keys from Bitcoin block hashes, BHTP eliminates the cryptographic handshakes and traffic signatures exploited by Deep Packet Inspection (DPI) systems for protocol identification and censorship.

## Key Features

- **Handshake-Free Encryption:** Keys derived from publicly observable blockchain data—no key exchange to fingerprint
- **Traffic Indistinguishability:** AES-256-GCM ciphertext with standardized padding appears as random bytes
- **Layered Security:** "Russian Doll" architecture separates transport obfuscation from payload confidentiality
- **Automatic Key Rotation:** ~10-minute key lifecycle tied to Bitcoin block production
- **Synchronization Tolerance:** 3-block lookback window handles propagation latency
- **Minimal Overhead:** ~0.2ms computational cost per message

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    BHTP Message                         │
├─────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────┐  │
│  │           Outer Layer (Transport)                 │  │
│  │           AES-256-GCM + BLAKE3(Blockchain)        │  │
│  │           Key Lifetime: ~10 minutes               │  │
│  │           Purpose: Censorship Resistance          │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │         Inner Layer (Payload)               │  │  │
│  │  │         NIP-44 / XChaCha20-Poly1305         │  │  │
│  │  │         Key Lifetime: Indefinite            │  │  │
│  │  │         Purpose: Confidentiality            │  │  │
│  │  │  ┌───────────────────────────────────────┐  │  │  │
│  │  │  │         Original Message              │  │  │  │
│  │  │  └───────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Quick Start

### Key Derivation

```rust
use blake3::Hasher;

pub fn derive_transport_key(
    block_hash: &[u8; 32],
    prev_hash: &[u8; 32],
    timestamp: u64,
) -> [u8; 32] {
    let mut hasher = Hasher::new();
    hasher.update(block_hash);
    hasher.update(prev_hash);
    hasher.update(&timestamp.to_be_bytes());
    *hasher.finalize().as_bytes()
}
```

### Encryption Flow

1. Encrypt message with NIP-44 (recipient's public key)
2. Pad to bucket size (1KB / 16KB / 256KB / 1MB)
3. Fetch current Bitcoin block header
4. Derive transport key: `K = BLAKE3(Hₙ ‖ Hₙ₋₁ ‖ Tₙ)`
5. Encrypt with AES-256-GCM
6. Publish as Nostr Kind 10059 event

### Decryption Flow

1. Extract block hash from `h` tag
2. Derive transport key (with lookback fallback)
3. Decrypt outer layer (AES-256-GCM)
4. Strip padding
5. Decrypt inner layer (NIP-44)

## Event Structure

```json
{
  "kind": 10059,
  "created_at": 1702300800,
  "tags": [
    ["h", "000000000000000000024bead8df69990852c202db0e0097c1a12ea637d7e96d"],
    ["p", "recipient_pubkey_hex"],
    ["iv", "random_nonce_hex"]
  ],
  "content": "base64_encoded_ciphertext...",
  "pubkey": "sender_pubkey_hex",
  "sig": "schnorr_signature_hex"
}
```

## Security Model

| Property | Outer Layer | Inner Layer |
|----------|-------------|-------------|
| Algorithm | AES-256-GCM | XChaCha20-Poly1305 |
| Key Source | BLAKE3(Blockchain) | ECDH (secp256k1) |
| Key Lifetime | ~10 minutes | Indefinite |
| Provides | Obfuscation | Confidentiality |
| Recoverable By | Anyone (public chain) | Private key holder only |

**Critical:** The outer layer provides censorship resistance, NOT long-term secrecy. All sensitive data MUST be encrypted with NIP-44 before BHTP wrapping.

## Padding Buckets

| Level | Size | Use Case |
|-------|------|----------|
| 1 | 1 KiB | Text messages, chat |
| 2 | 16 KiB | Rich text, code snippets |
| 3 | 256 KiB | Images, documents |
| 4 | 1 MiB | High-resolution media |

Clients MUST pad to the next largest bucket. A 1.1 KiB message pads to 16 KiB.

## Requirements

### Rust
```toml
[dependencies]
blake3 = "1.5"
aes-gcm = "0.10"
bitcoin = "0.31"
```

### JavaScript
```bash
npm install blake3 @noble/ciphers bitcoinjs-lib
```

### Python
```bash
pip install blake3 cryptography python-bitcoinlib
```

## Performance

| Metric | Value |
|--------|-------|
| Key derivation | ~50 ns |
| Encryption (1 KB) | ~150 ns |
| Total overhead | ~0.2 ms |
| Throughput | 5,000+ msg/s |

Benchmarks on AMD Ryzen 5 3600 with AES-NI.

## Specification

The complete technical specification includes:

- Formal cryptographic construction
- Threat model and security analysis
- Protocol procedures (RFC 2119 language)
- Lookback window algorithm
- Failure mode handling
- Security proofs
- Reference implementation

See the full specification document for details.

## Related Work

- **Tor Pluggable Transports:** obfs4, meek, snowflake
- **Steganographic Systems:** StegoTorus, SkypeMorph
- **Decoy Routing:** Telex, TapDance, Refraction Networking
- **Mix Networks:** Mixmaster, Nym

BHTP differs by eliminating key exchange entirely via blockchain-derived keys.

## Citation

```bibtex
@techreport{mcgirl2025bhtp,
  author = {McGirl, Timothy},
  title = {The Bitcoin-Hashed Transport Protocol: A First-Principles Approach to Metadata-Resistant Communication},
  year = {2025},
  month = {December},
  institution = {Independent Research},
  type = {Technical Specification},
  version = {1.1}
}
```

## License

This work is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

## Author

**Timothy McGirl**  
Independent Researcher  
December 2025
