# `ktrace-dump` utility is not built

`ktrace-dump` is a small utility that convert Fuchsia's ktrace record into human-readable format.

## Reproduce

TODO

## Solution

Using `fx host-tool` command.

```bash
$ cd path/to/fuchsia
#---------------#
# build fuchsia #
#---------------#

# This command will try to build ktrace-dump automatically
$ fx host-tool ktrace-dump
```

![solve-ktrace-dump](../img/solve-ktrace-dump.png)
