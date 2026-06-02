# Errors and reverts

Fourier has one error mechanism: `REVERT`. There are no try/catch
constructs. A reverting sub-call returns `0` from `call_b` /
`delegatecall_b` / `staticcall_b`, and the parent contract decides
whether to propagate.

## `require`

```fourier
require(cond);
require(cond, msg_bytes);     // optional bytes payload
```

`cond` is any expression; the assertion passes when it produces a
non-zero value. `msg_bytes` is a `bytes` value, typically constructed
with `pack_sel(...)` or returned by a helper.

Bytecode:

```text
<cond>
PUSHLABEL ok
JUMPI
; --- no msg ---
PUSH 0
PUSH 0
REVERT
; --- with msg ---
<msg_ptr>
DUP 1
MLOAD              ; length at ptr
SWAP 1
PUSH 32
ADD                 ; data offset = ptr + 32
REVERT
ok:
```

## REVERT semantics

The `REVERT` opcode in the VM (`vm/machine.py`):

```python
if op == Op.REVERT:
    offset, length = self._spop(f), self._spop(f)
    self._mem_expand(f, offset, length)
    raise Revert(bytes(f.memory[offset:offset + length]))
```

A `Revert` propagates up to the call boundary that caught a frame
snapshot:

```python
except Revert as r:
    self.world.restore(snapshot)
    return ExecutionResult(
        success=False,
        gas_used=...,            # whatever was consumed so far
        return_data=r.return_data,
        error="REVERT",
    )
```

What this means:

- **All state changes inside the reverted frame are rolled back.**
  Storage writes, balance transfers, and emitted logs are dropped.
- The frame's **gas consumed up to the REVERT is still spent.** The
  remaining gas is refunded to the caller, but the work done before
  the REVERT is not.
- The frame's **return data is preserved.** The caller can inspect it
  via `MLOAD(RETURN_AT)` after a `call_b` that returned 0.

## Hard errors vs explicit reverts

Two failure modes are distinct (`vm/errors.py`):

| Error type | Source | State rollback | Gas refund |
|---|---|---|---|
| `Revert` | `REVERT` opcode (explicit, from `require` or hand-written) | Yes | Yes (unused gas refunded) |
| `VMError` and subclasses (`OutOfGas`, `StackOverflow`, `StackUnderflow`, `InvalidJump`, `InvalidOpcode`, `CallDepthExceeded`) | Bytecode hit an illegal state | Yes | **No** — all gas is consumed |

A sub-call hitting either failure path:

- Sets parent's stack-top to `0` (the success word from `CALL`).
- Restores the world state to the pre-call snapshot.
- For `Revert`: parent can read the revert payload via memory at
  `RETURN_AT`.
- For `VMError`: parent gets empty return data, no refund.

## Sub-call return-value pattern

```fourier
let ok: uint = call_b(target, cd, 0, 50000);
require(ok == 1);
```

If `target` reverted, `ok == 0`, `require` reverts the parent
contract as well, and the whole tx unwinds. To **handle** the
sub-call failure (for example, log it and continue), omit the
`require`:

```fourier
let ok: uint = call_b(target, cd, 0, 50000);
if ok == 0 {
    emit CallFailed(target);
    // continue without reverting
}
```

## Inside a STATICCALL frame

`STATICCALL` (or `staticcall_b`) executes the target's code with state
mutation forbidden. Inside such a frame:

- `SSTORE` raises `VMError("SSTORE forbidden in STATICCALL frame")`.
- `LOG` raises `VMError("LOG forbidden in STATICCALL frame")`.
- A nested `CALL` with `value > 0` raises
  `VMError("CALL with value forbidden in STATICCALL frame")`.

These are hard errors — the caller sees `0` from `staticcall_b` and
loses all the gas spent in the static call.

## Dispatcher revert

If the calldata's first byte does not match any `pub fn` selector, the
dispatcher emits an unconditional `PUSH 0 / PUSH 0 / REVERT`. The
calling tx fails with empty return data.

## No errors-as-types

Fourier has no `Result`, no `Option`, and no error-typed return.
Failure modes are communicated either through a `bool` / `uint`
return value checked by the caller, or through a hard revert that
aborts the tx. The convention in the [stdlib](../stdlib/index.md) is
to return `1` for success and revert on any precondition failure.

## Revert payload conventions

The `bytes` payload to `require(cond, msg)` is opaque. V1 defines no
string-encoding convention. Common patterns:

- Encode an error code as a single 32-byte word:

  ```fourier
  let err: bytes = pack_sel(1);    // selector 1 = "insufficient balance"
  require(bal >= amount, err);
  ```

- Build a tagged bytes value with `pack_sel(error_code, context_word)`.

Off-chain consumers receive the payload in the transaction receipt's
`return_data` field and decide how to interpret it.
