## 🧪 Galaxy WASM Host Stub Spec v0.1

Deterministic reference host for **testing** cGalaxy contracts compiled to WASM. This host stub is **not production runtime**; it is the canonical behavior for compiler golden tests and SDK consistency.

### 1) Purpose

The host stub provides deterministic implementations of:

* **storage**: key/value bytes
* **events**: append-only logs
* **crypto**: sha256, blake2b256, verify_ed25519
* **context**: sender, chain_id, tx_hash, block_height
* **gas**: remaining + tick (metering)
* **memory allocation**: `env.alloc`

It MUST behave identically across machines for the same test vector inputs.

## A. Determinism Rules (Hard)

A.1 No wall-clock time. No randomness. No OS-dependent behavior.
A.2 All outputs depend only on:

* contract input bytes
* host stub initial state (storage map, context values, gas budget, memory allocation pointer)
  A.3 Storage iteration order must never affect results (use sorted keys only when needed for debugging output).
  A.4 Crypto must use standard deterministic algorithms.
  A.5 Gas behavior must be exact and integer-only.

## B. Host Imports (WASM ABI)

All imports are in module `"env"`.

### B.1 Memory Allocation

**`env.alloc(len: i32) -> i32`**

* Allocates `len` bytes in the contract linear memory.
* Returns pointer (offset) `ptr` to start of allocated region.
* Deterministic “bump allocator” semantics in host:

  * `heap_ptr` starts at a fixed value (recommended `65536` = 64KiB) for tests
  * `heap_ptr` is aligned to 8 bytes after each allocation
* Failure:

  * If `len < 0` return `0`
  * If allocation exceeds configured max memory limit, return `0`

**Alignment rule**

* `heap_ptr = (heap_ptr + 7) & ~7` after increment.

### B.2 Storage (KV)

Keys and values are raw bytes.

**`env.storage_has(key_ptr: i32, key_len: i32) -> i32`**

* Returns `1` if key exists, else `0`
* If key_len < 0 or key_ptr < 0 => return `0` (treat as missing; deterministic)

**`env.storage_get(key_ptr: i32, key_len: i32) -> i64`**

* If missing: returns packed slice `(ptr=0,len=0)` (packed i64 = 0)
* If present:

  * Allocate `val_len` bytes via `env.alloc`
  * Copy value bytes into contract memory at `ptr`
  * Return packed `(ptr,len)`
* If allocation fails: return `(0,0)` (same as missing) **and** the contract should treat this as internal error if it needs strictness; v0.1 keeps it simple/deterministic.

**`env.storage_put(key_ptr: i32, key_len: i32, val_ptr: i32, val_len: i32) -> i32`**

* Copies key/value bytes out of contract memory into host storage map.
* Returns:

  * `0` on success
  * `5` (`ERR_STORAGE`) on invalid pointers/lengths or size policy violation

**`env.storage_del(key_ptr: i32, key_len: i32) -> i32`**

* Deletes if present.
* Returns `0` always unless invalid pointers => `5`

**Size policies (test defaults)**

* max key size: 1024 bytes
* max value size: 262144 bytes (256KiB)

### B.3 Events

**`env.emit_event(topic_ptr: i32, topic_len: i32, data_ptr: i32, data_len: i32) -> i32`**

* Appends an event record to an in-memory list:

  * `{ topic_bytes, data_bytes }`
* Returns:

  * `0` on success
  * `8` (`ERR_INTERNAL`) on invalid pointers/lengths or policy violation

**Size policies (test defaults)**

* max topic size: 4096 bytes
* max data size: 32768 bytes (32KiB)
* max events per call: 1024 (to prevent runaway tests)

Events do not change contract-visible state except for their existence in the host log (used by tests).

### B.4 Crypto

All crypto functions are pure/deterministic.

**`env.sha256(data_ptr: i32, data_len: i32) -> i64`**

* Computes SHA-256 digest (32 bytes)
* Allocates 32 bytes in contract memory
* Writes digest bytes
* Returns packed `(ptr, 32)`
* If invalid pointer/len or alloc fails => returns (0,0)

**`env.blake2b256(data_ptr: i32, data_len: i32) -> i64`**

* Computes BLAKE2b-256 digest (32 bytes)
* Same allocation/copy rules as sha256

**`env.verify_ed25519(pub_ptr: i32, pub_len: i32, msg_ptr: i32, msg_len: i32, sig_ptr: i32, sig_len: i32) -> i32`**

* Expects:

  * pub_len = 32
  * sig_len = 64
* Returns:

  * `1` if signature valid
  * `0` otherwise (including invalid lengths, invalid pointers)

### B.5 Context

Returns deterministic, preconfigured bytes/values per test vector.

**`env.ctx_sender() -> i64`**

* Returns packed slice for sender bytes

**`env.ctx_chain_id() -> i64`**

* Returns packed slice for chain id bytes

**`env.ctx_tx_hash() -> i64`**

* Returns packed slice for tx hash bytes

**`env.ctx_block_height() -> i64`**

* Returns `u64` encoded in an `i64` (non-negative)

**Default test values**

* sender: 20 bytes `0x1111...11` (20 bytes)
* chain_id: ASCII `b"galaxy-testnet"` (or configured)
* tx_hash: 32 bytes `0x2222...22`
* block_height: 1

(Tests can override these per fixture.)

### B.6 Gas / Metering

**`env.gas_remaining() -> i64`**

* Returns current remaining gas as `u64` in `i64`

**`env.gas_tick(cost: i64) -> i32`**

* If `cost < 0`: return `0`
* If remaining >= cost:

  * remaining -= cost
  * return `1`
* Else:

  * remaining unchanged
  * return `0`

**Important**

* Host calls themselves may also be charged in the test harness.
  Recommended approach for deterministic tests:

  * contract IR already inserts `gas_tick` before loops/expensive ops
  * host stub charges additional “hostcall base costs” **only if** your runtime model requires it
    For v0.1 golden tests, keep it consistent: **either** (A) charge only via `gas_tick` inserted by compiler/runtime **or** (B) add hostcall costs in host stub.
    If you choose (B), apply the cost table deterministically before executing the hostcall and fail by returning the hostcall’s “failure shape” when gas runs out.

**Recommended v0.1 for simplicity:**

* Charge gas only via explicit `env.gas_tick` calls.

## C. Packed Slice Helpers

Packed `i64` format:

* low 32 bits: ptr (u32)
* high 32 bits: len (u32)

Pack:

* `packed = (u64(len) << 32) | u64(ptr)`

Unpack:

* `ptr = packed & 0xFFFF_FFFF`
* `len = packed >> 32`

## D. Host Stub State Model (for tests)

The harness should expose a deterministic state object:

* `storage: Map<Vec<u8>, Vec<u8>>`
* `events: Vec<{topic: Vec<u8>, data: Vec<u8>}>`
* `context: { sender: Vec<u8>, chain_id: Vec<u8>, tx_hash: Vec<u8>, block_height: u64 }`
* `gas_remaining: u64`
* `heap_ptr: u32`
* `memory_limit_bytes: u32`

The harness loads fixtures from a JSON file with base16 strings and numeric gas.

# ✅ Result Envelope Encode/Decode (Reference)

## 1) Envelope Format (canonical)

A contract returns a `Bytes` whose first byte is a tag:

* `0x00` = **Ok**
  Payload: `0x00 || <ok_bytes...>`
* `0x01` = **Err**
  Payload: `0x01 || <u32 error_code little-endian>`

Error codes are `u32` in the envelope. Your earlier standard codes map into these u32 values.

## 2) Rust Reference Code

This is dependency-free and works in `std` only.

```rust
// Galaxy Result Envelope v0.1
// Ok:  0x00 || payload
// Err: 0x01 || u32_le(error_code)

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum GalaxyResult {
    Ok(Vec<u8>),
    Err(u32),
}

pub const TAG_OK: u8 = 0x00;
pub const TAG_ERR: u8 = 0x01;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum EnvelopeError {
    Empty,
    UnknownTag(u8),
    ErrWrongLength(usize),
}

pub fn encode_envelope(res: GalaxyResult) -> Vec<u8> {
    match res {
        GalaxyResult::Ok(payload) => {
            let mut out = Vec::with_capacity(1 + payload.len());
            out.push(TAG_OK);
            out.extend_from_slice(&payload);
            out
        }
        GalaxyResult::Err(code) => {
            let mut out = Vec::with_capacity(1 + 4);
            out.push(TAG_ERR);
            out.extend_from_slice(&code.to_le_bytes());
            out
        }
    }
}

pub fn decode_envelope(bytes: &[u8]) -> Result<GalaxyResult, EnvelopeError> {
    if bytes.is_empty() {
        return Err(EnvelopeError::Empty);
    }
    match bytes[0] {
        TAG_OK => Ok(GalaxyResult::Ok(bytes[1..].to_vec())),
        TAG_ERR => {
            if bytes.len() != 1 + 4 {
                return Err(EnvelopeError::ErrWrongLength(bytes.len()));
            }
            let mut b = [0u8; 4];
            b.copy_from_slice(&bytes[1..5]);
            Ok(GalaxyResult::Err(u32::from_le_bytes(b)))
        }
        other => Err(EnvelopeError::UnknownTag(other)),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ok_roundtrip() {
        let x = GalaxyResult::Ok(vec![1, 2, 3]);
        let enc = encode_envelope(x.clone());
        let dec = decode_envelope(&enc).unwrap();
        assert_eq!(x, dec);
    }

    #[test]
    fn err_roundtrip() {
        let x = GalaxyResult::Err(7);
        let enc = encode_envelope(x.clone());
        let dec = decode_envelope(&enc).unwrap();
        assert_eq!(x, dec);
    }

    #[test]
    fn err_length_rejected() {
        let bad = [TAG_ERR, 0, 0, 0]; // 1+3
        assert!(matches!(decode_envelope(&bad), Err(EnvelopeError::ErrWrongLength(_))));
    }
}
```

## 3) TypeScript Reference Code

Works in Node and browser. Uses `Uint8Array` only (no Node-specific `Buffer` needed).

```ts
// Galaxy Result Envelope v0.1
// Ok:  0x00 || payload
// Err: 0x01 || u32_le(error_code)

export type GalaxyResult =
  | { ok: true; value: Uint8Array }
  | { ok: false; errorCode: number };

export const TAG_OK = 0x00;
export const TAG_ERR = 0x01;

export class EnvelopeError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "EnvelopeError";
  }
}

export function encodeEnvelope(res: GalaxyResult): Uint8Array {
  if (res.ok) {
    const out = new Uint8Array(1 + res.value.length);
    out[0] = TAG_OK;
    out.set(res.value, 1);
    return out;
  } else {
    const out = new Uint8Array(1 + 4);
    out[0] = TAG_ERR;
    const view = new DataView(out.buffer, out.byteOffset, out.byteLength);
    view.setUint32(1, res.errorCode >>> 0, true /* little-endian */);
    return out;
  }
}

export function decodeEnvelope(bytes: Uint8Array): GalaxyResult {
  if (bytes.length === 0) {
    throw new EnvelopeError("Empty envelope");
  }
  const tag = bytes[0];

  if (tag === TAG_OK) {
    return { ok: true, value: bytes.slice(1) };
  }

  if (tag === TAG_ERR) {
    if (bytes.length !== 1 + 4) {
      throw new EnvelopeError(`Err envelope wrong length: ${bytes.length}`);
    }
    const view = new DataView(bytes.buffer, bytes.byteOffset, bytes.byteLength);
    const code = view.getUint32(1, true);
    return { ok: false, errorCode: code };
  }

  throw new EnvelopeError(`Unknown tag: 0x${tag.toString(16)}`);
}

// Minimal tests (run in any TS test runner)
export function _testEnvelopeRoundtrip() {
  const ok = encodeEnvelope({ ok: true, value: new Uint8Array([1, 2, 3]) });
  const ok2 = decodeEnvelope(ok);
  if (!ok2.ok || ok2.value.length !== 3) throw new Error("OK roundtrip failed");

  const err = encodeEnvelope({ ok: false, errorCode: 7 });
  const err2 = decodeEnvelope(err);
  if (err2.ok || err2.errorCode !== 7) throw new Error("ERR roundtrip failed");
}
```

## Notes for SDK Consistency

* Treat the envelope as the **only** stable external contract return format in v0.1.
* Keep error codes as `u32` everywhere (even if language uses `I32` internally).
* When mapping internal `I32` error codes into envelope:

  * negative codes should be rejected or normalized by policy (recommended: reject and return `ERR_INTERNAL`).

