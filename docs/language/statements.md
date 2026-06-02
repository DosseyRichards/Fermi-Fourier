# Statements

| Form | Meaning |
|---|---|
| `let NAME : TYPE = EXPR ;` | Declare and initialize a local |
| `let (NAME, ...) : (TYPE, ...) = (EXPR, ...) ;` | Tuple destructuring (literal RHS only) |
| `NAME = EXPR ;` | Assign to local or storage scalar |
| `NAME[K1][K2]... = EXPR ;` | Write to a storage mapping or array slot |
| `NAME.FIELD = EXPR ;` | Write a storage struct field |
| `if EXPR { ... } else { ... }` | Conditional; `else` optional |
| `while EXPR { ... }` | Top-tested loop |
| `return EXPR? ;` | Exit the function (forbidden in `init`) |
| `require(EXPR) ;` | Revert if false |
| `require(EXPR, MSG) ;` | Revert with `bytes` payload if false |
| `emit NAME(EXPR, ...) ;` | Log an event |
| `EXPR ;` | Expression statement; result is popped |

## `let`

```fourier
let x: uint = 1;
let owner_addr: address = caller();
let bal: uint = balances[caller()];
```

A fresh memory slot is allocated for each `let`; see
[Functions / Locals](functions.md#function-locals).

Tuple destructuring requires a literal RHS:

```fourier
let (a, b): (uint, uint) = (10, 20);
```

The compiler emits one `MSTORE` per name. A non-literal RHS — for
example, binding a cross-contract return — is rejected with:

```
tuple destructuring requires a literal tuple RHS
```

## Assignment

```fourier
x = x + 1;                 // local
value = 42;                 // storage scalar (resolved by name lookup)
balances[caller()] = 0;     // storage mapping
queue[i] = 0;               // storage array element
cfg.fee = 100;              // storage struct field
```

For `NAME = EXPR;` the codegen looks up `NAME` in this order:

1. Locals → emit `PUSH offset, MSTORE`.
2. Storage scalars → emit `PUSH slot, SSTORE`.
3. Storage mappings used without `[...]` → compile error
   ("cannot assign to mapping '...' without index").
4. Otherwise → unknown identifier error.

## `if` / `else`

```fourier
if balance >= amount {
    balance = balance - amount;
} else {
    require(false);
}
```

Compiles to:

```text
<cond>
ISZERO
PUSHLABEL else_lbl
JUMPI
<then body>
PUSHLABEL end_lbl
JUMP
else_lbl:
<else body>
end_lbl:
```

The `else` branch is optional; if omitted, the JUMP destination is the
end label directly.

## `while`

```fourier
let i: uint = 0;
while i < 10 {
    sum = sum + arr[i];
    i = i + 1;
}
```

Compiles to:

```text
head:
<cond>
ISZERO
PUSHLABEL end
JUMPI
<body>
PUSHLABEL head
JUMP
end:
```

There is no `break` and no `continue`. Exit by branching out via
`return` or by making the condition false.

## `return`

```fourier
return;                     // void function
return value;
return (a, b);              // tuple return
```

Rules:

- Forbidden inside `init` (the init body must fall through to the
  dispatcher prologue).
- `return EXPR;` in a function declared without `-> TYPE` is rejected.
- Tuple returns require parens.

## `require`

```fourier
require(caller() == owner);
require(amount > 0);
require(verify_sig(1, pk, msg, sig) == 1, error_payload);
```

Compiles to:

```text
<cond>
PUSHLABEL ok
JUMPI
; if a msg arg was given:
<msg_ptr>
DUP 1
MLOAD                ; length
SWAP 1
PUSH 32
ADD                  ; data_offset = ptr + 32
REVERT
; else:
PUSH 0
PUSH 0
REVERT
ok:
```

The `bytes` payload (if any) becomes the revert return-data buffer; the
caller can read it via `return_data` after a failed `call_b` /
`staticcall_b` / `delegatecall_b`.

See [Errors and reverts](errors.md) for details on revert propagation
and state rollback.

## `emit`

```fourier
emit Transfer(from, to, amount);
```

The compiler maps each event to a `SHA3-256` topic of its signature
string (`"Transfer(address,address,uint)"` for the line above) and
emits a `LOG<n>` opcode. Up to 3 args become indexed topics; the rest
go in `data`. See [Events](events.md).

## `expr;`

```fourier
call_b(target, cd, 0, 50000);     // return value discarded
inc_counter();                     // hypothetical — would be discarded
```

The expression is evaluated and its top-of-stack result is popped.
Used for side-effecting calls whose return value is discarded.

## Array builtins

`len`, `push`, `pop` are statement-callable via expression statements:

```fourier
let n: uint = push(queue, 42);    // returns new length
let last: uint = pop(queue);       // returns popped element
let count: uint = len(queue);
```

The first argument must be a bare storage-array name (a `Var` resolving
to a storage decl with `array[T]` type). Anything else raises:

```
push(...) requires a storage array as its first arg
```

`pop` on an empty array reverts; the bounds check is emitted as a
runtime `ISZERO` + conditional revert.
