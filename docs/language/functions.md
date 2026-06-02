# Functions

```fourier
fn      name(p1: T1, p2: T2) -> Tret { ... }        // private
pub fn  name(p1: T1, p2: T2) -> Tret { ... }        // selector-dispatched
```

The `-> Tret` return type is optional; if absent, the function is
void and `return EXPR;` is rejected with "return with value in void
function".

## `fn` vs `pub fn`

| Property | `fn` (private) | `pub fn` (public) |
|---|---|---|
| Externally callable | No | Yes |
| Internally callable | No (v1 has no internal-call op) | No |
| Selector assigned | No | Yes (`0x01`+) |
| Bytecode emitted | Only if name is `init` | Yes |
| Allowed signatures | `init(...) { ... }` — params allowed, no return | Any |

Private functions other than `init` compile to nothing in v1. With
no internal-call mechanism, they are dead code. The grammar permits
them, but the codegen path that walks `c.functions` emits bodies only
for `is_pub` functions and inlines `init`'s body in the prologue.

## Selector assignment

```python
sel = 0x01
for fn in contract.functions:
    if fn.is_pub:
        fn.selector = sel
        sel += 1
```

(`_ContractGen.__init__` in `fourier/codegen.py`)

- Selectors are assigned in **declaration order**.
- The first `pub fn` gets `0x01`. Reordering source = breaking ABI.
- `init` is **not** included; it has no selector.
- There is no `0x00` selector; it would collide with the
  empty-calldata deploy short-circuit.
- Maximum 255 `pub fn`s per contract (selectors are one byte; `0xFF`
  is the last valid).

A function's selector equals its position among `pub fn`
declarations, counting from 1.

## Parameters

Parameters are decoded from calldata at fixed offsets. The layout
depends on whether this is a regular `pub fn` call or an `init`
invocation:

```text
pub fn call:
  calldata[ 0 ..  1]  = selector
  calldata[ 1 .. 33]  = param[0]   (32-byte word)
  calldata[33 .. 65]  = param[1]
  calldata[65 .. 97]  = param[2]
  ...

init invocation (deploy tx's init_calldata):
  calldata[ 0 .. 32]  = init_param[0]
  calldata[32 .. 64]  = init_param[1]
  calldata[64 .. 96]  = init_param[2]
  ...
```

The difference: a `pub fn` call carries a 1-byte selector; the init
payload does not.

Each param is loaded into a local memory slot during the function
prologue (see `_emit_fn_body`):

```text
PUSH 1
CALLDATALOAD
PUSH 0x80           ; first param goes at 0x80
MSTORE
PUSH 33
CALLDATALOAD
PUSH 0xa0           ; second param goes at 0xa0
MSTORE
...
```

Locals start at memory offset `0x80` (`LOCAL_MEM_BASE`) and grow
upward. Each local — parameter or `let` — takes 32 bytes.

## Return types

| Return type | Bytecode emitted |
|---|---|
| (absent) | `STOP` at function tail; `return;` valid, `return EXPR;` rejected |
| `uint` / `address` / `bool` | Compute value, `MSTORE 0x40`, `RETURN 0x40, 32` |
| `bytes` | Compute pointer; emit a length-aware return (length at `[ptr]`, data at `[ptr+32]`) |
| `(T1, T2, ...)` | Compute each element, store at `0x40`, `0x60`, `0x80`, ...; `RETURN 0x40, 32*n` |

`RETURN_AT = 0x40` (`fourier/codegen.py`).

## Function locals

Each `let` allocates a fresh 32-byte slot in memory starting from the
local-memory cursor (initialized at `LOCAL_MEM_BASE = 0x80`). Locals
are referenced by their memory offset, not by a register or stack
position.

```fourier
pub fn example(a: uint) -> uint {
    let b: uint = a + 1;          // b at offset 0xa0 (after a at 0x80)
    let c: uint = b * 2;          // c at offset 0xc0
    return c;
}
```

Locals persist for the lifetime of the call frame. No scope is
narrower than the function: a `let` inside an `if` body remains
visible after the `if`, and redeclaring `let x` in the same function
body allocates a fresh slot. The prior binding is shadowed in the
symbol table, and its memory is leaked.

## Calling convention summary

| Aspect | Behavior |
|---|---|
| Param passing | Calldata, 32 bytes per parameter. `pub fn`s start at offset 1; init starts at offset 0 (no selector). |
| Return passing | Memory at `RETURN_AT` (0x40), `RETURN` opcode |
| Caller identity | `caller()` builtin returns immediate caller |
| Origin identity | `origin()` returns tx originator |
| Sub-call gas forwarding | Capped at 63/64 of remaining (EVM "1/64th rule") — see `vm/machine.py` |

## Examples

Read-only:

```fourier
pub fn balance_of(addr: address) -> uint {
    return balances[addr];
}
```

Mutating with check:

```fourier
pub fn transfer(to: address, amount: uint) -> bool {
    let sender: address = caller();
    let bal: uint = balances[sender];
    require(bal >= amount);
    balances[sender] = bal - amount;
    balances[to] = balances[to] + amount;
    emit Transfer(sender, to, amount);
    return true;
}
```

Tuple return:

```fourier
pub fn split_proposal(id: uint) -> (address, uint) {
    return (p_target[id], p_value[id]);
}
```
