# CLI usage

Fourier v1 does not ship a standalone `fourierc` binary. Until one
lands, compile from the command line by invoking the package directly.

## One-liner: stdin → stdout

```bash
python -c 'from fourier import compile_source; import sys; sys.stdout.buffer.write(compile_source(sys.stdin.read()))' \
  < counter.fou \
  > counter.bin
```

This reads the `.fou` file from stdin, writes the raw bytecode (binary)
to stdout. Redirect to a file or pipe to `xxd` to inspect.

## One-liner: hex output

```bash
python -c 'from fourier import compile_source; import sys; print(compile_source(sys.stdin.read()).hex())' \
  < counter.fou
```

Prints hex on a single line — useful for embedding in a deploy tx's
`data.code` field.

## Reusable shell function

Add to `~/.zshrc` or `~/.bashrc`:

```bash
fourierc() {
  python -c "
from fourier import compile_source
import sys
src = open(sys.argv[1]).read()
out = compile_source(src)
sys.stdout.buffer.write(out)
" "$1"
}

fourierc-hex() {
  python -c "
from fourier import compile_source
import sys
src = open(sys.argv[1]).read()
print(compile_source(src).hex())
" "$1"
}
```

Then:

```bash
fourierc-hex counter.fou
# 010280...
fourierc counter.fou > counter.bin
```

## What can go wrong

Compilation errors are raised as Python exceptions. Run with `-c` and
they'll print a traceback:

```text
Traceback (most recent call last):
  ...
fourier.parser.ParseError: 5:12: expected SEMI (at LBRACE)
```

The leading numbers are `line:col` from the source file.

See [Error reference](errors.md) for the full catalog of error types.

## Reading the bytecode

The output is raw VM bytecode (the same byte sequence that goes into
an account's `code` field). To inspect:

```bash
xxd counter.bin | head
# 00000000: 0102 ff00 0000 0000 0000 0000 0000 0000  ................
# ...
```

Or pipe through a disassembler — see the bytecode-walk in
`vm/machine.py` for the opcode table.

## Deploying compiled bytecode

Wrap the hex output in a deploy tx (per
[https://docs.waveledger.net/reference/tx/](https://docs.waveledger.net/reference/tx/#contract-deploy)):

```json
{
  "type": "deploy",
  "code": "<bytecode hex from fourierc-hex>",
  "gas_limit": 1000000,
  "value": 0,
  "init_calldata": ""
}
```

The recipient is `"contract"`. The deployer's nonce determines the
deterministic deploy address (see `vm/deploy.py::contract_address`).
