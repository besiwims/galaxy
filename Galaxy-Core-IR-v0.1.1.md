## 🔒 Opcode Set Locked (Galaxy Core IR v0.1.1 – Bitwise + Shifts Added)

This is a **backwards-compatible extension** of Core IR v0.1. Everything existing remains valid.

# 1) New IR Opcodes (Added and Locked)

### 1.1 Integer bitwise ops

These apply to: `i32, i64, u32, u64` (and later `u256` when supported by backend lowering).

**Opcodes (GALIR-T spellings)**

* `%v = band T %a %b`  (bitwise AND)
* `%v = bor  T %a %b`  (bitwise OR)
* `%v = bxor T %a %b`  (bitwise XOR)
* `%v = bnot T %a`     (bitwise NOT)

> Note: boolean ops remain as previously defined: `and/or/not` for `b1`.
> The `b*` prefix is reserved for bitwise integer operations.

### 1.2 Shift ops

**Opcodes**

* `%v = shl T %a %shift`
* `%v = shr T %a %shift`

**Shift semantics**

* Shift amount is taken from the RHS value’s low bits (backend-native behavior is acceptable as long as it is deterministic):

  * For `i32/u32`, only low 5 bits used (0..31)
  * For `i64/u64`, only low 6 bits used (0..63)
* `shr`:

  * `i32/i64`: arithmetic shift right (sign-propagating)
  * `u32/u64`: logical shift right (zero-fill)

**Why no “*_imm” opcodes?**

* You can always do `const_u32 8` and pass it. This keeps the opcode set smaller and the binary format simpler.

# 2) Validator Updates (GALIR-VAL delta)

### 2.1 Typing rules (new)

For each instruction:

* `band/bor/bxor`: operands must both be type `T` where `T ∈ {i32,i64,u32,u64}`; result type is `T`.
* `bnot`: operand must be integer type `T`; result is `T`.
* `shl/shr`: both operands must be type `T` and `T ∈ {i32,i64,u32,u64}`; result type is `T`.

If violated → `V0010 Type mismatch`.

### 2.2 Determinism

These ops are pure and deterministic. No new determinism concerns.

### 2.3 Gas metering

No changes to loop metering rules.

# 3) Gas Cost Table Update (v0.1 baseline delta)

Add these to “Instruction class costs”:

| Operation          | Cost (gas) |
| ------------------ | ---------: |
| band/bor/bxor/bnot |          1 |
| shl/shr            |          1 |

Everything else unchanged.

# 4) WASM Backend Mapping (GALIR→WASM delta)

### 4.1 Lowering for i32/u32

* `band i32/u32` → `i32.and`
* `bor  i32/u32` → `i32.or`
* `bxor i32/u32` → `i32.xor`
* `bnot i32/u32` → `i32.const -1; i32.xor` (or `i32.eqz` not applicable; must invert bits)
* `shl  i32/u32` → `i32.shl`
* `shr  i32`     → `i32.shr_s`
* `shr  u32`     → `i32.shr_u`

### 4.2 Lowering for i64/u64

* `band` → `i64.and`
* `bor`  → `i64.or`
* `bxor` → `i64.xor`
* `bnot` → `i64.const -1; i64.xor`
* `shl`  → `i64.shl`
* `shr i64` → `i64.shr_s`
* `shr u64` → `i64.shr_u`

# 5) ABI Helper Templates Rewritten Using Locked Opcodes Only

Below are the **exact IR helpers** without any “imm” shortcuts.

## 5.1 `abi_u32_le(x:u32) -> slice`

Writes 4 bytes little-endian into freshly allocated memory.

```
// Returns slice ptr/len for 4 bytes: [x0, x1, x2, x3] little-endian
func @abi_u32_le (x:u32) -> (out:slice) {
  block b0(x:u32):
    %len = const_u32 4
    %ptr = alloc %len

    %z = const_u32 0
    %ok_alloc = neq u32 %ptr %z
    br_if %ok_alloc b1(%ptr,%x) b_fail()

  block b_fail():
    %p0 = const_u32 0
    %l0 = const_u32 0
    %s0 = slice_make %p0 %l0
    ret %s0

  block b1(p:u32, x:u32):
    %mask = const_u32 255

    // byte0 = x & 0xFF
    %b0 = band u32 %x %mask
    mem_store_u8 %p %b0

    // byte1 = (x >> 8) & 0xFF
    %one = const_u32 1
    %p1  = add_wrap u32 %p %one
    %s8  = const_u32 8
    %x1  = shr u32 %x %s8
    %b1  = band u32 %x1 %mask
    mem_store_u8 %p1 %b1

    // byte2 = (x >> 16) & 0xFF
    %two = const_u32 2
    %p2  = add_wrap u32 %p %two
    %s16 = const_u32 16
    %x2  = shr u32 %x %s16
    %b2  = band u32 %x2 %mask
    mem_store_u8 %p2 %b2

    // byte3 = (x >> 24) & 0xFF
    %three = const_u32 3
    %p3    = add_wrap u32 %p %three
    %s24   = const_u32 24
    %x3    = shr u32 %x %s24
    %b3    = band u32 %x3 %mask
    mem_store_u8 %p3 %b3

    %out = slice_make %p %len
    ret %out
}
```

## 5.2 `abi_err(code:u32) -> slice`

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
    // tag 0x01
    mem_store_u8 %out_ptr (const_u32 1)

    %code_slice = call @abi_u32_le(%code)
    %cp = slice_ptr %code_slice

    %one = const_u32 1
    %dst = add_wrap u32 %out_ptr %one
    mem_copy %dst %cp (const_u32 4)

    %out = slice_make %out_ptr %out_len
    ret %out
}
```

## 5.3 `abi_ok(payload:slice) -> slice`

Allocates `1 + payload.len`: `0x00 || payload`.

```
// out = 0x00 || payload
func @abi_ok (pl:slice) -> (out:slice) {
  block b0(pl:slice):
    %pl_len = slice_len %pl
    (%ok1,%out_len) = add_chk u32 %pl_len (const_u32 1)
    br_if %ok1 b1(%pl,%out_len) b_fail()

  block b_fail():
    %s0 = slice_make (const_u32 0) (const_u32 0)
    ret %s0

  block b1(pl:slice, out_len:u32):
    %out_ptr = alloc %out_len
    %z = const_u32 0
    %ok2 = neq u32 %out_ptr %z
    br_if %ok2 b2(%pl,%out_ptr,%out_len) b_fail()

  block b2(pl:slice, out_ptr:u32, out_len:u32):
    mem_store_u8 %out_ptr (const_u32 0) // tag 0x00
    %pl_ptr = slice_ptr %pl

    %one = const_u32 1
    %dst = add_wrap u32 %out_ptr %one
    mem_copy %dst %pl_ptr %pl_len

    %out = slice_make %out_ptr %out_len
    ret %out
}
```

# 6) One Small Alignment Note (for your harness + contracts)

These helpers rely on:

* `alloc(len)` returning a valid pointer or `0`
* `mem_store_u8`, `mem_copy` behaving deterministically
* contract memory exported as `memory`

# 🔒 GALIR-B Opcode Enum v0.1.1 (Numeric IDs Locked)

## 0) Compatibility rule

* **v0.1** binaries remain valid under **v0.1.1**.
* New opcodes are appended; existing numeric IDs are unchanged.
* Validators/backends that only support v0.1 must reject v0.1.1 modules that contain new opcodes.

# 1) Versioning

In `GALIR-B` header:

* magic: `GALIR\0`
* major: `0`
* minor: `1` for v0.1
* minor: `2` for v0.1.1 **(recommended)**

To avoid confusion, lock:

* **v0.1.1 = (major=0, minor=2)**

So:

* `0.1` remains `0.1`
* `0.1.1` becomes `0.2` in the binary header (minor bump), while we still refer to it as “v0.1.1” in docs.

# 2) Typecode Enum (u8) Locked (unchanged)

These appear in `S_TYPES`, `S_HOSTCALLS`, `S_FUNCS` signatures.

| Type       | code |
| ---------- | ---: |
| b1         |    0 |
| i32        |    1 |
| i64        |    2 |
| u32        |    3 |
| u64        |    4 |
| u256       |    5 |
| unit       |    6 |
| slice      |    7 |
| layout_ref |    8 |

# 3) Cost Class Enum (u8) Locked (baseline)

Used in `S_HOSTCALLS`.

| Cost class            | code |
| --------------------- | ---: |
| GAS_TICK              |    0 |
| STORAGE_HAS           |    1 |
| STORAGE_GET           |    2 |
| STORAGE_PUT           |    3 |
| STORAGE_DEL           |    4 |
| EVENT_EMIT            |    5 |
| CRYPTO_SHA256         |    6 |
| CRYPTO_BLAKE2B256     |    7 |
| CRYPTO_VERIFY_ED25519 |    8 |
| CTX_SENDER            |    9 |
| CTX_CHAIN_ID          |   10 |
| CTX_TX_HASH           |   11 |
| CTX_BLOCK_HEIGHT      |   12 |
| MEM_ALLOC             |   13 |

(Your runtime can still ignore this if you use explicit `gas_tick` only, but the field is now locked.)

# 4) Operand Reference Encoding (unchanged)

Operand packed `uleb ref_kind_and_index`:

* low 2 bits: kind

  * `0=ssa_value`
  * `1=block_param`
  * `2=immediate_small (0..63)`
  * `3=reserved`
* remaining bits: index

**Rule:** v0.1.1 does **not** add new ref kinds.

# 5) Opcode Enum (u8) Locked

## 5.1 Core constants

| Opcode      |   ID |
| ----------- | ---: |
| CONST_I32   | 0x01 |
| CONST_I64   | 0x02 |
| CONST_U32   | 0x03 |
| CONST_U64   | 0x04 |
| CONST_BOOL  | 0x05 |
| CONST_BYTES | 0x06 |

> Note: `CONST_BYTES` uses opcode-specific immediates: `uleb string_id` OR raw bytes reference (see §6).

## 5.2 Integer arithmetic (checked)

All checked ops return `(b1 ok, T value)`.

| Opcode  |   ID |
| ------- | ---: |
| ADD_CHK | 0x10 |
| SUB_CHK | 0x11 |
| MUL_CHK | 0x12 |
| DIV_CHK | 0x13 |
| MOD_CHK | 0x14 |

## 5.3 Integer arithmetic (wrapping)

| Opcode   |   ID |
| -------- | ---: |
| ADD_WRAP | 0x18 |
| SUB_WRAP | 0x19 |
| MUL_WRAP | 0x1A |

## 5.4 Comparisons (return b1)

| Opcode |   ID |
| ------ | ---: |
| EQ     | 0x20 |
| NEQ    | 0x21 |
| LT     | 0x22 |
| LTE    | 0x23 |
| GT     | 0x24 |
| GTE    | 0x25 |

## 5.5 Boolean (b1)

| Opcode |   ID |
| ------ | ---: |
| AND_B1 | 0x28 |
| OR_B1  | 0x29 |
| NOT_B1 | 0x2A |

(Keep `and/or/not` for b1 distinct from integer bitwise ops.)

## 5.6 Memory and slices

| Opcode          |   ID |                        |
| --------------- | ---: | ---------------------- |
| SLICE_LEN       | 0x30 |                        |
| SLICE_PTR       | 0x31 |                        |
| SLICE_MAKE      | 0x32 |                        |
| SLICE_EQ        | 0x33 |                        |
| SLICE_SLICE_CHK | 0x34 | *(returns ok + slice)* |
| ALLOC           | 0x35 |                        |
| MEM_LOAD_U8     | 0x36 |                        |
| MEM_STORE_U8    | 0x37 |                        |
| MEM_COPY        | 0x38 |                        |
| CONCAT_CHK      | 0x39 | *(returns ok + slice)* |

## 5.7 Calls and metering

| Opcode    |   ID |
| --------- | ---: |
| HOST_CALL | 0x40 |
| GAS_TICK  | 0x41 |

* `HOST_CALL` opcode-specific immediate: `uleb hostcall_index`
* `GAS_TICK` is a dedicated opcode (and should also correspond to a hostcall entry `gas::tick`)

## 5.8 Terminator opcodes (appear only as terminators)

Terminators are **not** in the instruction stream list above; they are encoded separately in the block terminator field.

Terminator kind enum (u8):

| Terminator |   ID |
| ---------- | ---: |
| BR         | 0x01 |
| BR_IF      | 0x02 |
| RET        | 0x03 |

(These are not “opcodes”; they are terminator kinds.)

# 6) ✅ New v0.1.1 integer bitwise + shift opcodes (LOCKED)

These are appended; no collisions.

## 6.1 Bitwise integer ops

| Opcode |   ID |
| ------ | ---: |
| BAND   | 0x50 |
| BOR    | 0x51 |
| BXOR   | 0x52 |
| BNOT   | 0x53 |

## 6.2 Shifts

| Opcode |   ID |
| ------ | ---: |
| SHL    | 0x54 |
| SHR    | 0x55 |


# 7) Opcode-specific immediates (locked encoding)

## 7.1 CONST_BYTES immediate

To support deterministic binaries, `CONST_BYTES` must reference stable data. Lock this rule:

`CONST_BYTES` encoding uses a single `u8 mode` then payload:

* mode `0x00`: **string table reference**

  * payload: `uleb string_id`
  * string is interpreted as **hex string** without `0x` and must decode to bytes
* mode `0x01`: **inline bytes**

  * payload: `uleb len` + raw bytes

**Validator rules**

* For mode `0x00`, the referenced string must decode to bytes and match `^[0-9a-fA-F]*$` with even length.
* For mode `0x01`, length must be ≤ module max literal bytes policy.

> Recommendation: Use mode `0x01` in generated binaries; use mode `0x00` for hand-authored text compilation to keep references stable.

## 7.2 HOST_CALL immediate

* immediate: `uleb hostcall_index`

The HOST_CALL instruction’s operand list corresponds to hostcall parameters; result list corresponds to hostcall results.

# 8) Text spelling ↔ Binary opcode mapping (final)

Your GALIR-T printer/parser must map these exact spellings:

* `band` ↔ `BAND (0x50)`
* `bor` ↔ `BOR (0x51)`
* `bxor` ↔ `BXOR (0x52)`
* `bnot` ↔ `BNOT (0x53)`
* `shl` ↔ `SHL (0x54)`
* `shr` ↔ `SHR (0x55)`

And earlier spellings to earlier IDs exactly as listed.

# 9) Minimal Validator Additions (binary-level)

When decoding an instruction:

* read `opcode:u8`
* dispatch by opcode:

  * for `CONST_BYTES`, read `mode:u8`, then its payload
  * for `HOST_CALL`, read `hostcall_index:uleb`
  * otherwise opcode has no immediates (in v0.1.1)

If an opcode is unknown → `V0002`.

```rust
// galir_bin.rs
//
// Galaxy IR Binary (GALIR-B) v0.1.1 (binary header major=0 minor=2)
// Single authoritative enums + shared encode/decode helpers.
//
// This module is intended to be used by BOTH:
// - serializer/deserializer
// - validator
//
// It intentionally focuses on the “core plumbing”:
// - header + section framing
// - uleb128
// - enums (typecodes, cost classes, opcodes, terminators, caps)
// - instruction immediate decoding (CONST_BYTES + HOST_CALL)
//
// You can extend this with higher-level AST structs for full module parsing.
//
// -------------------- Cargo.toml deps --------------------
// [dependencies]
// anyhow = "1"
//
// ---------------------------------------------------------

use anyhow::{bail, Result};

pub const MAGIC: &[u8; 6] = b"GALIR\0";
pub const VERSION_MAJOR: u16 = 0;
pub const VERSION_MINOR: u16 = 2; // v0.1.1 locked as binary (0,2)

// ============================ Enums ============================

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TypeCode {
    B1 = 0,
    I32 = 1,
    I64 = 2,
    U32 = 3,
    U64 = 4,
    U256 = 5,
    Unit = 6,
    Slice = 7,
    LayoutRef = 8,
}
impl TypeCode {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0 => Self::B1,
            1 => Self::I32,
            2 => Self::I64,
            3 => Self::U32,
            4 => Self::U64,
            5 => Self::U256,
            6 => Self::Unit,
            7 => Self::Slice,
            8 => Self::LayoutRef,
            _ => bail!("invalid TypeCode: {}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Cap {
    Storage = 0,
    Events = 1,
    Crypto = 2,
    Context = 3,
    Gas = 4,
    Mem = 5,
}
impl Cap {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0 => Self::Storage,
            1 => Self::Events,
            2 => Self::Crypto,
            3 => Self::Context,
            4 => Self::Gas,
            5 => Self::Mem,
            _ => bail!("invalid Cap: {}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum CostClass {
    GasTick = 0,
    StorageHas = 1,
    StorageGet = 2,
    StoragePut = 3,
    StorageDel = 4,
    EventEmit = 5,
    CryptoSha256 = 6,
    CryptoBlake2b256 = 7,
    CryptoVerifyEd25519 = 8,
    CtxSender = 9,
    CtxChainId = 10,
    CtxTxHash = 11,
    CtxBlockHeight = 12,
    MemAlloc = 13,
}
impl CostClass {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0 => Self::GasTick,
            1 => Self::StorageHas,
            2 => Self::StorageGet,
            3 => Self::StoragePut,
            4 => Self::StorageDel,
            5 => Self::EventEmit,
            6 => Self::CryptoSha256,
            7 => Self::CryptoBlake2b256,
            8 => Self::CryptoVerifyEd25519,
            9 => Self::CtxSender,
            10 => Self::CtxChainId,
            11 => Self::CtxTxHash,
            12 => Self::CtxBlockHeight,
            13 => Self::MemAlloc,
            _ => bail!("invalid CostClass: {}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Opcode {
    // constants
    ConstI32 = 0x01,
    ConstI64 = 0x02,
    ConstU32 = 0x03,
    ConstU64 = 0x04,
    ConstBool = 0x05,
    ConstBytes = 0x06,

    // checked arithmetic
    AddChk = 0x10,
    SubChk = 0x11,
    MulChk = 0x12,
    DivChk = 0x13,
    ModChk = 0x14,

    // wrapping arithmetic
    AddWrap = 0x18,
    SubWrap = 0x19,
    MulWrap = 0x1A,

    // comparisons
    Eq = 0x20,
    Neq = 0x21,
    Lt = 0x22,
    Lte = 0x23,
    Gt = 0x24,
    Gte = 0x25,

    // boolean (b1)
    AndB1 = 0x28,
    OrB1 = 0x29,
    NotB1 = 0x2A,

    // memory/slice
    SliceLen = 0x30,
    SlicePtr = 0x31,
    SliceMake = 0x32,
    SliceEq = 0x33,
    SliceSliceChk = 0x34,
    Alloc = 0x35,
    MemLoadU8 = 0x36,
    MemStoreU8 = 0x37,
    MemCopy = 0x38,
    ConcatChk = 0x39,

    // host + gas
    HostCall = 0x40,
    GasTick = 0x41,

    // v0.1.1 bitwise + shift
    Band = 0x50,
    Bor = 0x51,
    Bxor = 0x52,
    Bnot = 0x53,
    Shl = 0x54,
    Shr = 0x55,
}
impl Opcode {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0x01 => Self::ConstI32,
            0x02 => Self::ConstI64,
            0x03 => Self::ConstU32,
            0x04 => Self::ConstU64,
            0x05 => Self::ConstBool,
            0x06 => Self::ConstBytes,

            0x10 => Self::AddChk,
            0x11 => Self::SubChk,
            0x12 => Self::MulChk,
            0x13 => Self::DivChk,
            0x14 => Self::ModChk,

            0x18 => Self::AddWrap,
            0x19 => Self::SubWrap,
            0x1A => Self::MulWrap,

            0x20 => Self::Eq,
            0x21 => Self::Neq,
            0x22 => Self::Lt,
            0x23 => Self::Lte,
            0x24 => Self::Gt,
            0x25 => Self::Gte,

            0x28 => Self::AndB1,
            0x29 => Self::OrB1,
            0x2A => Self::NotB1,

            0x30 => Self::SliceLen,
            0x31 => Self::SlicePtr,
            0x32 => Self::SliceMake,
            0x33 => Self::SliceEq,
            0x34 => Self::SliceSliceChk,
            0x35 => Self::Alloc,
            0x36 => Self::MemLoadU8,
            0x37 => Self::MemStoreU8,
            0x38 => Self::MemCopy,
            0x39 => Self::ConcatChk,

            0x40 => Self::HostCall,
            0x41 => Self::GasTick,

            0x50 => Self::Band,
            0x51 => Self::Bor,
            0x52 => Self::Bxor,
            0x53 => Self::Bnot,
            0x54 => Self::Shl,
            0x55 => Self::Shr,

            _ => bail!("unknown opcode: 0x{:02x}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TermKind {
    Br = 0x01,
    BrIf = 0x02,
    Ret = 0x03,
}
impl TermKind {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0x01 => Self::Br,
            0x02 => Self::BrIf,
            0x03 => Self::Ret,
            _ => bail!("invalid terminator kind: 0x{:02x}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SectionId {
    Module = 1,
    Caps = 2,
    Strings = 3,
    Types = 4,
    HostCalls = 5,
    Funcs = 6,
    Debug = 7,
}
impl SectionId {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            1 => Self::Module,
            2 => Self::Caps,
            3 => Self::Strings,
            4 => Self::Types,
            5 => Self::HostCalls,
            6 => Self::Funcs,
            7 => Self::Debug,
            _ => bail!("invalid section id: {}", x),
        })
    }
}

// CONST_BYTES mode
#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ConstBytesMode {
    StringRef = 0x00, // uleb string_id containing hex text
    Inline = 0x01,    // uleb len + bytes
}
impl ConstBytesMode {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0x00 => Self::StringRef,
            0x01 => Self::Inline,
            _ => bail!("invalid ConstBytesMode: 0x{:02x}", x),
        })
    }
}

// ============================ ULEB128 ============================

pub fn write_uleb(mut v: u64, out: &mut Vec<u8>) {
    loop {
        let mut byte = (v & 0x7F) as u8;
        v >>= 7;
        if v != 0 {
            byte |= 0x80;
        }
        out.push(byte);
        if v == 0 {
            break;
        }
    }
}

pub fn read_uleb(input: &[u8], i: &mut usize) -> Result<u64> {
    let mut result: u64 = 0;
    let mut shift: u32 = 0;
    loop {
        if *i >= input.len() {
            bail!("uleb: unexpected EOF");
        }
        let byte = input[*i];
        *i += 1;

        let low = (byte & 0x7F) as u64;
        result |= low << shift;

        if (byte & 0x80) == 0 {
            return Ok(result);
        }
        shift += 7;
        if shift > 63 {
            bail!("uleb: overflow");
        }
    }
}

// ============================ Fixed-width helpers ============================

pub fn write_u16_le(v: u16, out: &mut Vec<u8>) {
    out.extend_from_slice(&v.to_le_bytes());
}
pub fn write_u32_le(v: u32, out: &mut Vec<u8>) {
    out.extend_from_slice(&v.to_le_bytes());
}
pub fn write_u64_le(v: u64, out: &mut Vec<u8>) {
    out.extend_from_slice(&v.to_le_bytes());
}

pub fn read_u16_le(input: &[u8], i: &mut usize) -> Result<u16> {
    if *i + 2 > input.len() {
        bail!("u16: unexpected EOF");
    }
    let mut b = [0u8; 2];
    b.copy_from_slice(&input[*i..*i + 2]);
    *i += 2;
    Ok(u16::from_le_bytes(b))
}
pub fn read_u32_le(input: &[u8], i: &mut usize) -> Result<u32> {
    if *i + 4 > input.len() {
        bail!("u32: unexpected EOF");
    }
    let mut b = [0u8; 4];
    b.copy_from_slice(&input[*i..*i + 4]);
    *i += 4;
    Ok(u32::from_le_bytes(b))
}
pub fn read_u64_le(input: &[u8], i: &mut usize) -> Result<u64> {
    if *i + 8 > input.len() {
        bail!("u64: unexpected EOF");
    }
    let mut b = [0u8; 8];
    b.copy_from_slice(&input[*i..*i + 8]);
    *i += 8;
    Ok(u64::from_le_bytes(b))
}

// ============================ Header + Section framing ============================

pub fn write_header(out: &mut Vec<u8>) {
    out.extend_from_slice(MAGIC);
    write_u16_le(VERSION_MAJOR, out);
    write_u16_le(VERSION_MINOR, out);
}

pub fn read_header(input: &[u8], i: &mut usize) -> Result<()> {
    if *i + 6 > input.len() {
        bail!("header: unexpected EOF (magic)");
    }
    let magic = &input[*i..*i + 6];
    *i += 6;
    if magic != MAGIC {
        bail!("header: bad magic");
    }
    let maj = read_u16_le(input, i)?;
    let min = read_u16_le(input, i)?;
    if maj != VERSION_MAJOR || min != VERSION_MINOR {
        bail!("header: unsupported version {}.{}", maj, min);
    }
    Ok(())
}

/// Write a section: [section_id:u8][len:uleb][payload...]
pub fn write_section(id: SectionId, payload: &[u8], out: &mut Vec<u8>) {
    out.push(id as u8);
    write_uleb(payload.len() as u64, out);
    out.extend_from_slice(payload);
}

/// Read next section header, returns (id, payload_slice_range)
pub fn read_section_header(input: &[u8], i: &mut usize) -> Result<(SectionId, usize, usize)> {
    if *i >= input.len() {
        bail!("section: unexpected EOF (id)");
    }
    let id = SectionId::from_u8(input[*i])?;
    *i += 1;
    let len = read_uleb(input, i)? as usize;

    let start = *i;
    let end = start.checked_add(len).ok_or_else(|| anyhow::anyhow!("section len overflow"))?;
    if end > input.len() {
        bail!("section: payload out of bounds");
    }
    *i = end;
    Ok((id, start, end))
}

// ============================ Instruction Immediate Decoding ============================

/// A minimal view of an instruction “header” as parsed from the stream.
/// Higher-level code can decide how to interpret operand references and types.
#[derive(Debug, Clone)]
pub struct InstrHead {
    pub opcode: Opcode,
    pub out_types: Vec<TypeCode>,
    pub operands: Vec<u64>, // raw packed refs (uleb decoded)
    pub imm: InstrImm,
}

#[derive(Debug, Clone)]
pub enum InstrImm {
    None,
    HostCall { hostcall_index: u64 },
    ConstBytesStringRef { string_id: u64 },
    ConstBytesInline { bytes: Vec<u8> },
}

/// Decode one instruction header for GALIR-B:
/// [opcode:u8]
/// [out_count:uleb][out_types:u8*]
/// [operand_count:uleb][operands:uleb*]
/// [opcode-specific immediate...]
pub fn read_instr_head(input: &[u8], i: &mut usize) -> Result<InstrHead> {
    if *i >= input.len() {
        bail!("instr: unexpected EOF (opcode)");
    }
    let opcode = Opcode::from_u8(input[*i])?;
    *i += 1;

    let out_count = read_uleb(input, i)? as usize;
    let mut out_types = Vec::with_capacity(out_count);
    for _ in 0..out_count {
        if *i >= input.len() {
            bail!("instr: EOF reading out type");
        }
        out_types.push(TypeCode::from_u8(input[*i])?);
        *i += 1;
    }

    let op_count = read_uleb(input, i)? as usize;
    let mut operands = Vec::with_capacity(op_count);
    for _ in 0..op_count {
        let r = read_uleb(input, i)?;
        operands.push(r);
    }

    let imm = match opcode {
        Opcode::HostCall => {
            let idx = read_uleb(input, i)?;
            InstrImm::HostCall { hostcall_index: idx }
        }
        Opcode::ConstBytes => {
            if *i >= input.len() {
                bail!("const_bytes: EOF reading mode");
            }
            let mode = ConstBytesMode::from_u8(input[*i])?;
            *i += 1;
            match mode {
                ConstBytesMode::StringRef => {
                    let sid = read_uleb(input, i)?;
                    InstrImm::ConstBytesStringRef { string_id: sid }
                }
                ConstBytesMode::Inline => {
                    let len = read_uleb(input, i)? as usize;
                    if *i + len > input.len() {
                        bail!("const_bytes inline: out of bounds");
                    }
                    let bytes = input[*i..*i + len].to_vec();
                    *i += len;
                    InstrImm::ConstBytesInline { bytes }
                }
            }
        }
        _ => InstrImm::None,
    };

    Ok(InstrHead {
        opcode,
        out_types,
        operands,
        imm,
    })
}

/// Encode instruction header (no SSA assignment info, since that is implicit in out_types).
pub fn write_instr_head(h: &InstrHead, out: &mut Vec<u8>) {
    out.push(h.opcode as u8);

    write_uleb(h.out_types.len() as u64, out);
    for t in &h.out_types {
        out.push(*t as u8);
    }

    write_uleb(h.operands.len() as u64, out);
    for r in &h.operands {
        write_uleb(*r, out);
    }

    match &h.imm {
        InstrImm::None => {}
        InstrImm::HostCall { hostcall_index } => {
            // only valid when opcode=HOST_CALL
            write_uleb(*hostcall_index, out);
        }
        InstrImm::ConstBytesStringRef { string_id } => {
            out.push(ConstBytesMode::StringRef as u8);
            write_uleb(*string_id, out);
        }
        InstrImm::ConstBytesInline { bytes } => {
            out.push(ConstBytesMode::Inline as u8);
            write_uleb(bytes.len() as u64, out);
            out.extend_from_slice(bytes);
        }
    }
}

// ============================ Terminator helpers ============================

#[derive(Debug, Clone)]
pub enum Terminator {
    Br {
        target: u64,
        args: Vec<u64>, // packed refs
    },
    BrIf {
        cond: u64, // packed ref
        t_target: u64,
        t_args: Vec<u64>,
        f_target: u64,
        f_args: Vec<u64>,
    },
    Ret {
        values: Vec<u64>, // packed refs
    },
}

/// Encode terminator:
/// [kind:u8] + payload:
/// BR:    [target:uleb][argc:uleb][args:uleb*]
/// BR_IF: [cond:uleb][t_target:uleb][t_argc:uleb][t_args*][f_target:uleb][f_argc:uleb][f_args*]
/// RET:   [retc:uleb][rets:uleb*]
pub fn write_terminator(t: &Terminator, out: &mut Vec<u8>) {
    match t {
        Terminator::Br { target, args } => {
            out.push(TermKind::Br as u8);
            write_uleb(*target, out);
            write_uleb(args.len() as u64, out);
            for a in args {
                write_uleb(*a, out);
            }
        }
        Terminator::BrIf {
            cond,
            t_target,
            t_args,
            f_target,
            f_args,
        } => {
            out.push(TermKind::BrIf as u8);
            write_uleb(*cond, out);
            write_uleb(*t_target, out);
            write_uleb(t_args.len() as u64, out);
            for a in t_args {
                write_uleb(*a, out);
            }
            write_uleb(*f_target, out);
            write_uleb(f_args.len() as u64, out);
            for a in f_args {
                write_uleb(*a, out);
            }
        }
        Terminator::Ret { values } => {
            out.push(TermKind::Ret as u8);
            write_uleb(values.len() as u64, out);
            for v in values {
                write_uleb(*v, out);
            }
        }
    }
}

pub fn read_terminator(input: &[u8], i: &mut usize) -> Result<Terminator> {
    if *i >= input.len() {
        bail!("terminator: EOF (kind)");
    }
    let kind = TermKind::from_u8(input[*i])?;
    *i += 1;

    Ok(match kind {
        TermKind::Br => {
            let target = read_uleb(input, i)?;
            let argc = read_uleb(input, i)? as usize;
            let mut args = Vec::with_capacity(argc);
            for _ in 0..argc {
                args.push(read_uleb(input, i)?);
            }
            Terminator::Br { target, args }
        }
        TermKind::BrIf => {
            let cond = read_uleb(input, i)?;
            let t_target = read_uleb(input, i)?;
            let t_argc = read_uleb(input, i)? as usize;
            let mut t_args = Vec::with_capacity(t_argc);
            for _ in 0..t_argc {
                t_args.push(read_uleb(input, i)?);
            }

            let f_target = read_uleb(input, i)?;
            let f_argc = read_uleb(input, i)? as usize;
            let mut f_args = Vec::with_capacity(f_argc);
            for _ in 0..f_argc {
                f_args.push(read_uleb(input, i)?);
            }

            Terminator::BrIf {
                cond,
                t_target,
                t_args,
                f_target,
                f_args,
            }
        }
        TermKind::Ret => {
            let retc = read_uleb(input, i)? as usize;
            let mut values = Vec::with_capacity(retc);
            for _ in 0..retc {
                values.push(read_uleb(input, i)?);
            }
            Terminator::Ret { values }
        }
    })
}

// ============================ Small section helpers ============================

pub fn write_string_table(strings: &[String]) -> Vec<u8> {
    let mut out = Vec::new();
    write_uleb(strings.len() as u64, &mut out);
    for s in strings {
        let b = s.as_bytes();
        write_uleb(b.len() as u64, &mut out);
        out.extend_from_slice(b);
    }
    out
}

pub fn read_string_table(input: &[u8]) -> Result<Vec<String>> {
    let mut i = 0usize;
    let count = read_uleb(input, &mut i)? as usize;
    let mut out = Vec::with_capacity(count);
    for _ in 0..count {
        let len = read_uleb(input, &mut i)? as usize;
        if i + len > input.len() {
            bail!("strings: out of bounds");
        }
        let s = std::str::from_utf8(&input[i..i + len]).map_err(|_| anyhow::anyhow!("strings: invalid utf8"))?;
        out.push(s.to_string());
        i += len;
    }
    if i != input.len() {
        bail!("strings: trailing bytes");
    }
    Ok(out)
}

pub fn write_caps(caps: &[Cap]) -> Vec<u8> {
    let mut out = Vec::new();
    write_uleb(caps.len() as u64, &mut out);
    for c in caps {
        out.push(*c as u8);
    }
    out
}

pub fn read_caps(input: &[u8]) -> Result<Vec<Cap>> {
    let mut i = 0usize;
    let count = read_uleb(input, &mut i)? as usize;
    let mut out = Vec::with_capacity(count);
    for _ in 0..count {
        if i >= input.len() {
            bail!("caps: EOF");
        }
        out.push(Cap::from_u8(input[i])?);
        i += 1;
    }
    if i != input.len() {
        bail!("caps: trailing bytes");
    }
    Ok(out)
}

// ============================ Quick sanity tests ============================

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn uleb_roundtrip() {
        let vals = [0u64, 1, 2, 127, 128, 129, 16384, u32::MAX as u64, u64::MAX / 2];
        for v in vals {
            let mut b = Vec::new();
            write_uleb(v, &mut b);
            let mut i = 0usize;
            let x = read_uleb(&b, &mut i).unwrap();
            assert_eq!(x, v);
            assert_eq!(i, b.len());
        }
    }

    #[test]
    fn instr_head_roundtrip_const_bytes_inline() {
        let h = InstrHead {
            opcode: Opcode::ConstBytes,
            out_types: vec![TypeCode::Slice],
            operands: vec![],
            imm: InstrImm::ConstBytesInline { bytes: vec![1, 2, 3] },
        };
        let mut b = Vec::new();
        write_instr_head(&h, &mut b);
        let mut i = 0usize;
        let h2 = read_instr_head(&b, &mut i).unwrap();
        assert_eq!(i, b.len());
        match h2.imm {
            InstrImm::ConstBytesInline { bytes } => assert_eq!(bytes, vec![1,2,3]),
            _ => panic!("wrong imm"),
        }
        assert_eq!(h2.opcode, Opcode::ConstBytes);
        assert_eq!(h2.out_types, vec![TypeCode::Slice]);
    }

    #[test]
    fn terminator_roundtrip() {
        let t = Terminator::BrIf {
            cond: 5,
            t_target: 1,
            t_args: vec![10, 11],
            f_target: 2,
            f_args: vec![12],
        };
        let mut b = Vec::new();
        write_terminator(&t, &mut b);
        let mut i = 0usize;
        let t2 = read_terminator(&b, &mut i).unwrap();
        assert_eq!(i, b.len());
        match t2 {
            Terminator::BrIf { cond, t_target, t_args, f_target, f_args } => {
                assert_eq!(cond, 5);
                assert_eq!(t_target, 1);
                assert_eq!(t_args, vec![10,11]);
                assert_eq!(f_target, 2);
                assert_eq!(f_args, vec![12]);
            }
            _ => panic!("wrong terminator"),
        }
    }
}
```

### How you use this module

* **Serializer**: build payloads for sections (`strings`, `caps`, `hostcalls`, `funcs`…), then `write_header()` + `write_section()` in canonical order.
* **Deserializer/Validator**: `read_header()`, then loop with `read_section_header()` and dispatch parsing by `SectionId`.
* **Instruction stream**: inside function bodies, use `read_instr_head()` / `write_instr_head()` for instruction headers and `read_terminator()` / `write_terminator()` for block terminators.

If you want, I can extend this same file with:

* full `HostCallEntry` encode/decode
* full `FuncEntry` + block stream encode/decode
* and a “minimal validator skeleton” that walks the module and checks opcode/type legality using these enums.
```rust
// galir_bin_full.rs
//
// GALIR-B v0.1.1 (binary header major=0 minor=2)
// Authoritative enums + full-ish encode/decode for:
// - header + section framing
// - strings, caps
// - hostcalls table
// - funcs table (blocks + instructions + terminators)
// - minimal validator skeleton (structural + SSA refs + hostcall signature match + main signature)
//
// This is designed so serializer + validator share the same decoding logic.
//
// -------------------- Cargo.toml deps --------------------
// [dependencies]
// anyhow = "1"
//
// ---------------------------------------------------------

use anyhow::{anyhow, bail, Context, Result};
use std::collections::{BTreeSet, HashSet};

pub const MAGIC: &[u8; 6] = b"GALIR\0";
pub const VERSION_MAJOR: u16 = 0;
pub const VERSION_MINOR: u16 = 2; // v0.1.1 locked as binary (0,2)

// ============================ Enums ============================

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub enum TypeCode {
    B1 = 0,
    I32 = 1,
    I64 = 2,
    U32 = 3,
    U64 = 4,
    U256 = 5,
    Unit = 6,
    Slice = 7,
    LayoutRef = 8,
}
impl TypeCode {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0 => Self::B1,
            1 => Self::I32,
            2 => Self::I64,
            3 => Self::U32,
            4 => Self::U64,
            5 => Self::U256,
            6 => Self::Unit,
            7 => Self::Slice,
            8 => Self::LayoutRef,
            _ => bail!("invalid TypeCode: {}", x),
        })
    }
    pub fn is_int_32_64(self) -> bool {
        matches!(self, TypeCode::I32 | TypeCode::I64 | TypeCode::U32 | TypeCode::U64)
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Cap {
    Storage = 0,
    Events = 1,
    Crypto = 2,
    Context = 3,
    Gas = 4,
    Mem = 5,
}
impl Cap {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0 => Self::Storage,
            1 => Self::Events,
            2 => Self::Crypto,
            3 => Self::Context,
            4 => Self::Gas,
            5 => Self::Mem,
            _ => bail!("invalid Cap: {}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum CostClass {
    GasTick = 0,
    StorageHas = 1,
    StorageGet = 2,
    StoragePut = 3,
    StorageDel = 4,
    EventEmit = 5,
    CryptoSha256 = 6,
    CryptoBlake2b256 = 7,
    CryptoVerifyEd25519 = 8,
    CtxSender = 9,
    CtxChainId = 10,
    CtxTxHash = 11,
    CtxBlockHeight = 12,
    MemAlloc = 13,
}
impl CostClass {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0 => Self::GasTick,
            1 => Self::StorageHas,
            2 => Self::StorageGet,
            3 => Self::StoragePut,
            4 => Self::StorageDel,
            5 => Self::EventEmit,
            6 => Self::CryptoSha256,
            7 => Self::CryptoBlake2b256,
            8 => Self::CryptoVerifyEd25519,
            9 => Self::CtxSender,
            10 => Self::CtxChainId,
            11 => Self::CtxTxHash,
            12 => Self::CtxBlockHeight,
            13 => Self::MemAlloc,
            _ => bail!("invalid CostClass: {}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Opcode {
    // constants
    ConstI32 = 0x01,
    ConstI64 = 0x02,
    ConstU32 = 0x03,
    ConstU64 = 0x04,
    ConstBool = 0x05,
    ConstBytes = 0x06,

    // checked arithmetic
    AddChk = 0x10,
    SubChk = 0x11,
    MulChk = 0x12,
    DivChk = 0x13,
    ModChk = 0x14,

    // wrapping arithmetic
    AddWrap = 0x18,
    SubWrap = 0x19,
    MulWrap = 0x1A,

    // comparisons
    Eq = 0x20,
    Neq = 0x21,
    Lt = 0x22,
    Lte = 0x23,
    Gt = 0x24,
    Gte = 0x25,

    // boolean (b1)
    AndB1 = 0x28,
    OrB1 = 0x29,
    NotB1 = 0x2A,

    // memory/slice
    SliceLen = 0x30,
    SlicePtr = 0x31,
    SliceMake = 0x32,
    SliceEq = 0x33,
    SliceSliceChk = 0x34,
    Alloc = 0x35,
    MemLoadU8 = 0x36,
    MemStoreU8 = 0x37,
    MemCopy = 0x38,
    ConcatChk = 0x39,

    // host + gas
    HostCall = 0x40,
    GasTick = 0x41,

    // v0.1.1 bitwise + shift
    Band = 0x50,
    Bor = 0x51,
    Bxor = 0x52,
    Bnot = 0x53,
    Shl = 0x54,
    Shr = 0x55,
}
impl Opcode {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0x01 => Self::ConstI32,
            0x02 => Self::ConstI64,
            0x03 => Self::ConstU32,
            0x04 => Self::ConstU64,
            0x05 => Self::ConstBool,
            0x06 => Self::ConstBytes,

            0x10 => Self::AddChk,
            0x11 => Self::SubChk,
            0x12 => Self::MulChk,
            0x13 => Self::DivChk,
            0x14 => Self::ModChk,

            0x18 => Self::AddWrap,
            0x19 => Self::SubWrap,
            0x1A => Self::MulWrap,

            0x20 => Self::Eq,
            0x21 => Self::Neq,
            0x22 => Self::Lt,
            0x23 => Self::Lte,
            0x24 => Self::Gt,
            0x25 => Self::Gte,

            0x28 => Self::AndB1,
            0x29 => Self::OrB1,
            0x2A => Self::NotB1,

            0x30 => Self::SliceLen,
            0x31 => Self::SlicePtr,
            0x32 => Self::SliceMake,
            0x33 => Self::SliceEq,
            0x34 => Self::SliceSliceChk,
            0x35 => Self::Alloc,
            0x36 => Self::MemLoadU8,
            0x37 => Self::MemStoreU8,
            0x38 => Self::MemCopy,
            0x39 => Self::ConcatChk,

            0x40 => Self::HostCall,
            0x41 => Self::GasTick,

            0x50 => Self::Band,
            0x51 => Self::Bor,
            0x52 => Self::Bxor,
            0x53 => Self::Bnot,
            0x54 => Self::Shl,
            0x55 => Self::Shr,

            _ => bail!("unknown opcode: 0x{:02x}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TermKind {
    Br = 0x01,
    BrIf = 0x02,
    Ret = 0x03,
}
impl TermKind {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0x01 => Self::Br,
            0x02 => Self::BrIf,
            0x03 => Self::Ret,
            _ => bail!("invalid terminator kind: 0x{:02x}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SectionId {
    Module = 1,
    Caps = 2,
    Strings = 3,
    Types = 4,
    HostCalls = 5,
    Funcs = 6,
    Debug = 7,
}
impl SectionId {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            1 => Self::Module,
            2 => Self::Caps,
            3 => Self::Strings,
            4 => Self::Types,
            5 => Self::HostCalls,
            6 => Self::Funcs,
            7 => Self::Debug,
            _ => bail!("invalid section id: {}", x),
        })
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ConstBytesMode {
    StringRef = 0x00, // uleb string_id containing hex text
    Inline = 0x01,    // uleb len + bytes
}
impl ConstBytesMode {
    pub fn from_u8(x: u8) -> Result<Self> {
        Ok(match x {
            0x00 => Self::StringRef,
            0x01 => Self::Inline,
            _ => bail!("invalid ConstBytesMode: 0x{:02x}", x),
        })
    }
}

// ============================ Packed Ref helpers ============================

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RefKind {
    SsaValue = 0,
    BlockParam = 1,
    ImmediateSmall = 2,
    Reserved = 3,
}

pub fn ref_kind(packed: u64) -> RefKind {
    match (packed & 0x3) as u8 {
        0 => RefKind::SsaValue,
        1 => RefKind::BlockParam,
        2 => RefKind::ImmediateSmall,
        _ => RefKind::Reserved,
    }
}

pub fn ref_index(packed: u64) -> u64 {
    packed >> 2
}

pub fn pack_ref(kind: RefKind, index: u64) -> u64 {
    (index << 2) | (kind as u64)
}

// ============================ ULEB128 + primitives ============================

pub fn write_uleb(mut v: u64, out: &mut Vec<u8>) {
    loop {
        let mut byte = (v & 0x7F) as u8;
        v >>= 7;
        if v != 0 {
            byte |= 0x80;
        }
        out.push(byte);
        if v == 0 {
            break;
        }
    }
}

pub fn read_uleb(input: &[u8], i: &mut usize) -> Result<u64> {
    let mut result: u64 = 0;
    let mut shift: u32 = 0;
    loop {
        if *i >= input.len() {
            bail!("uleb: unexpected EOF");
        }
        let byte = input[*i];
        *i += 1;

        let low = (byte & 0x7F) as u64;
        result |= low << shift;

        if (byte & 0x80) == 0 {
            return Ok(result);
        }
        shift += 7;
        if shift > 63 {
            bail!("uleb: overflow");
        }
    }
}

pub fn write_u16_le(v: u16, out: &mut Vec<u8>) {
    out.extend_from_slice(&v.to_le_bytes());
}
pub fn read_u16_le(input: &[u8], i: &mut usize) -> Result<u16> {
    if *i + 2 > input.len() {
        bail!("u16: unexpected EOF");
    }
    let mut b = [0u8; 2];
    b.copy_from_slice(&input[*i..*i + 2]);
    *i += 2;
    Ok(u16::from_le_bytes(b))
}

// ============================ Header + Section framing ============================

pub fn write_header(out: &mut Vec<u8>) {
    out.extend_from_slice(MAGIC);
    write_u16_le(VERSION_MAJOR, out);
    write_u16_le(VERSION_MINOR, out);
}

pub fn read_header(input: &[u8], i: &mut usize) -> Result<()> {
    if *i + 6 > input.len() {
        bail!("header: unexpected EOF (magic)");
    }
    let magic = &input[*i..*i + 6];
    *i += 6;
    if magic != MAGIC {
        bail!("header: bad magic");
    }
    let maj = read_u16_le(input, i)?;
    let min = read_u16_le(input, i)?;
    if maj != VERSION_MAJOR || min != VERSION_MINOR {
        bail!("header: unsupported version {}.{}", maj, min);
    }
    Ok(())
}

/// Write a section: [section_id:u8][len:uleb][payload...]
pub fn write_section(id: SectionId, payload: &[u8], out: &mut Vec<u8>) {
    out.push(id as u8);
    write_uleb(payload.len() as u64, out);
    out.extend_from_slice(payload);
}

/// Read next section header, returns (id, payload_slice_range)
pub fn read_section_header(input: &[u8], i: &mut usize) -> Result<(SectionId, usize, usize)> {
    if *i >= input.len() {
        bail!("section: unexpected EOF (id)");
    }
    let id = SectionId::from_u8(input[*i])?;
    *i += 1;

    let len = read_uleb(input, i)? as usize;
    let start = *i;
    let end = start
        .checked_add(len)
        .ok_or_else(|| anyhow!("section len overflow"))?;
    if end > input.len() {
        bail!("section: payload out of bounds");
    }
    *i = end;
    Ok((id, start, end))
}

// ============================ Instruction encoding ============================

#[derive(Debug, Clone)]
pub struct InstrHead {
    pub opcode: Opcode,
    pub out_types: Vec<TypeCode>,
    pub operands: Vec<u64>, // packed refs (uleb decoded)
    pub imm: InstrImm,
}

#[derive(Debug, Clone)]
pub enum InstrImm {
    None,
    HostCall { hostcall_index: u64 },
    ConstBytesStringRef { string_id: u64 },
    ConstBytesInline { bytes: Vec<u8> },
}

/// Decode instruction head:
/// [opcode:u8]
/// [out_count:uleb][out_types:u8*]
/// [operand_count:uleb][operands:uleb*]
/// [opcode-specific immediate...]
pub fn read_instr_head(input: &[u8], i: &mut usize) -> Result<InstrHead> {
    if *i >= input.len() {
        bail!("instr: unexpected EOF (opcode)");
    }
    let opcode = Opcode::from_u8(input[*i])?;
    *i += 1;

    let out_count = read_uleb(input, i)? as usize;
    let mut out_types = Vec::with_capacity(out_count);
    for _ in 0..out_count {
        if *i >= input.len() {
            bail!("instr: EOF reading out type");
        }
        out_types.push(TypeCode::from_u8(input[*i])?);
        *i += 1;
    }

    let op_count = read_uleb(input, i)? as usize;
    let mut operands = Vec::with_capacity(op_count);
    for _ in 0..op_count {
        operands.push(read_uleb(input, i)?);
    }

    let imm = match opcode {
        Opcode::HostCall => {
            let idx = read_uleb(input, i)?;
            InstrImm::HostCall { hostcall_index: idx }
        }
        Opcode::ConstBytes => {
            if *i >= input.len() {
                bail!("const_bytes: EOF reading mode");
            }
            let mode = ConstBytesMode::from_u8(input[*i])?;
            *i += 1;
            match mode {
                ConstBytesMode::StringRef => {
                    let sid = read_uleb(input, i)?;
                    InstrImm::ConstBytesStringRef { string_id: sid }
                }
                ConstBytesMode::Inline => {
                    let len = read_uleb(input, i)? as usize;
                    if *i + len > input.len() {
                        bail!("const_bytes inline: out of bounds");
                    }
                    let bytes = input[*i..*i + len].to_vec();
                    *i += len;
                    InstrImm::ConstBytesInline { bytes }
                }
            }
        }
        _ => InstrImm::None,
    };

    Ok(InstrHead {
        opcode,
        out_types,
        operands,
        imm,
    })
}

pub fn write_instr_head(h: &InstrHead, out: &mut Vec<u8>) {
    out.push(h.opcode as u8);

    write_uleb(h.out_types.len() as u64, out);
    for t in &h.out_types {
        out.push(*t as u8);
    }

    write_uleb(h.operands.len() as u64, out);
    for r in &h.operands {
        write_uleb(*r, out);
    }

    match &h.imm {
        InstrImm::None => {}
        InstrImm::HostCall { hostcall_index } => write_uleb(*hostcall_index, out),
        InstrImm::ConstBytesStringRef { string_id } => {
            out.push(ConstBytesMode::StringRef as u8);
            write_uleb(*string_id, out);
        }
        InstrImm::ConstBytesInline { bytes } => {
            out.push(ConstBytesMode::Inline as u8);
            write_uleb(bytes.len() as u64, out);
            out.extend_from_slice(bytes);
        }
    }
}

// ============================ Terminators ============================

#[derive(Debug, Clone)]
pub enum Terminator {
    Br { target: u64, args: Vec<u64> },
    BrIf {
        cond: u64,
        t_target: u64,
        t_args: Vec<u64>,
        f_target: u64,
        f_args: Vec<u64>,
    },
    Ret { values: Vec<u64> },
}

pub fn write_terminator(t: &Terminator, out: &mut Vec<u8>) {
    match t {
        Terminator::Br { target, args } => {
            out.push(TermKind::Br as u8);
            write_uleb(*target, out);
            write_uleb(args.len() as u64, out);
            for a in args {
                write_uleb(*a, out);
            }
        }
        Terminator::BrIf {
            cond,
            t_target,
            t_args,
            f_target,
            f_args,
        } => {
            out.push(TermKind::BrIf as u8);
            write_uleb(*cond, out);

            write_uleb(*t_target, out);
            write_uleb(t_args.len() as u64, out);
            for a in t_args {
                write_uleb(*a, out);
            }

            write_uleb(*f_target, out);
            write_uleb(f_args.len() as u64, out);
            for a in f_args {
                write_uleb(*a, out);
            }
        }
        Terminator::Ret { values } => {
            out.push(TermKind::Ret as u8);
            write_uleb(values.len() as u64, out);
            for v in values {
                write_uleb(*v, out);
            }
        }
    }
}

pub fn read_terminator(input: &[u8], i: &mut usize) -> Result<Terminator> {
    if *i >= input.len() {
        bail!("terminator: EOF (kind)");
    }
    let kind = TermKind::from_u8(input[*i])?;
    *i += 1;

    Ok(match kind {
        TermKind::Br => {
            let target = read_uleb(input, i)?;
            let argc = read_uleb(input, i)? as usize;
            let mut args = Vec::with_capacity(argc);
            for _ in 0..argc {
                args.push(read_uleb(input, i)?);
            }
            Terminator::Br { target, args }
        }
        TermKind::BrIf => {
            let cond = read_uleb(input, i)?;

            let t_target = read_uleb(input, i)?;
            let t_argc = read_uleb(input, i)? as usize;
            let mut t_args = Vec::with_capacity(t_argc);
            for _ in 0..t_argc {
                t_args.push(read_uleb(input, i)?);
            }

            let f_target = read_uleb(input, i)?;
            let f_argc = read_uleb(input, i)? as usize;
            let mut f_args = Vec::with_capacity(f_argc);
            for _ in 0..f_argc {
                f_args.push(read_uleb(input, i)?);
            }

            Terminator::BrIf {
                cond,
                t_target,
                t_args,
                f_target,
                f_args,
            }
        }
        TermKind::Ret => {
            let retc = read_uleb(input, i)? as usize;
            let mut values = Vec::with_capacity(retc);
            for _ in 0..retc {
                values.push(read_uleb(input, i)?);
            }
            Terminator::Ret { values }
        }
    })
}

// ============================ Sections: strings, caps ============================

pub fn write_string_table(strings: &[String]) -> Vec<u8> {
    let mut out = Vec::new();
    write_uleb(strings.len() as u64, &mut out);
    for s in strings {
        let b = s.as_bytes();
        write_uleb(b.len() as u64, &mut out);
        out.extend_from_slice(b);
    }
    out
}

pub fn read_string_table(input: &[u8]) -> Result<Vec<String>> {
    let mut i = 0usize;
    let count = read_uleb(input, &mut i)? as usize;
    let mut out = Vec::with_capacity(count);
    for _ in 0..count {
        let len = read_uleb(input, &mut i)? as usize;
        if i + len > input.len() {
            bail!("strings: out of bounds");
        }
        let s = std::str::from_utf8(&input[i..i + len])
            .map_err(|_| anyhow!("strings: invalid utf8"))?;
        out.push(s.to_string());
        i += len;
    }
    if i != input.len() {
        bail!("strings: trailing bytes");
    }
    Ok(out)
}

pub fn write_caps(caps: &[Cap]) -> Vec<u8> {
    let mut out = Vec::new();
    write_uleb(caps.len() as u64, &mut out);
    for c in caps {
        out.push(*c as u8);
    }
    out
}

pub fn read_caps(input: &[u8]) -> Result<Vec<Cap>> {
    let mut i = 0usize;
    let count = read_uleb(input, &mut i)? as usize;
    let mut out = Vec::with_capacity(count);
    for _ in 0..count {
        if i >= input.len() {
            bail!("caps: EOF");
        }
        out.push(Cap::from_u8(input[i])?);
        i += 1;
    }
    if i != input.len() {
        bail!("caps: trailing bytes");
    }
    Ok(out)
}

// ============================ HostCalls Section ============================

#[derive(Debug, Clone)]
pub struct HostCallEntry {
    pub name_sid: u64,            // string table id for "storage_get" etc.
    pub cap: Cap,                 // required cap
    pub cost: CostClass,          // cost class
    pub params: Vec<TypeCode>,    // parameter types
    pub results: Vec<TypeCode>,   // result types
}

/// Encoding:
/// [count:uleb]
/// repeated:
///   [name_sid:uleb]
///   [cap:u8]
///   [cost_class:u8]
///   [param_count:uleb][param_types:u8*]
///   [result_count:uleb][result_types:u8*]
pub fn write_hostcalls(hc: &[HostCallEntry]) -> Vec<u8> {
    let mut out = Vec::new();
    write_uleb(hc.len() as u64, &mut out);
    for e in hc {
        write_uleb(e.name_sid, &mut out);
        out.push(e.cap as u8);
        out.push(e.cost as u8);

        write_uleb(e.params.len() as u64, &mut out);
        for t in &e.params {
            out.push(*t as u8);
        }

        write_uleb(e.results.len() as u64, &mut out);
        for t in &e.results {
            out.push(*t as u8);
        }
    }
    out
}

pub fn read_hostcalls(input: &[u8]) -> Result<Vec<HostCallEntry>> {
    let mut i = 0usize;
    let count = read_uleb(input, &mut i)? as usize;
    let mut out = Vec::with_capacity(count);
    for _ in 0..count {
        let name_sid = read_uleb(input, &mut i)?;
        if i + 2 > input.len() {
            bail!("hostcalls: EOF reading cap/cost");
        }
        let cap = Cap::from_u8(input[i])?;
        i += 1;
        let cost = CostClass::from_u8(input[i])?;
        i += 1;

        let pc = read_uleb(input, &mut i)? as usize;
        let mut params = Vec::with_capacity(pc);
        for _ in 0..pc {
            if i >= input.len() {
                bail!("hostcalls: EOF reading param type");
            }
            params.push(TypeCode::from_u8(input[i])?);
            i += 1;
        }

        let rc = read_uleb(input, &mut i)? as usize;
        let mut results = Vec::with_capacity(rc);
        for _ in 0..rc {
            if i >= input.len() {
                bail!("hostcalls: EOF reading result type");
            }
            results.push(TypeCode::from_u8(input[i])?);
            i += 1;
        }

        out.push(HostCallEntry {
            name_sid,
            cap,
            cost,
            params,
            results,
        });
    }
    if i != input.len() {
        bail!("hostcalls: trailing bytes");
    }
    Ok(out)
}

// ============================ Funcs Section ============================

#[derive(Debug, Clone)]
pub struct FuncEntry {
    pub name_sid: u64,           // string table id (e.g., "main")
    pub is_export: bool,         // exported function
    pub params: Vec<TypeCode>,
    pub results: Vec<TypeCode>,
    pub blocks: Vec<BlockEntry>,
}

/// Block encoding:
/// [param_count:uleb][param_types:u8*]
/// [instr_count:uleb]
///   instr_count times: InstrHead
/// [terminator:Terminator]
#[derive(Debug, Clone)]
pub struct BlockEntry {
    pub params: Vec<TypeCode>,
    pub instrs: Vec<InstrHead>,
    pub term: Terminator,
}

/// Funcs section encoding:
/// [func_count:uleb]
/// repeated:
///   [name_sid:uleb]
///   [is_export:u8] (0/1)
///   [param_count:uleb][param_types:u8*]
///   [result_count:uleb][result_types:u8*]
///   [block_count:uleb]
///     blocks...
pub fn write_funcs(funcs: &[FuncEntry]) -> Vec<u8> {
    let mut out = Vec::new();
    write_uleb(funcs.len() as u64, &mut out);
    for f in funcs {
        write_uleb(f.name_sid, &mut out);
        out.push(if f.is_export { 1 } else { 0 });

        write_uleb(f.params.len() as u64, &mut out);
        for t in &f.params {
            out.push(*t as u8);
        }

        write_uleb(f.results.len() as u64, &mut out);
        for t in &f.results {
            out.push(*t as u8);
        }

        write_uleb(f.blocks.len() as u64, &mut out);
        for b in &f.blocks {
            write_uleb(b.params.len() as u64, &mut out);
            for t in &b.params {
                out.push(*t as u8);
            }

            write_uleb(b.instrs.len() as u64, &mut out);
            for ins in &b.instrs {
                write_instr_head(ins, &mut out);
            }

            write_terminator(&b.term, &mut out);
        }
    }
    out
}

pub fn read_funcs(input: &[u8]) -> Result<Vec<FuncEntry>> {
    let mut i = 0usize;
    let count = read_uleb(input, &mut i)? as usize;
    let mut out = Vec::with_capacity(count);

    for _ in 0..count {
        let name_sid = read_uleb(input, &mut i)?;
        if i >= input.len() {
            bail!("funcs: EOF reading export flag");
        }
        let is_export = match input[i] {
            0 => false,
            1 => true,
            x => bail!("funcs: invalid export flag: {}", x),
        };
        i += 1;

        let pc = read_uleb(input, &mut i)? as usize;
        let mut params = Vec::with_capacity(pc);
        for _ in 0..pc {
            if i >= input.len() {
                bail!("funcs: EOF reading param type");
            }
            params.push(TypeCode::from_u8(input[i])?);
            i += 1;
        }

        let rc = read_uleb(input, &mut i)? as usize;
        let mut results = Vec::with_capacity(rc);
        for _ in 0..rc {
            if i >= input.len() {
                bail!("funcs: EOF reading result type");
            }
            results.push(TypeCode::from_u8(input[i])?);
            i += 1;
        }

        let bc = read_uleb(input, &mut i)? as usize;
        let mut blocks = Vec::with_capacity(bc);

        for _ in 0..bc {
            let bpc = read_uleb(input, &mut i)? as usize;
            let mut bparams = Vec::with_capacity(bpc);
            for _ in 0..bpc {
                if i >= input.len() {
                    bail!("funcs: EOF reading block param type");
                }
                bparams.push(TypeCode::from_u8(input[i])?);
                i += 1;
            }

            let ic = read_uleb(input, &mut i)? as usize;
            let mut instrs = Vec::with_capacity(ic);
            for _ in 0..ic {
                let ins = read_instr_head(input, &mut i)?;
                instrs.push(ins);
            }

            let term = read_terminator(input, &mut i)?;
            blocks.push(BlockEntry {
                params: bparams,
                instrs,
                term,
            });
        }

        out.push(FuncEntry {
            name_sid,
            is_export,
            params,
            results,
            blocks,
        });
    }

    if i != input.len() {
        bail!("funcs: trailing bytes");
    }
    Ok(out)
}

// ============================ Whole module container ============================

#[derive(Debug, Clone)]
pub struct ModuleBin {
    pub module_section_raw: Vec<u8>, // reserved; may be empty
    pub caps: Vec<Cap>,
    pub strings: Vec<String>,
    pub types_section_raw: Vec<u8>, // v0.1.1 leaves layouts/complex types for later
    pub hostcalls: Vec<HostCallEntry>,
    pub funcs: Vec<FuncEntry>,
    pub debug_section_raw: Vec<u8>,
}

impl ModuleBin {
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut out = Vec::new();
        write_header(&mut out);

        // canonical section order
        write_section(SectionId::Module, &self.module_section_raw, &mut out);
        write_section(SectionId::Caps, &write_caps(&self.caps), &mut out);
        write_section(SectionId::Strings, &write_string_table(&self.strings), &mut out);
        write_section(SectionId::Types, &self.types_section_raw, &mut out);
        write_section(SectionId::HostCalls, &write_hostcalls(&self.hostcalls), &mut out);
        write_section(SectionId::Funcs, &write_funcs(&self.funcs), &mut out);

        if !self.debug_section_raw.is_empty() {
            write_section(SectionId::Debug, &self.debug_section_raw, &mut out);
        }

        out
    }

    pub fn from_bytes(bytes: &[u8]) -> Result<Self> {
        let mut i = 0usize;
        read_header(bytes, &mut i)?;

        let mut module_section_raw = None;
        let mut caps = None;
        let mut strings = None;
        let mut types_section_raw = None;
        let mut hostcalls = None;
        let mut funcs = None;
        let mut debug_section_raw = Vec::new();

        // enforce canonical order (strict, for determinism)
        let mut seen = Vec::<SectionId>::new();

        while i < bytes.len() {
            let (sid, s, e) = read_section_header(bytes, &mut i)?;
            seen.push(sid);

            let payload = &bytes[s..e];
            match sid {
                SectionId::Module => module_section_raw = Some(payload.to_vec()),
                SectionId::Caps => caps = Some(read_caps(payload)?),
                SectionId::Strings => strings = Some(read_string_table(payload)?),
                SectionId::Types => types_section_raw = Some(payload.to_vec()),
                SectionId::HostCalls => hostcalls = Some(read_hostcalls(payload)?),
                SectionId::Funcs => funcs = Some(read_funcs(payload)?),
                SectionId::Debug => debug_section_raw = payload.to_vec(),
            }
        }

        // required sections
        let module_section_raw = module_section_raw.ok_or_else(|| anyhow!("missing S_MODULE"))?;
        let caps = caps.ok_or_else(|| anyhow!("missing S_CAPS"))?;
        let strings = strings.ok_or_else(|| anyhow!("missing S_STRINGS"))?;
        let types_section_raw = types_section_raw.ok_or_else(|| anyhow!("missing S_TYPES"))?;
        let hostcalls = hostcalls.ok_or_else(|| anyhow!("missing S_HOSTCALLS"))?;
        let funcs = funcs.ok_or_else(|| anyhow!("missing S_FUNCS"))?;

        // strict order check (exact order, debug optional at end)
        let expected = [
            SectionId::Module,
            SectionId::Caps,
            SectionId::Strings,
            SectionId::Types,
            SectionId::HostCalls,
            SectionId::Funcs,
        ];
        for (idx, exp) in expected.iter().enumerate() {
            if seen.get(idx) != Some(exp) {
                bail!("section order invalid: expected {:?} at index {}", exp, idx);
            }
        }
        if seen.len() > expected.len() {
            // only allow Debug as the final extra section(s) - but v0.1.1 expects at most one
            if seen.len() != expected.len() + 1 || seen[expected.len()] != SectionId::Debug {
                bail!("unexpected extra sections after S_FUNCS");
            }
        }

        Ok(Self {
            module_section_raw,
            caps,
            strings,
            types_section_raw,
            hostcalls,
            funcs,
            debug_section_raw,
        })
    }
}

// ============================ Minimal validator skeleton ============================

#[derive(Debug, Clone)]
pub struct ValidationError {
    pub code: &'static str,
    pub where_: String,
    pub msg: String,
}

impl ValidationError {
    fn new(code: &'static str, where_: impl Into<String>, msg: impl Into<String>) -> Self {
        Self {
            code,
            where_: where_.into(),
            msg: msg.into(),
        }
    }
}

pub type VResult<T> = std::result::Result<T, ValidationError>;

fn vfail(code: &'static str, where_: impl Into<String>, msg: impl Into<String>) -> ValidationError {
    ValidationError::new(code, where_, msg)
}

/// Minimal structural validation:
/// - string id ranges
/// - caps uniqueness
/// - hostcall indices/sigs for HOST_CALL instructions
/// - CONST_BYTES output type must be [slice] and operand_count=0
/// - BR/BR_IF targets in range + block args match block param counts
/// - SSA refs: ssa indices < defs so far; block params in range; immediates <= 63
/// - exported "main" must exist with signature (u32,u32)->(u64) (WASM ABI expectation)
///
/// This is intentionally minimal and safe to start.
/// You can layer on full typing + gas-metering proofs later.
pub fn validate_module_minimal(m: &ModuleBin) -> VResult<()> {
    // caps unique
    {
        let mut set = HashSet::new();
        for c in &m.caps {
            if !set.insert(*c as u8) {
                return Err(vfail("V0001", "module", "duplicate capability"));
            }
        }
    }

    // hostcall name sids + cap/cost ok already by parsing; also ensure string ids in range
    let str_len = m.strings.len() as u64;
    for (idx, hc) in m.hostcalls.iter().enumerate() {
        if hc.name_sid >= str_len {
            return Err(vfail(
                "V0001",
                format!("hostcall[{}]", idx),
                "name_sid out of range",
            ));
        }
        // Optionally: enforce unique hostcall names for determinism
    }

    // func name sids in range + find main
    let mut main_found = false;
    for (fi, f) in m.funcs.iter().enumerate() {
        if f.name_sid >= str_len {
            return Err(vfail("V0001", format!("func[{}]", fi), "name_sid out of range"));
        }
        let fname = &m.strings[f.name_sid as usize];
        if f.is_export && fname == "main" {
            main_found = true;
            // ABI: main(u32,u32)->u64
            if f.params != vec![TypeCode::U32, TypeCode::U32] || f.results != vec![TypeCode::U64] {
                return Err(vfail(
                    "V0015",
                    "func main",
                    "main must have signature (u32,u32)->(u64) for WASM ABI v0.1.1",
                ));
            }
        }

        if f.blocks.is_empty() {
            return Err(vfail("V0008", format!("func[{}]", fi), "function has zero blocks"));
        }

        validate_func_minimal(m, fi, f)?;
    }

    if !main_found {
        return Err(vfail("V0015", "module", "missing exported main"));
    }

    Ok(())
}

fn validate_func_minimal(m: &ModuleBin, fi: usize, f: &FuncEntry) -> VResult<()> {
    // For SSA: defs are assigned by instruction outputs sequentially per function.
    // Block params are separate index space local to each block (as per packed ref kind).
    // This validator only checks "use-before-def" and range.
    let func_where = format!("func[{}]", fi);

    // precompute block param counts
    let block_param_counts: Vec<usize> = f.blocks.iter().map(|b| b.params.len()).collect();
    let block_count = f.blocks.len();

    // track defs count as we walk blocks in order; this implies a canonical block ordering.
    // (Full SSA across blocks usually needs per-block dominance checks; v0.1 minimal keeps it simple.)
    let mut defs_so_far: u64 = 0;

    for (bi, b) in f.blocks.iter().enumerate() {
        let bwhere = format!("{}.block[{}]", func_where, bi);

        // Validate each instruction structurally + refs
        for (ii, ins) in b.instrs.iter().enumerate() {
            let iwhere = format!("{}.instr[{}]", bwhere, ii);

            // Basic opcode-specific structure checks
            match ins.opcode {
                Opcode::ConstBytes => {
                    // must produce exactly one slice
                    if ins.out_types != vec![TypeCode::Slice] {
                        return Err(vfail("V0010", iwhere, "CONST_BYTES must output [slice]"));
                    }
                    if !ins.operands.is_empty() {
                        return Err(vfail("V0010", iwhere, "CONST_BYTES must have 0 operands"));
                    }
                    match ins.imm {
                        InstrImm::ConstBytesInline { .. } | InstrImm::ConstBytesStringRef { .. } => {}
                        _ => return Err(vfail("V0002", iwhere, "CONST_BYTES missing immediate")),
                    }
                }
                Opcode::HostCall => {
                    // must have HostCall imm
                    let hidx = match ins.imm {
                        InstrImm::HostCall { hostcall_index } => hostcall_index,
                        _ => return Err(vfail("V0002", iwhere, "HOST_CALL missing hostcall_index")),
                    };
                    if hidx as usize >= m.hostcalls.len() {
                        return Err(vfail("V0005", iwhere, "hostcall index out of range"));
                    }
                    let hc = &m.hostcalls[hidx as usize];

                    // operand count must match params count, out_types must match results count/types
                    if ins.operands.len() != hc.params.len() {
                        return Err(vfail("V0005", iwhere, "hostcall operand count mismatch"));
                    }
                    if ins.out_types != hc.results {
                        return Err(vfail("V0005", iwhere, "hostcall result types mismatch"));
                    }

                    // cap must be declared in module caps
                    if !m.caps.iter().any(|c| *c == hc.cap) {
                        return Err(vfail("V0004", iwhere, "hostcall capability not enabled"));
                    }
                }
                _ => {
                    // minimal: nothing else enforced here
                }
            }

            // Validate all operand refs are in range
            for (oi, r) in ins.operands.iter().enumerate() {
                validate_ref(*r, defs_so_far, b.params.len() as u64)
                    .map_err(|e| vfail(e.code, format!("{}.op[{}]", iwhere, oi), e.msg))?;
            }

            // For CONST_BYTES string ref mode, ensure string decodes as hex (validator-level determinism)
            if let Opcode::ConstBytes = ins.opcode {
                if let InstrImm::ConstBytesStringRef { string_id } = ins.imm {
                    if string_id as usize >= m.strings.len() {
                        return Err(vfail("V0001", iwhere, "CONST_BYTES string_id out of range"));
                    }
                    let s = &m.strings[string_id as usize];
                    validate_hex_string(s).map_err(|msg| vfail("V0001", iwhere, msg))?;
                }
            }

            // Assign defs for this instruction (SSA outputs)
            defs_so_far = defs_so_far
                .checked_add(ins.out_types.len() as u64)
                .ok_or_else(|| vfail("V0013", iwhere, "SSA defs overflow"))?;
        }

        // Validate terminator refs + targets + arg counts
        validate_terminator_minimal(
            &bwhere,
            &b.term,
            defs_so_far,
            block_count,
            &block_param_counts,
            b.params.len(),
        )?;
    }

    Ok(())
}

fn validate_ref(packed: u64, defs_so_far: u64, block_param_count: u64) -> VResult<()> {
    match ref_kind(packed) {
        RefKind::SsaValue => {
            let idx = ref_index(packed);
            if idx >= defs_so_far {
                return Err(vfail("V0009", "ref", "SSA use-before-def or out of range"));
            }
            Ok(())
        }
        RefKind::BlockParam => {
            let idx = ref_index(packed);
            if idx >= block_param_count {
                return Err(vfail("V0009", "ref", "block param out of range"));
            }
            Ok(())
        }
        RefKind::ImmediateSmall => {
            let idx = ref_index(packed);
            if idx > 63 {
                return Err(vfail("V0009", "ref", "immediate_small out of range (>63)"));
            }
            Ok(())
        }
        RefKind::Reserved => Err(vfail("V0009", "ref", "reserved ref kind")),
    }
}

fn validate_terminator_minimal(
    where_: &str,
    t: &Terminator,
    defs_so_far: u64,
    block_count: usize,
    block_param_counts: &[usize],
    this_block_param_count: usize,
) -> VResult<()> {
    match t {
        Terminator::Br { target, args } => {
            if (*target as usize) >= block_count {
                return Err(vfail("V0008", where_, "BR target out of range"));
            }
            let expected = block_param_counts[*target as usize];
            if args.len() != expected {
                return Err(vfail("V0008", where_, "BR arg count mismatch for target params"));
            }
            for (i, r) in args.iter().enumerate() {
                validate_ref(*r, defs_so_far, this_block_param_count as u64)
                    .map_err(|e| vfail(e.code, format!("{}.br_arg[{}]", where_, i), e.msg))?;
            }
            Ok(())
        }
        Terminator::BrIf {
            cond,
            t_target,
            t_args,
            f_target,
            f_args,
        } => {
            validate_ref(*cond, defs_so_far, this_block_param_count as u64)
                .map_err(|e| vfail(e.code, format!("{}.cond", where_), e.msg))?;

            if (*t_target as usize) >= block_count || (*f_target as usize) >= block_count {
                return Err(vfail("V0008", where_, "BR_IF target out of range"));
            }

            let t_expected = block_param_counts[*t_target as usize];
            if t_args.len() != t_expected {
                return Err(vfail("V0008", where_, "BR_IF true arg count mismatch"));
            }
            let f_expected = block_param_counts[*f_target as usize];
            if f_args.len() != f_expected {
                return Err(vfail("V0008", where_, "BR_IF false arg count mismatch"));
            }

            for (i, r) in t_args.iter().enumerate() {
                validate_ref(*r, defs_so_far, this_block_param_count as u64)
                    .map_err(|e| vfail(e.code, format!("{}.t_arg[{}]", where_, i), e.msg))?;
            }
            for (i, r) in f_args.iter().enumerate() {
                validate_ref(*r, defs_so_far, this_block_param_count as u64)
                    .map_err(|e| vfail(e.code, format!("{}.f_arg[{}]", where_, i), e.msg))?;
            }
            Ok(())
        }
        Terminator::Ret { values } => {
            for (i, r) in values.iter().enumerate() {
                validate_ref(*r, defs_so_far, this_block_param_count as u64)
                    .map_err(|e| vfail(e.code, format!("{}.ret[{}]", where_, i), e.msg))?;
            }
            Ok(())
        }
    }
}

fn validate_hex_string(s: &str) -> Result<()> {
    // even length
    if s.len() % 2 != 0 {
        bail!("CONST_BYTES string hex must have even length");
    }
    // only hex chars
    for ch in s.bytes() {
        let ok = matches!(ch, b'0'..=b'9' | b'a'..=b'f' | b'A'..=b'F');
        if !ok {
            bail!("CONST_BYTES string contains non-hex characters");
        }
    }
    Ok(())
}

// ============================ Optional: stricter opcode-shape checks ============================

/// If you want extra safety early: enforce certain opcode out-counts.
/// Keep it optional so you can stage validator hardening.
pub fn validate_opcode_shapes_strict(m: &ModuleBin) -> VResult<()> {
    for (fi, f) in m.funcs.iter().enumerate() {
        for (bi, b) in f.blocks.iter().enumerate() {
            for (ii, ins) in b.instrs.iter().enumerate() {
                let w = format!("func[{}].block[{}].instr[{}]", fi, bi, ii);
                match ins.opcode {
                    Opcode::ConstI32 => expect_shape(&w, ins, 1, 0, Some(TypeCode::I32))?,
                    Opcode::ConstI64 => expect_shape(&w, ins, 1, 0, Some(TypeCode::I64))?,
                    Opcode::ConstU32 => expect_shape(&w, ins, 1, 0, Some(TypeCode::U32))?,
                    Opcode::ConstU64 => expect_shape(&w, ins, 1, 0, Some(TypeCode::U64))?,
                    Opcode::ConstBool => expect_shape(&w, ins, 1, 0, Some(TypeCode::B1))?,
                    Opcode::Alloc => expect_shape(&w, ins, 1, 1, Some(TypeCode::U32))?,
                    Opcode::SliceLen => expect_shape(&w, ins, 1, 1, Some(TypeCode::U32))?,
                    Opcode::SlicePtr => expect_shape(&w, ins, 1, 1, Some(TypeCode::U32))?,
                    Opcode::MemLoadU8 => expect_shape(&w, ins, 1, 1, Some(TypeCode::U32))?,
                    Opcode::MemStoreU8 => expect_shape(&w, ins, 0, 2, None)?,
                    Opcode::MemCopy => expect_shape(&w, ins, 0, 3, None)?,
                    Opcode::GasTick => {
                        // convention: gas_tick produces 1 b1 (ok) and has 1 operand (cost u64/u32)
                        if ins.out_types != vec![TypeCode::B1] || ins.operands.len() != 1 {
                            return Err(vfail("V0010", w, "GAS_TICK must be (out:[b1], ops:[cost])"));
                        }
                    }
                    Opcode::Band | Opcode::Bor | Opcode::Bxor | Opcode::Shl | Opcode::Shr => {
                        if ins.out_types.len() != 1 || ins.operands.len() != 2 {
                            return Err(vfail("V0010", w, "bitwise/shift must be (1 out, 2 ops)"));
                        }
                        if !ins.out_types[0].is_int_32_64() {
                            return Err(vfail("V0010", w, "bitwise/shift out type must be i32/i64/u32/u64"));
                        }
                    }
                    Opcode::Bnot => {
                        if ins.out_types.len() != 1 || ins.operands.len() != 1 {
                            return Err(vfail("V0010", w, "BNOT must be (1 out, 1 op)"));
                        }
                        if !ins.out_types[0].is_int_32_64() {
                            return Err(vfail("V0010", w, "BNOT out type must be i32/i64/u32/u64"));
                        }
                    }
                    _ => {}
                }
            }
        }
    }
    Ok(())
}

fn expect_shape(where_: &str, ins: &InstrHead, outc: usize, opc: usize, out_ty: Option<TypeCode>) -> VResult<()> {
    if ins.out_types.len() != outc {
        return Err(vfail("V0010", where_, "wrong out count"));
    }
    if ins.operands.len() != opc {
        return Err(vfail("V0010", where_, "wrong operand count"));
    }
    if let Some(t) = out_ty {
        if outc != 1 || ins.out_types[0] != t {
            return Err(vfail("V0010", where_, "wrong out type"));
        }
    }
    Ok(())
}

// ============================ Tests ============================

#[cfg(test)]
mod tests {
    use super::*;

    fn minimal_module() -> ModuleBin {
        ModuleBin {
            module_section_raw: vec![],
            caps: vec![Cap::Gas, Cap::Mem],
            strings: vec![
                "main".to_string(),
                "gas_tick".to_string(),
            ],
            types_section_raw: vec![],
            hostcalls: vec![
                HostCallEntry {
                    name_sid: 1,
                    cap: Cap::Gas,
                    cost: CostClass::GasTick,
                    params: vec![TypeCode::U64],
                    results: vec![TypeCode::B1],
                }
            ],
            funcs: vec![
                FuncEntry {
                    name_sid: 0,
                    is_export: true,
                    params: vec![TypeCode::U32, TypeCode::U32],
                    results: vec![TypeCode::U64],
                    blocks: vec![
                        BlockEntry {
                            params: vec![],
                            instrs: vec![
                                // const_u64 0 (pretend immediate is elsewhere; instr head doesn't encode const payloads yet)
                                InstrHead { opcode: Opcode::ConstU64, out_types: vec![TypeCode::U64], operands: vec![], imm: InstrImm::None },
                            ],
                            term: Terminator::Ret { values: vec![pack_ref(RefKind::SsaValue, 0)] },
                        }
                    ],
                }
            ],
            debug_section_raw: vec![],
        }
    }

    #[test]
    fn roundtrip_module_container() {
        let m = minimal_module();
        let b = m.to_bytes();
        let m2 = ModuleBin::from_bytes(&b).unwrap();
        assert_eq!(m2.strings[0], "main");
        assert_eq!(m2.funcs.len(), 1);
    }

    #[test]
    fn minimal_validation_ok() {
        let m = minimal_module();
        validate_module_minimal(&m).unwrap();
    }
}
```


