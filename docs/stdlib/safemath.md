# SafeMath

SafeMath is not a separately deployable contract; it is a set of
compiler-level builtins. Each `safe_*` call expands inline to
bytecode that performs the underlying op and reverts on overflow,
underflow, or division by zero.

| Builtin | Semantics | Reverts when |
|---|---|---|
| `safe_add(a, b) -> uint` | `a + b` | `a + b < a` (wrap) |
| `safe_sub(a, b) -> uint` | `a - b` | `b > a` (underflow) |
| `safe_mul(a, b) -> uint` | `a * b` | `b != 0 && (a * b) / b != a` |
| `safe_div(a, b) -> uint` | `a / b` | `b == 0` |

Source: `_emit_expr` for the `safe_*` builtin names in
`fourier/codegen.py`.

## Why builtins, not a stdlib contract

EVM-style SafeMath libraries exist because checked-by-default
arithmetic arrived only in Solidity 0.8. Fourier keeps the raw
`+ - * /` opcodes wraparound / divide-to-zero (matching the
underlying VM semantics in `vm/machine.py`); checks are opt-in per
call.

Compiling to inlined opcodes — rather than a callout to a deployed
library — avoids cross-contract call cost (no `CALL`, no gas
forwarding overhead) and keeps the runtime entirely native.

## Behavior

### `safe_add`

```fourier
let z: uint = safe_add(x, y);
```

Computes `z = (x + y) mod 2**256`, then asserts `z >= x`. If the
addition wrapped (typical 256-bit overflow), `z < x` and the call
reverts.

### `safe_sub`

```fourier
let z: uint = safe_sub(x, y);
```

Asserts `y <= x` first; if not, reverts. Otherwise returns `x - y`.

### `safe_mul`

```fourier
let z: uint = safe_mul(x, y);
```

If `y == 0`, returns 0 (the only path that doesn't compute the product
explicitly). Otherwise computes `z = x * y mod 2**256` and asserts
`z / y == x`. Catches the case where multiplication wrapped.

### `safe_div`

```fourier
let z: uint = safe_div(x, y);
```

Reverts if `y == 0`; otherwise returns `x / y` (integer division —
remainder truncated).

Note: the raw `/` operator returns `0` when dividing by zero. Use
`safe_div` to surface that as an error instead of silently producing
0.

## Gas cost

Each `safe_*` expands to ~10–20 opcodes (arithmetic, a comparison, a
conditional jump, a possible revert). Approximate per-call cost is
30–80 gas — cheaper than a sub-call, more expensive than the raw
operator.

## When to use

- Token balance updates (overflow on add, underflow on sub).
- Reward / fee accumulations.
- Anywhere application semantics treat overflow as a bug rather than
  as modular arithmetic.

Skip when:

- Computing slot derivations, hashes, or bitwise mixing, where
  wraparound is the desired behavior.
- Inside a tight loop where every gas unit counts.

## Example

```fourier
contract Bank {
    storage balances: map[address, uint] @ 0;

    pub fn deposit() {
        let bal: uint = balances[caller()];
        balances[caller()] = safe_add(bal, callvalue());
    }

    pub fn transfer(to: address, amount: uint) {
        let sender_bal: uint = balances[caller()];
        balances[caller()] = safe_sub(sender_bal, amount);
        balances[to]       = safe_add(balances[to], amount);
    }
}
```

If a `transfer` would overflow the recipient or underflow the
sender, the whole tx reverts and balances remain unchanged.
