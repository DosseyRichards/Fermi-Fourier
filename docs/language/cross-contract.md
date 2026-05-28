# Cross-contract calls

Three call kinds are exposed from Fourier — `call_b`, `delegatecall_b`,
`staticcall_b` — plus a legacy one-word `call`. All return a `uint`:
`1` for success, `0` for revert / VM error.

## Calldata is `bytes`

Calldata for a cross-contract call is a `bytes` value built with
`pack_sel`:

```fourier
let cd: bytes = pack_sel(target_selector, arg1, arg2);
let ok: uint = call_b(target_addr, cd, /* value */ 0, /* gas */ 50000);
```

`pack_sel(sel, a1, ..., aN)` allocates `[length:32][sel:32][a1:32]...[aN:32]`
in scratch memory and returns the pointer. The total `bytes` length is
`1 + N * 32` — one selector byte plus 32 bytes per argument. When the
callee's dispatcher reads `CALLDATALOAD 0`, the high byte is the
selector; subsequent params come from offsets `1, 33, 65, ...`.

## `call_b`

```fourier
call_b(addr: address, calldata: bytes, value: uint, gas: uint) -> uint
```

Standard CALL semantics:

- New frame with `address = addr`, `caller = self`.
- Callee operates on **its own** storage.
- `value` WAVE transferred from caller to callee before code runs.
- Sub-call gets its own state snapshot; revert rolls back independently.

VM opcode: `CALL` (0x80). Stack order (`vm/machine.py`):

```text
gas, addr, value, in_off, in_len, out_off, out_len
```

The codegen path stashes `in_len` and `in_off + 32` in scratch slots,
then pushes the seven words in the order CALL pops them.

## `delegatecall_b`

```fourier
delegatecall_b(addr: address, calldata: bytes, gas: uint) -> uint
```

DELEGATECALL semantics — execute target's code in **caller's**
context:

- `address` stays as **self** (not target).
- Storage writes hit **self's** storage slots.
- `caller` and `call_value` carry over from the calling frame.
- No `value` parameter (forwarded as-is).

Use cases: proxy contracts, library-style callable code where the
library should mutate the caller's state directly.

VM opcode: `DELEGATECALL` (0x83).

## `staticcall_b`

```fourier
staticcall_b(addr: address, calldata: bytes, gas: uint) -> uint
```

Read-only call — any state-mutating op inside the callee reverts:

- `SSTORE` → VMError (no rollback refund).
- `LOG` → VMError.
- Nested `CALL` with `value > 0` → VMError.

VM opcode: `STATICCALL` (0x84).

## Return data

Whichever call kind you use, the callee's return data is copied into
**32 bytes** at `RETURN_AT (0x40)` of the caller's memory. To read it
from Fourier, you'd need a `bytes` deref, which isn't a first-class
op in v1 — instead, the convention is for callees to return a single
`uint` (or `bool` or `address`) and for callers to inspect via a
follow-up MLOAD that the language doesn't expose.

In practice, **the success word is the only result Fourier-level code
typically reads.** If you need the callee's return value, write a
small wrapper or use the call result as a precondition only:

```fourier
let ok: uint = call_b(target, cd, 0, 50000);
require(ok == 1);
// callee's return word sits at memory 0x40 but Fourier has no MLOAD primitive
```

## `lib_call` — library DELEGATECALL sugar

```fourier
lib_call(lib_addr: address, selector: uint, arg1: T1, ..., argN: TN, gas: uint) -> uint
```

The high-level form of "call a library via DELEGATECALL". One
builtin captures the four-line `pack_sel` + `delegatecall_b` +
`require(ok)` + `MLOAD(RETURN_AT)` pattern that you would otherwise
spell out by hand:

```fourier
let result: uint = lib_call(lib_addr, 0x01, a, b, 100_000);
```

is equivalent to:

```fourier
let cd: bytes = pack_sel(0x01, a, b);
let ok: uint  = delegatecall_b(lib_addr, cd, 100_000);
require(ok == 1);
let result: uint = /* the 32-byte word at RETURN_AT — Fourier has no MLOAD primitive */;
```

with the important difference that `lib_call` actually surfaces the
return word — there is no plain Fourier syntax to do this with the
lower-level builtins.

Argument ordering by position:

| Position | Meaning |
|---|---|
| 1 | Library contract address |
| 2 | Method selector (1-byte literal, usually) |
| 3..N-1 | Method arguments |
| N (last) | Gas limit for the sub-call |

Failure semantics:

- If the delegatecall reverts (or any sub-call error), `lib_call`
  reverts the **caller** with no return data. There is no success
  word to inspect.
- The library executes against the **caller's** storage (standard
  DELEGATECALL behavior). Writes the library makes hit the caller's
  slots, not the library's.

### Example: math library

```fourier
contract Math {
    pub fn add(a: uint, b: uint) -> uint { return a + b; }
    pub fn mul(a: uint, b: uint) -> uint { return a * b; }
}

contract MyContract {
    storage math_lib: address @ 0;

    pub fn init(lib_addr: address) {
        math_lib = lib_addr;
    }

    pub fn double_sum(a: uint, b: uint) -> uint {
        let s: uint = lib_call(math_lib, 0x01, a, b, 50_000);   // add
        return lib_call(math_lib, 0x02, s, 2, 50_000);           // mul by 2
    }
}
```

Selectors `0x01` / `0x02` follow the standard
[selector assignment](functions.md#selector-assignment) rule: first
`pub fn` in declaration order gets `0x01`.

## One-word `call`

```fourier
call(addr: address, calldata_word: uint, value: uint, gas: uint) -> uint
```

The legacy form. `calldata_word` is a 32-byte word that becomes the
entire calldata — first byte is the selector, remaining 31 are
ignored by the callee's dispatcher unless they comprise a single
argument. Use this when the callee takes zero or one arg:

```fourier
let ok: uint = call(target, 0x01_00..._00, 0, 50000);    // selector 0x01, no args
```

For multi-arg calls, use `pack_sel` + `call_b`.

## Gas forwarding

The VM caps forwarded gas at **63/64 of remaining** (EVM "1/64th
rule"; see `vm/machine.py`):

```python
avail = f.gas - f.gas // 64
fwd_gas = min(fwd_gas, avail)
```

Pass a generous gas budget; the cap ensures the caller always has
gas left to recover from a failed sub-call.

## Reentrancy

There is no built-in reentrancy guard. The
[`ReentrancyGuard` stdlib](../stdlib/reentrancy.md) shows the standard
mutex pattern:

```fourier
require(_locked == 0);
_locked = 1;
// ... external call ...
_locked = 0;
```

Set the flag **before** the external call; unset after. Any reentry
during the call hits the flag and reverts.

## Call depth

The VM enforces a maximum call depth of `MAX_CALL_DEPTH = 256`. The
257th nested call raises `CallDepthExceeded` (hard error, full gas
consumed).

## PQC precompile calls

`verify_sig(scheme_id, pk, msg, sig)` internally builds the precompile
calldata layout `[pk_len:4][msg_len:4][sig_len:4][pk][msg][sig]` and
issues a `STATICCALL` to the precompile address. The return word
(`1` valid / `0` invalid) is loaded into `RETURN_AT` and surfaced as
the builtin's expression value. See
[Expressions / PQC signature verify](expressions.md#pqc-signature-verify).

## Examples

### Forward a value transfer

```fourier
let cd: bytes = pack_sel(0);     // empty calldata (selector 0; will revert at callee
                                 // if no fn matches — useful for plain EOA transfers)
let ok: uint = call_b(target, cd, value_to_send, 50000);
require(ok == 1);
```

### Read a counter (staticcall)

```fourier
// counter.get() has selector 0x01
let cd: bytes = pack_sel(0x01);
let ok: uint = staticcall_b(counter_addr, cd, 30000);
require(ok == 1);
// Counter's return word now sits at memory 0x40 but Fourier can't read it directly in v1.
```

### Delegated upgrade

```fourier
// implementation.action(arg) has selector 0x01
let cd: bytes = pack_sel(0x01, arg);
let ok: uint = delegatecall_b(implementation_addr, cd, 100000);
require(ok == 1);
// storage at THIS contract has been mutated
```
