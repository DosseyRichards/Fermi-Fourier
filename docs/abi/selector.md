# Selector layout

The selector is the first byte of calldata. It picks one `pub fn` out
of the contract's dispatch table.

## Assignment rule

```python
sel = 0x01
for fn in contract.functions:
    if fn.is_pub:
        fn.selector = sel
        sel += 1
```

(`_ContractGen.__init__` in `fourier/codegen.py`)

- The first `pub fn` declared gets selector `0x01`.
- Each subsequent `pub fn` increments by 1.
- Private `fn`s (including `init`) are skipped.
- Maximum 255 `pub fn`s per contract (selector is one byte).

## Reading off a contract's selector table

The bytecode contains no metadata mapping selectors to names; the
mapping is derived from source. Walk the source top-to-bottom and
count `pub fn` declarations starting at 1:

```fourier
contract Token {
    storage total_supply: uint @ 0;
    storage balances: map[address, uint] @ 1;

    event Transfer(from: address, to: address, amount: uint);

    pub fn totalSupply() -> uint { ... }       // 0x01
    pub fn balanceOf(addr: address) -> uint { ... }   // 0x02
    pub fn transfer(to: address, amount: uint) -> bool { ... }   // 0x03
}
```

Reordering the source = breaking ABI. Adding a new `pub fn` between
existing ones shifts every later selector.

## Reserved selectors

| Selector | Reserved for |
|---|---|
| `0x00` | Empty-calldata short-circuit (deploy-time init); not a valid call selector |

`0x00` is treated specially: if the calldata is empty,
`CALLDATASIZE == 0` short-circuits the dispatcher and halts with
`STOP`. This is what makes init run atomically during deploy
(`vm/deploy.py`).

A tx with `calldata = "00"` (a single zero byte) does **not** hit
the short-circuit: that is 1 byte of calldata, so the dispatcher
reads `0x00` as the selector, finds no matching `pub fn`, and reverts.

## Dispatcher bytecode

For each `pub fn`, the dispatcher emits:

```text
DUP 1                  ; keep selector for later comparisons
PUSH <selector>
EQ
PUSHLABEL _fn_<name>
JUMPI                  ; if selector matches, jump to function body
```

The selector is extracted once at the top of the dispatcher:

```text
PUSH 0
CALLDATALOAD           ; load first 32 bytes of calldata
PUSH 1 << 248
SWAP 1
DIV                    ; divide by 2**248 → keep only the top byte
```

After all comparisons, if no selector matched, the dispatcher emits an
unconditional `REVERT`.

## Cross-contract calls

To build calldata in Fourier that calls another contract:

```fourier
let cd: bytes = pack_sel(0x03, target_addr, amount);
```

The first argument to `pack_sel` is the selector word. Internally it
shifts the selector to the top byte (`<< 248`) and writes it at
`cd + 32`, so the resulting `bytes` value reads `[0x03][arg1][arg2]`
from offset 0.

See [Cross-contract calls](../language/cross-contract.md) for the
full context.

## Selector collisions

There are no name-based selector collisions (selectors are
positional, not name-derived). When two different versions of a
contract are both deployed and called by the same off-chain client,
selectors line up only if the public-function order is identical
between versions.

Recommended practice when evolving a contract: **always append** new
`pub fn`s; never reorder. This keeps existing selectors stable.

## Why one byte?

EVM uses 4 bytes (the truncated `keccak256("name(types)")`) so that
selectors are name-derived and collision-resistant. Fourier uses one
byte for three reasons:

- The selector space (`pub fn` count) is bounded by what a single
  contract reasonably exposes.
- Removing the hash-derivation step makes calldata trivial to
  reason about and hand-construct.
- The dispatcher is a flat O(n) comparison chain (with small n)
  rather than a switch table — simpler bytecode, easier to audit.

The tradeoff: ABI stability is positional, not nominal. Contracts
that need long-term ABI stability must freeze the declaration order
of their `pub fn`s and document it.
