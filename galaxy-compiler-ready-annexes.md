# Annex A — Galaxy WASM Profile v0.1 (Deterministic WASM32)

## A.1 Purpose and scope

Galaxy MUST define a single executable profile called **Galaxy WASM Profile v0.1**. This profile exists to guarantee that a module produced from **any supported source language** executes deterministically across all Galaxy nodes, including offline nodes, and produces identical results and identical side effects when given the same inputs and the same chain context.

Galaxy WASM Profile v0.1 applies to all **contracts** and **application modules** that execute under Galaxy consensus or affect any state that will be synchronized as a “global unified source of truth.”

## A.2 Target and ABI invariants

A Galaxy WASM module MUST target **wasm32** (32-bit linear memory addressing). A module MUST export:

1. A linear memory export named **`memory`**.
2. A function export named **`main`** with signature:
   **`main(in_ptr: i32, in_len: i32) -> i64`**.

Galaxy interprets the returned `i64` as a packed `(out_ptr:u32, out_len:u32)` as follows:

* `out_ptr = (ret & 0xFFFF_FFFF)`
* `out_len = (ret >> 32)`

The bytes `memory[out_ptr .. out_ptr+out_len)` MUST be a valid **Result envelope**:

* `0x00 || payload_bytes` for success, or
* `0x01 || u32_le(error_code)` for failure.

A module MUST NOT depend on WASI, operating system calls, clocks, randomness, network, filesystem, or host environment details. Galaxy nodes MUST reject modules that import any forbidden functionality.

## A.3 Determinism rules

Galaxy determinism MUST be enforced by both **static validation** and **runtime constraints**.

### A.3.1 Forbidden nondeterminism sources

A module MUST NOT be allowed to:

* observe wall-clock time or timestamps from the host (unless provided as deterministic chain context, see A.6),
* access randomness or entropy,
* perform I/O (network, filesystem, environment variables),
* create threads or use shared memory,
* use floating point instructions in MVP (see A.4),
* trap as a normal control-flow mechanism.

If a module traps anyway, the runtime MUST convert the execution result into a deterministic error envelope with a fixed error code (for example `ERR_TRAP = 0x0000_0001`) and MUST apply a deterministic side-effect policy (see A.7).

### A.3.2 Replay determinism

For the same `(input_bytes, context, initial_storage_snapshot)` the module MUST produce:

* the same returned envelope bytes,
* the same ordered event log,
* the same storage writes and deletes,
* the same gas usage,
  on any node, on any machine.

Galaxy MUST enforce deterministic behavior by restricting WASM features and by specifying hostcall semantics precisely.

## A.4 WASM feature whitelist and blacklist (exact rules)

### A.4.1 Whitelisted core features (allowed)

A module MAY use:

* integer instructions (`i32`, `i64`) and control flow (`block`, `loop`, `if`, `br`, `br_if`, `return`, `call`),
* linear memory load/store instructions for integers, including `load8_u` / `store8`,
* tables and `call_indirect` ONLY under strict rules (A.4.3),
* multiple functions, locals, and immutable or mutable globals,
* `memory.grow` ONLY if the runtime permits it and accounts gas deterministically (recommended: forbid in v0.1 and require allocation via host, see A.6.5).

A module MUST be valid under the **WebAssembly Core Specification MVP (Wasm 1.0)** subset defined by this annex.

### A.4.2 Blacklisted features (forbidden in v0.1)

A module MUST be rejected if it contains any of the following:

* floating point types or operators: any `f32.*` or `f64.*`,
* SIMD: any `v128` or SIMD instructions,
* threads: shared memory, atomic instructions, `memory.atomic.*`,
* exceptions and stack switching: `try`, `catch`, `throw`, `delegate`,
* reference types / GC / externref features beyond MVP requirements,
* WASI imports (any `wasi_*` namespace),
* non-deterministic host imports (anything outside `env.*` in the allowed import list),
* `start` function (module start),
* multiple memories (more than one linear memory),
* importing memory from the host (memory MUST be module-defined and exported as `memory`),
* importing tables from the host (tables MUST be module-defined if used).

### A.4.3 Conditional allowance: tables and indirect calls

Galaxy allows tables and `call_indirect` only because many mainstream compilers emit them for dynamic dispatch. However, tables increase attack surface.

Therefore, if a module uses tables, the module MUST satisfy all of the following:

* it MUST define the table internally and MUST NOT import a table,
* it MUST NOT export the table,
* table element initializers MUST be fully static,
* table maximum size MUST be declared and MUST be ≤ a chain parameter `MAX_TABLE_ELEMS`,
* `call_indirect` targets MUST resolve to functions defined in the module, not host imports.

If these conditions are not met, the validator MUST reject the module.

## A.5 Resource limits (mandatory)

Galaxy MUST define hard limits for:

* maximum linear memory pages (`MAX_MEM_PAGES`),
* maximum input size (`MAX_INPUT_BYTES`),
* maximum output size (`MAX_OUTPUT_BYTES`),
* maximum storage bytes written per call (`MAX_STORAGE_WRITE_BYTES`),
* maximum storage key length and value length,
* maximum event count per call and maximum event size,
* maximum call stack depth and instruction budget (or gas-metering).

These limits MUST be enforced deterministically. When a limit is exceeded, the runtime MUST produce a deterministic error envelope and MUST apply the deterministic side-effect policy.

## A.6 Allowed host imports (exact list and signatures)

A module MUST import only from namespace **`env`**. Any other import namespace MUST be rejected.

All host functions MUST have deterministic behavior. All pointers are `i32` interpreted as `u32` offsets into `memory`. All lengths are `i32` interpreted as `u32`.

### A.6.1 Gas

* `env.gas_tick(cost: i64) -> i32`
  Returns `1` for ok, `0` if out of gas. The runtime MUST stop further execution deterministically if out of gas, returning `ERR_OOG`.

### A.6.2 Memory allocation

* `env.alloc(len: i32) -> i32`
  Returns a pointer to `len` bytes or `0` on failure. Allocation MUST be deterministic given the same execution path and memory state. Allocation MUST charge gas deterministically.

### A.6.3 Storage (key/value)

* `env.storage_has(k_ptr:i32, k_len:i32) -> i32` returns `1` if present else `0`
* `env.storage_get(k_ptr:i32, k_len:i32, out_ptr:i32, out_cap:i32) -> i32`
  Returns number of bytes written, or `-1` if not found, or `-2` if `out_cap` insufficient.
* `env.storage_put(k_ptr:i32, k_len:i32, v_ptr:i32, v_len:i32) -> i32` returns `1` ok else `0`
* `env.storage_del(k_ptr:i32, k_len:i32) -> i32` returns `1` deleted else `0`

Storage MUST be scoped to the current contract instance and the transaction being executed, and writes MUST be buffered until commit (A.7).

### A.6.4 Events

* `env.event_emit(topic_ptr:i32, topic_len:i32, data_ptr:i32, data_len:i32) -> i32` returns `1` ok else `0`

Events MUST be appended in a deterministic order. Events MUST be included in receipts or receipt commitments.

### A.6.5 Crypto

* `env.sha256(in_ptr:i32, in_len:i32, out32_ptr:i32) -> i32`
* `env.blake2b256(in_ptr:i32, in_len:i32, out32_ptr:i32) -> i32`
* `env.ed25519_verify(msg_ptr:i32, msg_len:i32, sig64_ptr:i32, pk32_ptr:i32) -> i32` returns `1` valid else `0`

### A.6.6 Context (deterministic chain context)

* `env.ctx_sender(out32_ptr:i32) -> i32`
* `env.ctx_chain_id(out32_ptr:i32) -> i32`
* `env.ctx_tx_hash(out32_ptr:i32) -> i32`
* `env.ctx_block_height() -> i64`

Context values MUST be deterministic as part of the block being executed.

## A.7 Deterministic side-effect policy

Galaxy MUST buffer all side effects during execution. Galaxy MUST commit side effects only if the returned envelope is `Ok` and gas was not exceeded. If the contract returns `Err` or traps or runs out of gas, Galaxy MUST discard buffered writes and MUST still retain any deterministic receipt metadata required by the chain (such as error code and gas used). Event retention policy MUST be deterministic; the recommended rule is “events are discarded on failure,” unless Galaxy defines “failure events” explicitly.

# Annex B — Official Language Adapters v0.1 (exact APIs per language)

## B.1 One unified contract model across languages

Galaxy MUST provide official adapters so that developers in different languages all follow the same conceptual pattern:

1. The developer implements a handler function that accepts `input_bytes` and returns `Ok(payload_bytes)` or `Err(error_code)`.
2. The adapter marshals pointers and lengths to and from WASM linear memory.
3. The adapter calls the deterministic `env.*` imports and exposes them as idiomatic APIs.
4. The adapter always returns a correct Result envelope and never traps intentionally.

Galaxy MUST ship day-one support for: **Rust, TypeScript, Go, C/C++, Python, C#, Java, PHP, JavaScript.**
“Support from day one” means that every one of these languages MUST have an official toolchain path that produces a valid Galaxy WASM module conforming to Annex A, even if some languages use AI translation into cGalaxy as their initial path.

## B.2 Two supported compilation routes (both official)

Galaxy MUST officially support two routes:

### Route 1: Native WASM compilation (preferred where possible)

This route compiles source language directly to WASM. It MUST be used for:

* Rust, TypeScript, Go, C/C++ at minimum in MVP.

### Route 2: AI translation into cGalaxy (guaranteed for all listed languages)

This route translates the developer’s source code into cGalaxy (or a deterministic Galaxy subset) and then compiles that to WASM. This route MUST exist for:

* Python, C#, Java, PHP, JavaScript, and MAY also exist for the others as an alternate workflow.

The AI translation pipeline MUST be reproducible under Annex C rules, and its output MUST be validated by the same validator and golden test harness used for native compilation.

## B.3 The canonical Galaxy SDK API surface (language-agnostic)

Every adapter MUST expose the following logical modules and functions. The exact signatures differ per language, but the behaviors MUST be identical.

### B.3.1 Result envelope helpers

* `ok(payload: bytes) -> bytes`
* `err(code: u32) -> bytes`

### B.3.2 Storage API

* `storage.has(key: bytes) -> bool`
* `storage.get(key: bytes) -> Option<bytes>`
* `storage.put(key: bytes, value: bytes) -> void` (or Result)
* `storage.del(key: bytes) -> bool`

### B.3.3 Events API

* `events.emit(topic: bytes, data: bytes) -> void` (or Result)

### B.3.4 Crypto API

* `crypto.sha256(data: bytes) -> [u8;32]`
* `crypto.blake2b256(data: bytes) -> [u8;32]`
* `crypto.ed25519_verify(msg: bytes, sig: [u8;64], pk: [u8;32]) -> bool`

### B.3.5 Context API

* `ctx.sender() -> [u8;32]`
* `ctx.chain_id() -> [u8;32]`
* `ctx.tx_hash() -> [u8;32]`
* `ctx.block_height() -> u64`

### B.3.6 Gas API

* `gas.tick(cost: u64) -> void` (throws/returns error if OOG)

## B.4 Exact adapter APIs by language (v0.1)

### B.4.1 Rust adapter (galaxy-sdk-rs)

Rust MUST target `wasm32-unknown-unknown` and MUST not link WASI.

**Developer entrypoint**

```rust
pub fn handle(input: &[u8]) -> core::result::Result<Vec<u8>, u32>;
```

**Adapter export**

```rust
#[no_mangle]
pub extern "C" fn main(in_ptr: u32, in_len: u32) -> u64;
```

**Host imports**

```rust
extern "C" {
  fn alloc(len: u32) -> u32;
  fn gas_tick(cost: u64) -> u32;

  fn storage_has(k_ptr:u32,k_len:u32)->u32;
  fn storage_get(k_ptr:u32,k_len:u32,out_ptr:u32,out_cap:u32)->i32;
  fn storage_put(k_ptr:u32,k_len:u32,v_ptr:u32,v_len:u32)->u32;
  fn storage_del(k_ptr:u32,k_len:u32)->u32;

  fn event_emit(t_ptr:u32,t_len:u32,d_ptr:u32,d_len:u32)->u32;

  fn sha256(p:u32,l:u32,out32:u32)->u32;
  fn blake2b256(p:u32,l:u32,out32:u32)->u32;
  fn ed25519_verify(m:u32,ml:u32,sig64:u32,pk32:u32)->u32;

  fn ctx_sender(out32:u32)->u32;
  fn ctx_chain_id(out32:u32)->u32;
  fn ctx_tx_hash(out32:u32)->u32;
  fn ctx_block_height()->u64;
}
```

The Rust adapter MUST provide safe wrappers and MUST guarantee correct envelope encoding on success or error.

### B.4.2 TypeScript adapter (galaxy-sdk-ts)

TypeScript MUST compile using a deterministic WASM toolchain (initially AssemblyScript or a Galaxy TS→cGalaxy route).

**Developer entrypoint**

```ts
export function handle(input: Uint8Array): Uint8Array | GalaxyError;
export class GalaxyError { code: u32 }
```

**Adapter export**

```ts
// compiled to wasm export
export function main(inPtr: i32, inLen: i32): i64;
```

The adapter MUST expose `storage/events/crypto/ctx/gas` as TS classes with the behaviors defined in B.3.

### B.4.3 Go adapter (galaxy-sdk-go)

Go MUST compile to WASM using a pinned deterministic toolchain (TinyGo recommended for MVP).

**Developer entrypoint**

```go
func Handle(input []byte) ([]byte, uint32 /*errCode*/, bool /*ok*/ )
```

**Adapter export**

```go
//export main
func main(inPtr uint32, inLen uint32) uint64
```

The adapter MUST ensure no garbage-collector nondeterminism leaks into consensus. If GC introduces timing dependence, Galaxy MUST require a restricted profile (bounded allocations, deterministic alloc via host) or route Go through AI→cGalaxy for consensus modules until proven deterministic.

### B.4.4 C/C++ adapter (galaxy-sdk-c)

C/C++ MUST compile via clang to wasm32 without WASI.

**Developer entrypoint**

```c
// returns 1 if ok, 0 if err; out parameters set accordingly
int galaxy_handle(const uint8_t* in, uint32_t in_len,
                  uint8_t** out_ptr, uint32_t* out_len,
                  uint32_t* err_code);
```

**Adapter export**

```c
__attribute__((export_name("main")))
uint64_t main(uint32_t in_ptr, uint32_t in_len);
```

The C adapter MUST provide safe helpers to build envelopes and MUST validate pointer arithmetic defensively to prevent accidental traps.

### B.4.5 Python adapter (galaxy-sdk-py) — day one via AI→cGalaxy

Python cannot be assumed to compile to deterministic WASM safely in MVP. Therefore Galaxy MUST support Python from day one via the **AI translation route** into cGalaxy.

**Developer entrypoint (Python source)**

```python
def handle(input: bytes) -> tuple[bool, bytes | int]:
    ...
```

The build system MUST translate this Python module into cGalaxy deterministically (Annex C), then compile to WASM. The adapter MUST present the same SDK surface in Python source, but the executed artifact is cGalaxy-compiled WASM.

### B.4.6 C# adapter — day one via AI→cGalaxy (or pinned AOT WASM if validated)

C# MUST be supported from day one. If a pinned deterministic AOT WASM pipeline is not proven, C# MUST compile via AI→cGalaxy for MVP.

**Developer entrypoint**

```csharp
public static GalaxyResult Handle(byte[] input);
public struct GalaxyResult { public bool Ok; public byte[] Payload; public uint ErrorCode; }
```

### B.4.7 Java adapter — day one via AI→cGalaxy

Java MUST be supported from day one using AI translation into cGalaxy unless a deterministic Java→WASM pipeline is validated for consensus use.

**Developer entrypoint**

```java
public static GalaxyResult handle(byte[] input);
```

### B.4.8 PHP adapter — day one via AI→cGalaxy

PHP MUST be supported from day one via AI translation into cGalaxy.

**Developer entrypoint**

```php
function handle(string $input): array; // [ok=>bool, payload=>string] or [ok=>false, code=>int]
```

### B.4.9 JavaScript adapter — day one via AI→cGalaxy or TS toolchain

JavaScript MUST be supported from day one. If native JS→WASM is not deterministic, JS MUST be translated to cGalaxy or to the TS subset that compiles deterministically.

**Developer entrypoint**

```js
export function handle(inputBytes) { return { ok:true, payload:... } or { ok:false, code:... }; }
```

## B.5 Adapter conformance tests (mandatory)

Galaxy MUST ship a conformance test suite that every adapter must pass, including:

* envelope encoding roundtrips,
* storage put/get/del determinism,
* event ordering determinism,
* crypto vector determinism,
* out-of-gas determinism,
* trap-to-error determinism.

No adapter may be considered “supported from day one” unless it passes this suite and produces WASM that passes the Galaxy WASM Profile validator.

# Annex C — Playground + Prompt Compilation Workflow (reproducible artifacts)

## C.1 Purpose

Galaxy MUST ship an online playground and a local CLI that implement the same compilation workflow. The workflow MUST be reproducible, meaning that a given input (prompt + source + template + toolchain versions) MUST produce identical build artifacts, identical hashes, and identical test results across machines.

Galaxy requires reproducibility because AI assistance is only trustworthy when it is verifiable, and because enterprises and regulators require build provenance.

## C.2 Canonical project inputs

A Galaxy build MUST be defined by a single canonical object called the **Galaxy Build Intent (GBI)**. The GBI MUST be a canonical JSON object with sorted keys and UTF-8 encoding.

GBI MUST include:

* `language`: one of `rust|ts|go|c|cpp|python|csharp|java|php|js|cgalaxy`
* `source_files`: filename → bytes (or content hash references)
* `prompt`: optional natural language prompt
* `template_id`: optional
* `abi_version`: fixed string, e.g. `galaxy/wasm-abi@0.1`
* `wasm_profile`: fixed string, e.g. `galaxy/wasm32-det@0.1`
* `toolchain_lock`: exact versions and hashes (see C.4)
* `ai_lock`: model id, decoding settings, and policy version if AI is used (see C.5)
* `build_seed`: a fixed u64 used only for non-consensus UI decisions (never in execution)
* `expected_tests`: golden expectations if provided

The playground MUST show the GBI to the user and MUST allow exporting it. The CLI MUST accept it as input. This is how Galaxy ensures that “what you built” can always be rebuilt.

## C.3 Deterministic compilation pipeline (exact stages)

The pipeline MUST be executed in this order:

1. **Normalize inputs**: canonicalize line endings, UTF-8, file ordering, and JSON canonicalization.
2. **(Optional) AI translation**: if `language` is not natively compiled to WASM in the current toolchain lock, the pipeline MUST translate source into **cGalaxy** using the AI process in C.5.
3. **Compile**: compile cGalaxy or native language into WASM conforming to Annex A.
4. **Validate**: run the Galaxy WASM Profile validator. If validation fails, the build MUST fail.
5. **Link ABI shim**: ensure `memory` export exists, ensure `main` export exists, ensure imports match `env.*`.
6. **Run deterministic harness**: run golden tests in a deterministic host harness that records storage/events.
7. **Package**: produce a `.gxc` package that includes the wasm, manifest, and provenance file.
8. **Content-address**: compute and publish hashes for wasm and package.

Each stage MUST produce a stage hash so the pipeline is auditable.

## C.4 Toolchain lock (mandatory and exact)

Galaxy MUST maintain a **Toolchain Lock** document that pins:

* compiler version(s),
* code generator version(s),
* validator version,
* runtime version (Wasmtime version for MVP),
* adapter versions per language,
* standard library version,
* canonical formatter version.

The toolchain lock MUST be included in the build provenance. The playground MUST build inside a sandbox that uses only pinned toolchain versions. The CLI MUST be able to fetch exactly those pinned versions or run them inside a container.

## C.5 AI translation lock (mandatory for AI use)

If AI is used in the pipeline, the AI process MUST be reproducible in the following sense:

* The model identifier MUST be pinned (for example `ai_model_id`).
* Decoding settings MUST be fixed, with temperature set to **0**.
* The prompt template MUST be versioned and hashed.
* The translation MUST output a canonical cGalaxy source form (auto-formatted).
* The pipeline MUST store the AI input (prompt + source + constraints) and output (cGalaxy) in the provenance file.

Galaxy MUST treat AI output as untrusted until it passes:

* compilation,
* validator checks,
* conformance tests,
* and project golden tests.

AI MUST be used to improve developer productivity and safety, but protocol truth MUST come from deterministic validation and execution.

## C.6 The “one-command local run” contract (exact)

Galaxy MUST provide a single command that performs a full local cycle. The canonical CLI behavior MUST be:

* `galaxy run` MUST:

  1. compile the project to WASM under the locked profile,
  2. validate it against Annex A,
  3. run the deterministic harness,
  4. display the returned envelope in decoded form,
  5. display emitted events,
  6. display a storage diff,
  7. report gas used,
  8. exit nonzero on failure.

This command MUST work without internet access except when the user explicitly requests AI translation from a remote model. Galaxy SHOULD support a local AI model option or an offline translation cache so that offline development remains possible.

## C.7 Reproducible artifacts (exact output files)

A successful build MUST produce:

1. `contract.wasm`
2. `manifest.json` (canonical JSON)
3. `provenance.json` containing:

   * GBI
   * toolchain lock versions/hashes
   * AI lock info (if used)
   * stage hashes
4. `contract.gxc` (archive containing the above)

The `.gxc` hash MUST be stable and MUST be computed over the exact byte content of the archive. Archive ordering and timestamps MUST be normalized so builds are identical across machines.

# Annex D — Galaxy Error Codes v0.1 (locked)

## D.1 Purpose

Galaxy MUST use a single global error code namespace so that failures are interpretable across languages, nodes, offline proof bundles, explorers, wallets, and audits. The error codes MUST be stable and MUST NOT change meanings once released. The error codes MUST be encoded in the Result envelope as `0x01 || u32_le(code)`.

## D.2 Encoding rules (normative)

* All error codes are `u32`.
* `0` MUST NOT be used as an error code (reserved).
* Error codes MUST be deterministic and MUST be returned for all failures, including traps and out-of-gas.
* The runtime MUST map internal failures to the specified codes below.

## D.3 Error code ranges (locked)

Galaxy reserves the following ranges:

* `0x0000_0001 .. 0x0000_0FFF` = VM and ABI failures (universal)
* `0x0000_1000 .. 0x0000_1FFF` = Hostcall failures (universal)
* `0x0000_2000 .. 0x0000_2FFF` = Determinism / validation failures
* `0x0000_3000 .. 0x0000_3FFF` = Identity / KYC / policy zone failures
* `0x0000_4000 .. 0x0000_4FFF` = Exchange / market rail failures
* `0x0000_8000 .. 0x0000_FFFF` = Application-reserved (developer space, must not clash with Galaxy core if used carefully)
* `0x8000_0000 .. 0xFFFF_FFFF` = Vendor/institution/reserved extensions (discouraged in MVP)

## D.4 Core error codes (locked list)

### VM / ABI failures

* `0x0000_0001 ERR_TRAP`
  The module trapped or crashed unexpectedly.
* `0x0000_0002 ERR_OOG`
  Out of gas. The runtime MUST stop execution deterministically and discard side effects.
* `0x0000_0003 ERR_OOB`
  Out-of-bounds memory access or invalid pointer/length supplied to hostcalls.
* `0x0000_0004 ERR_BAD_ENVELOPE`
  Returned bytes were not a valid Galaxy Result envelope.
* `0x0000_0005 ERR_BAD_ABI`
  Missing `memory` export, missing `main`, wrong signature, or invalid packed return.
* `0x0000_0006 ERR_BAD_INPUT`
  Input bytes failed required decoding rules for the called contract (MVP standard).
* `0x0000_0007 ERR_PANIC`
  Language-level panic/exception occurred and was mapped to a deterministic error.

### Hostcall failures

* `0x0000_1001 ERR_CAP_DENIED`
  Contract attempted a hostcall without the required capability enabled.
* `0x0000_1002 ERR_STORAGE_NOT_FOUND`
  Storage get requested a missing key (only used when API requires hard fail; `Option` APIs should not surface this).
* `0x0000_1003 ERR_STORAGE_LIMIT`
  Storage write violates key/value size limits or write quota.
* `0x0000_1004 ERR_EVENT_LIMIT`
  Event count or event size exceeds limits.
* `0x0000_1005 ERR_CRYPTO_FAIL`
  Crypto hostcall failed due to invalid input format.
* `0x0000_1006 ERR_ALLOC_FAIL`
  Host allocator failed to allocate requested memory.

### Determinism / validation failures

* `0x0000_2001 ERR_WASM_PROFILE`
  Module violates Galaxy WASM Profile (whitelist/blacklist rules).
* `0x0000_2002 ERR_IMPORTS_INVALID`
  Module imports forbidden functions or wrong signatures.
* `0x0000_2003 ERR_STATE_ROOT_MISMATCH`
  Re-execution produced a different state root than declared.
* `0x0000_2004 ERR_RECEIPT_MISMATCH`
  Re-execution produced different receipts/events than declared.
* `0x0000_2005 ERR_NONCE`
  Transaction nonce invalid.
* `0x0000_2006 ERR_SIG`
  Signature invalid.

### Identity / KYC / policy zone failures

* `0x0000_3001 ERR_KYC_REQUIRED`
  Operation requires a valid KYC attestation.
* `0x0000_3002 ERR_KYC_INVALID`
  Presented KYC attestation is invalid, expired, or revoked.
* `0x0000_3003 ERR_POLICY_DENIED`
  Operation denied by deterministic policy rules for the zone.

### Exchange / market rail failures

* `0x0000_4001 ERR_PRICE_BAND`
  Conversion request violates deterministic pricing corridor.
* `0x0000_4002 ERR_LIQUIDITY_LIMIT`
  Conversion request exceeds risk-limited liquidity policy.
* `0x0000_4003 ERR_MARKET_HALTED`
  Market rail temporarily halted by deterministic circuit breaker policy.

### Reserved generic application errors (recommended)

Galaxy reserves a small conventional subset for apps:

* `0x0000_8001 ERR_UNAUTHORIZED`
* `0x0000_8002 ERR_INSUFFICIENT_FUNDS`
* `0x0000_8003 ERR_NOT_FOUND`
* `0x0000_8004 ERR_INVALID_STATE`

Apps MAY define additional codes in `0x0000_8000..0x0000_FFFF` but SHOULD publish a codebook in their manifest.

## D.5 Side-effect policy on errors (locked)

* On any error (including `ERR_TRAP` and `ERR_OOG`), all buffered storage writes and deletes MUST be discarded.
* On any error, events MUST be discarded in v0.1 (locked), unless the event is explicitly a system “receipt event” emitted by the runtime (not by the contract). Contract-emitted events do not persist on failure in v0.1.

# Annex E — `.gxc` Packaging Format v0.1 (locked) + Canonical JSON

## E.1 Purpose

Galaxy MUST package deployable modules into a single deterministic artifact called a **GXC package** (`.gxc`). The `.gxc` must be reproducible and content-addressable, so that any node can verify that a given package corresponds to a given source intent and toolchain lock.

## E.2 Archive container (locked)

A `.gxc` file MUST be a **TAR archive** (ustar) with:

* no compression in v0.1 (locked), to avoid tool differences,
* normalized metadata.

The archive MUST contain exactly these files at the root:

1. `contract.wasm`
2. `manifest.json`
3. `provenance.json`
4. `checksums.txt`

No additional files are permitted in v0.1. Packages with extra files MUST be rejected.

## E.3 File ordering (locked)

Files MUST appear in this exact order inside the TAR:

1. `contract.wasm`
2. `manifest.json`
3. `provenance.json`
4. `checksums.txt`

Any other ordering MUST be rejected by strict validators.

## E.4 Normalized TAR metadata (locked)

For reproducibility, TAR headers MUST be normalized as follows:

* `mtime` MUST be `0`
* `uid` MUST be `0`
* `gid` MUST be `0`
* `uname` MUST be empty
* `gname` MUST be empty
* file permissions MUST be:

  * `contract.wasm`: `0644`
  * `manifest.json`: `0644`
  * `provenance.json`: `0644`
  * `checksums.txt`: `0644`

## E.5 Hashing (locked)

Galaxy MUST compute:

* `wasm_sha256 = sha256(contract.wasm bytes)`
* `gxc_sha256 = sha256(.gxc bytes)`

The `.gxc` hash is the canonical identity of the deployable package.

## E.6 `checksums.txt` format (locked)

`checksums.txt` MUST be ASCII with LF newlines and exactly these lines:

wasm_sha256 <HEX64>
manifest_sha256 <HEX64>
provenance_sha256 <HEX64>
gxc_sha256 <HEX64>
```

Each `<HEX64>` MUST be lowercase hex.

## E.7 Canonical JSON rules (locked)

Both `manifest.json` and `provenance.json` MUST be canonical JSON with these rules:

* UTF-8 encoding.
* Objects MUST have keys in lexicographic order.
* No trailing whitespace.
* Arrays preserve declared order.
* Numbers MUST be integers where specified; no floats.
* Booleans are `true/false`.
* Strings MUST use standard JSON escapes; no nonstandard unicode forms.

If canonicalization fails, packaging MUST fail.

## E.8 `manifest.json` schema (locked v0.1)

`manifest.json` MUST contain at minimum:

* `abi_version`: `"galaxy/wasm-abi@0.1"`
* `wasm_profile`: `"galaxy/wasm32-det@0.1"`
* `entry`: `"main"`
* `wasm_sha256`: `<HEX64 lowercase>`
* `package_sha256`: `<HEX64 lowercase>` (this is `gxc_sha256`)
* `required_caps`: array of strings from `{storage,events,crypto,context,gas,mem}`
* `contract_id`: `<HEX64 lowercase>` where `contract_id = sha256(wasm_sha256 || manifest_sha256)` (concatenated raw bytes)

The manifest MAY also include:

* `name`, `version`, `publisher`, `homepage`, `license`
* `app_error_codebook`: a map of app error codes to names
* `schemas`: input/output schema hashes

## E.9 `provenance.json` schema (locked v0.1)

`provenance.json` MUST contain:

* `gbi`: the Galaxy Build Intent object (canonical)
* `toolchain_lock`: pinned versions and hashes
* `ai_lock`: present only if AI used; includes model id, prompt template hash, decoding settings (temperature MUST be 0)
* `stage_hashes`: map from stage name to sha256
* `build_machine`: optional metadata (not used in hashing rules for contract identity; but still included in canonical JSON and checksum)

# Annex F — Adapter Conformance Suite v0.1 (locked vectors and expected results)

## F.1 Purpose

Galaxy MUST ship a conformance suite that every adapter must pass. This suite ensures that code written in different source languages produces identical behavior, and it ensures that hostcalls and envelope encoding are consistent across SDKs.

In v0.1, all conformance tests MUST be executed in the deterministic host harness and MUST produce identical:

* returned envelopes (exact bytes),
* storage diffs,
* emitted events (if successful),
* gas usage behavior (at least pass/fail, and recommended exact gas counts if locked).

## F.2 Canonical test harness assumptions (locked)

The harness MUST provide:

* empty initial storage unless specified,
* deterministic context values (fixed test vectors),
* gas limit = sufficiently high unless testing OOG,
* memory limit = sufficiently high unless testing OOM/OOB.

### F.2.1 Fixed deterministic context values (locked)

For conformance tests, the host MUST return:

* `ctx_sender` = 32 bytes of `0x11`
* `ctx_chain_id` = 32 bytes of `0x22`
* `ctx_tx_hash` = 32 bytes of `0x33`
* `ctx_block_height` = `12345`

## F.3 Envelope encoding tests (locked)

### Test F3.1 Ok envelope

Input: none (SDK-level unit test)
Expected bytes: `00 01 02 03` when payload is `[0x01,0x02,0x03]`.

### Test F3.2 Err envelope (ERR_UNAUTHORIZED)

Code: `0x0000_8001`
Expected bytes (hex): `01 01 80 00 00`
Explanation: `0x00008001` little-endian is `01 80 00 00`.

## F.4 Crypto test vectors (locked)

### Test F4.1 SHA-256

Input bytes: ASCII `"abc"` = `61 62 63`
Expected SHA-256 (hex, lowercase):
`ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad`

The adapter MUST call `env.sha256` and MUST return exactly these 32 bytes.

### Test F4.2 Blake2b-256

Input bytes: ASCII `"abc"` = `61 62 63`
Expected blake2b-256 (hex, lowercase):
`bddd813c634239723171ef3fee98579b94964e3bb1cb3e427262c8c068d52319`

The adapter MUST call `env.blake2b256` and MUST return exactly these 32 bytes.

### Test F4.3 Ed25519 verify

This test validates deterministic signature verification behavior.

* msg = ASCII `"test"` = `74 65 73 74`
* pk (32 bytes) = `000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f`
* sig (64 bytes) = `00` repeated 64 times (invalid signature)

Expected: `false` (0)

The adapter MUST call `env.ed25519_verify` and MUST report `false`.

## F.5 Storage determinism tests (locked)

### Test F5.1 Put/Get/Has/Del

Contract behavior:

1. `storage.put("k", "v")`
2. `storage.has("k")` must be true
3. `storage.get("k")` must return "v"
4. `storage.del("k")` must be true
5. `storage.has("k")` must be false

Key = bytes `"k"` = `6b`
Value = bytes `"v"` = `76`

Expected result envelope: Ok with payload = `01` (single byte success marker).
Expected envelope bytes: `00 01`

Expected storage diff after execution: empty (because key was deleted).
Expected events: none.

## F.6 Event ordering test (locked)

### Test F6.1 Two events order preserved

Contract behavior:

* emit event topic `"t1"` data `"a"`
* emit event topic `"t2"` data `"b"`

Expected:

* event[0] topic=`"t1"` data=`"a"`
* event[1] topic=`"t2"` data=`"b"`
* returned Ok payload = `01` so envelope bytes = `00 01`

## F.7 OOG determinism test (locked)

### Test F7.1 Out-of-gas must be deterministic

Harness config:

* gas limit extremely low (chain parameter for test)
  Contract behavior:
* calls `gas.tick(999999)` (or repeatedly ticks) until OOG

Expected:

* returned Err envelope with code `ERR_OOG = 0x0000_0002`
* envelope bytes = `01 02 00 00 00`
* storage diff MUST be empty
* events MUST be empty

## F.8 Trap mapping test (locked)

### Test F8.1 Trap must map to ERR_TRAP

Contract behavior:

* intentionally triggers a trap (for example division by zero in languages where it traps, or invalid memory access)
  Expected:
* returned Err envelope with code `ERR_TRAP = 0x0000_0001`
* envelope bytes = `01 01 00 00 00`
* storage diff MUST be empty
* events MUST be empty

## F.9 Pointer/length validation test (locked)

### Test F9.1 Invalid hostcall pointers must map to ERR_OOB

Contract behavior:

* calls storage_get with out_ptr beyond memory or out_cap invalid
  Expected:
* returned Err envelope with code `ERR_OOB = 0x0000_0003`
* envelope bytes = `01 03 00 00 00`
* storage diff MUST be empty
* events MUST be empty

## Implementation note (still locked behavior, but not extra features)

To make the conformance suite executable across all languages on day one, Galaxy MUST provide:

* a reference “conformance contract” per language route (native or AI→cGalaxy),
* a shared harness that runs the same tests and compares exact bytes.

Below are the **locked, canonical binary encodings** for **Tx / Receipt / Block / ProofBundle**, plus the **cross-cell chunking format** suitable for cellular/low-bandwidth transport. This is end-to-end enough that independent teams can implement node, wallet, bundler, and verifier without “interpretation gaps.”

# Annex G — Canonical Binary Encoding v0.1 (Tx / Receipt / Block / ProofBundle) **LOCKED**

## G.1 Global rules (apply to all objects)

Galaxy binary objects MUST be encoded using the rules below.

**Endianness**

* All fixed-width integers MUST be **little-endian**.

**Variable-length integers**

* All variable-length integers MUST be **ULEB128** (unsigned LEB128), the same encoding already used in GALIR-B.

**Byte strings**

* Byte strings MUST be encoded as:
  `len: uleb128` + `len bytes`

**Fixed-size primitives**

* `Hash32` = 32 bytes
* `PubKey32` = 32 bytes (Ed25519 public key)
* `Sig64` = 64 bytes (Ed25519 signature)

**Domain-separated hashing**

* Every hash in this annex MUST be computed as:
  `sha256(DOMAIN_TAG || encoded_bytes)`
* `DOMAIN_TAG` is ASCII bytes specified per object (below).
* This prevents cross-type collision and ensures canonical identities.

**Length framing**

* When an object is embedded inside another (e.g., Tx inside Block), it MUST be framed as:
  `obj_len: uleb128` + `obj_bytes`

**Signature rule**

* All signatures use **Ed25519** over a **signing preimage** defined per object.
* Signatures MUST be validated before accepting any object as truth.

## G.2 Transaction (Tx) v0.1 **LOCKED**

### G.2.1 Purpose

A Tx represents a deterministic call into a contract/module executed in the Galaxy WASM VM. Tx validity MUST be verifiable offline.

### G.2.2 Encoding: `TxBytes`

Tx is encoded as:

1. `tx_version: u8` = `0x01`
2. `cell_id: Hash32`
3. `sender_pk: PubKey32`
4. `nonce: u64_le`
5. `gas_limit: u64_le`
6. `fee_asset_id: Hash32` (the “planet/star/moon…” token id paying fees)
7. `contract_id: Hash32` (from `.gxc` manifest rule: contract identity)
8. `input: bytes` (uleb len + bytes)
9. `flags: u32_le` (MVP: MUST be `0`)
10. `sig: Sig64`

### G.2.3 Signing preimage and tx_id

**Signing preimage** is:

* `TX_SIGN_DOMAIN = b"GALXTX\0"`
* `preimage = TX_SIGN_DOMAIN || TxCoreBytes`

Where `TxCoreBytes` is `TxBytes` without the final `sig` field (i.e., fields 1–9).

**Signature**

* `sig = ed25519_sign(sender_sk, sha256(preimage))`

**Transaction id**

* `TX_ID_DOMAIN = b"GALXTXID"`
* `tx_id = sha256(TX_ID_DOMAIN || TxBytes)`
  (TxBytes includes the signature. This makes tx_id bind to the final signed transaction.)

### G.2.4 Validity rules (locked)

A Tx MUST be rejected if:

* `tx_version != 0x01`
* signature invalid
* nonce invalid for sender (chain rule)
* gas_limit exceeds chain maximum
* input length exceeds `MAX_INPUT_BYTES`
* fee_asset_id is not recognized as a valid fee token for the current zone (policy)
* contract_id not found or not permitted by policy zone rules

## G.3 Event and StorageDiff primitives (used in Receipt) **LOCKED**

### G.3.1 Event encoding

An Event is encoded as:

1. `topic: bytes`
2. `data: bytes`

Event hash:

* `EVT_DOMAIN = b"GALXEVT\0"`
* `event_hash = sha256(EVT_DOMAIN || EventBytes)`

Event merkle root:

* `events_root = merkle_root_sha256(event_hashes)`
* Merkle rule: pairwise sha256 of concatenated child hashes; if odd count, duplicate last. Empty list root = `sha256(b"GALXEVTEMPTY")`.

### G.3.2 StorageDiffItem encoding

A StorageDiffItem represents a committed change.

Encoding:

1. `op: u8` where `0x00 = PUT`, `0x01 = DEL`
2. `key: bytes`
3. If `op == PUT`: `value: bytes` else omitted.

Storage diff item hash:

* `SDI_DOMAIN = b"GALXSDI\0"`
* `sdi_hash = sha256(SDI_DOMAIN || StorageDiffItemBytes)`

Storage diff merkle root:

* `diff_root = merkle_root_sha256(sdi_hashes)`
* Empty list root = `sha256(b"GALXSDIEMPTY")`.

## G.4 Receipt v0.1 **LOCKED**

### G.4.1 Purpose

A Receipt commits to the deterministic outcome of executing one Tx: status, error code, gas used, event commitments, storage commitments, and post-state root.

### G.4.2 Encoding: `ReceiptBytes`

Receipt is encoded as:

1. `rcpt_version: u8` = `0x01`
2. `tx_id: Hash32`
3. `status: u8` where `0x00 = OK`, `0x01 = ERR`
4. `error_code: u32_le` (MUST be `0` if status OK; else one of Annex D codes)
5. `gas_used: u64_le`
6. `return_envelope_sha256: Hash32`

   * OK: hash of returned envelope bytes
   * ERR: hash of the returned envelope bytes (which is the error envelope)
7. `events_root: Hash32` (as defined above; on failure in v0.1 MUST be the empty root)
8. `storage_diff_root: Hash32` (as defined above; on failure MUST be empty root)
9. `post_state_root: Hash32`

### G.4.3 Receipt hash

* `RCPT_DOMAIN = b"GALXRCPT"`
* `receipt_hash = sha256(RCPT_DOMAIN || ReceiptBytes)`

### G.4.4 Locked failure side-effects (consistent with Annex D)

If execution results in any error (including `ERR_OOG` or `ERR_TRAP`):

* events_root MUST be empty root
* storage_diff_root MUST be empty root
* post_state_root MUST equal the pre-state root of the Tx execution (no changes)

## G.5 BlockHeader v0.1 **LOCKED**

### G.5.1 Purpose

The BlockHeader is the signed commitment to a block’s position and contents.

### G.5.2 Encoding: `HeaderBytes`

Header is encoded as:

1. `hdr_version: u8` = `0x01`
2. `cell_id: Hash32`
3. `epoch: u64_le` (MVP: MAY be `0`; used to segment sync eras)
4. `height: u64_le`
5. `prev_block_hash: Hash32` (genesis uses all-zero)
6. `slot: u64_le` (MVP: MUST equal height; logical time)
7. `tx_root: Hash32`
8. `receipt_root: Hash32`
9. `state_root: Hash32`
10. `producer_pk: PubKey32`
11. `producer_sig: Sig64`

### G.5.3 Roots and hashes (locked)

Tx merkle root:

* `tx_hash[i] = sha256(b"GALXTXH\0" || TxBytes_i)`
* `tx_root = merkle_root_sha256(tx_hashes)`
* Empty root = `sha256(b"GALXTXEMPTY")`

Receipt merkle root:

* `receipt_root = merkle_root_sha256(receipt_hashes)`
* Empty root forbidden in v0.1 blocks (a block with 0 txs is allowed only if chain explicitly permits it; if permitted, use empty root constant).

Block hash:

* `BLK_DOMAIN = b"GALXBLK\0"`
* `block_hash = sha256(BLK_DOMAIN || HeaderCoreBytes)`
  Where `HeaderCoreBytes` is HeaderBytes without `producer_sig`.

Header signature:

* `HDR_SIGN_DOMAIN = b"GALXHDR\0"`
* `producer_sig = ed25519_sign(producer_sk, sha256(HDR_SIGN_DOMAIN || HeaderCoreBytes))`

## G.6 Block v0.1 **LOCKED**

### G.6.1 Encoding: `BlockBytes`

Block is encoded as:

1. `block_version: u8` = `0x01`
2. `header: HeaderBytes` (fixed fields including signature)
3. `tx_count: uleb128`
4. `txs: [ tx_len + TxBytes ] * tx_count`
5. `receipt_count: uleb128` (MUST equal tx_count)
6. `receipts: [ rcpt_len + ReceiptBytes ] * receipt_count`

### G.6.2 Block validity rules (locked)

A block MUST be rejected if:

* `block_version != 0x01`
* header signature invalid
* `tx_count != receipt_count`
* header `tx_root` does not match computed tx_root
* header `receipt_root` does not match computed receipt_root
* re-executing txs in order does not produce:

  * receipts matching each tx (status, error_code, gas_used, return hash, roots)
  * final `state_root` matching the header

## G.7 ProofBundle v0.1 **LOCKED**

### G.7.1 Purpose

A ProofBundle is the portable, offline-syncable object that allows one cell/node to transfer a verified segment of chain history to another.

### G.7.2 Encoding: `BundleBytes`

Bundle is encoded as:

1. `bundle_version: u8` = `0x01`
2. `src_cell_id: Hash32`
3. `dst_cell_id: Hash32`

   * broadcast bundles MUST use all-zero dst_cell_id
4. `epoch: u64_le`
5. `segment_start_height: u64_le`
6. `segment_end_height: u64_le` (inclusive; MUST be ≥ start)
7. `checkpoint_block_hash: Hash32` (hash of block at height start-1, or all-zero if start=0)
8. `mode: u8` where:

   * `0x00 = FULL_BLOCKS` (MVP REQUIRED)
   * `0x01 = HEADERS_ONLY` (reserved; not allowed in v0.1)
9. `block_count: uleb128` (MUST equal `end-start+1`)
10. `blocks: [ block_len + BlockBytes ] * block_count`
11. `sender_pk: PubKey32`
12. `sender_sig: Sig64`

### G.7.3 Bundle hash and signature (locked)

Bundle hash:

* `BNDL_DOMAIN = b"GALXBNDL"`
* `bundle_hash = sha256(BNDL_DOMAIN || BundleCoreBytes)`
  Where `BundleCoreBytes` is BundleBytes without `sender_sig`.

Signature:

* `BNDL_SIGN_DOMAIN = b"GALXBNDLSIGN"`
* `sender_sig = ed25519_sign(sender_sk, sha256(BNDL_SIGN_DOMAIN || bundle_hash))`

### G.7.4 Bundle validity rules (locked)

A bundle MUST be rejected if:

* mode is not `FULL_BLOCKS` in v0.1
* bundle signature invalid
* block_count mismatch
* chain continuity fails within the segment (prev_hash mismatch)
* checkpoint_block_hash does not match receiver’s known checkpoint (if receiver has it)
* any block in the bundle fails block validation and deterministic replay rules

# Annex H — Cross-Cell Bundle Chunking Format v0.1 (cellular transport) **LOCKED**

## H.1 Purpose

Chunking exists because proof bundles can be large, and cellular/low-bandwidth transport requires splitting into bounded frames that can be reassembled offline.

This annex defines a canonical chunk format that can be carried over:

* TCP/UDP on cellular data
* store-and-forward file transfer
* Bluetooth / local Wi-Fi
* and later SMS/USSD gateways (with an extra text encoding layer)

## H.2 Chunk object: `ChunkBytes` **LOCKED**

A chunk is encoded as:

1. `chunk_version: u8` = `0x01`
2. `bundle_hash: Hash32` (from Annex G)
3. `bundle_len: u32_le` (exact bytes of BundleBytes)
4. `chunk_count: u32_le`
5. `chunk_index: u32_le` (0-based; MUST be < chunk_count)
6. `chunk_offset: u32_le` (offset into BundleBytes)
7. `payload_len: u32_le`
8. `payload_bytes: payload_len bytes`
   These bytes MUST equal `BundleBytes[chunk_offset .. chunk_offset+payload_len)`.
9. `chunk_payload_sha256: Hash32`

   * `CHUNK_PAY_DOMAIN = b"GALXCHP\0"`
   * `chunk_payload_sha256 = sha256(CHUNK_PAY_DOMAIN || bundle_hash || chunk_index_le || payload_bytes)`
10. `chunk_crc32c: u32_le`
    CRC32C of bytes (1..9) for quick corruption detection. CRC32C is optional in many systems, but in cellular it is extremely helpful; therefore it is locked into v0.1 chunk format.

### H.2.1 Chunk size rules (locked)

* `payload_len` MUST be ≤ `MAX_CHUNK_PAYLOAD` (network policy; recommended default 32 KiB).
* Chunks MUST cover the entire bundle with no gaps and no overlaps:

  * chunk 0 offset MUST be 0
  * all offsets and lengths MUST be consistent so reassembly is unambiguous

### H.2.2 Reassembly rules (locked)

A receiver MUST:

1. group chunks by `bundle_hash`
2. validate CRC32C first for fast rejection
3. validate `chunk_payload_sha256`
4. place payload into an in-memory or on-disk buffer at `chunk_offset`
5. once all bytes are present, validate:

   * computed `bundle_hash` matches the advertised `bundle_hash`
   * bundle signature inside BundleBytes validates (Annex G)
6. only then attempt block verification and replay

A receiver MUST reject if:

* any chunk fails CRC32C or sha256 validation
* any chunk claims an invalid offset/length
* bundle signature invalid after reassembly

## H.3 Transport framing (binary) **LOCKED**

When chunks are sent over a stream (TCP) or stored in a file containing multiple chunks, each chunk MUST be framed as:

1. `frame_magic: [u8;4]` = `b"GCX0"`
2. `frame_len: u32_le` (length of ChunkBytes)
3. `ChunkBytes`

This allows a receiver to scan and recover from partial frames.

## H.4 Text transport layer (SMS / USSD / manual copy) **LOCKED**

If chunks must be carried as text, ChunkBytes MUST be encoded using **Base32 (RFC 4648, uppercase)** with no padding. The text message MUST contain:

* prefix: `"GCX0:"`
* then base32 payload

Receivers MUST:

* strip prefix
* base32 decode
* parse as ChunkBytes
* validate as in H.2.2

This rule is locked so that offline regions can share proof bundles even via constrained channels.

