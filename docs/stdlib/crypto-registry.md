# CryptoRegistry

Source: `fourier/stdlib/crypto_registry.fou`.

The on-chain mapping from `scheme_id` to PQC precompile address.
Deployed once per chain and held by a [Timelock](timelock.md) for
governance.

## Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `owner` | `address` | Initial deployer; transferred to the Timelock |
| `1` | `scheme_to_addr` | `map[uint, uint]` | `scheme_id → precompile_addr` |

Slots 0–1 are reserved.

## Source

```fourier
contract CryptoRegistry {
    storage owner: address @ 0;
    storage scheme_to_addr: map[uint, uint] @ 1;

    event SchemeRegistered(scheme_id: uint, precompile: uint);
    event SchemeRetired(scheme_id: uint);
    event OwnerTransferred(new_owner: address);

    fn init() {
        owner = caller();
        scheme_to_addr[1] = 2;       // ML-DSA-87 at 0x02
        scheme_to_addr[2] = 3;       // SLH-DSA at 0x03
    }

    pub fn set_scheme(scheme_id: uint, precompile: uint) {
        require(caller() == owner);
        scheme_to_addr[scheme_id] = precompile;
        emit SchemeRegistered(scheme_id, precompile);
    }

    pub fn retire_scheme(scheme_id: uint) {
        require(caller() == owner);
        scheme_to_addr[scheme_id] = 0;
        emit SchemeRetired(scheme_id);
    }

    pub fn get_scheme(scheme_id: uint) -> uint {
        return scheme_to_addr[scheme_id];
    }

    pub fn transfer_ownership(new_owner: address) {
        require(caller() == owner);
        owner = new_owner;
        emit OwnerTransferred(new_owner);
    }

    pub fn get_owner() -> address {
        return owner;
    }
}
```

## Selectors

| Selector | Function |
|---|---|
| `0x01` | `set_scheme(uint, uint)` |
| `0x02` | `retire_scheme(uint)` |
| `0x03` | `get_scheme(uint) -> uint` |
| `0x04` | `transfer_ownership(address)` |
| `0x05` | `get_owner() -> address` |

## Pre-registered schemes

At deploy time, `init` populates:

| `scheme_id` | `precompile_addr` | Algorithm |
|---|---|---|
| `1` | `2` (`0x02`) | ML-DSA-87 (FIPS 204) |
| `2` | `3` (`0x03`) | SLH-DSA-SHA2-128s (FIPS 205) |

These match the constants in `fourier/codegen.py::CRYPTO_SCHEMES` and
the precompile addresses in `vm/precompiles.py`.

## Role

The compiler hard-codes `CRYPTO_SCHEMES` because `verify_sig(scheme_id,
...)` requires a compile-time literal `scheme_id` to select the
precompile address. The CryptoRegistry serves a complementary role: it
is the **public, on-chain** record of which scheme identifiers are
valid, so:

1. Clients and indexers can resolve `scheme_id → precompile_addr`
   without trusting compiler internals.
2. Governance can add new schemes — after node operators upgrade their
   VMs to recognize the precompile — through Timelock proposals.
3. Retired schemes can be removed so client SDKs stop emitting
   signatures with them.

The registry does **not** prevent contracts from referencing an
unknown or retired scheme at compile time; that is a compiler-level
decision. Clients querying the registry observe what the chain
considers authoritative.

## Governance flow

```text
1. Foundation operator deploys CryptoRegistry.
   init() sets owner = operator and registers schemes 1 + 2.

2. Foundation operator deploys Timelock.
   init() sets owner = operator, delay = 14 days, grace = 14 days.

3. Foundation operator calls registry.transfer_ownership(timelock).
   The registry is now governed by the Timelock.

4. Foundation operator queues a registry update through the Timelock:
   timelock.queue(
       target   = registry_addr,
       value    = 0,
       selector = 0x01,                // set_scheme
       arg      = packed_args,         // see "Packing two arguments" below
       eta      = now + 14 days
   )

5. After the delay window, anyone calls timelock.execute(id).
   The Timelock invokes registry.set_scheme(scheme_id, precompile)
   with the Timelock as caller, satisfying the owner check.
```

### Packing two arguments

`set_scheme` takes two arguments; the shipped Timelock stores one
argument per proposal (`p_arg: map[uint, uint]`). Two compatible
patterns:

- **Argument packing.** Pack both values into a single 32-byte word —
  for example, `(scheme_id << 160) | precompile_addr` — and unpack
  inside a wrapper function called by the Timelock.
- **Wrapper contract.** Deploy a small contract whose single
  `set(scheme_id, precompile)` function the Timelock calls; the
  wrapper unpacks and re-calls `set_scheme` on the registry.

A multi-argument Timelock variant is a planned addition.

## Adding a new scheme

The compiler must recognize the new scheme before contracts can invoke
`verify_sig(N, ...)`. The end-to-end procedure:

1. Implement the algorithm in `vm/precompiles.py`. Reserve a
   precompile address and a gas cost.
2. Add the `(scheme_id, precompile_addr)` pair to
   `fourier/codegen.py::CRYPTO_SCHEMES`.
3. Node operators upgrade to the new VM release. Operators are the
   final veto — nodes that decline the upgrade do not honor the new
   precompile.
4. Governance queues `registry.set_scheme(N, addr)` through the
   Timelock.
5. After the delay window, the registry reflects the new scheme and
   clients reading the registry begin trusting `scheme_id = N`.

The registry is **declarative**: on-chain verification still requires
nodes to ship the precompile in their VM binary. The dual veto
between node operators and on-chain governance is intentional.

## Composition notes

CryptoRegistry is intentionally small — two storage slots and five
public functions — and is not designed to be inherited or copied. Its
purpose is to serve as a single canonical record per chain. Deploy it
once and reference its address from contracts that need to resolve
scheme metadata.
