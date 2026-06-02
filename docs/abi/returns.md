# Return values

A function's return data is whatever bytes the function writes to
memory before issuing `RETURN`. The return-data buffer is then made
available to the caller via the transaction receipt (for top-level
calls) or via memory at `RETURN_AT (0x40)` (for sub-calls).

## Per-return-type encoding

| Fourier return type | Bytes | Encoding |
|---|---|---|
| (void) | 0 | `RETURN 0, 0` — empty return |
| `uint` | 32 | Big-endian unsigned integer, left-zero-padded |
| `address` | 32 | Address right-aligned in low 20 bytes |
| `bool` | 32 | `0x00..01` or `0x00..00` |
| `bytes` | variable | `[length:32][data:length]` (length-prefixed) |
| Tuple `(T1, T2, ..., Tk)` | `32 * k` | Each element a 32-byte word, concatenated |

## Bytecode for each shape

### Void

```text
PUSH 0
PUSH 0
RETURN
```

Stack push order: `(offset=0, length=0)` to satisfy RETURN's pop order
(offset on top).

### Scalar (uint / address / bool)

```text
<compute value>
PUSH 0x40       ; RETURN_AT
MSTORE
PUSH 32
PUSH 0x40
RETURN
```

The value is written to memory at offset `0x40` and `RETURN` reads 32
bytes from that offset.

### Tuple

For `return (a, b, c)`:

```text
<compute a>
PUSH 0x40
MSTORE
<compute b>
PUSH 0x60
MSTORE
<compute c>
PUSH 0x80
MSTORE
PUSH 96
PUSH 0x40
RETURN
```

The tuple is laid out contiguously starting at `RETURN_AT`. Total
length = `32 * tuple_arity`.

Tuple return is the only way to get more than 32 bytes back from a
function whose return type isn't `bytes`.

### `bytes`

A `bytes` return value carries a length prefix. The function computes
a memory pointer `ptr`; the `bytes` "value" at `ptr` is:

```text
mem[ptr      .. ptr+32 ] = length (uint, big-endian)
mem[ptr+32   .. ptr+32+length] = data
```

The return opcode writes the entire region:

```text
<compute ptr>
DUP 1
MLOAD                ; length
SWAP 1
PUSH 32
ADD                  ; data offset = ptr + 32
RETURN
```

(For variable-length returns, the `return EXPR;` codegen path includes
a length-aware variant; see `_emit_stmt` for `Return` in
`fourier/codegen.py`.)

## Reading a return value off-chain

For top-level calls (a contract call tx), the return data lands in the
transaction receipt:

```json
{
  "tx_id": "<hex>",
  "status": "ok",
  "gas_used": 21342,
  "return_data": "<hex string of the bytes returned>"
}
```

(See
[https://docs.fermi.world/reference/receipt/](https://docs.fermi.world/reference/receipt/)
for receipt fields.)

To decode:

- Single `uint`: `int(return_data, 16)`.
- Single `address`: `return_data[-40:]` (the low 20 bytes hex).
- Single `bool`: `return_data == "00...01"`.
- Tuple `(uint, address)`: split into 32-byte chunks; decode each per
  its type.
- `bytes`: read the first 32-byte word as length `L`, then the next
  `L` bytes are the payload.

## Reading a return value inside Fourier

When a sub-call returns, the VM copies up to `out_len` bytes of the
callee's return data to `out_off` in the caller's memory. The
Fourier codegen for `call_b` / `delegatecall_b` / `staticcall_b` sets
`out_off = RETURN_AT (0x40)` and `out_len = 32`. **Only the first 32
bytes** of the callee's return are available — sufficient for a
single `uint`, `address`, or `bool` result.

Fourier v1 does not expose `MLOAD` directly, so reading the callee's
return word from Fourier source requires a workaround. Common patterns:

1. **Treat the call-success word as the only signal.** Have the
   callee revert on failure; if the call returns `1`, the callee
   succeeded and the return value is unnecessary.

2. **Wrap the call in a hand-rolled assembly snippet.** Drop to
   bytecode (outside the Fourier source surface) when the callee's
   return word is required on-chain.

3. **Parse the receipt off-chain.** Return-data-on-call patterns are
   most useful when the consumer lives off-chain.

## Examples

### Scalar return

```fourier
pub fn get_owner() -> address {
    return owner;
}
```

Call receipt:

```json
{ "return_data": "0000000000000000000000001234567890abcdef1234567890abcdef12341234" }
```

### Bool return

```fourier
pub fn is_owner(addr: address) -> bool {
    return addr == owner;
}
```

Call receipt:

```json
{ "return_data": "0000000000000000000000000000000000000000000000000000000000000001" }
```

### Tuple return

```fourier
pub fn split_proposal(id: uint) -> (address, uint) {
    return (p_target[id], p_value[id]);
}
```

Call receipt — 64 hex bytes (128 hex chars):

```text
0000000000000000000000001234567890abcdef1234567890abcdef12341234   // target
00000000000000000000000000000000000000000000000000000000000003e8   // value (1000)
```
