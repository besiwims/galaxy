# ✅ Galaxy Compiler Build Pack v0.1

**(1) IR Validator Spec + (2) WASM Backend Mapping Spec + (3) Reference Test Suite Layout**
**Project: Galaxy (cGalaxy + Galaxy Core)**
**By Bernard Sibanda — Company: Satoshi — Date: 20 Jan 2026**

## 📚 Table of Contents

1. 🧪 Core IR Validator Specification (GALIR-VAL v0.1)
   1.1 Goals and trust boundaries
   1.2 Validation phases
   1.3 Error model and codes
   1.4 Canonical checks (module-level)
   1.5 Capability and hostcall checks
   1.6 Function/block/SSA checks
   1.7 Control-flow and terminator checks
   1.8 Gas metering checks
   1.9 Memory safety checks (static)
   1.10 Determinism checks
   1.11 Pseudocode reference implementation
2. 🧩 WASM Backend Mapping Specification (GALIR→WASM v0.1)
   2.1 Goals and supported WASM profile
   2.2 WASM module sections and layout
   2.3 Type mapping
   2.4 Memory model and allocator strategy
   2.5 ABI mapping (`main` signature + packed return)
   2.6 Instruction-by-instruction lowering
   2.7 Control flow lowering (blocks, br, br_if, ret)
   2.8 Hostcall lowering (`env.*`)
   2.9 Gas charging integration
   2.10 Deterministic bytecode rules
3. 🧰 Reference Test Suite Layout (Compiler CI Plan v0.1)
   3.1 Repository structure
   3.2 Golden tests (source→IR→WASM)
   3.3 Negative tests (expected validator failures)
   3.4 Property-based tests
   3.5 Fuzzing plan
   3.6 Differential tests (future EVM/UPLC backends)
   3.7 Test vectors for host functions and ABI envelopes

# 1. 🧪 Core IR Validator Specification (GALIR-VAL v0.1)

## 1.1 Goals and trust boundaries

The IR validator is a mandatory gate that ensures:

* IR is **well-formed** (types, SSA, blocks, terminators).
* IR is **deterministic** (no forbidden operations, only approved host calls).
* IR is **metered** (no unmetered loops, bounded resource behavior).
* IR is **safe enough** to lower to WASM without traps caused by malformed structure.

**Trust boundary:**

* The validator must assume the IR input may be malicious.
* The validator must not rely on compiler correctness.
* The runtime must only execute IR that passes validation.

## 1.2 Validation phases

Validation occurs in fixed phases to produce specific errors:

1. **Header & container phase** (GALIR-B parsing + section order)
2. **Module phase** (caps, hostcalls, strings, layouts)
3. **Function signature phase** (types, export rules, entrypoint rules)
4. **CFG phase** (block structure, terminators, reachability)
5. **SSA phase** (definitions, uses, operand kinds)
6. **Typing phase** (instruction signatures, operand type matching)
7. **Metering phase** (loop back-edge coverage)
8. **Determinism phase** (forbidden ops/calls)
9. **Resource guardrails phase** (limits, sizes)

A module is accepted only if all phases pass.

## 1.3 Error model and codes

The validator returns:

* `OK`
* or `ERR(code, location, message)`

### 1.3.1 Error code enum (v0.1)

* `V0001` Invalid container (magic/version/section order)
* `V0002` Unknown or unsupported opcode
* `V0003` Invalid type code or layout reference
* `V0004` Capability violation (used but not imported)
* `V0005` Hostcall table invalid or mismatch
* `V0006` Function signature invalid (including `main`)
* `V0007` Block/terminator invalid
* `V0008` CFG invalid (unreachable entry, invalid targets)
* `V0009` SSA invalid (use-before-def, bad ref kind)
* `V0010` Type mismatch (operand/result types)
* `V0011` Loop not metered (missing gas_tick)
* `V0012` Determinism violation (forbidden op/call)
* `V0013` Resource limits exceeded
* `V0014` Memory unsafe pattern (static-detectable)
* `V0015` ABI violation (export/packed return mismatch)

### 1.3.2 Location format

* `module`
* `func_index`
* `block_index`
* `instr_index`
  This location is stable for GALIR-B and supports good debugging.

## 1.4 Canonical checks (module-level)

The validator MUST enforce:

1. **Version match:** only `v0.1` accepted.
2. **Section order:** must follow canonical ordering from GALIR-B spec.
3. **String table sanity:** UTF-8 valid, max string length enforced (default 4096).
4. **Caps table sanity:** caps are unique, known enums only.
5. **Type/layout sanity:** no invalid typecodes; layout refs in range.
6. **Hostcall table sanity:** hostcall entries have valid cap enum and signatures.

## 1.5 Capability and hostcall checks

5.1 **Capability gating**

* If a hostcall uses capability `storage`, then `storage` must exist in `caps`.
* If `gas_tick` opcode exists, then `gas` capability must exist and `gas::tick` must be declared as a hostcall.

5.2 **Hostcall signature checks**

* Each hostcall entry includes param/result typecodes.
* Any `host_call #id` instruction must match exactly:

  * operand count == hostcall param count
  * result count == hostcall result count
  * operand types match param types
  * produced types match result types

5.3 **Allowed hostcalls list**
For Galaxy WASM target v0.1, only these hostcalls are permitted (names are string IDs but must match exactly):

* `mem::alloc`
* `storage::has`, `storage::get`, `storage::put`, `storage::del`
* `events::emit`
* `crypto::sha256`, `crypto::blake2b256`, `crypto::verify_ed25519`
* `context::sender`, `context::block_height`, `context::chain_id`, `context::tx_hash`
* `gas::remaining`, `gas::tick`

Any other hostcall name is `V0012`.

## 1.6 Function/block/SSA checks

6.1 **Function signature**

* `export` functions must be in function table with export flag.
* Exactly one export named `"main"` is required for a deployable contract.
* `main` must have signature: `(u32,u32)->(u64)` in Core IR for WASM backend.

6.2 **Block structure**

* Block indices 0..N-1, each with:

  * param list types
  * instruction list
  * exactly one terminator
* Terminator must be last.

6.3 **SSA rules**

* Every SSA value ID referenced must be defined earlier in the same function.
* Block parameters are valid within their block.
* Operand references must decode to a valid kind:

  * `ssa_value` id in range
  * `block_param` id in range for that block
  * `immediate_small` allowed only where opcode permits it (validator checks)

Violation is `V0009`.

## 1.7 Control-flow and terminator checks

7.1 **Targets**

* `br` and `br_if` targets must be valid block indices.
* The number of arguments passed to a target must equal the target block parameter count.
* Each argument type must match the parameter type.

7.2 **Reachability**

* Entry block must be reachable.
* Optional strict mode: reject any unreachable blocks (recommended for v0.1 contracts to avoid hidden logic).

7.3 **Return**

* `ret` must supply exactly the function’s declared return count and types.

## 1.8 Gas metering checks

The validator must prove that all loops are metered.

8.1 **Loop definition**
A loop exists if the CFG contains a back-edge (edge from block A to block B where B dominates A, or more simply: any edge that participates in a cycle).

8.2 **Metering requirement**
For every cycle, the validator must ensure:

* every back-edge path includes a `gas_tick` executed at least once per iteration.

**Practical rule v0.1 (simpler and strict):**

* Identify all edges `A -> B` where `B` is in the current DFS stack (cycle edge).
* Require that block `A` contains a `gas_tick` instruction **before the terminator** on every path that can reach that edge.

This is conservative but simple and secure.

8.3 **Gas tick semantics requirement**

* `gas_tick` must use a constant or SSA `u64` cost.
* If `gas_tick` result is `false`, control flow must lead to an error return within a bounded number of steps (validator checks the immediate pattern; see below).

8.4 **Mandatory failure pattern**
Immediately after `gas_tick`, the following must occur in the same block:

* `br_if %ok b_continue(...) b_oog(...)`
  and `b_oog` must return ABI `Err(ERR_GAS_EXHAUSTED)`.

If this pattern is absent: `V0011`.

## 1.9 Memory safety checks (static)

The validator cannot prove all memory safety statically because slices and pointers are dynamic. However, it must enforce static rules to prevent obviously unsafe IR.

9.1 **Forbidden raw pointer arithmetic**

* Core IR v0.1 has no general pointer arithmetic opcode; it uses `slice_slice`, `mem_copy`, `mem_load_u8`, `mem_store_u8`.
* The validator permits `mem_*` only if:

  * pointers originate from `alloc`, `const_bytes`, `slice_ptr` of a slice returned by `alloc/const_bytes/host_call` (taint tracking)
  * and are not modified by unknown ops

9.2 **Taint tracking (lightweight)**
Maintain a boolean `ptr_is_valid[%v]` for SSA u32 values. Mark valid when derived from:

* `alloc`
* `const_bytes` pointer output
* `slice_ptr` of a slice already marked valid

If `mem_*` uses a pointer not marked valid: `V0014`.

9.3 **Slice creation rules**
`slice_make(ptr,len)` is only allowed if `ptr` is valid (as above). If not: `V0014`.

This does not guarantee bounds safety, but it blocks “invented pointers.”

## 1.10 Determinism checks

* No forbidden opcodes (floats don’t exist; recursion is already forbidden at language level; IR forbids direct recursion by rejecting call graph cycles if internal calls are introduced later).
* Only allowed hostcalls (Section 1.5.3).
* No `S_DEBUG` content can affect execution (already enforced by canonical digest rules).

## 1.11 Pseudocode reference implementation

Below is a precise, implementable pseudocode outline.

```text
validate_module(bytes):
  m = parse_galir_b(bytes)
  check_header(m) else V0001
  check_sections_order(m) else V0001
  check_limits(m) else V0013

  check_caps(m.caps) else V0004
  check_strings(m.strings) else V0001
  check_types(m.types) else V0003
  check_hostcalls(m.hostcalls, m.caps) else V0005

  main_found = false
  for each func in m.funcs:
    validate_func_signature(func, m) else V0006
    if func.export and func.name == "main":
       main_found = true
       require func.sig == (u32,u32)->(u64) else V0015

    cfg = build_cfg(func)
    validate_blocks_and_terminators(func, cfg) else V0007/V0008

    defmap = init_ssa_def_table()
    validate_ssa_and_types(func, m.hostcalls, defmap) else V0009/V0010
    validate_hostcalls_usage(func, m.hostcalls, m.caps) else V0004/V0012
    validate_ptr_taint_and_memops(func) else V0014

    validate_loop_metering(func, cfg) else V0011

  require main_found else V0015
  return OK
```

# 2. 🧩 WASM Backend Mapping Specification (GALIR→WASM v0.1)

## 2.1 Goals and supported WASM profile

2.1 **Target WASM profile**

* WASM MVP (no threads, no GC)
* Single linear memory
* Deterministic imports only (`env.*`)
* No floating-point instructions emitted

2.2 **Backend output**

* `.wasm` module
* optional `.wat` for debugging
* optional source maps (in custom section; must not affect execution)

## 2.2 WASM module sections and layout

The backend emits these standard sections in canonical order:

1. Type section
2. Import section
3. Function section
4. Memory section
5. Export section
6. Code section
7. Data section (for `const_bytes`)

Custom debug sections may follow.

## 2.3 Type mapping

### 2.3.1 Scalar mapping

* `b1` → `i32` (0 or 1)
* `i32/u32` → `i32`
* `i64/u64` → `i64`
* `u64` computations are done in `i64` but treated as unsigned where needed
* `u256` is **not supported in WASM v0.1 backend** unless lowered before backend.
  If `u256` survives to WASM backend: reject at compile stage (tooling error) because it needs a bigint strategy.

### 2.3.2 Slice mapping

* `slice` → two locals/stack values: `(i32 ptr, i32 len)`
* When the IR instruction returns a slice, the backend maintains this pair ordering consistently:

  * top of stack: `len`, under it: `ptr` (or vice versa), but it must be consistent everywhere.
    **Rule v0.1:** adopt `(ptr, len)` ordering at every boundary for clarity.

## 2.4 Memory model and allocator strategy

### 2.4.1 Memory

* Define one linear memory with initial pages (chain configurable, e.g., 1 page).
* The backend inserts a global mutable `heap_ptr: i32`.

### 2.4.2 Bump allocator (default)

If `env.alloc` is provided by host, the backend may import it. If not, the backend emits an internal function `$alloc`:

**Internal alloc behavior**

* returns current heap_ptr
* increments heap_ptr by requested len
* aligns heap_ptr to 8 bytes (optional but recommended)
* does not free (bump allocator)

**Determinism**

* Deterministic because it depends only on prior allocations in this execution.

**Failure**

* If memory exceeds max pages, allocator returns 0 and compiler must treat as error (v0.1 maps to `ERR_INTERNAL`).

## 2.5 ABI mapping (`main` signature + packed return)

### 2.5.1 Export

Export `main` with signature:

* `(param i32, param i32) (result i64)`

### 2.5.2 Packed return encoding

To pack `(out_ptr,out_len)` into `i64`:

* `(i64.extend_u/i32 out_len) << 32 | (i64.extend_u/i32 out_ptr)`

In WASM:

* `i64.extend_i32_u`
* `i64.shl`
* `i64.or`

## 2.6 Instruction-by-instruction lowering

This section defines exact lowering patterns. “Stack result” means the WASM value(s) left on the stack.

### 2.6.1 const_*

* `const_i32 N` → `i32.const N`
* `const_i64 N` → `i64.const N`
* `const_bool` → `i32.const 0/1`
* `const_bytes "0x.."` → data segment + produce `(ptr,len)`

  * backend places bytes into data section at constant offset
  * emits `i32.const <offset>` and `i32.const <len>`

### 2.6.2 add/sub/mul

For wrapping ops:

* `add_wrap i32 a b` → `local.get a; local.get b; i32.add`

For checked ops (`add_chk`, etc.), v0.1 uses a standard lowering:

* compute in wider type when possible
* check overflow using comparisons
* return `(ok,value)` as `(i32 ok, T value)`

**Important simplification:**
For v0.1, implement checked arithmetic using compiler-emitted helper functions in WASM:

* `$i32_add_chk(a:i32,b:i32)->(i32 ok,i32 v)`
* `$i64_add_chk(a:i64,b:i64)->(i32 ok,i64 v)`
  Same for sub/mul/div/mod.

This reduces backend complexity and keeps correctness centralized.

### 2.6.3 div_chk / mod_chk

* If divisor is zero → ok=0
* Else ok=1 and compute result
* For signed min/-1 overflow on division, ok=0 (for i32/i64)

### 2.6.4 comparisons

* `lt i32 a b` → `i32.lt_s` for signed, `i32.lt_u` for unsigned
  Rule: IR op `lt T` implies signedness from T:
* `i32/i64` → signed compare
* `u32/u64` → unsigned compare

### 2.6.5 slice ops

* `slice_make ptr len` → push ptr, push len (no runtime work)

* `slice_len s` → extract len local/stack

* `slice_ptr s` → extract ptr local/stack

* `slice_eq a b` → call helper `$slice_eq(ptrA,lenA,ptrB,lenB)->i32`

* `slice_slice s start len`:

  * compute `start+len`, compare to `s.len`
  * if out of bounds → ok=0, return dummy slice (0,0)
  * else ok=1, return `(s.ptr+start, len)`
  * implement as helper `$slice_slice_chk(...)`

### 2.6.6 mem_load_u8 / mem_store_u8

* `mem_load_u8 ptr` → `local.get ptr; i32.load8_u align=1 offset=0`
* `mem_store_u8 ptr val` → `local.get ptr; local.get val; i32.store8 align=1`

### 2.6.7 mem_copy

Implement as helper `$mem_copy(dst,src,len)->()`

* loop copy with `i32.load8_u` + `i32.store8`
* metering: charge via `gas_tick` inserted by IR before calls to mem_copy if desired, or include internal tick per N bytes (recommended later).
  v0.1 approach: charge gas at IR level based on len, then mem_copy is “free” at wasm level.

### 2.6.8 alloc

* If host alloc is used: import `env.alloc(i32)->i32`
* Else: call `$alloc(i32)->i32`
  Return pointer i32.

### 2.6.9 concat

Lower to:

* compute out_len = a.len + b.len (checked u32)
* alloc(out_len)
* mem_copy(out_ptr, a.ptr, a.len)
* mem_copy(out_ptr+a.len, b.ptr, b.len)
* return ok and out slice

This is easiest as helper `$concat_chk(a_ptr,a_len,b_ptr,b_len)->(i32 ok,i32 out_ptr,i32 out_len)`

## 2.7 Control flow lowering (blocks, br, br_if, ret)

Because Core IR is structured as basic blocks with parameters, the simplest WASM lowering uses:

* one WASM function per IR function
* locals for each SSA value
* structured blocks with `block` / `loop` and explicit `br` indices

**However**, mapping arbitrary CFG into structured WASM blocks can be complex. v0.1 chooses a reliable method:

### 2.7.1 v0.1 strategy: “Block Dispatcher” lowering

Emit a single `loop` with a `block_id` local acting as a program counter:

* Each IR block becomes a `case` in a `br_table` dispatch.
* Block parameters are stored in locals before jumping.

Pattern:

1. initialize `pc = entry_block`
2. `loop $dispatch`
3. `block $cases` with nested blocks matching count
4. `br_table` jumps to case blocks based on `pc`
5. each case block executes and sets `pc` then `br $dispatch`
6. return via `return` when encountering IR `ret`

This produces deterministic WASM and avoids complex structuring logic.

## 2.8 Hostcall lowering (`env.*`)

Each hostcall table entry maps to a WASM import:

* module name: `"env"`
* function name: specific import string
* signature derived from hostcall param/result types mapped to WASM types

### 2.8.1 Return slice from host

Host returns packed `i64` for bytes (ptr,len). Backend unpacks:

* `ptr = (i32)(ret & 0xFFFFFFFF)`
* `len = (i32)(ret >> 32)`

The backend must provide helper functions `$unpack_ptr`, `$unpack_len` or inline ops.

## 2.9 Gas charging integration

v0.1 requires deterministic metering. The backend enforces:

* Every IR `gas_tick cost` becomes a hostcall `env.gas_tick(cost:i64)->i32` and yields `ok` boolean.

If gas_tick fails, control must flow to an error return (enforced by validator).

## 2.10 Deterministic bytecode rules

To ensure all nodes produce identical results:

* No floats emitted.
* No imported nondeterministic functions.
* No reliance on table ordering that differs across compilation.
* Data segments are emitted in a deterministic order:

  * sorted by appearance order of `const_bytes` in IR.

# 3. 🧰 Reference Test Suite Layout (Compiler CI Plan v0.1)

## 3.1 Repository structure

Recommended monorepo layout:

```
galaxy/
  compiler/
    lexer/
    parser/
    ast/
    sema/
    ir/
      galir_text/
      galir_binary/
      validator/
    codegen_wasm/
    tests/
      fixtures/
      golden/
      negative/
      fuzz/
  runtime/
    wasm_host/
    abi/
  sdk/
    typescript/
    go/
    rust/
  docs/
    specs/
```

## 3.2 Golden tests (source→IR→WASM)

Golden tests prove compiler stability and prevent regressions.

Each golden test includes:

* `.cg` source
* expected `.galir` (GALIR-T)
* expected `.wasm` hash (or `.wat` snapshot)
* expected runtime output for a stub host

Example:

```
tests/golden/001_addition/
  contract.cg
  expected.galir
  expected.wat
  run.json
```

### 3.2.1 Recommended golden scenarios

1. arithmetic checked overflow → returns `ERR_OVERFLOW`
2. div by zero → returns `ERR_DIV_BY_ZERO`
3. if/else join with mut variables
4. while loop with gas ticking
5. bytes slicing OOB → `ERR_OOB`
6. concat large bytes → correct output, correct gas cost applied
7. storage_get missing key → deterministic handling
8. verify_ed25519 true/false path
9. events emitted with expected topic/data bytes

## 3.3 Negative tests (expected validator failures)

Negative tests ensure validator blocks unsafe IR.

Structure:

```
tests/negative/V0011_loop_unmetered/
  bad.galir
  expected_error.json
```

Must include cases for:

* V0001 bad container
* V0004 capability missing
* V0009 SSA use-before-def
* V0010 type mismatch
* V0011 unmetered loop
* V0012 forbidden hostcall
* V0015 invalid main signature/export

## 3.4 Property-based tests

Use property tests on:

* GALIR-T parser/printer roundtrip: `parse(print(ir)) == ir`
* GALIR-B serialize/deserialize roundtrip
* Deterministic hash invariant: `hash(GALIR-B without debug)` stable
* Checked arithmetic helpers: match reference big-int behavior (offchain test harness)

## 3.5 Fuzzing plan

* Lexer fuzzing: random bytes → must not crash
* Parser fuzzing: random tokens → must produce error or AST safely
* IR validator fuzzing: random GALIR-B → must not crash, must reject safely
* WASM backend fuzzing: valid IR corpus → must produce valid wasm or fail gracefully

## 3.6 Differential tests (future EVM/UPLC backends)

Once EVM/UPLC emitters exist:

* run the same contract logic across backends using a shared “host simulation”
* assert identical Result envelope bytes for the same inputs
  This is the long-term correctness strategy for “compile everywhere.”

## 3.7 Test vectors for host functions and ABI envelopes

Provide fixed vectors:

* Input payload bytes
* Expected output bytes (Ok/Err envelope)
* Expected gas remaining (optional)
* Expected storage/events side effects (in host stub log)

A host stub should record:

* calls made (name + args)
* returned values
* emitted events
* storage reads/writes

This makes offline debugging straightforward.

## ✅ Immediate next step (build order recommendation)

Build order:

1. Implement **GALIR-T parser/printer** + IR in-memory model.
2. Implement **GALIR-B serializer/deserializer**.
3. Implement **Validator** (must be robust; fuzz early).
4. Implement **AST→IR lowering** for a tiny subset.
5. Implement **WASM backend** using the “PC dispatcher” strategy.
6. Add the **host stub runtime** to run golden tests.


