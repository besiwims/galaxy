# 🧱 cGalaxy v0.1 Language Spec + 🔩 Galaxy Core IR v0.1 Spec

**Compiler-ready technical specifications**
**Project: Galaxy (cGalaxy + Galaxy Core)**
**By Bernard Sibanda — Company: Satoshi — Date: 20 Jan 2026**

## 📚 Table of Contents

1. 🎯 Scope and Design Goals
2. 🧭 Determinism and Safety Rules
3. 🧠 Execution Model and ABI
4. 📝 cGalaxy v0.1 Language Specification
   4.1 Lexical Rules
   4.2 Module Structure
   4.3 Types
   4.4 Expressions and Statements
   4.5 Control Flow
   4.6 Error Handling (`Result`)
   4.7 Storage/Event/Crypto Interfaces (Language Level)
   4.8 Restrictions (v0.1)
   4.9 Standard Library (v0.1 Minimal)
5. 🔩 Galaxy Core IR v0.1 Specification
   5.1 IR Overview and Invariants
   5.2 IR Types
   5.3 IR Values and SSA Rules
   5.4 IR Instructions (Opcodes)
   5.5 Control Flow and Blocks
   5.6 Memory Model and Byte Slices
   5.7 Effects and Host Calls
   5.8 Validation Rules (Must Pass to Compile)
6. 🌐 Host Function Definitions (Galaxy ABI v0.1)
   6.1 Common Host Imports
   6.2 Storage
   6.3 Events
   6.4 Crypto
   6.5 Context
   6.6 Gas / Metering
   6.7 Memory Allocation
7. 🧪 Deterministic Encoding Rules
8. 🔁 Mapping Notes to Backends (WASM / EVM / Plutus/UPLC)
9. 📖 Glossary

# 1. 🎯 Scope and Design Goals

1.1 **Primary Goal**
Define a minimal, Go-simple contract language (**cGalaxy v0.1**) and a portable compiler intermediate representation (**Galaxy Core IR v0.1**) that can generate contracts for:

* **Galaxy WASM runtime** (first backend; canonical)
* **EVM** (via IR→Yul pipeline later)
* **Plutus/UPLC** (via IR→UPLC pipeline later)

1.2 **v0.1 Philosophy**

* Keep the surface language **small and predictable**.
* Make determinism **enforced by design**, not “best effort.”
* Keep the IR **stable** and **portable** across many blockchains.
* Treat all chain/environment functionality as **explicit host capabilities** (storage, emit, crypto, context, gas).

# 2. 🧭 Determinism and Safety Rules

2.1 **Determinism Rules (Hard Requirements)**

* No floating-point types or operations in v0.1.
* No access to time, randomness, filesystem, network, threads.
* Iteration order is always defined (no hash-map iteration in v0.1).
* All serialization is canonical and deterministic (defined in Section 7).
* All host functions must be deterministic for the same inputs and same chain state.

2.2 **Safety Rules**

* No exceptions. No implicit panics. All fallible operations must return `Result`.
* Division/modulo by zero must return `Err` (not trap).
* Integer overflow behavior is defined: **checked by default** in language mode; IR exposes both checked and wrapping ops (compiler chooses based on policy).
* Recursion is disallowed in v0.1 (simplifies metering and backend portability).

2.3 **Metering Rules**

* Every contract call is metered with a `gas` budget.
* Loops must be metered per-iteration (IR has explicit `gas_tick` hook).
* Host calls charge gas deterministically.

# 3. 🧠 Execution Model and ABI

3.1 **Contract Unit**
A contract is a single compiled module with:

* one exported entrypoint `main`
* optional internal helper functions

3.2 **Entrypoint Signature (Logical)**

* `main(input: Bytes) -> Result<Bytes, ErrorCode>`

3.3 **Entrypoint ABI (WASM Canonical)**
Because WASM cannot return dynamic-sized Bytes directly, v0.1 uses a standard “slice” ABI.

**WASM export:**

* `main(in_ptr: i32, in_len: i32) -> i64`

**Return value encoding (`i64`):**

* low 32 bits: `out_ptr` (unsigned)
* high 32 bits: `out_len` (unsigned)

So: `ret = (u64(out_len) << 32) | u64(out_ptr)`

3.4 **Result Encoding**
The function returns `Bytes` which itself encodes either:

* `Ok(Bytes)` or
* `Err(ErrorCode)`

v0.1 standard:

* output `Bytes` begins with a 1-byte tag:

  * `0x00` = Ok
  * `0x01` = Err
* Ok payload: `0x00 || <bytes...>`
* Err payload: `0x01 || <u32 error_code LE>`

This ensures **every backend can represent Result** even if it lacks complex types.

3.5 **Memory Ownership**

* Contract memory is linear WASM memory.
* Host may allocate output buffers or contract may allocate using `env.alloc`.
* All byte slices returned from host are allocated in contract memory via host allocator functions to keep a single ownership model.

# 4. 📝 cGalaxy v0.1 Language Specification

## 4.1 Lexical Rules

4.1.1 **Character Set**
UTF-8 source allowed, but identifiers are restricted to ASCII letters, digits, and `_` in v0.1.

4.1.2 **Tokens**

* Identifiers: `[A-Za-z_][A-Za-z0-9_]*`
* Integers: decimal only in v0.1 (`0` or `[1-9][0-9]*`)
* Hex bytes literal: `0x` followed by even number of hex chars (for `Bytes` literals)
* Comments:

  * line: `// ...`
  * block: `/* ... */` (nesting not allowed in v0.1)

4.1.3 **Keywords**
`module, import, fn, let, mut, if, else, while, for, return, struct, enum, match, true, false, type`

(Notes: `for` is reserved but optional; `match` is allowed but minimal in v0.1.)

## 4.2 Module Structure

A file is a module.

### 4.2.1 High-level layout

1. `module` declaration (required)
2. `import` declarations (optional)
3. type declarations (`struct`, `enum`, `type`)
4. function declarations (`fn`)
5. exactly one `fn main(...)` exported entrypoint

### 4.2.2 Imports

Imports are **host capability bindings**, not arbitrary module linking.

Example:

```cg
module Payments

import storage
import crypto
import events
import context
import gas
```

The compiler maps these to specific host functions (Section 6). If a module does not import a capability, it cannot use it.

## 4.3 Types

### 4.3.1 Primitive Types

* `Bool`
* `I32`
* `I64`
* `U32`
* `U64`
* `U256` (defined, but may compile to backend-specific representation)
* `Bytes` (byte slice)
* `Unit` (no value)

### 4.3.2 Composite Types

* `struct` (product type)
* `enum` (tagged union)
* `Result<T, E>` (built-in generic type, *special-cased* in v0.1)

**v0.1 restriction:** only `Result<Bytes, I32>` is guaranteed portable across all backends. Other `Result` instantiations are allowed at language level but may be rejected by backend targets. The canonical external interface uses `Result<Bytes, I32>` via Section 3.4 encoding.

### 4.3.3 Type Aliases

* `type Name = ExistingType;`

### 4.3.4 Bytes Type Semantics

`Bytes` is an immutable slice:

* represented as `(ptr: U32, len: U32)` at runtime
* slicing produces another `Bytes` view
* concatenation allocates a new buffer

## 4.4 Expressions and Statements

### 4.4.1 Expressions (v0.1)

* literals: `true`, `false`, integers, bytes literals `0x...`
* variable references
* unary: `!`, `-`
* binary: `+ - * / %`, comparisons, boolean ops `&& ||`
* function calls
* field access: `x.field`
* indexing bytes: `b[i]` returns `U8` (note: `U8` exists logically but compiles as `U32` in v0.1 with range enforced; see 4.9)

### 4.4.2 Statements

* variable declaration: `let name: Type = expr;`
* mutable declaration: `let mut name: Type = expr;`
* assignment: `name = expr;`
* if: `if cond { ... } else { ... }`
* loops: `while cond { ... }`
* return: `return expr;`
* block: `{ ... }`

**v0.1 restriction:** declarations must include explicit types (`let x: I32 = ...;`) to keep compiler simple and error messages clear.

## 4.5 Control Flow

### 4.5.1 `if/else`

Condition must be `Bool`.

### 4.5.2 `while`

* Allowed but must be metered.
* Compiler inserts a `gas.tick(LOOP_TICK_COST)` call each iteration (Section 6.6).
* If gas is exhausted, execution returns `Err(GAS_EXHAUSTED)`.

### 4.5.3 `match` (minimal)

Allowed only for enums and `Result`:

```cg
match r {
  Ok(v) => { ... }
  Err(e) => { ... }
}
```

Compiler lowers to IR `switch` on tag.

## 4.6 Error Handling (`Result`)

### 4.6.1 No exceptions

All fallible operations return `Result`.

### 4.6.2 Built-in helpers

* `ok(bytes: Bytes) -> Result<Bytes, I32>`
* `err(code: I32) -> Result<Bytes, I32>`
* `?` operator is **not** included in v0.1 (keeps grammar simple). Use explicit match.

## 4.7 Storage/Event/Crypto Interfaces (Language Level)

When a capability is imported, the following namespaces become available:

* `storage.get(key: Bytes) -> Result<Bytes, I32>`
* `storage.put(key: Bytes, val: Bytes) -> Result<Unit, I32>`
* `events.emit(topic: Bytes, data: Bytes) -> Result<Unit, I32>`
* `crypto.sha256(data: Bytes) -> Bytes`
* `crypto.blake2b256(data: Bytes) -> Bytes`
* `crypto.verify_ed25519(pub: Bytes, msg: Bytes, sig: Bytes) -> Bool`
* `context.sender() -> Bytes`
* `context.block_height() -> U64`
* `gas.remaining() -> U64`
* `gas.tick(cost: U64) -> Result<Unit, I32>`

These map 1:1 to host functions (Section 6).

## 4.8 Restrictions (v0.1)

* No recursion.
* No generics other than built-in `Result` (special-cased).
* No trait system.
* No exceptions or panic.
* No floating point.
* No dynamic dispatch.
* No user-defined operator overloading.

## 4.9 Standard Library (v0.1 Minimal)

v0.1 provides these intrinsic helpers (compiler-known):

* `bytes.len(b: Bytes) -> U32`
* `bytes.slice(b: Bytes, start: U32, len: U32) -> Result<Bytes, I32>`
* `bytes.concat(a: Bytes, b: Bytes) -> Result<Bytes, I32>`
* `bytes.eq(a: Bytes, b: Bytes) -> Bool`

**Note:** `bytes.concat` may fail if allocation fails or gas is insufficient.

## 4.10 Grammar (EBNF — v0.1)

This grammar is intentionally strict.

```ebnf
Program        = ModuleDecl { ImportDecl } { TypeDecl | FuncDecl } ;

ModuleDecl     = "module" Ident ;

ImportDecl     = "import" Ident ;

TypeDecl       = StructDecl | EnumDecl | AliasDecl ;

AliasDecl      = "type" Ident "=" Type ";" ;

StructDecl     = "struct" Ident "{" { FieldDecl } "}" ;
FieldDecl      = Ident ":" Type ";" ;

EnumDecl       = "enum" Ident "{" { VariantDecl } "}" ;
VariantDecl    = Ident [ "(" TypeList ")" ] ";" ;

TypeList       = Type { "," Type } ;

FuncDecl       = "fn" Ident "(" [ ParamList ] ")" [ "->" Type ] Block ;
ParamList      = Param { "," Param } ;
Param          = Ident ":" Type ;

Type           = PrimType | Ident | ResultType ;
PrimType       = "Bool" | "I32" | "I64" | "U32" | "U64" | "U256" | "Bytes" | "Unit" ;
ResultType     = "Result" "<" Type "," Type ">" ;

Block          = "{" { Stmt } "}" ;

Stmt           = LetStmt | AssignStmt | IfStmt | WhileStmt | ReturnStmt | ExprStmt | Block ;
LetStmt        = "let" [ "mut" ] Ident ":" Type "=" Expr ";" ;
AssignStmt     = Ident "=" Expr ";" ;
IfStmt         = "if" Expr Block [ "else" Block ] ;
WhileStmt      = "while" Expr Block ;
ReturnStmt     = "return" [ Expr ] ";" ;
ExprStmt       = Expr ";" ;

Expr           = OrExpr ;
OrExpr         = AndExpr { "||" AndExpr } ;
AndExpr        = EqExpr { "&&" EqExpr } ;
EqExpr         = RelExpr { ("==" | "!=") RelExpr } ;
RelExpr        = AddExpr { ("<" | "<=" | ">" | ">=") AddExpr } ;
AddExpr        = MulExpr { ("+" | "-") MulExpr } ;
MulExpr        = UnaryExpr { ("*" | "/" | "%") UnaryExpr } ;
UnaryExpr      = [ "!" | "-" ] Primary ;
Primary        = Literal | Ident | Call | Field | "(" Expr ")" | MatchExpr ;

Call           = Ident "(" [ ArgList ] ")" ;
ArgList        = Expr { "," Expr } ;
Field          = Primary "." Ident ;

MatchExpr      = "match" Expr "{" { MatchArm } "}" ;
MatchArm       = Pattern "=>" Block ;
Pattern        = Ident [ "(" [ Ident { "," Ident } ] ")" ] ;

Literal        = BoolLit | IntLit | BytesLit ;
BoolLit        = "true" | "false" ;
IntLit         = "0" | (NonZeroDigit { Digit }) ;
BytesLit       = "0x" { HexPair } ;
HexPair        = HexDigit HexDigit ;

Ident          = Letter { Letter | Digit | "_" } ;
```

# 5. 🔩 Galaxy Core IR v0.1 Specification

## 5.1 IR Overview and Invariants

5.1.1 **Purpose**
Galaxy Core IR is the single portable internal representation used by cGalaxy to target many backends.

5.1.2 **Form**

* SSA-like (values assigned once)
* structured control-flow with basic blocks
* explicit effects through `host_call`
* explicit gas metering via `gas_tick`

5.1.3 **Determinism Invariants**

* No nondeterministic instructions exist in the IR.
* All external interactions must be represented as `host_call` with declared capability and deterministic signature.

## 5.2 IR Types

IR types are minimal and backend-friendly:

**Scalar Types**

* `b1` (boolean)
* `i32`, `i64`
* `u32`, `u64`
* `u256` (lowered by backend)
* `unit`

**Memory/Slice Types**

* `bytes` represented as:

  * `ptr: u32`
  * `len: u32`
* IR uses a named tuple type: `slice` = `(u32 ptr, u32 len)`

**Composite Types**

* `struct` lowered to field-packed memory or tuple values (compiler chooses)
* `enum` lowered to `(tag: u32, payload...)` (payload may be empty)

**Result Type**

* lowered to `(ok: b1, val_or_err: ...)` internally
* then encoded to Bytes at ABI boundary (Section 3.4)

## 5.3 IR Values and SSA Rules

* Every instruction that produces a value assigns a new SSA name: `%v1`, `%v2`, etc.
* Mutable variables from source are lowered into SSA through phi-like block parameters (see 5.5).

## 5.4 IR Instructions (Opcodes)

This is the canonical opcode list. Backends must support the semantics.

### 5.4.1 Constants

* `const_i32 N -> i32`
* `const_i64 N -> i64`
* `const_u32 N -> u32`
* `const_u64 N -> u64`
* `const_bool B -> b1`
* `const_bytes HEX -> slice`
  (compiler emits memory init + returns pointer/len)

### 5.4.2 Arithmetic (Checked and Wrapping)

Checked versions return `(ok: b1, value: T)`:

* `add_chk T a b -> (b1, T)`
* `sub_chk T a b -> (b1, T)`
* `mul_chk T a b -> (b1, T)`
* `div_chk T a b -> (b1, T)` (fails if div-by-zero; may fail for min/-1 on signed)
* `mod_chk T a b -> (b1, T)`

Wrapping versions:

* `add_wrap T a b -> T`
* `sub_wrap T a b -> T`
* `mul_wrap T a b -> T`

### 5.4.3 Comparisons

* `eq T a b -> b1`
* `neq T a b -> b1`
* `lt T a b -> b1`
* `lte T a b -> b1`
* `gt T a b -> b1`
* `gte T a b -> b1`

### 5.4.4 Boolean

* `and a b -> b1`
* `or a b -> b1`
* `not a -> b1`

### 5.4.5 Bytes and Memory

* `slice_len s -> u32`
* `slice_ptr s -> u32`
* `slice_make ptr len -> slice`
* `slice_eq a b -> b1`
* `slice_slice s start len -> (b1 ok, slice out)` (fails if out of bounds)
* `mem_load_u8 ptr -> u32`
* `mem_store_u8 ptr val_u32 -> unit`
* `mem_copy dst_ptr src_ptr len_u32 -> unit`
* `alloc len_u32 -> u32` (calls host allocator or internal allocator)
* `concat a b -> (b1 ok, slice out)` (alloc + copy)

### 5.4.6 Struct/Enum Lowering Helpers

* `make_struct <layout> args... -> value`
* `get_field <layout> value field_index -> T`
* `make_enum tag payload... -> value`
* `get_tag value -> u32`
* `get_payload value -> ...`

(These are compiler-internal; backends may lower them into tuples/memory.)

### 5.4.7 Control Flow

* `br block_id (args...)`
* `br_if cond block_true(args...) block_false(args...)`
* `ret values...`

### 5.4.8 Host Interaction and Gas

* `host_call cap::name (args...) -> (results...)`
* `gas_tick cost_u64 -> (b1 ok)`
  If `ok=false`, compiler must lower to return `Err(GAS_EXHAUSTED)` at the nearest safe boundary.

## 5.5 Control Flow and Blocks

IR is function-based.

### 5.5.1 Function form

A function is:

* signature (params, returns)
* set of blocks
* each block has parameters (phi replacement), instructions, terminator

Example block shape:

* `block B0(%x: i32, %y: i32):`

  * `%c = lt i32 %x %y`
  * `br_if %c B1(%x) B2(%y)`

This makes SSA and backend lowering straightforward.

## 5.6 Memory Model and Byte Slices

* One linear memory in WASM backend.
* `slice` is always `(ptr,len)` where `ptr` is a byte address.
* All pointers are `u32` in v0.1.
* Alignment is 1 for bytes; structured data alignment is compiler-defined but must be consistent within a module.

## 5.7 Effects and Host Calls

Every host call must declare:

* capability group (storage/events/crypto/context/gas/mem)
* deterministic signature
* gas charge rule

Example:

* `host_call storage::get(key_slice) -> (b1 ok, slice val, i32 err)`

## 5.8 Validation Rules (Must Pass to Compile)

A module is rejected if:

1. It uses any capability that was not imported.
2. It contains recursion (direct or indirect).
3. It contains any loop without a `gas_tick` on the back-edge path.
4. It uses unchecked arithmetic unless policy allows it (default: forbid).
5. It returns an ABI output that violates Result encoding rules.
6. It performs out-of-bounds memory operations (must be guarded by checked slicing and bounds checks).
7. It calls any host function not in the allowed host function table for the chosen target (Galaxy/WASM, EVM, Plutus).

# 6. 🌐 Host Function Definitions (Galaxy ABI v0.1)

All host functions are imported under module name: `env`.

## 6.1 Common Host Imports

### 6.1.1 Memory allocation

* `env.alloc(len: i32) -> i32`
  Allocates `len` bytes and returns pointer. Must be deterministic given current memory state.

Optional (future):

* `env.free(ptr: i32, len: i32) -> i32` (v0.1 not required; bump alloc acceptable)

## 6.2 Storage (KV)

Storage is key/value bytes. All keys and values are raw bytes.

* `env.storage_get(key_ptr: i32, key_len: i32) -> i64`
  Returns packed slice `(ptr,len)` of the value, or `(0,0)` if missing.
  Missing vs empty is distinguished via `env.storage_has` or by returning an error code in v0.2; in v0.1 we add:

* `env.storage_has(key_ptr: i32, key_len: i32) -> i32`
  Returns `1` if exists, `0` if not.

* `env.storage_put(key_ptr: i32, key_len: i32, val_ptr: i32, val_len: i32) -> i32`
  Returns `0` on success, nonzero error code on failure.

* `env.storage_del(key_ptr: i32, key_len: i32) -> i32`
  Returns `0` on success.

**Determinism:** storage functions are deterministic given chain state at call time.

## 6.3 Events

* `env.emit_event(topic_ptr: i32, topic_len: i32, data_ptr: i32, data_len: i32) -> i32`
  Returns `0` on success.

Events must not affect consensus state other than being included in receipts/logs.

## 6.4 Crypto

All crypto functions are deterministic pure functions.

* `env.sha256(data_ptr: i32, data_len: i32) -> i64`
  Returns packed slice to 32-byte digest.

* `env.blake2b256(data_ptr: i32, data_len: i32) -> i64`
  Returns packed slice to 32-byte digest.

* `env.verify_ed25519(pub_ptr: i32, pub_len: i32, msg_ptr: i32, msg_len: i32, sig_ptr: i32, sig_len: i32) -> i32`
  Returns `1` if valid, else `0`.

* `env.verify_secp256k1(...) -> i32`
  (reserved; optional per deployment)

## 6.5 Context

Context is deterministic information about the current transaction and environment.

* `env.ctx_sender() -> i64`
  Returns packed slice for sender/address bytes.

* `env.ctx_block_height() -> i64`
  Returns `u64` in `i64` slot (non-negative).

* `env.ctx_chain_id() -> i64`
  Returns bytes identifying chain.

* `env.ctx_tx_hash() -> i64`
  Returns bytes of the transaction hash.

## 6.6 Gas / Metering

* `env.gas_remaining() -> i64`
  Returns remaining gas as `u64` in `i64`.

* `env.gas_tick(cost: i64) -> i32`
  If enough gas: subtract and return `1`. Otherwise return `0`.

Compiler must:

* insert `gas_tick(loop_cost)` in loops
* insert `gas_tick(host_call_cost)` before/after host calls per policy table

## 6.7 Error Codes (v0.1 Standard)

These error codes are reserved:

* `1` = `ERR_GAS_EXHAUSTED`
* `2` = `ERR_DIV_BY_ZERO`
* `3` = `ERR_OVERFLOW`
* `4` = `ERR_OOB` (out of bounds)
* `5` = `ERR_STORAGE`
* `6` = `ERR_INVALID_INPUT`
* `7` = `ERR_UNAUTHORIZED`
* `8` = `ERR_INTERNAL`

Contracts may define custom errors starting from `1000`.

# 7. 🧪 Deterministic Encoding Rules

7.1 **Canonical Integers**

* `I32/I64/U32/U64` are little-endian when serialized into bytes.

7.2 **Canonical Bytes**

* `Bytes` are raw bytes; no implicit encoding.

7.3 **Canonical Struct/Enum (v0.1)**
To remain portable across backends, the canonical external interface uses only the `Bytes` envelope. If structured types are used internally, they must be encoded/decoded using deterministic helper functions provided by templates/SDKs (v0.1 does not standardize a full schema system yet).

7.4 **Result Envelope**
As defined in Section 3.4.

# 8. 🔁 Mapping Notes to Backends (WASM / EVM / Plutus/UPLC)

8.1 **WASM (Galaxy canonical)**

* Core IR lowers almost 1:1 to WASM stack code.
* `host_call` maps to `env.*` imports.
* `slice` maps to `(ptr,len)` pairs; packed `i64` is used only at ABI boundary.

8.2 **EVM**

* Core IR should be lowered to an EVM-friendly IR where:

  * `u256` becomes native word
  * memory operations map to EVM memory
  * host calls map to “precompiles” or ABI calls depending on execution environment
* Practical path: Core IR → Yul (later phase).
* In EVM backend, `Bytes` slicing and concat must be careful about gas and memory costs.

8.3 **Plutus/UPLC**

* UPLC is functional; Core IR can be converted by:

  * lowering SSA blocks into lambda + case expressions
  * converting memory/bytes operations into UPLC-compatible primitives and encodings
* Host calls become Plutus builtins or script-context reads (adapter layer required).
* Because the Plutus execution model differs, not all host calls are universally portable; this is why v0.1 keeps the external interface as `Bytes` and uses a chain adapter policy.

# 9. 📖 Glossary

* **ABI:** The calling convention between host and contract.
* **Capability:** A named permission (storage, events, crypto, context, gas, mem) imported explicitly.
* **Core IR:** The portable intermediate representation used for multi-backend compilation.
* **Determinism:** Same inputs + same state must produce the same outputs everywhere.
* **Gas:** Metering budget that limits computation and resource use.
* **Slice:** Runtime representation of `Bytes` as `(ptr,len)`.
* **SSA:** Static single assignment form; each value assigned once.
* **UPLC:** Untyped Plutus Core, Cardano’s on-chain execution format.
* **Yul:** A low-level IR language often used to generate EVM bytecode.
