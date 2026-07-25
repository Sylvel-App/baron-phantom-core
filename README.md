# BHP — Baron Handshake Protocol

`baron-phantom-core` — a post-quantum, hybrid end-to-end encryption core in Rust.
One implementation, usable from Android, iOS, Web (WASM), Desktop, Python and Node.js
through `flutter_rust_bridge` / UniFFI / wasm-bindgen.

> **Status:** v0.2 (development) · 37 tests green · **not yet audited — do not ship to production.**

---

## What it is

BHP is the crypto core behind the [Sylvel](https://sylvel.com) messenger. It bundles
three things that a secure 1:1 channel needs, in a single crate:

- a **4-round mutually-authenticated handshake** (post-quantum + classical hybrid),
- a **message ratchet** for forward secrecy and post-compromise security,
- an **envelope layer** that hides metadata (size, timing, cover traffic).

It composes **only standard primitives** — ML-KEM-768 (FIPS 203), ML-DSA-65 (FIPS 204),
X25519, BLAKE3, HKDF-SHA-256, ChaCha20-Poly1305. Nothing home-grown at the primitive
level; the design work is in how they are put together (see the security notice below).

---

## Design

### Handshake — 4 rounds, SIGMA-style

```
R1  COMMIT     Alice → Bob   commit to entropy + ephemeral keys, no identity yet
R2  CHALLENGE  Bob → Alice   Bob's ephemeral KEM/DH + his challenge
R3  REVEAL     Alice → Bob   open the commitment, prove identity (ML-DSA over transcript)
R4  CONFIRM    Bob → Alice   Bob proves identity, both derive the session keys
```

- **Entropy commitment before identity reveal** — Alice binds her entropy (BLAKE3 commit)
  before anyone learns who she is.
- **Hybrid key exchange** — ML-KEM-768 and X25519 run in parallel; the channel only breaks
  if *both* are broken.
- **Transcript-bound auth** — ML-DSA-65 signs the handshake transcript, not the derived key
  (Alice cannot know the final key until R4, when Bob's entropy is revealed).
- **Five-source IKM → HKDF-SHA-256 → `SessionKeys` (`key_send` / `key_recv` / `key_ratchet`).**

### Phantom Ratchet — message layer

- **Level 1:** BLAKE3 symmetric chain, one key per message → forward secrecy.
- **Level 2:** Kyber768 KEM ratchet, strict ping-pong → post-compromise security.
- AEAD ChaCha20-Poly1305, nonce derived from the message key. Header encryption (v0.2, opt-in).

### Phantom Envelope — metadata hiding

Fixed-size bucket padding · cover-traffic frames indistinguishable from real ones ·
randomized send jitter.

### Identity

Long-term `IdentityBundle` — ML-DSA-65 signing keys + X25519 DH keys. Generated on install,
never sent over the network. QR payload ≈ 2017 bytes.

---

## Security properties

| Property | How |
|---|---|
| Confidentiality + integrity | AEAD ChaCha20-Poly1305 under ratchet keys |
| Mutual authentication | ML-DSA-65 signatures over the handshake transcript |
| Forward secrecy | BLAKE3 symmetric-chain ratchet (per message) |
| Post-compromise security | Kyber768 KEM ratchet (ping-pong) |
| Post-quantum (harvest-now-decrypt-later) | ML-KEM-768 + X25519 hybrid, ML-DSA-65 signatures |
| Downgrade resistance | negotiated suite/version bound into the transcript |
| Metadata resistance | Phantom Envelope (padding + cover + jitter) |

Non-goals for now: **group crypto** (Sylvel uses MLS in the interim), multi-device fan-out,
`no_std`.

---

## Layout

```
src/
├── lib.rs          crate root, public re-exports
├── identity.rs     IdentityBundle (long-term keys + QR)
├── ephemeral.rs    per-session ephemeral KEM/DH bundle
├── kdf.rs          five-source IKM + HKDF → SessionKeys
├── handshake/      round1..round4 + transcript/signing (mod.rs)
├── ratchet/        Phantom Ratchet — mod.rs, header.rs, state.rs
├── envelope.rs     Phantom Envelope — padding, cover, jitter
├── api.rs          flutter_rust_bridge FFI surface
├── transport.rs    QUIC transport (feature "transport")
└── frb_generated.rs  auto-generated — do not hand-edit
tests/   handshake_e2e.rs · transport_e2e.rs (needs --features transport)
benches/ bhp_bench.rs (criterion)
bhp_dart/ generated Dart/Flutter package
```

Protocol source of truth: [`A_baron-phantom-core.md`](A_baron-phantom-core.md) ·
[`B_phantom-ratchet.md`](B_phantom-ratchet.md) · [`C_wire-format.md`](C_wire-format.md).

---

## Build & test

```bash
cargo build                        # core
cargo build --features transport   # + QUIC transport
cargo test                         # 33 tests (unit + handshake e2e)
cargo test --features transport    # + transport e2e
cargo bench                        # criterion → target/criterion/report/index.html

# Cross-compile / bindings
cargo build --target aarch64-linux-android --release
wasm-pack build --target web       # WASM
flutter_rust_bridge_codegen generate   # regenerate Dart bindings
```

## Benchmarks (reference hardware, July 2026)

| Op | Time |
|---|---|
| `identity_generate` | ~163 µs |
| `full_handshake` (4 rounds + finalize, 2× sign + 2× verify) | ~2.1 ms |
| `ratchet_message_roundtrip_256b` | ~3.4 µs |
| `envelope_seal_200b` | ~161 ns |

---

## ⚠️ Security notice

BHP is a **custom protocol composition**. The primitives are standard and vetted; the
handshake, entropy fusion and ratchet composition are **not**. Before any production use:

1. external cryptographic **protocol review**,
2. external **implementation audit**,
3. keep crypto-agility (suite ID) so a suite can be swapped without a protocol break.

Until then: **experimental. Do not rely on it for real secrets.**

---

## License

**Proprietary — All Rights Reserved.** © 2026 Baron. See spec docs for design rationale.
