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
| `fn` decl | 0+ | Private helper functions are not externally callable. V1 has no internal-call mechanism; private `fn`s are used primarily to define `init` |
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
fn init(p1: T1, p2: T2, ...) {
    // ... runs exactly once ...
}
```

Rules enforced by `_ContractGen.__init__`:

- `init` must **not** be declared `pub` — it runs automatically, never
  by selector.
- `init` **may** take typed parameters. They are unpacked from the
  deploy transaction's `init_calldata` payload.
- `init` must **not** return a value.
- `return` is not allowed inside `init` (init must fall through to the
  dispatcher when paramless, or jump to `_deploy_stop` when it has
  params — neither path lets the body emit an explicit return).

The one-shot flag lives at storage slot `2**256 - 1`
(`INIT_FLAG_SLOT`). Declaring a `storage NAME: ... @ <2**256-1>;` is a
compile error.

### Calldata layout

Unlike `pub fn` calls, init invocations have **no leading selector
byte**. Init parameters are packed as 32-byte words starting at
calldata offset 0:

```text
calldata[ 0 .. 32]  = init_param[0]
calldata[32 .. 64]  = init_param[1]
calldata[64 .. 96]  = init_param[2]
...
```

The deploy transaction supplies this payload as the `init_calldata`
field on the deploy tx data dict; the contract engine forwards it as
the VM's calldata for the deploy-time invocation.

### Atomicity + the deploy_stop short-circuit

Init runs atomically inside the deploy transaction. The VM
[deploy](../vm.md) path immediately invokes the freshly stored code
with `init_calldata`.

- **Paramless init**: deploy supplies empty calldata. The init prologue
  marks the flag, runs the init body, then falls through to the
  dispatcher — which sees `CALLDATASIZE == 0` and jumps to
  `_deploy_stop`, halting cleanly.
- **Init with params**: deploy supplies the packed params as
  calldata. The init prologue marks the flag, the body unpacks the
  params from offsets `0, 32, 64, ...`, then codegen emits an
  unconditional `JUMP _deploy_stop` — so the dispatcher never gets a
  chance to misread param bytes as a selector. After the first
  invocation, `init` will never run again.

### Idiom for capturing the deployer

The deployer may be captured via `caller()` (which works whether or
not init takes parameters) or via an explicit parameter:

```fourier
contract Ownable {
    storage owner: address @ 0;

    fn init() {
        owner = caller();
    }
}
```

Or with an explicit parameter, which gives the deployer more control
over what enters storage:

```fourier
contract Vault {
    storage owner: address @ 0;
    storage cap: uint @ 1;

    fn init(initial_owner: address, initial_cap: uint) {
        owner = initial_owner;
        cap = initial_cap;
    }
}
```

The first form works because the deploy transaction triggers init
within the same frame, so `caller()` is the deployer EOA. The second
form is clearer when initial state originates somewhere other than
the deployer EOA (a factory contract, a multisig).

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
[counter example](../examples/counter.md) for a line-by-line trace.
