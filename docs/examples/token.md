# Token (ERC-20-flavored)

Source: `fourier/examples/token.fou`.

A minimal fungible-token contract: total supply, per-address
balances, a `transfer` function with a `require` precondition and a
`Transfer` event. ERC-20-flavored, but using the Fourier ABI
(1-byte selectors rather than 4-byte function IDs).

## Full source

```fourier
// ERC-20-style token written in Fourier.
// Compiles to the same VM bytecode interface as vm/examples/token.py
// (selectors 0x01/0x02/0x03 for totalSupply/balanceOf/transfer).

contract Token {
    storage total_supply: uint @ 0;
    storage balances: map[address, uint] @ 1;

    event Transfer(from: address, to: address, amount: uint);

    pub fn totalSupply() -> uint {
        return total_supply;
    }

    pub fn balanceOf(addr: address) -> uint {
        return balances[addr];
    }

    pub fn transfer(to: address, amount: uint) -> bool {
        let sender: address = caller();
        let sender_bal: uint = balances[sender];
        require(sender_bal >= amount);
        balances[sender] = sender_bal - amount;
        balances[to] = balances[to] + amount;
        emit Transfer(sender, to, amount);
        return true;
    }
}
```

Note: this file has no `init`. No minting occurs in this version;
all balances start at 0. For a deployer-funded version, add:

```fourier
fn init() {
    total_supply = 1_000_000;
    balances[caller()] = total_supply;
}
```

## Annotated source

### Storage

| Slot | Name | Type |
|---|---|---|
| `0` | `total_supply` | `uint` |
| `1` | `balances` | `map[address, uint]` |

`balances[addr]` is stored at storage key `SHA3-256(addr_word ||
slot_1_word)`. Two addresses can never collide because SHA3 collisions
are infeasible.

### Event

```fourier
event Transfer(from: address, to: address, amount: uint);
```

Topic 0: `SHA3-256("Transfer(address,address,uint)")`.

Because `transfer` has 3 args, all three become indexed topics:
`topic_1 = sender`, `topic_2 = to`, `topic_3 = amount`. Data is empty.

This makes the event efficient to filter: wallets querying for
inbound transfers to address X match on `topic_2 == X` without
parsing data.

### `totalSupply() -> uint` (selector `0x01`)

```fourier
pub fn totalSupply() -> uint {
    return total_supply;
}
```

Single `SLOAD` of slot 0, write to `RETURN_AT`, `RETURN 32`.

### `balanceOf(addr) -> uint` (selector `0x02`)

```fourier
pub fn balanceOf(addr: address) -> uint {
    return balances[addr];
}
```

Slot derivation:

```text
PUSH addr           ; loaded from calldata at offset 1
PUSH 0              ; SCRATCH_A
MSTORE
PUSH 1              ; slot
PUSH 0x20           ; SCRATCH_B
MSTORE
PUSH 64
PUSH 0              ; SCRATCH_A start
SHA3                ; → derived key for balances[addr]
SLOAD
```

Then write to `RETURN_AT`, `RETURN 32`.

### `transfer(to, amount) -> bool` (selector `0x03`)

Five statements:

```fourier
let sender: address = caller();
let sender_bal: uint = balances[sender];
require(sender_bal >= amount);
balances[sender] = sender_bal - amount;
balances[to] = balances[to] + amount;
emit Transfer(sender, to, amount);
return true;
```

Step by step:

1. `let sender = caller();` — single CALLER opcode, store at local
   memory offset 0xc0 (params occupy 0x80 and 0xa0).
2. `let sender_bal = balances[sender];` — derive `balances[sender]`
   slot via SHA3, SLOAD, store at local 0xe0.
3. `require(sender_bal >= amount);` — bytecode `LT`, `ISZERO`
   (sender_bal < amount → bad), JUMPI to ok-label or PUSH 0 PUSH 0
   REVERT.
4. `balances[sender] = sender_bal - amount;` — derive slot,
   compute `sender_bal - amount`, SSTORE.
5. `balances[to] = balances[to] + amount;` — derive slot, SLOAD,
   add `amount`, derive slot **again** (compiler doesn't cache), SSTORE.
6. `emit Transfer(sender, to, amount);` — LOG4 with sig topic +
   3 indexed args, no data.
7. `return true;` — PUSH 1, MSTORE 0x40, RETURN 0x40 32.

Note: step 5 re-derives the slot. V1 performs no expression-level
caching: each occurrence of `balances[to]` in the same statement
triggers an SHA3 plus SLOAD.

## Calldata for each selector

```text
totalSupply()                                # 01
balanceOf(0xAAAA...AAAA)                     # 02 || pad12 || addr20
transfer(0xBBBB...BBBB, 1000)                 # 03 || pad12 || addr20 || pad28 || 0x3e8
```

(`||` is concatenation; `pad12` is 12 zero bytes, `pad28` is 28 zero
bytes.)

## Gas cost approximation

Per-call estimates assuming all storage slots already exist:

| Call | Approx gas |
|---|---|
| `totalSupply()` | ~250 (SLOAD + RETURN + dispatch) |
| `balanceOf(addr)` | ~280 (SHA3 + SLOAD + RETURN) |
| `transfer(to, amount)` | ~12,000 (2× SHA3, 1× SLOAD, 2× SSTORE existing→existing, 1× LOG4, 1× RETURN) |

A `transfer` to a **fresh** recipient — whose `balances[to]` slot
holds zero — costs ~27,000 because the SSTORE upgrades from
`zero → non-zero` (20,000 gas vs 5,000 gas).

## Variants

- Add `init()` to seed `total_supply` and the deployer's balance.
- Add `approve(spender, amount)` and `transferFrom(from, to, amount)`
  to match the ERC-20 surface. Requires a `storage allowances:
  map[address, map[address, uint]] @ 2;`.
- Wrap `transfer` with a pausable guard using
  [`Pausable`](../stdlib/pausable.md).
- Use `safe_add` / `safe_sub` in place of raw `+` / `-` to revert on
  overflow instead of wrapping.
