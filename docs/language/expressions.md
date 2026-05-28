# Expressions

Expressions form the right-hand side of `let`, assignments, `return`,
`require`, `emit`, and the condition slots of `if` / `while`. Every
expression evaluates to a single 256-bit word.

## Literals

```fourier
0
42
1_000_000           // underscores ignored
0xCAFE_BABE
true                 // == 1
false                // == 0
```

## Variables

```fourier
NAME                 // local memory load OR storage SLOAD
NAME[KEY]            // storage mapping/array load
NAME[K1][K2]         // nested mapping load
NAME.FIELD           // storage struct field load
```

Identifier resolution at expression position:

1. Local? → `PUSH offset, MLOAD`
2. Storage scalar (not map/array)? → `PUSH slot, SLOAD`
3. Storage struct field access (`name.field`)? → `PUSH slot+i, SLOAD`
4. Storage map/array indexed (`name[k]`)? → compute derived slot, `SLOAD`
5. Bare storage map name → compile error ("mapping '...' used as scalar")

## Operators

### Precedence

Highest to lowest binding (from `_BIN_OPS` in `fourier/parser.py`):

| Level | Operators | Notes |
|---|---|---|
| 9 | `*` `/` `%` | DIV / MOD by 0 produce 0 (not a revert) |
| 8 | `+` `-` | Wraps modulo `2**256` |
| 7 | `<` `>` `<=` `>=` | Unsigned |
| 6 | `==` `!=` | |
| 5 | `&` | Bitwise AND |
| 4 | `^` | Bitwise XOR |
| 3 | `\|` | Bitwise OR |
| 2 | `&&` | Logical AND (no short-circuit) |
| 1 | `\|\|` | Logical OR (no short-circuit) |

Prefix unary (`-`, `!`, `~`) binds tighter than any binary operator.

### Short-circuit?

**Both sides are always evaluated.** `&&` and `||` collapse to bitwise
ops over normalized booleans:

```text
a && b   →   ISZERO(ISZERO(a)) AND ISZERO(ISZERO(b))
a || b   →   ISZERO(ISZERO(a)) OR  ISZERO(ISZERO(b))
```

If you need short-circuit for side-effect safety, use an `if`:

```fourier
if cond1 {
    if cond2 {
        // ...
    }
}
```

### Operator → opcode mapping

| Source | Bytecode |
|---|---|
| `+` | `ADD` |
| `-` | `SWAP 1` then `SUB` (operand order correction) |
| `*` | `MUL` |
| `/` | `SWAP 1` then `DIV` |
| `%` | `SWAP 1` then `MOD` |
| `==` | `EQ` |
| `!=` | `EQ` then `ISZERO` |
| `<` | `SWAP 1` then `LT` |
| `>` | `SWAP 1` then `GT` |
| `<=` | `SWAP 1` then `GT` then `ISZERO` |
| `>=` | `SWAP 1` then `LT` then `ISZERO` |
| `&` | `AND` |
| `\|` | `OR` |
| `^` | `XOR` |
| `&&` | normalized AND (see above) |
| `\|\|` | normalized OR |

Unary:

| Source | Bytecode |
|---|---|
| `-x` | `PUSH 0, SWAP 1, SUB` |
| `!x` | `ISZERO` |
| `~x` | `NOT` |

## Calls

Function call syntax: `NAME(ARG, ARG, ...)`. Resolved against:

1. **Builtins** (see [Builtins](#builtins) below)
2. **Otherwise** → `unknown function 'NAME/ARITY'` compile error.

There is no user-defined `fn` call mechanism in v1. Every `NAME(...)`
in an expression must be a builtin.

## Builtins

### Environment

```fourier
caller()         -> address      // CALLER
callvalue()      -> uint          // CALLVALUE
origin()         -> address       // ORIGIN
block_height()   -> uint          // BLOCKHEIGHT
timestamp()      -> uint          // TIMESTAMP
balance(addr)    -> uint          // BALANCE
```

### Crypto

```fourier
sha3(word) -> uint
```

Hashes a single 32-byte word and returns the 256-bit result. The word
is written to scratch memory before `SHA3` runs:

```text
PUSH word
PUSH 0
MSTORE
PUSH 32
PUSH 0
SHA3
```

For multi-word hashes, build a `bytes` value and use the SHA3-512
precompile (`STATICCALL 0x01`) directly or via a stdlib helper.

### Checked arithmetic

```fourier
safe_add(a, b) -> uint
safe_sub(a, b) -> uint
safe_mul(a, b) -> uint
safe_div(a, b) -> uint
```

Reverts on overflow / underflow / div-by-zero. See
[SafeMath](../stdlib/safemath.md) for semantics.

### Storage array operations

```fourier
len(arr)        -> uint
push(arr, v)    -> uint    // returns new length
pop(arr)        -> uint    // returns popped element; reverts if empty
```

First arg must be a bare storage array name.

### Calldata construction

```fourier
pack_sel(selector_word, arg1, arg2, ..., argN) -> bytes
```

Allocates a memory region holding `[length:32][selector_word:32][arg1:32]...`.
Returns the pointer (memory offset) as the `bytes` value. Layout
description in `_emit_expr` for `pack_sel`:

```text
heap[off+0 .. off+32]      = packed length (1 + N*32)
heap[off+32 .. off+64]     = selector_word << 248 (top byte = selector)
heap[off+64 .. off+96]     = arg1 (whole 32-byte word, but only first 31 of arg are used due to selector's prefix byte)
heap[off+96 .. off+128]    = arg2
...
```

Note: arguments live at offsets `33, 65, 97, ...` from the start of the
data, matching the calldata layout the callee's dispatcher expects.

### Cross-contract calls

```fourier
call_b(addr, calldata: bytes, value, gas)    -> uint   // 1 = success, 0 = fail
delegatecall_b(addr, calldata: bytes, gas)   -> uint
staticcall_b(addr, calldata: bytes, gas)     -> uint

call(addr, calldata_word, value, gas) -> uint    // one-word legacy form
```

Detail in [Cross-contract calls](cross-contract.md).

### PQC signature verify

```fourier
verify_sig(scheme_id, pk: bytes, msg: bytes, sig: bytes) -> uint
```

`scheme_id` must be a literal int (not a variable). Known schemes:

| `scheme_id` | Precompile | Algorithm |
|---|---|---|
| `1` | `0x02` | ML-DSA-87 |
| `2` | `0x03` | SLH-DSA-SHA2-128s |

Returns `1` if valid, `0` if invalid (the precompile call itself does
not revert on bad signature; the return word distinguishes).

## Parenthesized expressions

```fourier
(a + b) * c
```

Standard. Parentheses around a comma-separated list become a tuple
literal:

```fourier
(a, b, c)
```

Tuples are only legal as the RHS of a `let (...) = (...)` or in
`return (...)`. Anywhere else → compile error.

## Address-mismatch enforcement

Arithmetic and ordering ops between an `address` and a non-`address`
local raise a compile error. See
[Types / Address vs uint enforcement](types.md#address-vs-uint-enforcement).
