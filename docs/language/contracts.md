# Contracts

A Fourier source file holds exactly one `contract` block.

## Syntax

```fourier
contract Name {
    // any mix, any order:
    storage  NAME: TYPE @ SLOT;
    event    NAME(field: TYPE, ...);
    struct   NAME { field: TYPE; ... }
    fn       NAME(...) { ... }              // private; not callable externally
    pub fn   NAME(...) -> TYPE { ... }      // assigned a 1-byte selector
}
```

The parser entry point is `parse()` in `fourier/parser.py`; it requires
the top-level `CONTRACT` token followed by an identifier and a single
brace-delimited body.

## What goes inside

| Item | Multiplicity | Notes |
|---|---|---|
| `storage` decl | 0+ | Each must pin a unique slot via `@ N` |
| `event` decl | 0+ | Topic = `SHA3-256("Name(t1,t2,...)")` |
| `struct` decl | 0+ | Used for typed storage layouts |
| `fn` decl | 0+ | Private helper functions are not callable externally — there's no internal-call mechanism in v1, so private fns are mostly used to define `init` |
| `pub fn` decl | 0+ | Each gets a selector starting at `0x01` |

## The dispatcher

Every compiled contract has a fixed dispatcher prologue at the top of
its bytecode (see `_ContractGen.emit` in `fourier/codegen.py`):

```text
; --- init guard (only emitted if `fn init` exists) ---
PUSH (2**256 - 1)   ; INIT_FLAG_SLOT
SLOAD
PUSHLABEL _after_init
JUMPI                ; already initialized → skip init
PUSH 1
PUSH (2**256 - 1)
SSTORE               ; mark initialized
<init body inlined>
_after_init:

; --- deploy-time empty-calldata short circuit ---
CALLDATASIZE
ISZERO
PUSHLABEL _deploy_stop
JUMPI                ; if no calldata, halt successfully

; --- selector dispatch ---
PUSH 0
CALLDATALOAD         ; read first 32 bytes of calldata
PUSH (1 << 248)
SWAP 1
DIV                  ; divide by 2**248 == take top byte
; for each pub fn:
DUP 1
PUSH <selector>
EQ
PUSHLABEL _fn_<name>
JUMPI
; no match → revert
PUSH 0
PUSH 0
REVERT

_deploy_stop:
STOP
```

Each `pub fn` body sits after the dispatcher, prefixed by its
`_fn_<name>:` label and a `POP` that discards the selector left on the
stack.

## init

```fourier
fn init() {
    // ... runs exactly once ...
}
```

Rules enforced by `_ContractGen.__init__`:

- `init` must **not** be declared `pub` — it runs automatically, never
  by selector.
- `init` must take **no parameters**.
- `init` must **not** return a value.
- `return` is not allowed inside `init` (init must fall through to the
  dispatcher).

The one-shot flag lives at storage slot `2**256 - 1`
(`INIT_FLAG_SLOT`). Declaring a `storage NAME: ... @ <2**256-1>;` is a
compile error.

Init runs atomically inside the deploy transaction: the VM
[deploy](../vm.md) path immediately invokes the freshly stored code
with empty calldata, the init prologue marks the flag and runs the
init body, then the empty-calldata short-circuit halts before falling
through to the dispatcher (which would otherwise revert on no
selector). After this single invocation, `init` will never run again.

### Idiom for capturing the deployer

Because `init` cannot take params, capture the deployer via `caller()`:

```fourier
contract Ownable {
    storage owner: address @ 0;

    fn init() {
        owner = caller();
    }
}
```

This works because the deploy transaction triggers init in the same
frame, so `caller()` is the deployer EOA.

## A complete minimum contract

```fourier
contract Counter {
    storage value: uint @ 0;

    fn init() {
        value = 42;
    }

    pub fn get() -> uint {
        return value;
    }

    pub fn inc() {
        value = value + 1;
    }
}
```

Source: `fourier/examples/counter.fou`. See the
[counter walkthrough](../examples/counter.md) for a line-by-line trace.
