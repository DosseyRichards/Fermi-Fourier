# Quick reference

Single-page summary of the language. Each section links to the full semantics.

## Types

```fourier
uint        // 256-bit unsigned integer
address     // 20-byte address; stored as uint at runtime
bool        // 0 or 1
bytes       // length-prefixed byte string (function returns only)

map[K, V]   // backed by SHA3(key_word || slot_word)
array[T]    // dynamic storage array
```

## Literals

```fourier
0  42  1_000_000  0xCAFE_BABE   // integers
true  false                      // booleans
"alice"                          // string → uint, right-padded to 32 bytes
""                               // empty string == 0
```

See [Types](language/types.md).

## Contract skeleton

```fourier
contract Name {
    // storage declarations (each pinned to a slot)
    storage value: uint @ 0;
    storage balances: map[address, uint] @ 1;

    // events
    event Updated(by: address, new_value: uint);

    // optional init — runs once on first call, no params, no return
    fn init() {
        value = 0;
    }

    // public entry points get selectors 0x01, 0x02, ... in declaration order
    pub fn set(v: uint) {
        require(caller() == 0);          // example check
        value = v;
        emit Updated(caller(), v);
    }

    pub fn get() -> uint {
        return value;
    }
}
```

See [Contracts](language/contracts.md).

## Statements

```fourier
let x: uint = 1;                         // declare local
x = x + 1;                                // assign local
value = 42;                               // assign storage scalar
balances[addr] = 100;                     // assign mapping slot

if cond { ... } else { ... }              // else optional
while cond { ... }
return EXPR;                              // or `return;` in void fn
require(cond);                            // revert if false
require(cond, msg_bytes);                 // revert with bytes payload
emit Updated(addr, val);
expr;                                     // expression statement
```

See [Statements](language/statements.md).

## Expressions

| Precedence | Operators (high → low rank shown bottom → top) |
|---|---|
| 1 | `\|\|` |
| 2 | `&&` |
| 3 | `\|` |
| 4 | `^` |
| 5 | `&` |
| 6 | `==`  `!=` |
| 7 | `<`  `>`  `<=`  `>=` |
| 8 | `+`  `-` |
| 9 | `*`  `/`  `%` |
| prefix | `-`  `!`  `~` |

Note: `&&` / `||` do **not** short-circuit — both sides are evaluated.
See [Expressions](language/expressions.md).

## Builtins

```fourier
caller()           -> address    // msg.sender
callvalue()        -> uint        // WAVE attached to this call
origin()           -> address     // tx.origin
block_height()     -> uint
timestamp()        -> uint
balance(addr)      -> uint        // address WAVE balance
sha3(word)         -> uint        // SHA3-256 of one 32-byte word

require(cond)
require(cond, msg)                // msg is a `bytes` value

// Checked arithmetic — revert on overflow / underflow / div-by-zero
safe_add(a, b)     -> uint
safe_sub(a, b)     -> uint
safe_mul(a, b)     -> uint
safe_div(a, b)     -> uint

// Q64.64 fixed-point math (scale = 2**64; 1.0 == 2**64)
from_int(n)         -> uint        // lift integer to fixed
to_int(f)           -> uint        // truncate fractional bits
fmul(a, b)          -> uint        // (a * b) / 2**64
fdiv(a, b)          -> uint        // (a * 2**64) / b

// Storage array
len(arr)           -> uint
push(arr, v)       -> uint        // returns new length
pop(arr)           -> uint        // returns popped element; reverts if empty

// Cross-contract (calldata is a `bytes` value built with pack_sel)
pack_sel(sel, arg1, ..., argN)  -> bytes
call_b(addr, calldata, value, gas)    -> uint   // 1 = success
delegatecall_b(addr, calldata, gas)   -> uint
staticcall_b(addr, calldata, gas)     -> uint

// Library-call sugar — DELEGATECALL with auto-pack + auto-revert
lib_call(lib_addr, selector, args..., gas) -> uint   // returns callee's return word

// One-word legacy call (single 32-byte calldata word)
call(addr, calldata_word, value, gas) -> uint

// PQC signature verify — scheme_id must be a literal int
verify_sig(scheme_id, pk: bytes, msg: bytes, sig: bytes) -> uint
```

See [Cross-contract](language/cross-contract.md) and
[Errors](language/errors.md).

## Crypto scheme IDs

| `scheme_id` | Precompile addr | Algorithm |
|---|---|---|
| `1` | `0x02` | ML-DSA-87 (FIPS 204) |
| `2` | `0x03` | SLH-DSA-SHA2-128s (FIPS 205) |

Defined in `fourier/codegen.py::CRYPTO_SCHEMES` and
`vm/precompiles.py`.

## Storage slot rules

- Each `storage` decl pins a slot with `@ N`.
- Slot `2**256 - 1` is reserved for the init flag. Pinning it raises a
  compile error.
- Mapping key: `SHA3-256(key_word || slot_word)` (each 32 bytes,
  concatenated, hashed).
- Nested mapping key: hash iteratively — `slot_i = SHA3(k_i || slot_{i-1})`.
- Dynamic array: length at slot `S`; element `i` at slot
  `SHA3(S_word) + i`.
- Struct field `i` at slot `S + i`. The compiler reserves
  `slot..slot+n_fields-1`.

See [Storage](language/storage.md).
