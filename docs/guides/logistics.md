# Logistics & chain of custody

## The problem

Something valuable moves through many hands: a pallet of vaccines from
factory to distributor to pharmacy; an evidence bag from crime scene to
lab to courtroom; a lab sample; a piece of art; a high-value part.

At the end, someone asks: **who held this, in what order, and was it
ever out of trusted hands?** Today that answer lives in a spreadsheet or
a vendor's database — editable, loseable, and only as honest as whoever
controls it. Disputes become one party's word against another's.

A blockchain turns the custody trail into a shared record no single
party can alter: each handoff is a signed transaction, the sequence is
enforced by code, and the history is permanent. But custody records
outlive the shipment — a drug's chain of custody may be scrutinized
years after delivery, evidence may be challenged decades later at appeal
— and a record is only as permanent as the signatures underneath it.
Each "I received this, intact" is a signature, and a quantum computer
will eventually be able to forge the signatures securing today's
systems, deriving private keys from the public ones already visible
on-chain. That would let an adversary **retroactively rewrite history**:
insert a handler who was never there, or erase one who was. To keep that
from ever happening, each handoff here is signed with a post-quantum
scheme (**ML-DSA-87, FIPS 204** today, with the chain able to swap in
newer post-quantum schemes without a hard fork) — the current holder,
and only the current holder, can pass custody on, and that authorization
stays unforgeable into the quantum era.

## The contract

Source: `fourier/examples/custody_chain.fou` (compiles on the WaveLedger
VM).

```fourier
// CustodyChain — a quantum-proof chain-of-custody / logistics tracker.
// Each handoff is a transaction signed by the current holder with
// ML-DSA-87, so the trail of who held the item, and when, cannot be
// forged or rewritten.
contract CustodyChain {
    storage shipment_count: uint @ 0;
    storage holder: map[uint, address] @ 1;                // id -> current holder
    storage origin: map[uint, address] @ 2;                // id -> creator
    storage delivered: map[uint, uint] @ 3;                // id -> 1 if delivered
    storage step_count: map[uint, uint] @ 4;               // id -> number of custody steps
    storage step_holder: map[uint, map[uint, address]] @ 5; // id -> step -> holder
    storage step_time: map[uint, map[uint, uint]] @ 6;     // id -> step -> timestamp

    event ShipmentCreated(id: uint, origin: address);
    event CustodyTransferred(id: uint, from: address, to: address, step: uint);
    event Delivered(id: uint, by: address);

    pub fn create_shipment() -> uint {
        let id: uint = shipment_count;
        holder[id] = caller();
        origin[id] = caller();
        delivered[id] = 0;
        step_holder[id][0] = caller();
        step_time[id][0] = timestamp();
        step_count[id] = 1;
        shipment_count = id + 1;
        emit ShipmentCreated(id, caller());
        return id;
    }

    pub fn transfer_custody(id: uint, to: address) {
        require(holder[id] == caller());
        require(delivered[id] == 0);
        holder[id] = to;
        let s: uint = step_count[id];
        step_holder[id][s] = to;
        step_time[id][s] = timestamp();
        step_count[id] = s + 1;
        emit CustodyTransferred(id, caller(), to, s);
    }

    pub fn mark_delivered(id: uint) {
        require(holder[id] == caller());
        delivered[id] = 1;
        emit Delivered(id, caller());
    }

    pub fn current_holder(id: uint) -> address {
        return holder[id];
    }

    pub fn get_step_holder(id: uint, step: uint) -> address {
        return step_holder[id][step];
    }

    pub fn get_step_count(id: uint) -> uint {
        return step_count[id];
    }
}
```

## How it works

### Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `shipment_count` | `uint` | Next shipment id |
| `1` | `holder` | `map[uint, address]` | id → who holds it *right now* |
| `2` | `origin` | `map[uint, address]` | id → who created it |
| `3` | `delivered` | `map[uint, uint]` | id → `1` once finalized |
| `4` | `step_count` | `map[uint, uint]` | id → number of custody steps |
| `5` | `step_holder` | `map[uint, map[uint, address]]` | id → step → holder (the audit trail) |
| `6` | `step_time` | `map[uint, map[uint, uint]]` | id → step → timestamp |

### Selector layout

| Selector | Function |
|---|---|
| `0x01` | `create_shipment() -> uint` |
| `0x02` | `transfer_custody(uint, address)` |
| `0x03` | `mark_delivered(uint)` |
| `0x04` | `current_holder(uint) -> address` |
| `0x05` | `get_step_holder(uint, uint) -> address` |
| `0x06` | `get_step_count(uint) -> uint` |

### The custody invariant

The whole guarantee rests on one line in `transfer_custody`:

```fourier
require(holder[id] == caller());
```

Only the account that *currently* holds the item can pass it on, and
`caller()` is post-quantum-authenticated. So the chain of custody is a
strict baton pass — you can't hand off something you don't hold, and you
can't insert yourself into someone else's trail. Every step appends an
immutable `(holder, timestamp)` pair to `step_holder` / `step_time` and
emits `CustodyTransferred`, so the full history is reconstructable from
events alone.

`mark_delivered` freezes the shipment: once `delivered[id] == 1`, further
transfers are rejected.

### Reconstructing the trail

To audit shipment `id`, read `get_step_count(id)`, then
`get_step_holder(id, i)` and the matching timestamp for each step from
`0` to `count - 1`. That sequence — origin first, current holder last —
is the complete, tamper-proof custody record.

## Driving it from your application

1. The origin (factory, evidence tech) calls `create_shipment()` and
   records the returned `id` — perhaps printed as a QR code on the box.
2. At each handoff, the **releasing** party calls
   `transfer_custody(id, next_holder_address)` from their own wallet.
   (A "confirm receipt" variant is in *Extending it* below.)
3. The final recipient calls `mark_delivered(id)`.
4. Auditors, regulators, or a counterparty read the trail — no access to
   anyone's database required.

## Extending it

- **Two-party handoff (accept step)** — instead of the sender assigning
  the next holder, have the sender *propose* and the receiver *accept*
  in a second transaction, so both signatures are on record for each
  step.
- **Condition attestations** — add a `note_hash` argument to
  `transfer_custody` holding the `sha3` of an off-chain inspection
  report ("temperature within range, seal intact"), anchoring the
  condition at each step without putting the report on-chain.
- **Authorized-handlers list** — gate `transfer_custody` on an
  `is_handler[to] == 1` roll so custody can only pass to vetted parties.
- **Geofencing / SLA timers** — record expected vs. actual timestamps
  and emit an alert event when a leg exceeds its window.
