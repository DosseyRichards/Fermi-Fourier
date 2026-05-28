# VM internals

Fourier compiles to the WaveLedger stack VM, defined in `vm/`. This
page mirrors the implementation directly. Source: `vm/opcodes.py`,
`vm/machine.py`, `vm/precompiles.py`, `vm/state.py`, `vm/deploy.py`.

## Word size

Every stack and storage value is a **256-bit unsigned integer**. All
arithmetic is modulo `2**256` unless explicitly checked (via
`safe_*` builtins). `WORD_MAX = (1 << 256) - 1`.

## Limits

| Constant | Value | Source |
|---|---|---|
| `MAX_CALL_DEPTH` | 256 | `vm/opcodes.py` |
| `MAX_STACK` | 1024 | `vm/opcodes.py` |
| `ADDRESS_BYTES` | 20 | `vm/state.py` |

Exceeding any of these raises a hard `VMError` — full gas consumed,
state rolled back.

## Opcode table

Opcodes are arranged in functional groups. Within each row, the
"gas" column is the base cost; some opcodes charge per-byte or
per-word adders (noted in the Adder column).

### Stack manipulation

| Op | Hex | Gas | Adder | Notes |
|---|---|---|---|---|
| `STOP` | `0x00` | 0 | — | Halt successfully, return empty data |
| `PUSH` | `0x01` | 3 | — | Immediate byte N (1..32) then N literal bytes |
| `POP` | `0x02` | 2 | — | Discard top |
| `DUP` | `0x03` | 3 | — | Immediate byte i (1..16); duplicates i-th item from top |
| `SWAP` | `0x04` | 3 | — | Immediate byte i (1..16); swap top with i+1-th |

### Arithmetic

All modulo `2**256`. Stack convention: `OP s_0 s_1` pops `s_0` first.
For `SUB`: `s_0 - s_1` (so PUSH a, PUSH b → top=b, result=b-a).

| Op | Hex | Gas | Notes |
|---|---|---|---|
| `ADD` | `0x10` | 3 | |
| `SUB` | `0x11` | 3 | |
| `MUL` | `0x12` | 5 | |
| `DIV` | `0x13` | 5 | Div by 0 → 0 |
| `MOD` | `0x14` | 5 | Mod by 0 → 0 |
| `EXP` | `0x15` | 10 | |

### Comparison and logic

Result is `1` or `0`.

| Op | Hex | Gas | Notes |
|---|---|---|---|
| `LT` | `0x20` | 3 | Unsigned |
| `GT` | `0x21` | 3 | Unsigned |
| `EQ` | `0x22` | 3 | |
| `ISZERO` | `0x23` | 3 | Unary: `1` if 0 else `0` |
| `AND` | `0x24` | 3 | Bitwise |
| `OR` | `0x25` | 3 | Bitwise |
| `XOR` | `0x26` | 3 | Bitwise |
| `NOT` | `0x27` | 3 | Unary bitwise (256-bit complement) |

### Cryptographic

| Op | Hex | Gas | Adder | Notes |
|---|---|---|---|---|
| `SHA3` | `0x30` | 30 | +6 per word hashed | SHA3-256 over `mem[offset:offset+length]`; push 256-bit truncated digest |

### Memory

Byte-addressable scratch, zero-initialized, grows on demand. Memory
expansion cost: `cost(words) = 3 * words + (words^2) / 512`; charged
as the delta on first access.

| Op | Hex | Gas | Notes |
|---|---|---|---|
| `MLOAD` | `0x40` | 3 | Push `mem[offset:offset+32]` as a uint (big-endian) |
| `MSTORE` | `0x41` | 3 | Pop offset, value; write 32-byte BE word |

### Storage

Per-contract persistent KV (`uint → uint`).

| Op | Hex | Base gas | Notes |
|---|---|---|---|
| `SLOAD` | `0x50` | 200 | Read storage slot |
| `SSTORE` | `0x51` | 5000 | Write storage slot. If prev was 0 and new is non-zero, additional 15000 charged (`SSTORE_SET_GAS = 20000`). Forbidden in STATICCALL frames |

There is no gas refund for clearing slots (`SSTORE_CLEAR_REFUND = 0`).

### Control flow

`JUMPI` pops `dest` then `cond`; jumps to `dest` if `cond != 0`. Both
`JUMP` and `JUMPI` require the destination to be a `JUMPDEST` opcode
position (pre-scanned).

| Op | Hex | Gas | Notes |
|---|---|---|---|
| `JUMP` | `0x60` | 8 | |
| `JUMPI` | `0x61` | 10 | |
| `PC` | `0x62` | 2 | Push current pc |
| `JUMPDEST` | `0x63` | 1 | Legal jump target; no-op otherwise |

### Environment

| Op | Hex | Gas | Notes |
|---|---|---|---|
| `ADDRESS` | `0x70` | 2 | This contract's address |
| `CALLER` | `0x71` | 2 | Immediate caller (msg.sender) |
| `ORIGIN` | `0x72` | 2 | Tx originator (tx.origin) |
| `CALLVALUE` | `0x73` | 2 | WAVE attached to this call |
| `CALLDATASIZE` | `0x74` | 2 | Length of calldata in bytes |
| `CALLDATALOAD` | `0x75` | 3 | Read 32 bytes from calldata at offset; zero-pad on right |
| `BALANCE` | `0x76` | 400 | Pop address; push balance |
| `BLOCKHEIGHT` | `0x77` | 2 | |
| `BLOCKHASH` | `0x78` | 20 | |
| `TIMESTAMP` | `0x79` | 2 | |

### Cross-contract

Stack order (top → bottom): each row shows what gets popped, top
first. See [Cross-contract calls](language/cross-contract.md) for the
Fourier wrappers.

| Op | Hex | Gas | Stack | Notes |
|---|---|---|---|---|
| `CALL` | `0x80` | 700 + forwarded | `gas, addr, value, in_off, in_len, out_off, out_len` | Push `1`/`0` for success |
| `RETURN` | `0x81` | 0 | `offset, length` | Halt successfully, return `mem[offset:offset+length]` |
| `REVERT` | `0x82` | 0 | `offset, length` | Halt unsuccessfully, all state changes undone, return data preserved |
| `DELEGATECALL` | `0x83` | 700 + forwarded | `gas, addr, in_off, in_len, out_off, out_len` | Execute target's code with caller's identity/storage/value |
| `STATICCALL` | `0x84` | 700 + forwarded | `gas, addr, in_off, in_len, out_off, out_len` | State-mutation forbidden in callee |

Gas forwarding cap: `min(requested, available - available/64)` —
the "1/64th rule." Caller always retains 1/64 of remaining gas to
recover.

### Logging

| Op | Hex | Gas | Adder | Notes |
|---|---|---|---|---|
| `LOG` | `0x90` | 375 | +375 per topic, +8 per data byte | Immediate byte: number of topics (0..4). Stack: `offset, length, topic_0, ..., topic_{n-1}`. Forbidden in STATICCALL frames |

## Precompiles

Addresses `0x00..00<01..FF>` are reserved. When `CALL` or
`DELEGATECALL` or `STATICCALL` targets one of these, the VM
short-circuits the code lookup and runs the native routine.

| Address | Gas | Adder | Algorithm | Output |
|---|---|---|---|---|
| `0x01` | 60 | +12 per input word | SHA3-512 | 64-byte hash |
| `0x02` | 30,000 | — | ML-DSA-87 verify (FIPS 204) | 32-byte word: `1` valid, `0` invalid |
| `0x03` | 50,000 | — | SLH-DSA-SHA2-128s verify (FIPS 205) | 32-byte word: `1` valid, `0` invalid |

Source: `vm/precompiles.py`.

### Verify-precompile calldata format

ML-DSA-87 and SLH-DSA share a common entry layout (`_verify_helper`):

```text
bytes 0..3    : pk_len   (uint32 BE)
bytes 4..7    : msg_len  (uint32 BE)
bytes 8..11   : sig_len  (uint32 BE)
bytes 12..    : pk || msg || sig
```

Return: 32 bytes, `0x00..01` for valid, `0x00..00` for invalid.

If the calldata is malformed (truncated headers or inconsistent
lengths), the precompile returns `0x00..00` (invalid) but does **not**
revert — that way contracts get a clean "no" answer rather than
having to handle a revert.

Note on SLH-DSA: requires `pqcrypto.sign.sphincs_sha2_128s_simple` to
be installed on the node. If not available, the precompile returns
`0` (reject all signatures) — fail-safe default.

## Frame model

Each call (top-level or sub-call) gets a fresh `_Frame`:

```python
@dataclass
class _Frame:
    code: bytes
    address: bytes             # the contract being executed
    storage_address: bytes     # where SSTORE/SLOAD read/write
    caller: bytes              # msg.sender for this frame
    call_value: int            # WAVE forwarded
    calldata: bytes
    gas: int
    pc: int = 0
    stack: List[int]
    memory: bytearray
    logs: List[LogEntry]
    return_data: bytes
    depth: int
    static: bool               # if True, SSTORE/LOG/CALL-with-value forbidden
```

Memory and stack are **per-frame** — not shared with parent. The
parent's `out_off..out_off+out_len` memory region is written with the
sub-call's return data on success.

`storage_address` differs from `address` only in `DELEGATECALL` frames:
the code being executed is the callee's, but storage I/O hits the
caller's storage.

## State model

`WorldState` (`vm/state.py`) holds all account state:

```python
class Account:
    balance: int = 0                   # WAVE; integer (no decimals)
    nonce: int = 0                     # only used by deployer for deterministic addr
    code: bytes = b""                  # contract bytecode (empty for EOAs)
    storage: Dict[int, int] = {}       # key → value
```

Writing 0 to a storage slot **deletes** the entry; subsequent SLOAD
returns 0 (the natural-zero default). This is why uninitialized slots
read as 0 — there's no separate "unset" sentinel.

## Snapshot / revert

Every top-level call (and every sub-call) takes a deep-copy snapshot
of the WorldState. On `REVERT` or `VMError`, the snapshot is restored:

```python
try:
    result = self._exec(frame)
    ...
except Revert as r:
    self.world.restore(snapshot)        # rollback
    return ExecutionResult(success=False, ..., return_data=r.return_data)
except VMError as e:
    self.world.restore(snapshot)
    return ExecutionResult(success=False, gas_used=gas_limit, ...)  # consume all
```

The deep-copy is expensive but conceptually simple. There's no MPT,
no journaled changes — just snapshot/restore on Python dicts.

## Deploy

Contracts are deployed via `vm/deploy.py::deploy`:

```python
def deploy(world, deployer, code) -> bytes:
    addr = contract_address(deployer, deployer.nonce)
    if world.get_code(addr):
        raise DeploymentCollision(...)
    deployer.nonce += 1
    world.set_code(addr, code)
    return addr
```

The deploy address is **deterministic**:

```python
def contract_address(deployer, nonce) -> bytes:
    h = sha3_256(deployer + nonce.to_bytes(32, "big")).digest()
    return h[:20]
```

(`vm/deploy.py::contract_address`)

The first 20 bytes of `SHA3-256(deployer || nonce)`. Two nodes
processing the same tx independently derive the same address.

After bytecode is stored, the deploy tx's caller-side path invokes the
new contract with empty calldata, which triggers the init prologue:

1. The init-flag at slot `2**256 - 1` is read; if non-zero, init has
   already run and the contract jumps past the init body.
2. Otherwise, the flag is set to 1 and the init body runs.
3. The empty-calldata short-circuit halts the contract before the
   dispatcher tries to read a selector.

This makes init atomic with deploy — there's no race window between
"contract exists" and "contract initialized."

## Execution loop summary

The VM main loop (`_exec` in `vm/machine.py`) is a flat `while True:`
that decodes the next opcode, charges base gas, and dispatches. Each
opcode either:

- Mutates `frame.stack` / `frame.memory` and falls through.
- Issues a sub-call (which recursively `_exec`s a new frame).
- Halts the frame (`STOP`, `RETURN`, `REVERT`) or raises a `VMError`.

There is no JIT, no superinstructions, no opcode fusion. The code is
straightforward to audit; speed is a secondary concern in v1.
