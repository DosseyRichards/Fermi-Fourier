# Pausable

Source: `fourier/stdlib/pausable.fou`.

Circuit-breaker pattern. An owner can pause and unpause the contract;
protected functions check the flag before doing work and revert if
paused.

## Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `owner` | `address` | Pause controller |
| `1` | `paused` | `uint` | `0` = running, `1` = paused |

Reserves slots 0 and 1. Inheriting contracts must start their
storage at slot 2 or higher.

## Source

```fourier
contract Pausable {
    storage owner: address @ 0;
    storage paused: uint @ 1;

    fn init() {
        owner = caller();
        paused = 0;
    }

    pub fn pause() {
        require(caller() == owner);
        paused = 1;
    }

    pub fn unpause() {
        require(caller() == owner);
        paused = 0;
    }

    pub fn is_paused() -> uint {
        return paused;
    }

    pub fn protected_action() -> uint {
        require(paused == 0);
        // ... your logic here ...
        return 1;
    }
}
```

## Selectors

| Selector | Function |
|---|---|
| `0x01` | `pause()` |
| `0x02` | `unpause()` |
| `0x03` | `is_paused() -> uint` |
| `0x04` | `protected_action() -> uint` |

## Usage: inherit-by-copy

Copy the storage decls and the pause/unpause functions, then gate
each mutating function with `require(paused == 0)`:

```fourier
contract Token {
    storage owner:        address @ 0;            // from Pausable
    storage paused:       uint    @ 1;            // from Pausable
    storage total_supply: uint    @ 2;
    storage balances:     map[address, uint] @ 3;

    fn init() {
        owner = caller();
        paused = 0;
        total_supply = 1_000_000;
        balances[caller()] = total_supply;
    }

    pub fn pause() {
        require(caller() == owner);
        paused = 1;
    }

    pub fn unpause() {
        require(caller() == owner);
        paused = 0;
    }

    pub fn transfer(to: address, amount: uint) -> bool {
        require(paused == 0);                      // pausable gate
        let bal: uint = balances[caller()];
        require(bal >= amount);
        balances[caller()] = bal - amount;
        balances[to]       = balances[to] + amount;
        return true;
    }
}
```

## Pattern: pause read paths too?

Whether to gate read-only functions on `paused` is
application-specific. Standard practice is to leave reads open
(users typically need to inspect their balance even when transfers
are paused) and gate only mutating paths.

## Combining with Ownable

Pausable's owner field is the same slot as Ownable's. To use both
`transfer_ownership` and `pause`, copy Ownable's functions alongside
Pausable's; they share storage slot 0:

```fourier
storage owner:  address @ 0;       // from both Pausable and Ownable
storage paused: uint    @ 1;       // from Pausable
// ... your storage at slot 2+ ...

pub fn transfer_ownership(new_owner: address) {     // Ownable
    require(caller() == owner);
    owner = new_owner;
}

pub fn pause() {                                     // Pausable
    require(caller() == owner);
    paused = 1;
}
// ...
```

## Events

Pausable as shipped does not emit events. Add them in the inherited
copy when indexers must track pause state:

```fourier
event Paused(by: address);
event Unpaused(by: address);

pub fn pause() {
    require(caller() == owner);
    paused = 1;
    emit Paused(caller());
}
```
