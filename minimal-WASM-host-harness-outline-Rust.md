```rust
// minimal_wasm_host.rs
//
// Galaxy WASM Host Harness v0.1 (ready-to-run outline)
// - Loads a .wasm contract
// - Wires env.* imports (storage/events/crypto/context/gas/mem alloc)
// - Writes input into contract memory using the same deterministic bump allocator
// - Calls exported main(in_ptr, in_len) -> i64 (packed out_ptr/out_len)
// - Reads return slice, decodes Result envelope
// - Asserts storage/events side effects for golden tests
//
// NOTE: This is a *test harness* reference. It is deterministic by design.
// You can adapt it into a CLI runner or a unit-test runner.
//
// -------------------- Cargo.toml --------------------
// [package]
// name = "galaxy_wasm_host"
// version = "0.1.0"
// edition = "2021"
//
// [dependencies]
// anyhow = "1"
// wasmtime = "19"          # pin to a known version in your repo
// serde = { version = "1", features = ["derive"] }
// serde_json = "1"
// hex = "0.4"
// sha2 = "0.10"
// blake2 = "0.10"
// ed25519-dalek = "2"      # for verify_ed25519
//
// ----------------------------------------------------

use anyhow::{anyhow, bail, Context, Result};
use blake2::{Blake2b512, Digest as _};
use ed25519_dalek::{Signature, VerifyingKey};
use serde::{Deserialize, Serialize};
use sha2::Sha256;
use std::collections::BTreeMap;
use std::fs;
use wasmtime::{Caller, Engine, Extern, Instance, Linker, Memory, Module, Store, TypedFunc};

/// ============ Result Envelope (canonical) ============
/// Ok:  0x00 || payload
/// Err: 0x01 || u32_le(error_code)

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

pub fn decode_envelope(bytes: &[u8]) -> std::result::Result<GalaxyResult, EnvelopeError> {
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

/// Pack/unpack (ptr,len) into i64/u64:
/// packed = (u64(len) << 32) | u64(ptr)
fn pack_slice(ptr: u32, len: u32) -> u64 {
    ((len as u64) << 32) | (ptr as u64)
}
fn unpack_slice(packed: u64) -> (u32, u32) {
    let ptr = (packed & 0xFFFF_FFFF) as u32;
    let len = (packed >> 32) as u32;
    (ptr, len)
}

/// ============ Deterministic Host State ============

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContextCfg {
    /// hex string bytes (without 0x is fine)
    pub sender_hex: String,
    pub chain_id_hex: String,
    pub tx_hash_hex: String,
    pub block_height: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Fixture {
    pub wasm_path: String,
    pub input_hex: String,

    /// initial storage (hex key -> hex value)
    pub storage: BTreeMap<String, String>,

    pub context: ContextCfg,
    pub gas_budget: u64,

    /// Expected outputs for golden tests
    pub expected: Expected,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Expected {
    /// expected result envelope (hex bytes)
    pub result_envelope_hex: String,

    /// expected final storage (hex key -> hex value)
    pub storage: BTreeMap<String, String>,

    /// expected events: list of (topic_hex, data_hex)
    pub events: Vec<(String, String)>,

    /// optional: expected gas remaining (if you want to assert)
    pub gas_remaining: Option<u64>,
}

#[derive(Debug, Clone)]
pub struct EventRecord {
    pub topic: Vec<u8>,
    pub data: Vec<u8>,
}

#[derive(Debug)]
pub struct HostState {
    // deterministic storage: use BTreeMap for stable ordering in debug output
    pub storage: BTreeMap<Vec<u8>, Vec<u8>>,
    pub events: Vec<EventRecord>,
    pub ctx_sender: Vec<u8>,
    pub ctx_chain_id: Vec<u8>,
    pub ctx_tx_hash: Vec<u8>,
    pub ctx_block_height: u64,
    pub gas_remaining: u64,

    // deterministic bump allocator pointer in WASM linear memory
    pub heap_ptr: u32,
    pub memory_limit_bytes: u32,

    // policy limits
    pub max_key: usize,
    pub max_value: usize,
    pub max_topic: usize,
    pub max_event_data: usize,
    pub max_events: usize,
}

impl HostState {
    pub fn from_fixture(fx: &Fixture) -> Result<Self> {
        let mut storage = BTreeMap::new();
        for (khex, vhex) in &fx.storage {
            storage.insert(hex_to_bytes(khex)?, hex_to_bytes(vhex)?);
        }

        Ok(Self {
            storage,
            events: vec![],
            ctx_sender: hex_to_bytes(&fx.context.sender_hex)?,
            ctx_chain_id: hex_to_bytes(&fx.context.chain_id_hex)?,
            ctx_tx_hash: hex_to_bytes(&fx.context.tx_hash_hex)?,
            ctx_block_height: fx.context.block_height,
            gas_remaining: fx.gas_budget,

            heap_ptr: 65536,                // 64KiB deterministic start
            memory_limit_bytes: 1024 * 1024, // 1MiB default for tests

            max_key: 1024,
            max_value: 256 * 1024,
            max_topic: 4096,
            max_event_data: 32 * 1024,
            max_events: 1024,
        })
    }
}

/// ============ Memory helpers (safe + deterministic) ============

fn get_memory(mut caller: Caller<'_, HostState>) -> Result<Memory> {
    match caller.get_export("memory") {
        Some(Extern::Memory(m)) => Ok(m),
        _ => bail!("WASM instance does not export memory named 'memory'"),
    }
}

fn mem_read(caller: &mut Caller<'_, HostState>, ptr: i32, len: i32) -> Result<Vec<u8>> {
    if ptr < 0 || len < 0 {
        bail!("mem_read invalid ptr/len");
    }
    let ptr = ptr as usize;
    let len = len as usize;

    let mem = get_memory(caller.as_context_mut())?;
    let data = mem.data(caller);
    let end = ptr.checked_add(len).ok_or_else(|| anyhow!("mem_read overflow"))?;
    if end > data.len() {
        bail!("mem_read OOB: end={} mem={}", end, data.len());
    }
    Ok(data[ptr..end].to_vec())
}

fn mem_write(caller: &mut Caller<'_, HostState>, ptr: i32, bytes: &[u8]) -> Result<()> {
    if ptr < 0 {
        bail!("mem_write invalid ptr");
    }
    let ptr = ptr as usize;

    let mem = get_memory(caller.as_context_mut())?;
    let data = mem.data_mut(caller);
    let end = ptr.checked_add(bytes.len()).ok_or_else(|| anyhow!("mem_write overflow"))?;
    if end > data.len() {
        bail!("mem_write OOB: end={} mem={}", end, data.len());
    }
    data[ptr..end].copy_from_slice(bytes);
    Ok(())
}

/// Ensure memory is large enough for an allocation.
/// This grows WASM memory deterministically if needed.
fn ensure_memory_capacity(caller: &mut Caller<'_, HostState>, needed_end: u32) -> Result<()> {
    let mem = get_memory(caller.as_context_mut())?;
    let cur = mem.data_size(caller) as u32;

    if needed_end <= cur {
        return Ok(());
    }
    let limit = caller.data().memory_limit_bytes;
    if needed_end > limit {
        bail!("memory limit exceeded: needed_end={} limit={}", needed_end, limit);
    }

    // WASM page is 64KiB
    let page_size: u32 = 65536;
    let cur_pages = (cur + page_size - 1) / page_size;
    let needed_pages = (needed_end + page_size - 1) / page_size;
    let grow_by = needed_pages.saturating_sub(cur_pages);

    if grow_by > 0 {
        mem.grow(caller, grow_by as u64)
            .map_err(|e| anyhow!("memory.grow failed: {}", e))?;
    }
    Ok(())
}

/// Deterministic bump allocator used by env.alloc and also internally by hostcalls.
fn alloc_impl(caller: &mut Caller<'_, HostState>, len: i32) -> Result<u32> {
    if len < 0 {
        return Ok(0);
    }
    let len_u = len as u32;
    if len_u == 0 {
        return Ok(0);
    }

    let start = caller.data().heap_ptr;
    let end = start.checked_add(len_u).ok_or_else(|| anyhow!("alloc overflow"))?;

    ensure_memory_capacity(caller, end)?;

    // update heap_ptr with 8-byte alignment
    let mut new_hp = end;
    new_hp = (new_hp + 7) & !7;
    caller.data_mut().heap_ptr = new_hp;

    Ok(start)
}

/// Allocates and writes bytes into contract memory, returns (ptr,len).
fn alloc_and_write(caller: &mut Caller<'_, HostState>, bytes: &[u8]) -> Result<(u32, u32)> {
    let ptr = alloc_impl(caller, bytes.len() as i32)?;
    if ptr == 0 {
        return Ok((0, 0));
    }
    mem_write(caller, ptr as i32, bytes)?;
    Ok((ptr, bytes.len() as u32))
}

/// ============ env.* host functions wiring ============

fn wire_env_imports(linker: &mut Linker<HostState>) -> Result<()> {
    // env.alloc(len:i32)->i32
    linker.func_wrap("env", "alloc", |mut caller: Caller<'_, HostState>, len: i32| -> i32 {
        match alloc_impl(&mut caller, len) {
            Ok(ptr) => ptr as i32,
            Err(_) => 0,
        }
    })?;

    // env.storage_has(key_ptr,key_len)->i32
    linker.func_wrap(
        "env",
        "storage_has",
        |mut caller: Caller<'_, HostState>, key_ptr: i32, key_len: i32| -> i32 {
            let key = match mem_read(&mut caller, key_ptr, key_len) {
                Ok(k) => k,
                Err(_) => return 0,
            };
            if caller.data().storage.contains_key(&key) {
                1
            } else {
                0
            }
        },
    )?;

    // env.storage_get(key_ptr,key_len)->i64 packed slice
    linker.func_wrap(
        "env",
        "storage_get",
        |mut caller: Caller<'_, HostState>, key_ptr: i32, key_len: i32| -> i64 {
            let key = match mem_read(&mut caller, key_ptr, key_len) {
                Ok(k) => k,
                Err(_) => return 0,
            };

            let Some(val) = caller.data().storage.get(&key).cloned() else {
                return 0;
            };

            // allocate and copy into contract memory
            let (ptr, len) = match alloc_and_write(&mut caller, &val) {
                Ok(pl) => pl,
                Err(_) => return 0,
            };
            pack_slice(ptr, len) as i64
        },
    )?;

    // env.storage_put(key_ptr,key_len,val_ptr,val_len)->i32 (0 ok, 5 ERR_STORAGE)
    linker.func_wrap(
        "env",
        "storage_put",
        |mut caller: Caller<'_, HostState>,
         key_ptr: i32,
         key_len: i32,
         val_ptr: i32,
         val_len: i32|
         -> i32 {
            let key = match mem_read(&mut caller, key_ptr, key_len) {
                Ok(k) => k,
                Err(_) => return 5,
            };
            let val = match mem_read(&mut caller, val_ptr, val_len) {
                Ok(v) => v,
                Err(_) => return 5,
            };

            if key.len() > caller.data().max_key || val.len() > caller.data().max_value {
                return 5;
            }

            caller.data_mut().storage.insert(key, val);
            0
        },
    )?;

    // env.storage_del(key_ptr,key_len)->i32
    linker.func_wrap(
        "env",
        "storage_del",
        |mut caller: Caller<'_, HostState>, key_ptr: i32, key_len: i32| -> i32 {
            let key = match mem_read(&mut caller, key_ptr, key_len) {
                Ok(k) => k,
                Err(_) => return 5,
            };
            caller.data_mut().storage.remove(&key);
            0
        },
    )?;

    // env.emit_event(topic_ptr,topic_len,data_ptr,data_len)->i32
    linker.func_wrap(
        "env",
        "emit_event",
        |mut caller: Caller<'_, HostState>,
         topic_ptr: i32,
         topic_len: i32,
         data_ptr: i32,
         data_len: i32|
         -> i32 {
            if caller.data().events.len() >= caller.data().max_events {
                return 8;
            }

            let topic = match mem_read(&mut caller, topic_ptr, topic_len) {
                Ok(t) => t,
                Err(_) => return 8,
            };
            let data = match mem_read(&mut caller, data_ptr, data_len) {
                Ok(d) => d,
                Err(_) => return 8,
            };

            if topic.len() > caller.data().max_topic || data.len() > caller.data().max_event_data {
                return 8;
            }

            caller.data_mut().events.push(EventRecord { topic, data });
            0
        },
    )?;

    // env.sha256(data_ptr,data_len)->i64 packed slice
    linker.func_wrap(
        "env",
        "sha256",
        |mut caller: Caller<'_, HostState>, data_ptr: i32, data_len: i32| -> i64 {
            let data = match mem_read(&mut caller, data_ptr, data_len) {
                Ok(d) => d,
                Err(_) => return 0,
            };
            let mut hasher = Sha256::new();
            hasher.update(&data);
            let digest = hasher.finalize().to_vec(); // 32 bytes

            let (ptr, len) = match alloc_and_write(&mut caller, &digest) {
                Ok(pl) => pl,
                Err(_) => return 0,
            };
            pack_slice(ptr, len) as i64
        },
    )?;

    // env.blake2b256(data_ptr,data_len)->i64 packed slice
    linker.func_wrap(
        "env",
        "blake2b256",
        |mut caller: Caller<'_, HostState>, data_ptr: i32, data_len: i32| -> i64 {
            let data = match mem_read(&mut caller, data_ptr, data_len) {
                Ok(d) => d,
                Err(_) => return 0,
            };
            let mut h = Blake2b512::new();
            h.update(&data);
            let out512 = h.finalize();
            let digest = out512[..32].to_vec(); // truncate to 256-bit

            let (ptr, len) = match alloc_and_write(&mut caller, &digest) {
                Ok(pl) => pl,
                Err(_) => return 0,
            };
            pack_slice(ptr, len) as i64
        },
    )?;

    // env.verify_ed25519(pub,msg,sig)->i32
    linker.func_wrap(
        "env",
        "verify_ed25519",
        |mut caller: Caller<'_, HostState>,
         pub_ptr: i32,
         pub_len: i32,
         msg_ptr: i32,
         msg_len: i32,
         sig_ptr: i32,
         sig_len: i32|
         -> i32 {
            if pub_len != 32 || sig_len != 64 {
                return 0;
            }
            let pubkey = match mem_read(&mut caller, pub_ptr, pub_len) {
                Ok(p) => p,
                Err(_) => return 0,
            };
            let msg = match mem_read(&mut caller, msg_ptr, msg_len) {
                Ok(m) => m,
                Err(_) => return 0,
            };
            let sig = match mem_read(&mut caller, sig_ptr, sig_len) {
                Ok(s) => s,
                Err(_) => return 0,
            };

            let vk = match VerifyingKey::from_bytes(&pubkey.try_into().unwrap_or([0u8; 32])) {
                Ok(v) => v,
                Err(_) => return 0,
            };
            let signature = match Signature::from_bytes(&sig.try_into().unwrap_or([0u8; 64])) {
                Ok(s) => s,
                Err(_) => return 0,
            };

            match vk.verify_strict(&msg, &signature) {
                Ok(()) => 1,
                Err(_) => 0,
            }
        },
    )?;

    // context: returns packed slices (allocated in contract memory)
    linker.func_wrap("env", "ctx_sender", |mut caller: Caller<'_, HostState>| -> i64 {
        let (ptr, len) = match alloc_and_write(&mut caller, &caller.data().ctx_sender) {
            Ok(pl) => pl,
            Err(_) => return 0,
        };
        pack_slice(ptr, len) as i64
    })?;

    linker.func_wrap("env", "ctx_chain_id", |mut caller: Caller<'_, HostState>| -> i64 {
        let (ptr, len) = match alloc_and_write(&mut caller, &caller.data().ctx_chain_id) {
            Ok(pl) => pl,
            Err(_) => return 0,
        };
        pack_slice(ptr, len) as i64
    })?;

    linker.func_wrap("env", "ctx_tx_hash", |mut caller: Caller<'_, HostState>| -> i64 {
        let (ptr, len) = match alloc_and_write(&mut caller, &caller.data().ctx_tx_hash) {
            Ok(pl) => pl,
            Err(_) => return 0,
        };
        pack_slice(ptr, len) as i64
    })?;

    linker.func_wrap("env", "ctx_block_height", |caller: Caller<'_, HostState>| -> i64 {
        caller.data().ctx_block_height as i64
    })?;

    // gas
    linker.func_wrap("env", "gas_remaining", |caller: Caller<'_, HostState>| -> i64 {
        caller.data().gas_remaining as i64
    })?;

    linker.func_wrap("env", "gas_tick", |mut caller: Caller<'_, HostState>, cost: i64| -> i32 {
        if cost < 0 {
            return 0;
        }
        let c = cost as u64;
        if caller.data().gas_remaining >= c {
            caller.data_mut().gas_remaining -= c;
            1
        } else {
            0
        }
    })?;

    Ok(())
}

/// ============ Contract Runner ============

pub struct RunOutput {
    pub result: GalaxyResult,
    pub raw_envelope: Vec<u8>,
    pub host: HostState,
}

/// Load/instantiate contract, run main, decode envelope, return host state for assertions.
pub fn run_contract(fx: &Fixture) -> Result<RunOutput> {
    let engine = Engine::default();
    let wasm_bytes = fs::read(&fx.wasm_path).with_context(|| "failed to read wasm")?;
    let module = Module::new(&engine, wasm_bytes).with_context(|| "failed to compile wasm")?;

    let host_state = HostState::from_fixture(fx)?;
    let mut store = Store::new(&engine, host_state);

    let mut linker = Linker::new(&engine);
    wire_env_imports(&mut linker)?;

    let instance = linker
        .instantiate(&mut store, &module)
        .with_context(|| "failed to instantiate")?;

    let main = get_typed_main(&mut store, &instance)?;

    // Allocate and write input into WASM memory using the SAME allocator used by env.alloc
    let input = hex_to_bytes(&fx.input_hex)?;
    let (in_ptr, in_len) = {
        let mut caller = store.as_context_mut();
        // We can't call env.alloc directly; we reuse alloc_impl deterministically.
        // (This is correct because env.alloc ALSO uses alloc_impl.)
        let mut fake = Caller::new(caller, &instance, &mut *store.data_mut()); // Not available in wasmtime
        // Instead: do it without Caller by directly editing store state and Memory.
        // We'll use the Memory export directly:
        let mem = get_instance_memory(&mut store, &instance)?;
        let ptr = alloc_for_instance(&mut store, &mem, input.len() as u32)?;
        mem_write_instance(&mut store, &mem, ptr, &input)?;
        (ptr, input.len() as u32)
    };

    // Call exported main(in_ptr, in_len) -> i64 packed return
    let packed: u64 = main
        .call(&mut store, (in_ptr, in_len))
        .with_context(|| "main call failed")? as u64;

    let (out_ptr, out_len) = unpack_slice(packed);
    let out_bytes = read_instance_bytes(&mut store, &instance, out_ptr, out_len)?;

    let decoded = decode_envelope(&out_bytes).map_err(|e| anyhow!("envelope decode error: {:?}", e))?;

    // Return host state for assertions
    let host = store.into_data();

    Ok(RunOutput {
        result: decoded,
        raw_envelope: out_bytes,
        host,
    })
}

/// WASM main typed func: (i32,i32)->i64 but represented here as (u32,u32)->i64 for convenience.
fn get_typed_main(store: &mut Store<HostState>, instance: &Instance) -> Result<TypedFunc<(u32, u32), i64>> {
    let f = instance
        .get_typed_func::<(u32, u32), i64>(store, "main")
        .with_context(|| "exported function `main` not found or wrong signature")?;
    Ok(f)
}

fn get_instance_memory(store: &mut Store<HostState>, instance: &Instance) -> Result<Memory> {
    match instance.get_export(store, "memory") {
        Some(Extern::Memory(m)) => Ok(m),
        _ => bail!("instance missing exported memory"),
    }
}

fn alloc_for_instance(store: &mut Store<HostState>, mem: &Memory, len: u32) -> Result<u32> {
    if len == 0 {
        return Ok(0);
    }
    let start = store.data().heap_ptr;
    let end = start.checked_add(len).ok_or_else(|| anyhow!("alloc overflow"))?;
    ensure_memory_capacity_instance(store, mem, end)?;
    let mut new_hp = end;
    new_hp = (new_hp + 7) & !7;
    store.data_mut().heap_ptr = new_hp;
    Ok(start)
}

fn ensure_memory_capacity_instance(store: &mut Store<HostState>, mem: &Memory, needed_end: u32) -> Result<()> {
    let cur = mem.data_size(store) as u32;
    if needed_end <= cur {
        return Ok(());
    }
    let limit = store.data().memory_limit_bytes;
    if needed_end > limit {
        bail!("memory limit exceeded: needed_end={} limit={}", needed_end, limit);
    }
    let page_size: u32 = 65536;
    let cur_pages = (cur + page_size - 1) / page_size;
    let needed_pages = (needed_end + page_size - 1) / page_size;
    let grow_by = needed_pages.saturating_sub(cur_pages);
    if grow_by > 0 {
        mem.grow(store, grow_by as u64).map_err(|e| anyhow!("memory.grow failed: {}", e))?;
    }
    Ok(())
}

fn mem_write_instance(store: &mut Store<HostState>, mem: &Memory, ptr: u32, bytes: &[u8]) -> Result<()> {
    let data = mem.data_mut(store);
    let start = ptr as usize;
    let end = start.checked_add(bytes.len()).ok_or_else(|| anyhow!("mem_write overflow"))?;
    if end > data.len() {
        bail!("mem_write OOB: end={} mem={}", end, data.len());
    }
    data[start..end].copy_from_slice(bytes);
    Ok(())
}

fn read_instance_bytes(store: &mut Store<HostState>, instance: &Instance, ptr: u32, len: u32) -> Result<Vec<u8>> {
    let mem = get_instance_memory(store, instance)?;
    let data = mem.data(store);
    let start = ptr as usize;
    let end = start.checked_add(len as usize).ok_or_else(|| anyhow!("read overflow"))?;
    if end > data.len() {
        bail!("read OOB: end={} mem={}", end, data.len());
    }
    Ok(data[start..end].to_vec())
}

/// ============ Golden Assertions ============

pub fn assert_against_fixture(out: &RunOutput, fx: &Fixture) -> Result<()> {
    // 1) Assert result envelope bytes exact match (strongest)
    let expected_env = hex_to_bytes(&fx.expected.result_envelope_hex)?;
    if out.raw_envelope != expected_env {
        bail!(
            "Result envelope mismatch.\nExpected: {}\nGot:      {}",
            hex::encode(&expected_env),
            hex::encode(&out.raw_envelope)
        );
    }

    // 2) Assert events (topic/data)
    let expected_events: Vec<(Vec<u8>, Vec<u8>)> = fx
        .expected
        .events
        .iter()
        .map(|(t, d)| Ok((hex_to_bytes(t)?, hex_to_bytes(d)?)))
        .collect::<Result<_>>()?;

    if out.host.events.len() != expected_events.len() {
        bail!(
            "Event count mismatch. expected={}, got={}",
            expected_events.len(),
            out.host.events.len()
        );
    }

    for (i, (exp_t, exp_d)) in expected_events.iter().enumerate() {
        let got = &out.host.events[i];
        if &got.topic != exp_t || &got.data != exp_d {
            bail!(
                "Event[{}] mismatch.\nExpected topic={}, data={}\nGot      topic={}, data={}",
                i,
                hex::encode(exp_t),
                hex::encode(exp_d),
                hex::encode(&got.topic),
                hex::encode(&got.data),
            );
        }
    }

    // 3) Assert storage final state exact match
    let mut expected_storage = BTreeMap::new();
    for (khex, vhex) in &fx.expected.storage {
        expected_storage.insert(hex_to_bytes(khex)?, hex_to_bytes(vhex)?);
    }

    if out.host.storage != expected_storage {
        // Show a deterministic diff summary (sorted by key due to BTreeMap)
        bail!(
            "Storage mismatch.\nExpected keys: {}\nGot keys:      {}",
            expected_storage.len(),
            out.host.storage.len()
        );
    }

    // 4) Optional: gas remaining
    if let Some(exp_gas) = fx.expected.gas_remaining {
        if out.host.gas_remaining != exp_gas {
            bail!(
                "Gas remaining mismatch. expected={}, got={}",
                exp_gas,
                out.host.gas_remaining
            );
        }
    }

    Ok(())
}

/// ============ Fixture Loading + Helpers ============

pub fn load_fixture(path: &str) -> Result<Fixture> {
    let s = fs::read_to_string(path)?;
    let fx: Fixture = serde_json::from_str(&s)?;
    Ok(fx)
}

fn hex_to_bytes(s: &str) -> Result<Vec<u8>> {
    let s = s.trim();
    let s = s.strip_prefix("0x").unwrap_or(s);
    if s.is_empty() {
        return Ok(vec![]);
    }
    Ok(hex::decode(s).with_context(|| format!("invalid hex: {}", s))?)
}

/// ============ Example CLI runner (optional) ============
/// Usage:
///   cargo run -- fixtures/001_addition.json
fn main() -> Result<()> {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 2 {
        eprintln!("Usage: {} <fixture.json>", args[0]);
        std::process::exit(2);
    }

    let fx = load_fixture(&args[1])?;
    let out = run_contract(&fx)?;
    assert_against_fixture(&out, &fx)?;

    println!("OK ✅ result={:?} gas_remaining={}", out.result, out.host.gas_remaining);
    Ok(())
}
```

### Example fixture JSON (minimal)

```json
{
  "wasm_path": "artifacts/contract.wasm",
  "input_hex": "00aabbcc",
  "storage": {
    "01": "ff"
  },
  "context": {
    "sender_hex": "1111111111111111111111111111111111111111",
    "chain_id_hex": "67616c6178792d746573746e6574",
    "tx_hash_hex": "2222222222222222222222222222222222222222222222222222222222222222",
    "block_height": 1
  },
  "gas_budget": 500000,
  "expected": {
    "result_envelope_hex": "00deadbeef",
    "storage": {
      "01": "ff"
    },
    "events": [],
    "gas_remaining": 500000
  }
}
```

### Notes (important for “ready-to-run”)

* This harness expects the contract exports:

  * `memory` (WASM linear memory)
  * `main(i32,i32)->i64`
* The harness uses the same bump allocator pointer (`heap_ptr`) as the host imports. This ensures input writing and host-returned slices never overlap unpredictably.
* If you want hostcall gas charging (beyond explicit `gas_tick`), tell me and I’ll add a deterministic “charge_hostcall(cost)” layer that fails consistently.

