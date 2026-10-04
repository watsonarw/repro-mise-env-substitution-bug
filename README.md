# Bug Repro: mise `.env` parameter substitution with overrides

## Problem

When `_.file` loads two `.env` files based on `MISE_ENV`, parameter substitution
in the second file resolves against the first file's values instead of its own.

Hypothesis: It seems like the interpolation is happening against the ambient
environment rather than the values in `.env.test`.

Example:

```sh
# .env
VAR_A="base"
VAR_B="base_${VAR_A}"
VAR_C="base_${VAR_B}"
```

```sh
# .env.test
VAR_A="test"
VAR_B="test_${VAR_A}"
VAR_C="test_${VAR_B}"
```

```toml
[env]
_.file = [
  ".env",
  ".env.{{ env.MISE_ENV | default(value='none') }}"
]

[tasks.vars]
run = [
  'echo "VAR_A is $VAR_A"',
  'echo "VAR_B is $VAR_B"',
  'echo "VAR_C is $VAR_C"',
]
```

`MISE_ENV=test mise run vars` gives:

```
VAR_A is test
VAR_B is test_base       # expected test_test
VAR_C is test_base_base  # expected test_test_test
```

`VAR_A` resolves correctly. `VAR_B`/`VAR_C` substitute with `.env`'s values, not
`.env.test`'s.

Output is also inconsistent across runs: editing `.env` and running twice
results in different results on the first compared to subsequent runs

## Reproduction

Create the 3 files above (or clone
[this repro](https://github.com/watsonarw/repro-mise-env-substitution-bug))

### `mise run vars` prints:

```sh
VAR_A is base
VAR_B is base_base
VAR_C is base_base_base
```

as expected.

### `MISE_ENV=test mise run vars` prints

```sh
VAR_A is test
VAR_B is test_base # expected test_test
VAR_C is test_base_base #expected test_test_test
```

### Changing variables on subsequent runs

Comment out `VAR_B="base_${VAR_A}"` in `.env`

Run `MISE_ENV=test mise run vars` prints

```sh
VAR_A is test
VAR_B is test_base
VAR_C is test_base_base # substitutes commented out VAR_B from `.env`
```

Run `MISE_ENV=test mise run vars` again, prints

```sh
VAR_A is test
VAR_B is test_base
VAR_C is test_test_base # substitutes VAR_B from `.env.test`, which is incorrectly substituting `VAR_A` from `.env`
```

## Suspected mechanism

`mise activate`'s hook exports `.env` (no `MISE_ENV`) on every shell prompt

```sh
VAR_A=base
VAR_B=base_base
VAR_C=base_base_base
```

`MISE_ENV=test mise run vars` parses `.env.test` with the ambient environment.

`VAR_B`'s `${VAR_A}` resolves against the ambient `VAR_A=base` rather than
`.env.test`'s own `VAR_A=test`, and `VAR_C` likewise against the ambient
`VAR_B=base_base`.

Editing `.env` changes what the hook exports on the next prompt, which explains
the run-to-run instability with otherwise-unchanged files.

Stripping the ambient vars before running
(`env -u VAR_A -u VAR_B -u VAR_C MISE_ENV=test mise run vars`) produces the
correct output, consistent with this mechanism.
