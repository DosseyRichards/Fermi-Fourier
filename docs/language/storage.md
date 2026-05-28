# Storage

Persistent state on a contract is declared with the `storage` keyword.
Every declaration must pin a slot:

```fourier
storage NAME: TYPE @ SLOT;
```

`SLOT` is a literal `uint` in `[0, 2**256)`. The compiler tracks slot
usage; reusing one in the same contract is a compile error.

## Permitted types

```fourier
storage scalar:  uint @ 0;
storage owner:   address @ 1;
storage paused:  bool @ 2;
storage by_addr: map[address, uint] @ 3;
storage table:   map[uint, map[address, uint]] @ 4;     // nested mapping
storage queue:   array[uint] @ 5;
storage cfg:     Config @ 6;                              // struct (see below)
```

`bytes` is not a permitted storage type in v1 — it only appears as a
function return / parameter / scratch value in memory.

## Reserved slot

| Slot | Use |
|---|---|
| `2**256 - 1` | Init-flag (set once by `init`); reserved |

Pinning a `storage` to slot `2**256 - 1` raises a `CompileError` from
`_ContractGen.__init__` (`fourier/codegen.py`):

```
storage slot ... collides with reserved init-flag slot (2**256 - 1)
```

## Slot derivation for compound types

### Scalar

```text
slot(scalar) == declared @ slot
```

### Mapping

```text
slot(m[k]) == SHA3-256(key_word || slot_word)
```

Both `key_word` and `slot_word` are 32 bytes, big-endian. Emitted by
`_emit_nested_map_slot`:

```text
MSTORE key   at offset 0
MSTORE slot  at offset 32
SHA3 over 64 bytes → derived slot
```

### Nested mapping

For `m: map[K1, map[K2, V]] @ S` and access `m[k1][k2]`:

```text
slot_1 = SHA3-256(k1 || S)
slot_2 = SHA3-256(k2 || slot_1)
```

This is iterative — each key collapses the running slot value with the
next one.

### Array

`array[T] @ S` stores `length` at slot `S`, elements at slots
`SHA3-256(S_word) + i`:

```text
slot(len)   == S
slot(arr[i]) == SHA3-256(pad32(S)) + i
```

Emitted by `_emit_array_slot`. Note that overflow risk is real for
large `i` — collisions with unrelated slots cannot occur, but you can
exceed `2**256` modulo. With realistic sizes this is not a concern.

### Struct

```fourier
struct Config {
    fee: uint;
    owner: address;
    paused: bool;
}

storage cfg: Config @ 6;
```

Field `i` (0-indexed) sits at slot `6 + i`. The compiler reserves
slots `6..(6+n_fields-1)` and reports a collision if any are already
used.

Field access is straight scalar:

```fourier
let f: uint = cfg.fee;             // SLOAD slot 6
cfg.paused = true;                  // SSTORE slot 8
```

## Collision risks

Two scenarios where you can corrupt state by accident:

1. **Pinning two storage decls to the same slot** — caught at compile
   time (`storage slot N already used by '<name>'`).

2. **Pinning a mapping/array slot equal to a struct's interior slot** —
   e.g. `struct Config @ 0` (3 fields, uses 0,1,2) and a separate
   `storage other: uint @ 1`. Caught at compile time.

3. **Picking a scalar slot that happens to equal a runtime mapping key**
   — not detectable statically. Mappings derive their keys via SHA3, so
   the probability is negligible (2^-256), but pinning a scalar at a
   slot like `42` is safe forever — no map can ever hash to that slot.

## Zero-value clearing

The VM removes entries from the underlying dict when an SSTORE writes
0 (see `WorldState.storage_set` in `vm/state.py`). That means writing
0 to a never-written slot is a no-op, and reading from an unset slot
returns 0 — there is no "unset" sentinel distinct from 0.

## Gas

| Op | Gas |
|---|---|
| SLOAD | 200 |
| SSTORE (existing → existing) | 5000 |
| SSTORE (zero → non-zero) | 20000 |
| SSTORE (any → zero) | 5000 (no refund) |

WaveLedger does not implement gas refunds for slot clearing. See
`SSTORE_SET_GAS`, `SSTORE_RESET_GAS`, `SSTORE_CLEAR_REFUND` in
`vm/opcodes.py`.

## Example: ERC-20-style layout

```fourier
contract Token {
    storage total_supply: uint @ 0;
    storage balances:     map[address, uint] @ 1;
    storage allowances:   map[address, map[address, uint]] @ 2;
}
```

- `total_supply` lives at slot 0.
- `balances[addr]` lives at `SHA3(addr || 1)`.
- `allowances[owner][spender]` lives at
  `SHA3(spender || SHA3(owner || 2))`.
