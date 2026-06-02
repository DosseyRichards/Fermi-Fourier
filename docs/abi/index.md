# ABI and calldata

The Fourier ABI is defined by three rules:

1. **The first byte of calldata is the selector.** It identifies the
   target `pub fn`.
2. **Each subsequent argument is a single 32-byte word, big-endian.**
   Calldata carries no variable-length arguments.
3. **Return values are 32 bytes per scalar, big-endian.** A `bytes`
   return is the only variable-length result type.

There is no Solidity-style 4-byte function selector, no
`keccak256("name(types)")` derivation, and no head/tail tuple
encoding. The design omits these in exchange for a calldata format
that can be hand-constructed in a single line.

## Calldata layout

```text
byte  0          : selector  (1 byte, e.g. 0x01)
bytes 1   .. 33  : arg[0]    (32 bytes, big-endian)
bytes 33  .. 65  : arg[1]    (32 bytes, big-endian)
bytes 65  .. 97  : arg[2]    (32 bytes, big-endian)
...
```

For a function with `N` args, calldata is exactly `1 + N * 32` bytes.

## Encoding a call by hand

Given a contract that declares:

```fourier
pub fn transfer(to: address, amount: uint) -> bool { ... }
```

`transfer` is the third `pub fn` declared, so its selector is `0x03`
(see [Selector layout](selector.md)).

A call to `transfer(0x1234..1234, 1000)` is constructed as:

```text
0x03                                                                      // selector
0000000000000000000000001234567890abcdef1234567890abcdef12341234        // to (address right-aligned in low 20 bytes)
00000000000000000000000000000000000000000000000000000000000003e8        // amount (1000)
```

Total: 65 bytes.

This goes into the `data.calldata` field of a chain-level
[contract call transaction](https://docs.fermi.world/reference/tx/#contract-call):

```json
{
  "type": "call",
  "to":   "<your contract address>",
  "calldata": "03000000000000000000000000_1234567890abcdef1234567890abcdef12341234_00000000000000000000000000000000000000000000000000000000000003e8",
  "gas_limit": 100000,
  "value": 0
}
```

(without the underscores — those are visual breaks).

## What's in this section

- [**Selector layout**](selector.md) — how the 1-byte selector is
  assigned to each `pub fn`.
- [**Argument encoding**](encoding.md) — how each Fourier type encodes
  into its 32-byte word.
- [**Return values**](returns.md) — how return data is laid out for
  scalar vs tuple vs `bytes` returns.

## When to encode by hand

- Building a wallet UI that talks to a contract.
- Off-chain indexers that recognize specific calls.
- Cross-contract calls **from within Fourier** via `pack_sel(...)`.
- Manual tx construction in tests.

The Python SDK at
[https://docs.fermi.world/sdk/](https://docs.fermi.world/sdk/)
provides helpers. This section is the normative specification those
helpers implement.
