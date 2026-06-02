# Standard library

Fourier ships six small contracts in `fourier/stdlib/` plus a set of
compiler-level checked-arithmetic builtins ("SafeMath"). Six of these
are full `.fou` source files; SafeMath is intrinsic to the compiler.

## Inheritance model

Fourier v1 has no inheritance keyword. A stdlib contract is reused in
one of two ways:

1. **Inherit-by-copy.** Paste the contract's storage decls, events,
   and functions into a consuming contract. Renumber `@ slot` to
   avoid collisions with other storage. This is the dominant pattern.

2. **Deploy + call.** Deploy the stdlib contract as its own address,
   then `call_b` / `delegatecall_b` from the consuming contract.
   Useful when multiple contracts share governance (for example, one
   `Timelock` per organization, many target contracts).

`DELEGATECALL` lets a stdlib contract mutate the caller's storage.
See [`Timelock`](timelock.md) for a concrete pattern where the stdlib
contract is a separately-deployed governance gate.

## Shipped contracts

| Contract | Source | Storage slots | One-line role |
|---|---|---|---|
| [SafeMath](safemath.md) | (compiler builtins) | none | Overflow-checked arithmetic — `safe_add`, `safe_sub`, `safe_mul`, `safe_div` |
| [Ownable](ownable.md) | `fourier/stdlib/ownable.fou` | `0` (owner) | Single-owner access control |
| [Pausable](pausable.md) | `fourier/stdlib/pausable.fou` | `0` (owner), `1` (paused flag) | Circuit-breaker pause flag, owner-controlled |
| [ReentrancyGuard](reentrancy.md) | `fourier/stdlib/reentrancy_guard.fou` | `0` (`_locked`), `1` (`balances`) | Mutex pattern for external-call functions |
| Multisig | `fourier/stdlib/multisig.fou` | `0`–`9` | M-of-N signer governance |
| Timelock | `fourier/stdlib/timelock.fou` | `0`–`10` | Owner-controlled delayed proposals |
| CryptoRegistry | `fourier/stdlib/crypto_registry.fou` | `0`–`1` | Authoritative `scheme_id → precompile_addr` table |

## Slot conventions

Under inherit-by-copy, the stdlib contract's `@ slot` numbers collide
with the consuming contract's slots unless rebased. The standard
practice is:

- Treat the stdlib's slots as a fixed prefix (the low slot numbers).
- Renumber the consuming contract's storage decls to sit above them.

Example: copying `Ownable` (uses slot 0) into a `Token` that needs
`total_supply` and `balances`:

```fourier
contract MyToken {
    storage owner:        address @ 0;            // from Ownable
    storage total_supply: uint    @ 1;            // your storage
    storage balances:     map[address, uint] @ 2;

    fn init() {
        owner = caller();
        total_supply = 1_000_000;
        balances[caller()] = total_supply;
    }

    pub fn transfer_ownership(new_owner: address) {  // from Ownable
        require(caller() == owner);
        owner = new_owner;
    }
    // ... your token functions ...
}
```

## Why these six (and only these)

The set covers the patterns most contracts need at the source level
without adding compiler complexity:

- Ownable, Pausable, ReentrancyGuard — pure source-level patterns,
  copy-and-paste.
- Multisig — exercises the PQC `verify_sig` builtin for signer
  attestation flows.
- Timelock — makes governance changes auditable and delayable.
- CryptoRegistry — the chain-level mapping that the multisig and
  client SDKs must agree on.

More complex contracts (oracles, AMMs, NFTs) are left to user code.
