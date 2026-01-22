# 🔩 Galaxy Core IR v0.1 — File Formats + Lowering Guide + Gas Cost Table

**Compiler tooling specification (implementation-ready)**
**Project: Galaxy (cGalaxy + Galaxy Core)**
**By Bernard Sibanda — Company: Satoshi — Date: 20 Jan 2026**

## 📚 Table of Contents

1. 🎯 Scope and Design Goals
2. 🧾 Core IR Text Format (GALIR-T)
   2.1 Design rules
   2.2 Module structure
   2.3 Type system encoding
   2.4 Function and block encoding
   2.5 Instruction syntax and rules
   2.6 Validation hooks in text form
3. 📦 Core IR Binary Format (GALIR-B)
   3.1 Container overview
   3.2 Primitive encodings
   3.3 Sections and IDs
   3.4 Tables (types, strings, host calls)
   3.5 Function bodies and instruction streams
   3.6 Deterministic hashing/canonicalization
4. 🧠 Reference Lowering Guide (AST → Core IR)
   4.1 Naming, SSA, and block parameters
   4.2 Expressions
   4.3 Variables (`let`, `mut`, assignment)
   4.4 `if/else`
   4.5 `while` loops (metering + safety)
   4.6 `match` (enums and Result)
   4.7 `Result` construction and propagation (v0.1 style)
   4.8 Bytes literals, slicing, concatenation
   4.9 Calls and host capability enforcement
   4.10 Entrypoint lowering and Result envelope encoding
5. ⛽ Host Cost Table v0.1 (Gas Accounting)
   5.1 Cost model principles
   5.2 Instruction class costs
   5.3 Memory and bytes costs
   5.4 Host call costs (storage/events/crypto/context)
   5.5 Guardrail limits and rejection thresholds
6. 📖 Glossary

# 1. 🎯 Scope and Design Goals

This document extends the previous cGalaxy/Core IR v0.1 specification with three implementation-critical elements:

* **An exact Core IR text format (GALIR-T)** for debugging, golden tests, and human review.
* **An exact Core IR binary format (GALIR-B)** for deterministic, compact packaging, caching, hashing, and distribution.
* **A reference lowering guide** that defines how the compiler must translate language constructs into IR blocks and opcodes.
* **A concrete gas cost table** that is deterministic, tunable by governance, and safe for offline-first environments.

This specification is written so a team can implement:

* a parser/printer for GALIR-T,
* a serializer/deserializer for GALIR-B,
* an AST→IR lowering pass,
* a gas metering injector and validator.

# 2. 🧾 Core IR Text Format (GALIR-T)

## 2.1 Design rules

* GALIR-T is **canonicalizable**: whitespace and comments do not affect semantics.
* Every identifier is explicit; there is no implicit type inference in IR.
* SSA values are named `%0`, `%1`, `%2`, etc. Names must be unique within a function.
* Blocks are numbered `b0`, `b1`, `b2`, etc. Block parameter lists replace phi-nodes.
* All host interactions appear only as `host_call cap::name(...)`.

## 2.2 Module structure

A module is a single text document:

* Header
* Capability imports
* String table (optional but recommended for deterministic IDs)
* Type declarations (for structs/enums lowered layouts)
* Host call table (declarations of signature and cost class)
* Function bodies

### 2.2.1 Example skeleton (GALIR-T)

```
// GALIR-T v0.1
galir.module v0.1
module_name "Payments"

// Capabilities must match the language `import` list.
caps [storage, events, crypto, context, gas, mem]

// Optional canonical string table.
// Each string has a stable integer ID used elsewhere.
strings {
  0: "main"
  1: "storage::get"
  2: "storage::put"
  3: "events::emit"
}

// Host calls are declared once and referenced by id.
// (This makes hashing stable and avoids repeating signatures.)
hostcalls {
  0: storage::get (slice) -> (b1, slice, i32) cost STORAGE_GET
  1: storage::put (slice, slice) -> (i32)     cost STORAGE_PUT
  2: events::emit (slice, slice) -> (i32)     cost EVENT_EMIT
  3: gas::tick (u64) -> (b1)                  cost GAS_TICK
}

func @main (in_ptr:u32, in_len:u32) -> (out_packed:u64) export {
  block b0(in_ptr:u32, in_len:u32):
    ...
}
```

## 2.3 Type system encoding

IR types must be written in a fixed spelling:

* `b1, i32, i64, u32, u64, u256, unit, slice`
* `slice` is always `(ptr:u32,len:u32)` semantically.

If the compiler needs internal layouts for `struct`/`enum`, it must declare them explicitly.

### 2.3.1 Layout declarations

Layouts are compiler metadata that backends can ignore if already lowered, but they must be present if referenced.

```
layout struct S0 fields [u64, slice, b1]
layout enum E0 tag:u32 variants [
  0: ()          // empty
  1: (slice)     // payload
  2: (i32, u64)  // payload
]
```

## 2.4 Function and block encoding

Functions use:

* `func @name (params...) -> (rets...) [export] { blocks... }`

Blocks use:

* `block bN(params...):`
* instructions
* exactly one terminator (`br`, `br_if`, `ret`)

### 2.4.1 SSA assignment form

Each instruction that returns a value assigns to a `%id`:

```
%3 = add_chk i64 %1 %2 -> (b1,i64)
```

If an instruction returns multiple values, the result is destructured:

```
(%ok, %sum) = add_chk i64 %a %b
```

## 2.5 Instruction syntax and rules

### 2.5.1 General instruction pattern

```
%out = OPCODE [type] operands...
```

### 2.5.2 Canonical instruction set (text spellings)

Use these spellings exactly (v0.1):

**Constants**

* `%v = const_i32 123`
* `%v = const_u64 1000`
* `%v = const_bool true`
* `%s = const_bytes "0xAABBCC"`  // quoted for stable parsing

**Arithmetic checked**

* `(%ok,%v) = add_chk i32 %a %b`
* `(%ok,%v) = div_chk i64 %a %b`

**Arithmetic wrapping**

* `%v = add_wrap u64 %a %b`

**Comparison**

* `%b = lt i32 %a %b`

**Boolean**

* `%b = and %x %y`

**Slice/Memory**

* `%n = slice_len %s`
* `%p = slice_ptr %s`
* `%s = slice_make %ptr %len`
* `%b = slice_eq %a %b`
* `(%ok,%out) = slice_slice %s %start %len`
* `%x = mem_load_u8 %ptr`
* `mem_store_u8 %ptr %val`
* `mem_copy %dst %src %len`
* `%ptr = alloc %len`
* `(%ok,%out) = concat %a %b`

**Control**

* `br b2(%x,%y)`
* `br_if %cond b1(%x) b2(%y)`
* `ret %value`

**Host + gas**

* `(%ok,%val,%err) = host_call #0 (%key)`
* `%ok = gas_tick %cost`
  (In v0.1, `gas_tick` is treated as a special form that must map to `gas::tick` hostcall; it exists as an IR opcode so validators can enforce loop metering without needing to interpret host tables.)

## 2.6 Validation hooks in text form

GALIR-T supports optional validator directives that do not change semantics but allow testing:

* `@assert_no_recursion`
* `@assert_loop_metered`
* `@assert_caps [storage, crypto]`

Example:

```
func @main (...) -> (...) export @assert_loop_metered {
  ...
}
```

These are ignored by runtime, used only by compiler tests and CI.

# 3. 📦 Core IR Binary Format (GALIR-B)

## 3.1 Container overview

GALIR-B is a deterministic binary container with:

* fixed magic + version,
* section-based encoding,
* LEB128 integer encoding,
* stable indexing into tables (strings, types, hostcalls),
* canonical ordering rules.

### 3.1.1 Magic and version

* Magic: ASCII `"GALIR\0"` (6 bytes)
* Version: `u16` major, `u16` minor
  v0.1 = major `0`, minor `1`

### 3.1.2 Endianness

* All fixed-width integers (u16/u32/u64) are **little-endian**.
* Variable-length integers use **unsigned LEB128** (ULEB128).

## 3.2 Primitive encodings

* `byte`: 1 byte
* `u16/u32/u64`: little-endian
* `uleb`: ULEB128
* `sleb`: reserved (not used in v0.1)
* `bool`: 1 byte (`0x00` false, `0x01` true)
* `bytes`: `uleb length` + raw bytes

## 3.3 Sections and IDs

A GALIR-B file is:

* Header
* Section directory (implicit by ordering)
* Sections in canonical order

### 3.3.1 Canonical section order (must be followed)

1. `S_MODULE` (module metadata)
2. `S_CAPS` (capabilities)
3. `S_STRINGS` (string table)
4. `S_TYPES` (type/layout table)
5. `S_HOSTCALLS` (host call table)
6. `S_FUNCS` (function table + bodies)
7. `S_DEBUG` (optional; may be stripped)

### 3.3.2 Section encoding

Each section:

* `u8 section_id`
* `uleb section_length`
* `section_payload...`

## 3.4 Tables (types, strings, host calls)

### 3.4.1 `S_STRINGS`

* `uleb count`
* repeated:

  * `uleb string_len`
  * `bytes string_utf8`

Strings are indexed by order (0..count-1). The compiler must keep this order stable.

### 3.4.2 `S_CAPS`

* `uleb cap_count`
* each cap is a single byte enum:

  * `0=storage,1=events,2=crypto,3=context,4=gas,5=mem`

### 3.4.3 `S_TYPES`

Defines additional layouts beyond primitives.

* `uleb type_count`
* each type entry:

  * `u8 kind`

    * `0=layout_struct`
    * `1=layout_enum`
  * `uleb name_string_id` (optional but recommended; can be 0xFFFFFFFF sentinel for none)
  * payload:

    * struct:

      * `uleb field_count`
      * `field_count` × `u8 typecode`
    * enum:

      * `uleb variant_count`
      * for each variant:

        * `uleb field_count`
        * `field_count` × `u8 typecode`

**Typecodes (u8)**

* `0=b1,1=i32,2=i64,3=u32,4=u64,5=u256,6=unit,7=slice,8=layout_ref`
  If `layout_ref`, the next item is `uleb layout_index`.

### 3.4.4 `S_HOSTCALLS`

* `uleb hostcall_count`
* for each hostcall:

  * `uleb cap_enum` (same mapping as caps)
  * `uleb name_string_id` (e.g., “storage::get”)
  * `uleb param_count` then param typecodes
  * `uleb result_count` then result typecodes
  * `u8 cost_class` (see Section 5; stable enum)

Host calls are indexed by order and referenced by `host_call` instructions.

## 3.5 Function bodies and instruction streams

### 3.5.1 `S_FUNCS` layout

* `uleb func_count`
* func headers:

  * `uleb name_string_id`
  * `u8 flags` (bit0=export)
  * `uleb param_count` + param typecodes
  * `uleb result_count` + result typecodes
  * `uleb block_count`
  * `uleb entry_block_index`
  * `uleb body_size_bytes`
  * `body_bytes...`

### 3.5.2 Block encoding in function body

A function body encodes blocks sequentially (0..block_count-1):

* `uleb block_param_count` + types
* `uleb instruction_count`
* instruction stream
* `u8 terminator_kind` + terminator payload

### 3.5.3 Instruction encoding

Each instruction:

* `u8 opcode`
* `uleb out_count` + out typecodes (out_count may be 0)
* `uleb operand_count` + operand references
* opcode-specific immediates

**Operand references**

* `uleb ref_kind_and_index` packed:

  * low 2 bits: kind

    * `0=ssa_value`
    * `1=block_param`
    * `2=immediate_small` (0..63)
    * `3=reserved`
  * remaining bits: index
    This keeps the format compact and deterministic.

### 3.5.4 Terminators

* `BR`: target block index + arg list refs
* `BR_IF`: condition ref + true block + args + false block + args
* `RET`: value refs

## 3.6 Deterministic hashing/canonicalization

* For signing and caching, the canonical digest is computed over the **GALIR-B bytes excluding `S_DEBUG`**.
* `S_DEBUG` is permitted to contain source maps, symbol names, and comments but must not affect execution.

# 4. 🧠 Reference Lowering Guide (AST → Core IR)

## 4.1 Naming, SSA, and block parameters

The compiler must translate source-level mutable variables into SSA using block parameters.

**Rule:** If a variable is assigned in different control-flow paths and later read, its “current value” must be passed as a block argument.

This makes the IR backend-friendly and avoids phi-instruction complexity.

## 4.2 Expressions

### 4.2.1 Literal integers

`123` becomes `const_i32` or `const_i64` depending on declared type.

### 4.2.2 Binary arithmetic

For `I32/I64/U32/U64`, the default lowering uses checked ops:

Source:

```cg
let x: I64 = a + b;
```

IR:

```
(%ok,%sum) = add_chk i64 %a %b
br_if %ok b_ok(%sum) b_err()
```

The error path must return `Err(ERR_OVERFLOW)` unless the expression is explicitly marked as wrapping (v0.1 does not expose wrapping in the language surface; it exists only for later expansion).

## 4.3 Variables (`let`, `mut`, assignment)

### 4.3.1 `let` lowering

* `let x: T = expr;` assigns SSA value `%xN`.
* Later reads refer to the latest SSA value.

### 4.3.2 `let mut` and `x = expr`

Mutable variables are a source-level convenience only. Lowering rule:

* each assignment creates a new SSA value,
* the new value is threaded through subsequent blocks as an argument.

**Example conceptually:**

```
block b0():
  %x0 = const_i32 0
  br b1(%x0)

block b1(%x1:i32):
  ...
  %x2 = const_i32 5
  br b2(%x2)

block b2(%x3:i32):
  ...
```

## 4.4 `if/else`

Source:

```cg
if cond { s1 } else { s2 }
```

Lowering pattern:

1. Evaluate `cond` to `%c`.
2. `br_if %c b_then(...) b_else(...)`
3. Both branches end by branching to a join block `b_join(...)` with the updated variable values.

This is where block parameters implement phi behavior.

## 4.5 `while` loops (metering + safety)

Source:

```cg
while cond { body }
```

Lowering must produce:

* a loop header block,
* a condition block (or reuse header),
* a body block,
* a back-edge that includes `gas_tick`.

**Mandatory rule:** Every back-edge path must execute `gas_tick(loop_cost)`.

Pattern (high-level):

* `b_entry(vars...) -> br b_loop(vars...)`
* `b_loop(vars...) -> gas_tick; if cond then br b_body(vars...) else br b_exit(vars...)`
* `b_body(vars...) -> ... -> br b_loop(vars_updated...)`
* `b_exit(vars...) -> continue`

If `gas_tick` returns false, the compiler must branch to a block that returns `Err(ERR_GAS_EXHAUSTED)` immediately.

## 4.6 `match` (enums and Result)

### 4.6.1 Result match

Source:

```cg
match r {
  Ok(v) => { ... }
  Err(e) => { ... }
}
```

Lowering:

* `get_tag` or direct boolean for ok flag
* `br_if` to two blocks
* pass payload values as block args

## 4.7 `Result` construction and propagation (v0.1 style)

v0.1 does not include `?`. Therefore propagation is explicit:

Source:

```cg
match storage.get(k) {
  Ok(v) => { ... }
  Err(e) => { return err(e); }
}
```

Lowering:

* call host `storage::get` returning `(ok,val,err)`
* branch on `%ok`
* in error block: return encoded Err envelope (Section 4.10)

## 4.8 Bytes literals, slicing, concatenation

### 4.8.1 Bytes literal

`0xAABB` becomes `const_bytes`, which the backend lowers into:

* embedded data segment in WASM
* memory initialization in other targets

### 4.8.2 Slicing

`bytes.slice(b,start,len)` lowers to `slice_slice` (checked) which returns `(ok, slice)`. On `ok=false`, return `Err(ERR_OOB)`.

### 4.8.3 Concatenation

`bytes.concat(a,b)` lowers to `concat` (checked), which allocates new memory and copies. If allocation fails or gas insufficient, return `Err(ERR_GAS_EXHAUSTED)` or `Err(ERR_INTERNAL)` depending on the failure source. In v0.1, allocation failure maps to `ERR_INTERNAL` unless governance later defines a dedicated code.

## 4.9 Calls and host capability enforcement

When lowering a call like `storage.get(k)`:

* the module must have imported `storage`.
* the IR must reference the declared hostcall index for `storage::get`.
* the IR validator must reject undeclared host call usage.

This rule ensures “capability by import” is enforced at compile time and remains visible for audits.

## 4.10 Entrypoint lowering and Result envelope encoding

Entrypoint `main(in_ptr,in_len)` returns packed `(out_ptr,out_len)` in a single `u64`.

Steps required:

1. Create `slice` from input: `slice_make(in_ptr,in_len)`.
2. Decode/interpret input bytes if needed (templates/SDK handle this; v0.1 does not standardize a schema).
3. Produce either:

   * Ok payload: allocate `1 + payload_len`, write tag `0x00`, copy payload
   * Err payload: allocate `1 + 4`, write tag `0x01`, write error code as little-endian u32
4. Pack the output pointer and length into `u64` in the ABI-defined format.

# 5. ⛽ Host Cost Table v0.1 (Gas Accounting)

## 5.1 Cost model principles

* Gas costs are deterministic and must not depend on local wall-clock or machine speed.
* Costs reflect:

  * CPU-like work (instruction count),
  * memory work (bytes copied),
  * state work (storage read/write sizes),
  * cryptographic work (hashing/signature verification).
* The schedule below is **the initial proposed baseline**, and Galaxy governance can tune it, but the compiler and runtime must treat it as a strict table for the chosen chain configuration.

## 5.2 Instruction class costs (IR-level)

These are charged by the runtime or by injected `gas_tick` where appropriate.

| Instruction class       | Cost (gas) | Notes                                |
| ----------------------- | ---------: | ------------------------------------ |
| const / local ops       |          1 | constants, moves, basic SSA plumbing |
| integer add/sub/compare |          1 | includes eq/lt/lte/gt/gte            |
| mul                     |          3 | higher than add/sub                  |
| div/mod (checked)       |          5 | includes div-by-zero check           |
| boolean ops             |          1 | and/or/not                           |
| br / br_if              |          1 | control transfer overhead            |
| ret                     |          1 | function return overhead             |

## 5.3 Memory and bytes costs

| Operation             |                                Cost (gas) | Notes                                         |
| --------------------- | ----------------------------------------: | --------------------------------------------- |
| alloc(len)            |                                10 + len/8 | linear cost; integer division rounds up       |
| mem_load_u8           |                                         1 |                                               |
| mem_store_u8          |                                         1 |                                               |
| mem_copy(len)         |                                 2 + len/8 | rounds up; includes bounds checks if inserted |
| slice_slice (checked) |                                         2 | plus cost of computing indices                |
| bytes.eq(len)         |                                2 + len/16 | compare cost proportional to length           |
| concat(a,b)           | alloc + mem_copy(a.len) + mem_copy(b.len) | fails if gas insufficient                     |

**Rounding rule:** `len/k` means `(len + (k-1)) / k`.

## 5.4 Host call costs (baseline)

These costs are charged **in addition to** any memory costs the contract performs for copying/encoding.

### 5.4.1 Context (cheap)

| Host call        | Cost (gas) |
| ---------------- | ---------: |
| ctx_sender       |         50 |
| ctx_block_height |         20 |
| ctx_chain_id     |         20 |
| ctx_tx_hash      |         50 |

### 5.4.2 Events

| Host call                       |                     Cost (gas) | Notes                  |
| ------------------------------- | -----------------------------: | ---------------------- |
| emit_event(topic_len, data_len) | 200 + topic_len/8 + data_len/8 | log writes are metered |

### 5.4.3 Storage (state access)

Storage costs scale with payload size, because offline-first networks must protect limited cellular and disk resources.

| Host call                     |                   Cost (gas) | Notes                                     |
| ----------------------------- | ---------------------------: | ----------------------------------------- |
| storage_has(key_len)          |              300 + key_len/8 |                                           |
| storage_get(key_len, val_len) |  600 + key_len/8 + val_len/8 | val_len is charged based on returned size |
| storage_put(key_len, val_len) | 1200 + key_len/8 + val_len/4 | writes are more expensive                 |
| storage_del(key_len)          |              800 + key_len/8 |                                           |

**Important:** For `storage_get`, the runtime charges based on actual returned value size, not a guessed size.

### 5.4.4 Crypto

| Host call               |       Cost (gas) | Notes                               |
| ----------------------- | ---------------: | ----------------------------------- |
| sha256(len)             |      800 + len/8 |                                     |
| blake2b256(len)         |      700 + len/8 |                                     |
| verify_ed25519(msg_len) | 5000 + msg_len/8 | signature verification is expensive |

## 5.5 Guardrail limits and rejection thresholds

These are compile-time and runtime limits for safety.

**Compile-time rejection limits (v0.1 defaults):**

* Maximum function count per module: **256**
* Maximum block count per function: **4096**
* Maximum instruction count per function: **200,000**
* Maximum static bytes literal total: **1 MB**
* Maximum nesting depth (`if/match`): **256**
* Recursion: **not allowed**

**Runtime limits (chain-configurable):**

* Max memory pages for contract: e.g., **16 pages** (1 MB) to start
* Max output bytes from `main`: e.g., **64 KB**
* Max event bytes per call: e.g., **32 KB**
* Max storage value size: e.g., **256 KB** (policy; chain-specific)

These defaults protect offline-first cellular deployments from denial-of-service.

# 6. 📖 Glossary

* **GALIR-T:** Galaxy IR Text format used for debugging and tests.
* **GALIR-B:** Galaxy IR Binary format used for signing, hashing, and execution packaging.
* **Cost class:** A stable enum assigned to host calls so costs can be applied deterministically.
* **Canonical digest:** Hash computed over GALIR-B excluding debug sections for stable verification.
