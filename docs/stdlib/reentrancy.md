# ReentrancyGuard

Source: `fourier/stdlib/reentrancy_guard.fou`.

A mutex flag that prevents a function from being re-entered during
its own execution. Set the flag before any external call, clear it
after.

## Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `_locked` | `uint` | `0` = unlocked, `1` = in protected section |
| `1` | `balances` | `map[address, uint]` | Example user balances |

The shipped contract includes a `balances` mapping at slot 1 as a
demonstration. When you inherit-by-copy, keep `_locked` (it's the
mutex you need) and replace the rest with your own storage.

## Source

```fourier
contract ReentrancyGuard {
    storage _locked: uint @ 0;
    storage balances: map[address, uint] @ 1;

    pub fn withdraw(amount: uint) -> uint {
        // ── Reentrancy guard: enter ──
        require(_locked == 0);
        _locked = 1;

        let bal: uint = balances[caller()];
        require(bal >= amount);
        balances[caller()] = bal - amount;

        // External call (the danger zone)
        let cd: bytes = pack_sel(1, amount);
        let ok: uint = call_b(caller(), cd, amount, 50000);
        require(ok == 1);

        // ── Reentrancy guard: exit ──
        _locked = 0;
        return 1;
    }

    pub fn deposit() -> uint {
        balances[caller()] = balances[caller()] + callvalue();
        return 1;
    }

    pub fn balance_of(addr: address) -> uint {
        return balances[addr];
    }
}
```

## Selectors

| Selector | Function |
|---|---|
| `0x01` | `withdraw(uint) -> uint` |
| `0x02` | `deposit() -> uint` |
| `0x03` | `balance_of(address) -> uint` |

## Why this matters

A classic reentrancy attack works like this:

1. Attacker calls a victim contract's `withdraw(100)`.
2. Victim reads attacker's balance (100), then sends 100 to attacker
   via `call_b`.
3. The "send" reaches the attacker's contract, whose fallback /
   selector-matching function calls `withdraw(100)` **again**.
4. Victim's balance read still shows 100 (the first decrement hasn't
   happened yet) and pays out another 100.
5. Step 3–4 repeats until victim is drained.

The classic fix is the "checks-effects-interactions" pattern: do all
state mutations **before** the external call. ReentrancyGuard is a
belt-and-suspenders second line.

## Pattern: minimal guard

For any function that makes an external `call_b` / `delegatecall_b`,
copy this skeleton:

```fourier
storage _locked: uint @ 0;

pub fn risky_action(...) -> uint {
    require(_locked == 0);            // enter
    _locked = 1;

    // ... read state, mutate state ...
    let ok: uint = call_b(target, cd, value, gas);
    require(ok == 1);
    // ... possibly more state changes ...

    _locked = 0;                       // exit
    return 1;
}
```

## What the guard prevents

- A reentrant call into the same function (the `_locked == 0`
  precondition fails on the second entry).
- A reentrant call into a **different** function that shares the same
  `_locked` flag — useful for inter-function protection.

What it does **not** prevent:

- Read-only reentrancy. A `staticcall_b` into the contract during the
  external call will still succeed, exposing inconsistent state to
  the caller. If you care about read-consistency, restructure so the
  external call happens last.
- Cross-function attacks where another contract you call back into
  yourself with a different selector — only if you don't share the
  `_locked` flag across functions. The standard is to use one flag for
  every state-mutating function on the contract.

## Gas notes

The guard adds:

- 1× `SLOAD` (200 gas) + 1× equality check on entry.
- 1× `SSTORE` non-zero → existing-non-zero (5000 gas) on entry.
- 1× `SSTORE` non-zero → zero (5000 gas, no refund) on exit.

Roughly **10,200 gas per protected call**. Cheap relative to the cost
of an external call (700+ forwarded gas), free relative to the cost of
a reentrancy-induced drain.

## Combining with Pausable

If you've also copied Pausable (slot 0 = owner, slot 1 = paused),
move `_locked` to a free slot:

```fourier
storage owner:    address @ 0;       // from Pausable
storage paused:   uint    @ 1;       // from Pausable
storage _locked:  uint    @ 2;       // from ReentrancyGuard, relocated
storage balances: map[address, uint] @ 3;
```

The protected-function pattern remains identical — just reference the
relocated `_locked`.
