**v0.1 fixture schema** that supports **both hex and utf8 and (optionally) canonical CBOR**, plus an updated **Rust golden generator** that outputs/patches `expected.result_envelope_hex`.

# 1) v0.1 “Expected Outcome” schema (supports hex + utf8 + cbor)

## 1A) Outcome JSON format

Top-level is exactly one of `ok` or `err`.

### ✅ Ok payload as HEX

```json
{
  "ok": { "hex": "deadbeef" }
}
```

### ✅ Ok payload as UTF-8

```json
{
  "ok": { "utf8": "hello galaxy" }
}
```

### ✅ Ok payload as Canonical CBOR (deterministic)

```json
{
  "ok": { "cbor": { "map": [["name", {"text":"Bernard"}], ["id", {"u64": 7}]] } }
}
```

### ✅ Err as numeric code

```json
{
  "err": { "code": 7 }
}
```

### Optional: convenience shortcut for error symbolic names

```json
{
  "err": { "name": "ERR_UNAUTHORIZED" }
}
```

## 1B) Canonical CBOR model (deterministic subset)

To guarantee determinism, we define a strict subset:

### CBOR Value forms

* `{ "u64": N }` where 0 ≤ N ≤ 2^64-1
* `{ "i64": N }` where -2^63 ≤ N ≤ 2^63-1
* `{ "bytes": "hex..." }`
* `{ "text": "utf8 string" }`
* `{ "array": [ <cbor>, <cbor>, ... ] }`
* `{ "map": [ [<textKey>, <cbor>], ... ] }`

### Canonicalization rules

* Map keys are **text strings only** (simplifies canonical ordering).
* Map entries must be **sorted by UTF-8 lexicographic key order** by the generator, regardless of input order.
* Use CBOR canonical integer encoding (standard).
* No indefinite-length items.
* No floats.

This makes your test outputs stable everywhere.

# 2) Updated Golden Test Generator (Rust) — hex + utf8 + cbor + patch fixture

This program:

* reads `outcome.json`
* builds the deterministic payload bytes
* wraps it into the Galaxy envelope:

  * Ok: `00 || payload`
  * Err: `01 || u32_le(code)`
* prints the final envelope hex
* optionally patches a fixture JSON file.

## 2A) Cargo.toml dependencies

```toml
[dependencies]
anyhow = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
hex = "0.4"
```

(We implement CBOR encoding ourselves to keep it deterministic and avoid library canonicalization differences.)

## 2B) Generator code

```rust
// golden_envelope_gen.rs
//
// Galaxy Golden Envelope Generator v0.1
// Supports expected outcomes in HEX, UTF-8, and Canonical CBOR (strict subset).
//
// Usage:
//   cargo run --bin golden_envelope_gen -- outcome.json
//   cargo run --bin golden_envelope_gen -- outcome.json --patch path/to/fixture.json
//
// Envelope:
//   Ok:  0x00 || payload
//   Err: 0x01 || u32_le(code)

use anyhow::{anyhow, bail, Context, Result};
use serde::{Deserialize, Serialize};
use std::fs;

const TAG_OK: u8 = 0x00;
const TAG_ERR: u8 = 0x01;

// ---------------- Outcome schema ----------------

#[derive(Debug, Clone, Deserialize)]
#[serde(deny_unknown_fields)]
struct OutcomeFile {
    #[serde(default)]
    ok: Option<OkPayload>,
    #[serde(default)]
    err: Option<ErrPayload>,
}

#[derive(Debug, Clone, Deserialize)]
#[serde(deny_unknown_fields)]
struct OkPayload {
    #[serde(default)]
    hex: Option<String>,
    #[serde(default)]
    utf8: Option<String>,
    #[serde(default)]
    cbor: Option<CborValue>,
}

#[derive(Debug, Clone, Deserialize)]
#[serde(deny_unknown_fields)]
struct ErrPayload {
    #[serde(default)]
    code: Option<u32>,
    #[serde(default)]
    name: Option<String>,
}

// ---------------- CBOR strict subset schema ----------------

#[derive(Debug, Clone, Deserialize)]
#[serde(deny_unknown_fields)]
#[serde(rename_all = "lowercase")]
enum CborValue {
    // {"u64": N}
    U64 { u64: u64 },
    // {"i64": N}
    I64 { i64: i64 },
    // {"bytes": "hex..."}
    Bytes { bytes: String },
    // {"text": "string"}
    Text { text: String },
    // {"array": [ ... ]}
    Array { array: Vec<CborValue> },
    // {"map": [ ["key", <cbor>], ... ]}
    Map { map: Vec<(String, CborValue)> },
}

// ---------------- Fixture patching (optional) ----------------

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

// ---------------- Helpers ----------------

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

fn require_exactly_one_ok_or_err(of: &OutcomeFile) -> Result<()> {
    let ok = of.ok.is_some();
    let err = of.err.is_some();
    if ok == err {
        bail!("outcome.json must contain exactly one of: {ok:...} or {err:...}");
    }
    Ok(())
}

fn resolve_err_code(e: &ErrPayload) -> Result<u32> {
    if let Some(code) = e.code {
        return Ok(code);
    }
    if let Some(name) = &e.name {
        return match name.as_str() {
            "ERR_GAS_EXHAUSTED" => Ok(1),
            "ERR_DIV_BY_ZERO" => Ok(2),
            "ERR_OVERFLOW" => Ok(3),
            "ERR_OOB" => Ok(4),
            "ERR_STORAGE" => Ok(5),
            "ERR_INVALID_INPUT" => Ok(6),
            "ERR_UNAUTHORIZED" => Ok(7),
            "ERR_INTERNAL" => Ok(8),
            _ => bail!("unknown error name: {}", name),
        };
    }
    bail!("err must contain either `code` or `name`");
}

fn build_ok_payload(ok: &OkPayload) -> Result<Vec<u8>> {
    let mut count = 0;
    if ok.hex.is_some() { count += 1; }
    if ok.utf8.is_some() { count += 1; }
    if ok.cbor.is_some() { count += 1; }
    if count != 1 {
        bail!("ok must contain exactly one of: hex, utf8, cbor");
    }

    if let Some(h) = &ok.hex {
        return hex_to_bytes(h);
    }
    if let Some(s) = &ok.utf8 {
        return Ok(s.as_bytes().to_vec());
    }
    if let Some(c) = &ok.cbor {
        let mut out = Vec::new();
        encode_cbor_canonical(c, &mut out)?;
        return Ok(out);
    }

    unreachable!()
}

// ---------------- Canonical CBOR encoder (strict subset) ----------------

// Major types:
// 0 unsigned, 1 negative, 2 bytes, 3 text, 4 array, 5 map
fn cbor_write_type_and_len(major: u8, len: u64, out: &mut Vec<u8>) {
    // canonical shortest encoding
    if len <= 23 {
        out.push((major << 5) | (len as u8));
    } else if len <= 0xFF {
        out.push((major << 5) | 24);
        out.push(len as u8);
    } else if len <= 0xFFFF {
        out.push((major << 5) | 25);
        out.extend_from_slice(&(len as u16).to_be_bytes());
    } else if len <= 0xFFFF_FFFF {
        out.push((major << 5) | 26);
        out.extend_from_slice(&(len as u32).to_be_bytes());
    } else {
        out.push((major << 5) | 27);
        out.extend_from_slice(&len.to_be_bytes());
    }
}

fn encode_cbor_canonical(v: &CborValue, out: &mut Vec<u8>) -> Result<()> {
    match v {
        CborValue::U64 { u64: n } => {
            cbor_write_type_and_len(0, *n, out);
            Ok(())
        }
        CborValue::I64 { i64: n } => {
            if *n >= 0 {
                cbor_write_type_and_len(0, *n as u64, out);
            } else {
                // CBOR negative integer encodes (-1 - n) as unsigned in major type 1
                let m = (-1i128 - (*n as i128)) as u64;
                cbor_write_type_and_len(1, m, out);
            }
            Ok(())
        }
        CborValue::Bytes { bytes } => {
            let b = hex_to_bytes(bytes)?;
            cbor_write_type_and_len(2, b.len() as u64, out);
            out.extend_from_slice(&b);
            Ok(())
        }
        CborValue::Text { text } => {
            let b = text.as_bytes();
            cbor_write_type_and_len(3, b.len() as u64, out);
            out.extend_from_slice(b);
            Ok(())
        }
        CborValue::Array { array } => {
            cbor_write_type_and_len(4, array.len() as u64, out);
            for item in array {
                encode_cbor_canonical(item, out)?;
            }
            Ok(())
        }
        CborValue::Map { map } => {
            // Deterministic: sort by UTF-8 key lexicographic order
            let mut items = map.clone();
            items.sort_by(|a, b| a.0.as_bytes().cmp(b.0.as_bytes()));

            cbor_write_type_and_len(5, items.len() as u64, out);
            for (k, val) in &items {
                // keys must be text strings (already)
                let kb = k.as_bytes();
                cbor_write_type_and_len(3, kb.len() as u64, out);
                out.extend_from_slice(kb);
                encode_cbor_canonical(val, out)?;
            }
            Ok(())
        }
    }
}

// ---------------- Envelope builder ----------------

fn build_envelope(of: &OutcomeFile) -> Result<Vec<u8>> {
    require_exactly_one_ok_or_err(of)?;

    if let Some(ok) = &of.ok {
        let payload = build_ok_payload(ok)?;
        let mut out = Vec::with_capacity(1 + payload.len());
        out.push(TAG_OK);
        out.extend_from_slice(&payload);
        return Ok(out);
    }

    let err = of.err.as_ref().unwrap();
    let code = resolve_err_code(err)?;
    let mut out = Vec::with_capacity(1 + 4);
    out.push(TAG_ERR);
    out.extend_from_slice(&code.to_le_bytes());
    Ok(out)
}

// ---------------- Patch fixture ----------------

fn patch_fixture(fixture_path: &str, new_hex: &str) -> Result<()> {
    let s = fs::read_to_string(fixture_path)?;
    let mut fx: Fixture = serde_json::from_str(&s).context("bad fixture.json")?;
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
    let s = fs::read_to_string(outcome_path).with_context(|| "failed to read outcome.json")?;
    let of: OutcomeFile = serde_json::from_str(&s).with_context(|| "bad outcome.json")?;

    let env = build_envelope(&of)?;
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

# 3) Example outcomes (hex / utf8 / cbor)

## 3A) ok hex

```json
{ "ok": { "hex": "deadbeef" } }
```

Envelope hex output: `00deadbeef`

## 3B) ok utf8

```json
{ "ok": { "utf8": "hello" } }
```

Envelope hex output: `0068656c6c6f`

## 3C) ok cbor

```json
{
  "ok": {
    "cbor": {
      "map": [
        ["id", { "u64": 7 }],
        ["name", { "text": "Bernard" }]
      ]
    }
  }
}
```

The generator will sort keys (`id` before `name`), encode canonical CBOR, then envelope it with `0x00 || cbor_bytes`.

## 3D) err name

```json
{ "err": { "name": "ERR_UNAUTHORIZED" } }
```

Envelope bytes: `01 07 00 00 00` → hex: `0107000000`

# 4) Contract-side ABI helper (practical recommendation)

For v0.1, the most reliable approach is:

* Provide `abi_ok(payload:Bytes)->Bytes` and `abi_err(code:I32)->Bytes` in a **standard library module** (or template) that compiles down to the IR helpers shown earlier (`alloc + mem_store_u8 + mem_copy`).
* This avoids needing `bytes.concat` and keeps gas predictable.

