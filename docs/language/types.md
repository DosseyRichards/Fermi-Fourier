# Types

Fourier has four primitive types and three compound types. All values
on the VM stack are 256-bit unsigned integers; types only constrain
what operations are legal in the source.

## Primitives

| Keyword | Storage width | Notes |
|---|---|---|
| `uint` | 256 bits | The universal numeric type. All arithmetic is modulo `2**256` |
| `address` | 20 bytes | Stored as `uint` at runtime, but compiler rejects arithmetic mixing with non-address operands |
| `bool` | 1 bit logical | Encoded as `1` or `0` in a 256-bit word |
| `bytes` | variable | Length-prefixed memory region. Pointer is a `uint` (memory offset). Permitted only as a function parameter, local, or return value — **not** as a storage type |

## Compound types

### `map[K, V]`

```fourier
storage balances: map[address, uint] @ 1;
storage nested:   map[uint, map[address, uint]] @ 2;
```

Permitted as a storage declaration only — not as a function parameter,
local, or return. Mapping key derivation lives in
[Storage / Mapping](storage.md#mapping).

### `array[T]`

```fourier
storage queue: array[uint] @ 3;
```

A dynamic storage array. `len(queue)`, `push(queue, v)`, `pop(queue)`
are the only operations. See
[Statements / Array builtins](statements.md#array-builtins).

### Structs

```fourier
struct Config {
    fee: uint;
    owner: address;
    paused: bool;
}

storage cfg: Config @ 6;
```

Structs are flat record types over primitive fields. Used as storage
types only in v1. Field `i` occupies slot `base + i`.

## Tuples

```fourier
fn split() -> (uint, address) {
    return (1, caller());
}

let (n, addr): (uint, address) = (42, caller());
```

Tuples appear only as:

- Return types of a `pub fn`
- The RHS of a `let (...) = (...)`

A tuple literal in any other expression position raises `CompileError`
("tuple literal only allowed in `return (...)` or `let (...) = (...)`").
Tuple destructuring requires a literal tuple RHS in v1 — you cannot
destructure a cross-contract call's return.

## String literals

String literals — `"hello"`, `"alice"`, `""` — are syntactic sugar for
a 256-bit `uint`: the UTF-8 bytes right-padded with zeros to 32 bytes
and interpreted big-endian. Same packing as Solidity's
`bytes32("...")`.

```fourier
storage name: uint @ 0;

pub fn set() {
    name = "alice";                              // = 0x616c696365000...0
}

pub fn check(n: uint) -> bool {
    return n == "alice";                         // compare 32-byte form
}

pub fn h() -> uint {
    return sha3("hello");                        // hashes "hello\0\0...\0"
}
```

Rules:

- UTF-8 encoded source bytes.
- Max 32 bytes — longer literals are a lex error.
- No escape sequences in v1 (no `\"`, no `\\`, no `\n`).
- No newlines inside a literal.
- The empty string `""` is `0`.

Strings are typed as `uint` after parsing — there is no separate
`string` or `bytes32` type; they all collapse to `uint` in the
expression layer.

## What's NOT supported

- **No floating point.** Compute fixed-point manually with
  appropriate shift constants.
- **No signed integers.** All comparisons are unsigned; no SDIV/SMOD.
- **No fixed-size byte arrays** (`bytes32` etc). Use `uint` for any
  32-byte value; use `bytes` for variable-length.
- **No generics.** Each mapping spells out its key and value type.
- **No type aliases.** Repeat the type wherever it appears.
- **No nullable / optional** wrappers. Use a separate `bool` or rely
  on the natural-zero convention (uninitialized storage reads 0).
- **`bytes` cannot be stored.** It's a memory-only type.

## Address vs uint enforcement

The codegen runs a soft type check on binary ops. For arithmetic and
comparison ops (`+ - * / % < > <= >=`), if one operand has known type
`address` and the other is a known non-`address`, compilation fails
with:

```
type mismatch: cannot apply '<op>' between 'address' and 'uint'
(address values aren't arithmetic)
```

The check sees only locally-typed values (locals declared with a
type, params). Storage reads return `_` (unknown), so mixed comparisons
involving storage variables compile silently. See
[Address type](address.md) for the full discussion.

`==` and `!=` between `address` and other types are always allowed.

## Boolean values

`true` and `false` are reserved tokens that compile to `1` and `0`
respectively (see `_emit_expr` for `BoolLit`). There is no separate
boolean opcode; comparison opcodes already produce `0` or `1`.

## Integer literals

```fourier
let a: uint = 0;
let b: uint = 1_000_000;
let c: uint = 0xCAFE_BABE;
let d: uint = 0xff;
```

- Underscores are ignored inside numeric literals.
- Hex literals are recognized via `0x` / `0X` prefix.
- All literals are `uint`. There is no negative literal — `-1` is a
  unary minus over the literal `1`, producing `2**256 - 1`.
