# CryptoRegistry

Source: `fourier/stdlib/crypto_registry.fou`.

The authoritative on-chain mapping from `scheme_id` to PQC precompile
address. Designed to be deployed once per chain and owned by a
[Timelock](timelock.md) for governance.

## Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `owner` | `address` | Initial deployer; transferred to Timelock |
| `1` | `scheme_to_addr` | `map[uint, uint]` | `scheme_id → precompile_addr` |

Reserves slots 0–1.

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

At deploy time, init populates:

| `scheme_id` | `precompile_addr` | Algorithm |
|---|---|---|
| `1` | `2` (i.e. `0x02`) | ML-DSA-87 (FIPS 204) |
| `2` | `3` (i.e. `0x03`) | SLH-DSA-SHA2-128s (FIPS 205) |

These match the constants in `fourier/codegen.py::CRYPTO_SCHEMES` and
the precompile addresses in `vm/precompiles.py`.

## Why this exists

The compiler hard-codes `CRYPTO_SCHEMES` because `verify_sig(scheme_id,
...)` needs a compile-time literal `scheme_id` to select the right
precompile address. The CryptoRegistry serves a different role: it's
the **public, on-chain** record of which scheme ids are valid, so:

1. Clients and indexers can look up `scheme_id → precompile_addr`
   without trusting compiler internals.
2. Governance can add new schemes (after the operator network upgrades
   their VMs to recognize the new precompile) via Timelock proposals.
3. Retired schemes (e.g. one broken by a future cryptanalysis attack)
   can be removed from the registry so client SDKs stop generating
   signatures with them.

The registry **does not** prevent contracts from using an unknown or
retired scheme at compile time — that's a compiler-level decision. But
clients querying the registry will see what the chain considers
valid.

## Governance flow

```text
1. BDFL deploys CryptoRegistry.
   init() sets owner = BDFL, registers schemes 1 + 2.

2. BDFL deploys Timelock.
   init() sets owner = BDFL, delay = 14d, grace = 14d.

3. BDFL calls registry.transfer_ownership(timelock_address).
   Now the registry is governed by the Timelock.

4. BDFL queues a registry update through the Timelock:
   timelock.queue(
       target   = registry_addr,
       value    = 0,
       selector = 0x01,            // set_scheme
       arg      = ???,             // need to pack (scheme_id, precompile_addr)
       eta      = now + 14 days
   )

5. After 14 days, anyone calls timelock.execute(id).
   The Timelock calls registry.set_scheme(scheme_id, precompile)
   with the Timelock as caller — passes the owner check.
```

Note step 4's caveat: `set_scheme` takes **two** arguments, but the
shipped Timelock supports proposals with only one arg
(`p_arg: map[uint, uint]`). To make this work in v1:

- Pack both args into a single 32-byte word (e.g. `(scheme_id << 160)
  | precompile_addr`) and have `set_scheme` unpack.
- Or extend the Timelock to store multiple args per proposal.

A two-arg Timelock variant is on the
[follow-up list](https://github.com/DosseyRichards/Fermi-Fourier/blob/main/TODO.md);
for now, governance updates use a custom Timelock variant or a
separate "two-arg setter" wrapper contract.

## Adding a new scheme

The compiler must learn the new scheme before contracts can use
`verify_sig(N, ...)`. The end-to-end procedure:

1. Implement the new PQC algorithm in `vm/precompiles.py`. Reserve a
   precompile address (next free byte) and a gas cost.
2. Add the `(scheme_id, precompile_addr)` pair to
   `fourier/codegen.py::CRYPTO_SCHEMES`.
3. Node operators upgrade to the new VM version (the ultimate veto —
   nodes refusing the upgrade won't honor the new precompile).
4. Governance queues `registry.set_scheme(N, addr)` in the Timelock.
5. After the delay, the registry reflects the new scheme; clients
   reading the registry start trusting `scheme_id = N`.

The registry's role is **declarative** — it's the canonical record;
actual on-chain verification still requires nodes to ship the
precompile in their VM binary. The dual veto (operators + governance)
is intentional.

## Inherit-by-copy notes

CryptoRegistry is small (2 slots, 5 public functions) but rarely
useful to inherit-by-copy — its purpose is to be a single canonical
record per chain. Deploy it once and reference its address from any
contract that needs to look up scheme metadata.
