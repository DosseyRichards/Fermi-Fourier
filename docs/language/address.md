# Address type

`address` is a 20-byte identifier referring to an account on chain.
At runtime it lives in a 256-bit word, right-aligned (the address
bytes occupy the low 160 bits, the top 96 bits are zero).

## Width

| Layer | Width |
|---|---|
| Source semantics | 20 bytes (`ADDRESS_BYTES = 20`) |
| Runtime stack value | 256-bit `uint` |
| Storage / memory | 256-bit `uint` |
| Network address representation | 40-char hex string (no `0x` prefix) |

The VM `vm/state.py` enforces 20-byte width at the WorldState API
level; values exceeding that are truncated to the low 20 bytes when
passed through `BALANCE` and `CALL`.

## Compile-time enforcement

The codegen tracks a "best-effort" type for each local. When a binary
op has `address` on one side and `uint` / `bool` on the other, it
rejects arithmetic and ordering ops:

```fourier
let a: address = caller();
let n: uint = 1;

let bad: uint = a + n;          // compile error
let bad2: bool = a < n;          // compile error
```

Allowed (equality / inequality always works):

```fourier
let same: bool = a == caller();
let diff: bool = caller() != owner;
```

The compile-time check is limited to **typed locals**. Storage reads
return `_` (unknown), so this passes the static check silently:

```fourier
let counter: uint = some_storage_uint + 1;       // ok if some_storage_uint is uint
let bad: uint = owner_storage + 1;                // not caught — runtime treats as uint add
```

Source: `_emit_expr` for `BinOp` in `fourier/codegen.py`.

## Zero address

There is no special `address(0)` constructor. Compare against the
literal `0`:

```fourier
require(addr != 0);
```

Writing `0` to a `storage owner: address` slot deletes the entry from
storage (the VM treats writing zero as a delete).

`vm/state.py` has no concept of a "burn" or "null" address — `0x00 *
20` is an ordinary account that simply receives any funds sent to it.
The chain-level convention (per the
[WaveLedger transaction docs](https://docs.fermi.world/reference/tx/#sentinel-senders-recipients))
treats `"0" * 32` as a burn convention; that's a docs convention, not
a VM rule.

## Address-producing builtins

| Builtin | Returns | VM op |
|---|---|---|
| `caller()` | Immediate caller (msg.sender) | `CALLER` |
| `origin()` | Tx originator (tx.origin) | `ORIGIN` |

There is no `address(this)` builtin in v1, but the VM exposes
`ADDRESS` (0x70). It's not currently surfaced to Fourier — tracked
on the
[follow-up list](https://github.com/DosseyRichards/Fermi-Fourier/blob/main/TODO.md).

## Comparing addresses

```fourier
require(caller() == owner);
require(addr != 0);
```

`==` and `!=` compile to `EQ` / `EQ + ISZERO`. Because both operands
become full 256-bit words, no padding concerns apply — addresses are
identical iff their underlying ints are identical.

## Address as map key

```fourier
storage balances: map[address, uint] @ 1;
```

The address (as a 256-bit word, right-padded with zero-prefix bytes)
is hashed with the slot to derive the storage key:

```text
slot(balances[a]) = SHA3-256(a_as_word || slot_1)
```

Because two addresses differ in their low 160 bits but share all-zero
top 96 bits, the hash domain is effectively unique per address — no
collision risk.

## Address from a uint

If you need to convert a runtime `uint` to address (e.g. from a
calldata-supplied recipient), no cast is required — `address` and
`uint` share the same runtime representation. Just bind to an
`address`-typed local:

```fourier
pub fn pay(to: address, amount: uint) {
    let receiver: address = to;
    // ...
}
```

The codegen treats both as 256-bit words; the address-vs-uint type
check is purely static.

## tx.origin caveat

`origin()` returns the EOA that signed the tx — the very first caller
in the chain. Use with care: any contract you call can see your
origin, and code that uses `origin()` as an authorization check is
vulnerable to phishing-style attacks (a malicious intermediate
contract can do something on your behalf because your origin matches).
Prefer `caller()` for authorization, except when you specifically
need to recover the EOA at the bottom of the call stack.
