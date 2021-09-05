# Syscalls

The syscalls in Fuchsia is bit of weird. The implementations have different function names from users' entries.

Fuchsia uses [kazoo](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/tools/kazoo) to generate syscall logic.


## Enforcement

The vDSO entry points are the only means to enter the kernel for system calls.

[vdso-loading](../img/vdso-loading.png)

## Implementation

The syscall implementation is a hand-written function with the naming convention `sys_<syscall>`




## Reference

[Life of a Fuchsia syscall](https://fuchsia.dev/fuchsia-src/concepts/kernel/life_of_a_syscall?hl=en)
