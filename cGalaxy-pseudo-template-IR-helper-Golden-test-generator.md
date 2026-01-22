Below are **two deliverables**:

1. A **contract-side ABI helper template** (both **cGalaxy pseudo** and **GALIR-style IR**) that builds the **Ok/Err Result envelope** exactly as specified.

2. A **golden test generator** (Rust) that takes a higher-level “expected Ok/Err” description and produces the exact `expected.result_envelope_hex` you put into your fixture JSON.

# 1) Contract-side ABI helper template

## 1A) cGalaxy pseudo-template (v0.1 style)

This uses only the v0.1 model: `Bytes` literals + `bytes.concat`. We add one small helper `abi.u32_le` that you can implement either as an intrinsic or as a tiny IR helper (see next section).

```cg
module ABIHelpers

import mem    // needed if you implement u32_le via alloc + stores; otherwise omit
import gas    // optional
// No storage/events/crypto needed

// Constants for tags
// NOTE: Bytes literal supports hex; 0x00 is single byte.
fn abi_ok(payload: Bytes) -> Bytes {
  // 0x00 || payload
  // bytes.concat returns Result; v0.1 uses explicit match.
  let tag: Bytes = 0x00;
  match bytes.concat(tag, payload) {
    Ok(out) => { return out; }
    Err(_)  => { return abi_err(8); } // ERR_INTERNAL
  }
}

fn abi_err(code: I32) -> Bytes {
  // 0x01 || u32_le(code)
  // Policy: negative code maps to ERR_INTERNAL
  let c: U32 = if code < 0 { 8 } else { (code as U32) };

  let tag: Bytes = 0x01;
  let code_bytes: Bytes = abi_u32_le(c);  // implemented below (intrinsic or IR helper)

  match bytes.concat(tag, code_bytes) {
    Ok(out) => { return out; }
    Err(_)  => {
      // If even this fails, return empty (last resort).
      // v0.1 golden tests should not force allocator failure.
      return 0x;
    }
  }
}

// You can implement abi_u32_le(c) in one of two ways:
//
// Option A (simplest for v0.1): Make it a compiler intrinsic or stdlib primitive:
//   abi_u32_le(u: U32) -> Bytes
//
// Option B: Implement it using IR-level alloc + mem_store_u8 (see IR template below).
fn abi_u32_le(u: U32) -> Bytes {
  // Placeholder. Implement via IR template below or as intrinsic.
  return 0x00000000;
}
```

### How you use it in `main`

```cg
fn main(input: Bytes) -> Result<Bytes, I32> {
  // ...do stuff...
  // For success:
  let out: Bytes = 0xDEADBEEF;
  return Ok(abi_ok(out));

  // For error:
  // return Ok(abi_err(7));
}
```

**Important:** In Galaxy v0.1 ABI, `main` still returns a packed slice pointer/len at the WASM boundary; the payload bytes returned from language-level `main` must already be the envelope bytes.

## 1B) “Compiler-ready” IR helper (GALIR-T style)

This is the cleanest way to guarantee correctness today: implement envelope construction in IR using `alloc` + `mem_store_u8` and return a `slice`.

### Helper 1: `abi_u32_le(u32) -> slice(4 bytes)`

Writes little-endian u32 into a fresh 4-byte buffer.

```
// Returns slice ptr/len for 4 bytes: [u0, u1, u2, u3] little-endian
func @abi_u32_le (x:u32) -> (out:slice) {
  block b0(x:u32):
    %len = const_u32 4
    %ptr = alloc %len
    // If ptr == 0, return (0,0) (out of memory)
    %z = const_u32 0
    %ok = neq u32 %ptr %z
    br_if %ok b1(%ptr,%x) b_fail()

  block b_fail():
    %p0 = const_u32 0
    %l0 = const_u32 0
    %s0 = slice_make %p0 %l0
    ret %s0

  block b1(p:u32, x:u32):
    // byte0 = x & 0xFF
    %b0 = and_u32_imm %x 255        // (If you don't have and_imm, use const+and)
    mem_store_u8 %p %b0

    // byte1 = (x >> 8) & 0xFF
    %p1 = add_wrap u32 %p (const_u32 1)
    %x1 = shr_u32_imm %x 8
    %b1 = and_u32_imm %x1 255
    mem_store_u8 %p1 %b1

    // byte2 = (x >> 16) & 0xFF
    %p2 = add_wrap u32 %p (const_u32 2)
    %x2 = shr_u32_imm %x 16
    %b2 = and_u32_imm %x2 255
    mem_store_u8 %p2 %b2

    // byte3 = (x >> 24) & 0xFF
    %p3 = add_wrap u32 %p (const_u32 3)
    %x3 = shr_u32_imm %x 24
    %b3 = and_u32_imm %x3 255
    mem_store_u8 %p3 %b3

    %s = slice_make %p %len
    ret %s
}
```

### Helper 2: `abi_ok(payload:slice) -> slice`

Allocates `1 + payload.len`, writes tag `0x00`, copies payload.

```
// out = 0x00 || payload
func @abi_ok (pl:slice) -> (out:slice) {
  block b0(pl:slice):
    %pl_len = slice_len %pl
    (%ok1,%out_len) = add_chk u32 %pl_len (const_u32 1)
    br_if %ok1 b1(%pl,%out_len) b_fail_overflow()

  block b_fail_overflow():
    // If overflow occurs (practically impossible with u32 len limits), return empty
    %s0 = slice_make (const_u32 0) (const_u32 0)
    ret %s0

  block b1(pl:slice, out_len:u32):
    %out_ptr = alloc %out_len
    %z = const_u32 0
    %ok2 = neq u32 %out_ptr %z
    br_if %ok2 b2(%pl,%out_ptr,%out_len) b_fail_alloc()

  block b_fail_alloc():
    %s0 = slice_make (const_u32 0) (const_u32 0)
    ret %s0

  block b2(pl:slice, out_ptr:u32, out_len:u32):
    // write tag byte
    mem_store_u8 %out_ptr (const_u32 0)  // 0x00

    // copy payload bytes after tag
    %pl_ptr = slice_ptr %pl
    %dst = add_wrap u32 %out_ptr (const_u32 1)
    mem_copy %dst %pl_ptr %pl_len

    %out = slice_make %out_ptr %out_len
    ret %out
}
```

### Helper 3: `abi_err(code:u32) -> slice`

Allocates 5 bytes: `0x01 || u32_le(code)`.

```
// out = 0x01 || u32_le(code)
func @abi_err (code:u32) -> (out:slice) {
  block b0(code:u32):
    %out_len = const_u32 5
    %out_ptr = alloc %out_len
    %z = const_u32 0
    %ok = neq u32 %out_ptr %z
    br_if %ok b1(%out_ptr,%code) b_fail()

  block b_fail():
    %s0 = slice_make (const_u32 0) (const_u32 0)
    ret %s0

  block b1(out_ptr:u32, code:u32):
    mem_store_u8 %out_ptr (const_u32 1) // tag 0x01
    %code_slice = call @abi_u32_le(%code)
    %cp = slice_ptr %code_slice
    %dst = add_wrap u32 %out_ptr (const_u32 1)
    mem_copy %dst %cp (const_u32 4)
    %out = slice_make %out_ptr (const_u32 5)
    ret %out
}
```

> Note: the IR snippets use a couple of helper ops like `shr_u32_imm` / `and_u32_imm`. If your IR v0.1 doesn’t have immediates, replace them with `const_u32` + normal `and` and a `shr` opcode (add `shr` to IR if needed). If you want, I’ll rewrite these helpers using only the exact opcodes you already locked in—just tell me if you prefer **adding `shr/and_imm`** or **pure const+and**.

# 2) Golden test generator (Rust)

This generator takes a high-level JSON:

* either `{ "ok": { "value_hex": "..." } }`
* or `{ "err": { "code": 7 } }`

…and outputs the **exact envelope hex** you paste into fixture `expected.result_envelope_hex`.

It can also **update your fixture JSON in-place** if you want.

## 2A) `expected_outcome.json` format

```json
{ "ok": { "value_hex": "deadbeef" } }
```

or

```json
{ "err": { "code": 7 } }
```

## 2B) Rust generator code

```rust
// golden_envelope_gen.rs
//
// Build expected.result_envelope_hex from a higher-level Ok/Err description.
//
// Cargo.toml deps:
// anyhow = "1"
// serde = { version = "1", features = ["derive"] }
// serde_json = "1"
// hex = "0.4"

use anyhow::{bail, Context, Result};
use serde::{Deserialize, Serialize};
use std::fs;

const TAG_OK: u8 = 0x00;
const TAG_ERR: u8 = 0x01;

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
enum Outcome {
    Ok { value_hex: String },
    Err { code: u32 },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct OutcomeFile {
    #[serde(flatten)]
    outcome: Outcome,
}

fn hex_to_bytes(s: &str) -> Result<Vec<u8>> {
    let s = s.trim();
    let s = s.strip_prefix("0x").unwrap_or(s);
    if s.is_empty() {
        return Ok(vec![]);
    }
    Ok(hex::decode(s).with_context(|| format!("invalid hex: {}", s))?)
}

fn bytes_to_hex(bytes: &[u8]) -> String {
    hex::encode(bytes)
}

fn build_envelope(outcome: &Outcome) -> Result<Vec<u8>> {
    match outcome {
        Outcome::Ok { value_hex } => {
            let payload = hex_to_bytes(value_hex)?;
            let mut out = Vec::with_capacity(1 + payload.len());
            out.push(TAG_OK);
            out.extend_from_slice(&payload);
            Ok(out)
        }
        Outcome::Err { code } => {
            let mut out = Vec::with_capacity(1 + 4);
            out.push(TAG_ERR);
            out.extend_from_slice(&code.to_le_bytes());
            Ok(out)
        }
    }
}

/// Optional: patch a fixture JSON file by setting expected.result_envelope_hex
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Fixture {
    wasm_path: String,
    input_hex: String,
    storage: serde_json::Value,
    context: serde_json::Value,
    gas_budget: u64,
    expected: Expected,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Expected {
    result_envelope_hex: String,
    storage: serde_json::Value,
    events: serde_json::Value,
    gas_remaining: Option<u64>,
}

fn patch_fixture(fixture_path: &str, new_hex: &str) -> Result<()> {
    let s = fs::read_to_string(fixture_path)?;
    let mut fx: Fixture = serde_json::from_str(&s)?;
    fx.expected.result_envelope_hex = new_hex.to_string();
    let out = serde_json::to_string_pretty(&fx)?;
    fs::write(fixture_path, out)?;
    Ok(())
}

fn main() -> Result<()> {
    let args: Vec<String> = std::env::args().collect();

    if args.len() < 2 || args.len() > 4 {
        eprintln!("Usage:");
        eprintln!("  {} <outcome.json>", args[0]);
        eprintln!("  {} <outcome.json> --patch <fixture.json>", args[0]);
        std::process::exit(2);
    }

    let outcome_path = &args[1];
    let s = fs::read_to_string(outcome_path)?;
    let of: OutcomeFile = serde_json::from_str(&s).context("bad outcome.json")?;

    let env = build_envelope(&of.outcome)?;
    let hex_env = bytes_to_hex(&env);

    if args.len() == 4 {
        if args[2] != "--patch" {
            bail!("unknown flag: {}", args[2]);
        }
        patch_fixture(&args[3], &hex_env)?;
        println!("patched fixture expected.result_envelope_hex = {}", hex_env);
    } else {
        println!("{}", hex_env);
    }

    Ok(())
}
```

### Example usage

* Just print expected envelope hex:

```bash
cargo run --bin golden_envelope_gen -- outcome.json
```

* Patch a fixture file in-place:

```bash
cargo run --bin golden_envelope_gen -- outcome.json --patch tests/golden/001_addition/run.json
```


## Optional improvement (if you want)

If you want higher-level expectations like:

```json
{ "ok": { "utf8": "hello" } }
```

or

```json
{ "ok": { "cbor": { ... } } }
```

