# Ownable

Source: `fourier/stdlib/ownable.fou`.

Single-owner access control. The contract that deploys becomes the
initial owner (`caller()` at `init` time); the owner can transfer or
renounce.

## Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `owner` | `address` | Current owner; `0` after renouncement |

Reserves slot 0. Your contract's storage must start at slot 1 or higher.

## Source

```fourier
contract Ownable {
    storage owner: address @ 0;

    fn init() {
        owner = caller();
    }

    pub fn get_owner() -> address {
        return owner;
    }

    pub fn transfer_ownership(new_owner: address) {
        require(caller() == owner);
        owner = new_owner;
    }

    pub fn renounce_ownership() {
        require(caller() == owner);
        owner = 0;
    }
}
```

## Selectors

| Selector | Function |
|---|---|
| `0x01` | `get_owner() -> address` |
| `0x02` | `transfer_ownership(address)` |
| `0x03` | `renounce_ownership()` |

## Usage: inherit-by-copy

Add the owner slot + the access check to your own contract:

```fourier
contract Vault {
    storage owner:    address @ 0;            // copied from Ownable
    storage balances: map[address, uint] @ 1;

    fn init() {
        owner = caller();
    }

    pub fn withdraw_to(addr: address, amount: uint) {
        require(caller() == owner);           // the only_owner check
        // ... withdraw logic ...
    }

    pub fn transfer_ownership(new_owner: address) {
        require(caller() == owner);
        owner = new_owner;
    }
}
```

## Usage: deploy + call

If you'd rather have a single shared Ownable record (e.g. an
organization's owner controls many contracts):

1. Deploy `Ownable` as its own contract; remember its address.
2. In each governed contract, store the ownable contract's address
   and STATICCALL it to read the current owner before granting
   privileges.

This is rare in v1 because there's no `bytes`-returning MLOAD
primitive at the Fourier level — you'd have to write your check in
hand-rolled bytecode or accept that the staticcall result lands at
memory `0x40` outside the source language. Inherit-by-copy is the
practical choice.

## Renouncement

`renounce_ownership()` writes `0` into the owner slot. Because writing
0 to a storage slot **deletes** the entry (`vm/state.py`), the slot
reverts to the natural-zero default. Subsequent `caller() == owner`
checks will fail for any non-zero caller — the contract becomes
permanently unmanaged.

This is permanent. There is no way to recover ownership after
renouncement; even calling `transfer_ownership(addr)` will fail
because the `require(caller() == owner)` check needs the caller to be
the zero address.

## Events

Ownable as shipped does not emit events. If you need an
`OwnershipTransferred(prev, new)` log for off-chain indexing, add it
in your copy:

```fourier
event OwnershipTransferred(prev: address, new_owner: address);

pub fn transfer_ownership(new_owner: address) {
    require(caller() == owner);
    emit OwnershipTransferred(owner, new_owner);
    owner = new_owner;
}
```
