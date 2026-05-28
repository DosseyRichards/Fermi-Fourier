# Counter

Source: `fourier/examples/counter.fou`.

The minimum interesting Fourier contract — one slot of storage, an
init, a read, a write.

## Full source

```fourier
// Minimal counter — demonstrates init + a single mutating method.

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

## Walkthrough

### `storage value: uint @ 0;`

One storage slot, pinned to slot 0. Holds a single 256-bit unsigned
integer. After init, this slot reads `42`; after each `inc()` call,
the value at slot 0 increases by 1.

The pin to slot 0 is arbitrary — any slot from 0 to `2**256 - 2` would
work. Slot `2**256 - 1` is reserved for the init guard.

### `fn init() { value = 42; }`

The optional initializer. Runs exactly once, atomically inside the
deploy transaction. After this runs, the contract reads `value == 42`
regardless of who calls `get()` first.

`init` is private (no `pub`). It is **not** assigned a selector. The
compiler enforces no params, no return, no early `return`. See
[Contracts / init](../language/contracts.md#init).

### `pub fn get() -> uint { return value; }`

Selector `0x01` (first declared `pub fn`). Reads slot 0 and returns
it.

Bytecode (approximate):

```text
_fn_get:
JUMPDEST
POP                ; discard selector
PUSH 0
SLOAD              ; read slot 0
PUSH 0x40
MSTORE
PUSH 32
PUSH 0x40
RETURN
```

### `pub fn inc() { value = value + 1; }`

Selector `0x02`. Reads slot 0, adds 1, writes back. No return value.

```text
_fn_inc:
JUMPDEST
POP
PUSH 0
SLOAD              ; current value
PUSH 1
ADD                ; +1
PUSH 0
SSTORE             ; write back
STOP               ; implicit terminator for void fn
```

Note: ADD wraps modulo `2**256`. Calling `inc()` after the counter
hits `2**256 - 1` rolls over to 0 with no error. Use `safe_add` if
you'd prefer to revert.

## Calldata for each selector

| Call | Calldata bytes |
|---|---|
| `init` | (never called by users; runs on deploy) |
| `get()` | `0x01` (1 byte) |
| `inc()` | `0x02` (1 byte) |

## Deployment + invocation transcript

Conceptual flow:

1. Compile `counter.fou` to bytecode.
2. Submit a deploy tx with `data.type = "deploy"`, `data.code =
   <hex>`. The deploy address is derived from
   `(deployer, deployer_nonce)`.
3. The VM stores the code; immediately invokes the new code with
   empty calldata; the init prologue checks the flag (unset), sets it,
   runs `value = 42`, falls through to the empty-calldata short-circuit
   that halts with `STOP`.
4. Submit a `get()` call tx: `data.calldata = "01"`. Receipt
   `return_data` = `0000...002a` (42 in hex).
5. Submit `inc()` calls. Each increments the slot.

## What to try next

- Add `pub fn dec() { value = value - 1; }`. What selector does it
  get? (Hint: it's the third `pub fn`.) Does `dec()` from `value == 0`
  revert or wrap?
- Add a precondition: `pub fn inc_if_owner() { require(caller() ==
  owner); value = value + 1; }`. You'll need to add a `storage owner:
  address @ 1;` and set it in init.
- Swap `value + 1` for `safe_add(value, 1)` and observe overflow
  behavior change from wrap to revert.
