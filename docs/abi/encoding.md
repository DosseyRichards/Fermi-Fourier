# Argument encoding

Each argument in calldata is exactly **32 bytes**. The encoding is the
big-endian representation of the underlying 256-bit word.

## Per-type encoding

| Fourier type | Bytes | Encoding |
|---|---|---|
| `uint` | 32 | Big-endian unsigned integer, left-zero-padded to 32 bytes |
| `address` | 32 | The 20 address bytes right-aligned (occupy bytes 12–31); top 12 bytes zero |
| `bool` | 32 | `0x00...01` for true, `0x00...00` for false |
| `bytes` | (not encodable as a calldata argument) | — |

There is no padding ambiguity: every type fits in a 32-byte word and
is read via `CALLDATALOAD offset` in the function prologue.

## How the VM reads it

In `_emit_fn_body` (`fourier/codegen.py`):

```python
for idx, param in enumerate(fn.params):
    calldata_off = 1 + idx * 32                # skip selector byte
    items.extend([
        ("PUSH", calldata_off), "CALLDATALOAD",   # read 32 bytes
        ("PUSH", offset), "MSTORE",               # write into local
    ])
```

`CALLDATALOAD offset` (VM op `0x75`) reads 32 bytes from calldata at
`offset` and pushes them as a single uint. If the calldata is shorter
than `offset + 32`, the missing bytes are zero-padded on the right
(`vm/machine.py`):

```python
chunk = f.calldata[offset:offset + 32]
chunk = chunk + b"\x00" * (32 - len(chunk))
self._spush(f, int.from_bytes(chunk, "big"))
```

## Padding conventions

### `uint`

Number is encoded big-endian, padded with leading zeros to 32 bytes:

```text
1000   →   0x00000000000000000000000000000000000000000000000000000000000003e8
2**32  →   0x0000000000000000000000000000000000000000000000000000000100000000
```

### `address`

The 20-byte address sits in the low 20 bytes of the word:

```text
addr = 1234567890abcdef1234567890abcdef12341234

word = 0x0000000000000000_0000000000000000_00000000_1234567890abcdef1234567890abcdef12341234
       └── 12 zero bytes ───────────────┴──────── 20 address bytes ─────────────────────────┘
```

This matches `vm/state.py`'s 20-byte address convention. Loaded from
calldata into a Fourier `address`-typed local, the high zeros are
preserved; comparisons and storage reads operate on the full 256-bit
word.

### `bool`

```text
true  →  0x000000...0000000000000001
false →  0x000000...0000000000000000
```

The codegen emits `PUSH 1` / `PUSH 0` for `true` / `false` literals,
and comparison opcodes natively produce these forms.

## Multiple arguments

For `fn(a: uint, b: address, c: bool)`:

```text
byte 0       : selector
bytes 1..33  : a (32 bytes)
bytes 33..65 : b (32 bytes; address in low 20)
bytes 65..97 : c (32 bytes; 0 or 1)
```

Total: 97 bytes.

## Encoding a call: full example

```fourier
pub fn approve(spender: address, amount: uint) -> bool {
    // ...
}
```

Selector: depends on declaration order, say `0x05`. To call
`approve(0xa0...a0, 1000)`:

```text
05                                                                        ; selector byte
0000000000000000000000000a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a        ; spender (32 bytes; address in low 20)
00000000000000000000000000000000000000000000000000000000000003e8        ; amount (1000)
```

Concatenated, this is a 65-byte calldata blob. As hex:

```
05_0000000000000000000000000a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a_00000000000000000000000000000000000000000000000000000000000003e8
```

(without underscores).

## Not encodable as a calldata arg: `bytes`

Variable-length `bytes` does not appear as a function parameter. The
parser and codegen accept `bytes` in a parameter position only via
the cross-contract calldata path, where `pack_sel` builds the
calldata as a `bytes` value in memory. `pub fn` parameters must fit
in a single 32-byte word, so types larger than 32 bytes are not
supported at the entry-point boundary.

Two workarounds exist for variable-length input from off-chain:

1. **Pass a hash plus a separate storage upload.** Send a SHA3-256
   hash as a `uint` argument; require the off-chain caller to have
   previously stored the payload.

2. **Use multiple `uint` arguments** to pack short byte strings (up
   to 32 bytes per argument).

## Multi-arg encoding from inside Fourier

`pack_sel(sel, a, b, c)` builds the same layout in memory:

```text
[sel:32][a:32][b:32][c:32]
```

with `sel` shifted to the top byte of its word. The returned `bytes`
pointer references `[length:32]` followed by `[1 + N*32]` bytes of
data. See [`pack_sel`](../language/expressions.md#calldata-construction)
for memory layout details.
