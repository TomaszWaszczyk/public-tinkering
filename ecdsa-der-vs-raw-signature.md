# The 69-vs-64 signature bug: DER-encoded ECDSA vs. raw `r‖s`

## The symptom

You see a signature verification fail with a message like:

```
data signature verification failed: invalid signature length: got 69, want 64
```

The verifier rejected the signature before it even did any cryptography — purely on the byte length. `64` is the length it expected; `69` is the length it got. Nothing is wrong with the key, the message, or the math. The two sides simply disagree on **how the signature is encoded**.

## Why 64 bytes?

A verifier that wants exactly **64 bytes** is expecting a *fixed-length raw* signature — the two components `r` and `s` concatenated, each padded to a fixed width:

- **ECDSA over the NIST P-256 curve (secp256r1):** `r` and `s` are each 256-bit (32-byte) integers, so the raw form is `r‖s` = 32 + 32 = **64 bytes**.
- **Ed25519:** the signature is defined by the spec as a fixed **64-byte** value (`R‖s`). Ed25519 has no other legal length, so an Ed25519 verifier will *only* ever accept 64 bytes.

Either way, "want 64" means the code expects a compact, fixed-width layout with no framing or metadata.

## Why 69 bytes?

A signature of ~70 bytes is the hallmark of **DER / ASN.1 encoding**, which is what most X.509, TLS, OpenSSL, Java `Signature`, and WebCrypto-in-some-modes stacks produce for ECDSA. DER wraps the same `r` and `s` in a self-describing structure:

```
SEQUENCE {
  INTEGER r
  INTEGER s
}
```

Serialized, that looks like:

```
30 <len> 02 <len_r> <r bytes> 02 <len_s> <s bytes>
```

The overhead comes from the tags and lengths:

- `30 <len>` — the SEQUENCE tag + total length (2 bytes)
- `02 <len_r>` — INTEGER tag + length for `r` (2 bytes)
- `02 <len_s>` — INTEGER tag + length for `s` (2 bytes)

That's ~6 bytes of framing on top of the value bytes. Because DER integers are **signed** (two's complement), a value whose top bit is set gets a leading `0x00` byte so it isn't read as negative. Depending on how many leading zero bytes `r` and `s` need, a P-256 DER signature lands anywhere in roughly the **68–72 byte** range. **69 bytes is squarely in that window.**

So `got 69` is almost certainly a DER-encoded ECDSA signature that was handed to a verifier expecting raw `r‖s` (or expecting Ed25519).

## The root cause

This is an **encoding mismatch, not a cryptographic failure**. It usually happens at an integration seam where a signature crosses between two ecosystems that made different default choices:

- The **signer** (a device, an HSM, OpenSSL, Java, a certificate toolchain) emits **DER**.
- The **verifier** (often Go's `ed25519.Verify`, a raw-P256 check, WebCrypto's `ECDSA`, or a fixed-size buffer parser) expects **raw 64 bytes**.

Both are "correct" in isolation. Neither auto-detects the other's format, so the length check trips immediately.

A close cousin of this bug is an **algorithm mismatch**: the data was signed with ECDSA but the verifier was configured for Ed25519 (or vice versa). Because both algorithms share the "64 bytes for raw" number, a length error can also be the first visible sign that the *wrong verifier* was selected for the key type. Always confirm the signing key's algorithm, not just the length.

## The fix

Fix the cause — make both sides agree on one encoding. Do **not** paper over it with a try/catch that swallows the error.

1. **Confirm the algorithm first.** Is the signing key ECDSA-P256 or Ed25519? Ed25519 is *only* ever 64 raw bytes and never DER-wrapped, so if you're seeing 69 bytes the signer is doing ECDSA. Make sure the verifier uses the matching algorithm.

2. **Normalize the encoding at the boundary.** If the verifier wants raw `r‖s`, convert the incoming DER signature once, right where it enters your system:
   - Parse the DER `SEQUENCE` into the two integers `r` and `s`.
   - Left-pad each to the curve's fixed width (32 bytes for P-256), stripping any DER leading-zero sign byte.
   - Concatenate to the 64-byte raw form and hand *that* to the verifier.
   - (The reverse — raw → DER — is equally valid if it's the *verifier* that wants DER.)

3. **Pick the direction deliberately.** Convert toward whatever the verifier can't change (e.g. a fixed hardware or spec-defined verifier), and do the conversion in one well-named function at the seam rather than scattering length checks around.

### Illustrative conversion (Go)

```go
import (
    "crypto/ecdsa"
    "encoding/asn1"
    "math/big"
)

// derToRaw converts a DER-encoded ECDSA signature into a fixed-length
// r||s signature for a curve whose order is `byteLen` bytes wide
// (32 for P-256). It returns an error on malformed input rather than
// silently accepting it.
func derToRaw(der []byte, byteLen int) ([]byte, error) {
    var sig struct{ R, S *big.Int }
    if _, err := asn1.Unmarshal(der, &sig); err != nil {
        return nil, err
    }
    out := make([]byte, 2*byteLen)
    sig.R.FillBytes(out[:byteLen])   // left-pads with zeros
    sig.S.FillBytes(out[byteLen:])   // left-pads with zeros
    return out, nil
}
```

`FillBytes` does the fixed-width left-padding for you and drops the DER sign byte, which is exactly the step naive `r.Bytes()` concatenation gets wrong (it produces variable-length output and reintroduces the mismatch).

## How to recognize it next time

| Observation | Likely meaning |
|---|---|
| `got 64, want 64` but verify still fails | Right encoding, wrong key/algorithm/message |
| `got 70–72, want 64` | DER ECDSA P-256 → needs raw `r‖s` conversion |
| **`got 69, want 64`** | **DER ECDSA (short `r`/`s`) → needs raw conversion** |
| `got 71, want 64` for Ed25519 verifier | Wrong algorithm entirely — key is ECDSA, verifier is Ed25519 |
| Length varies run-to-run (68–72) | Confirms DER (leading-zero trimming changes the length) |

The variable length is the giveaway: raw `r‖s` and Ed25519 are always a constant size, while DER breathes by a byte or two depending on the integer values. The moment you see a signature length that isn't your expected constant — and especially one that wobbles between runs — reach for a DER↔raw conversion at the boundary before suspecting the crypto itself.
