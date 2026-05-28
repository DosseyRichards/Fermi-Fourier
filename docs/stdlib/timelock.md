# Timelock

Source: `fourier/stdlib/timelock.fou`.

An owner-controlled gate with mandatory delay between queueing and
executing proposals. Anyone can execute a proposal once its `eta`
passes (so the owner being unavailable doesn't block upgrades), but
the owner can also cancel before execution.

## Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `owner` | `address` | Sole proposer / canceller |
| `1` | `delay` | `uint` | Min seconds between queue and earliest execution |
| `2` | `grace` | `uint` | Expiry window after eta |
| `3` | `next_id` | `uint` | Auto-incrementing proposal id |
| `4` | `p_target` | `map[uint, address]` | Target contract |
| `5` | `p_value` | `map[uint, uint]` | WAVE to forward |
| `6` | `p_selector` | `map[uint, uint]` | Selector byte for the call |
| `7` | `p_arg` | `map[uint, uint]` | Single argument (uint) |
| `8` | `p_eta` | `map[uint, uint]` | Earliest execution timestamp |
| `9` | `p_executed` | `map[uint, uint]` | 0/1 |
| `10` | `p_cancelled` | `map[uint, uint]` | 0/1 |

Reserves slots 0–10. Your storage starts at 11+ if inheriting.

## Source highlights

```fourier
fn init() {
    owner = caller();
    delay = 1209600;       // 14 days in seconds
    grace = 1209600;       // 14 days grace window
}

pub fn queue(target: address, value: uint, selector: uint, arg: uint,
              eta: uint) -> uint {
    require(caller() == owner);
    require(eta >= timestamp() + delay);
    // ... store proposal fields, emit, return id ...
}

pub fn cancel(id: uint) {
    require(caller() == owner);
    require(p_executed[id] == 0);
    require(p_cancelled[id] == 0);
    p_cancelled[id] = 1;
    emit ProposalCancelled(id);
}

pub fn execute(id: uint) -> uint {
    require(p_cancelled[id] == 0);
    require(p_executed[id] == 0);
    require(timestamp() >= p_eta[id]);
    require(timestamp() < p_eta[id] + grace);
    p_executed[id] = 1;
    let cd: bytes = pack_sel(p_selector[id], p_arg[id]);
    let ok: uint = call_b(p_target[id], cd, p_value[id], 500000);
    require(ok == 1);
    emit ProposalExecuted(id);
    return 1;
}
```

(Full source at `fourier/stdlib/timelock.fou`.)

## Selectors

| Selector | Function |
|---|---|
| `0x01` | `queue(address, uint, uint, uint, uint) -> uint` |
| `0x02` | `cancel(uint)` |
| `0x03` | `execute(uint) -> uint` |
| `0x04` | `transfer_ownership(address)` |
| `0x05` | `get_owner() -> address` |
| `0x06` | `get_delay() -> uint` |
| `0x07` | `get_proposal_eta(uint) -> uint` |
| `0x08` | `is_executed(uint) -> uint` |
| `0x09` | `is_cancelled(uint) -> uint` |

## Lifecycle

```text
queue(target, value, sel, arg, eta)
  ├── owner-only
  ├── require eta >= now + delay
  └── stores proposal, emits ProposalQueued

(wait until eta)

cancel(id)   ─── owner-only, before execute
execute(id)  ─── anyone, between eta and eta + grace
```

The `delay` enforces a minimum wait. The `grace` window prevents
indefinite execution — if no one calls `execute` within `grace`
seconds after `eta`, the proposal silently expires (the next
`execute(id)` will fail the upper-bound check `timestamp() < eta +
grace`).

## init runs at deploy

`init()` sets the deployer as owner and initializes `delay` and
`grace` to 14 days (1,209,600 seconds) each. These are constants —
to use different values, change the source before compiling.

`init` is private and runs atomically inside the deploy transaction.
See [Contracts / init](../language/contracts.md#init).

## One-arg constraint

Proposals carry a **single 32-byte argument** because there's only
one `p_arg` mapping. This covers most upgrade-flavored calls:

- `transfer_ownership(addr)` — one arg.
- `set_scheme(id, addr)` — two args; would require packing both into
  one word (e.g. `(id << 160) | addr`) or extending the contract to
  store a second arg mapping.
- Arbitrary multi-arg calls — extend the contract to store a
  `bytes`-hash commit/reveal scheme. Out of scope in v1.

## Governance pattern: Timelock + CryptoRegistry

The canonical governance setup (per the
[`CryptoRegistry`](https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software/blob/main/fourier/stdlib/crypto_registry.fou)
docstring):

1. BDFL deploys `CryptoRegistry` (init: owner = BDFL).
2. BDFL deploys `Timelock` (init: owner = BDFL, delay = 14d).
3. BDFL calls `registry.transfer_ownership(timelock_address)`.

After step 3:

- The registry's owner is the Timelock.
- The BDFL queues `set_scheme(id, addr)` proposals in the Timelock.
- After the 14-day delay, the BDFL (or anyone) calls
  `timelock.execute(id)`, which calls `registry.set_scheme(id, addr)`
  with the Timelock as `caller()`.
- The registry's `require(caller() == owner)` passes because the
  Timelock is the owner.

## Self-only ownership transfer

```fourier
pub fn transfer_ownership(new_owner: address) {
    require(caller() == owner);   // shortcut for v1; harden later
    owner = new_owner;
    emit OwnerTransferred(new_owner);
}
```

Note the source comment ("shortcut for v1; harden later"). A
production-grade Timelock would require `caller() == address(this)`
— i.e. ownership transfer would itself need to be queued and
timelocked. The current version trusts the owner to do the right
thing.

## Anybody-can-execute

After `eta`, **any** address can call `execute(id)`. This is by
design — even if the owner becomes unavailable, queued proposals can
still go through. The trade is that you lose the owner's veto window
after `eta`. If you want owner-only execution, gate `execute` with
`require(caller() == owner)`.

## Inherit-by-copy notes

Timelock is large (10+ storage slots, 9 public functions). Most
deployments use Timelock as a **separate** deployed contract rather
than inheriting it — the BDFL deploys one Timelock and points each
governable contract at the same Timelock address.

If you do inherit-by-copy, renumber all `@` slots to sit above your
own storage.
