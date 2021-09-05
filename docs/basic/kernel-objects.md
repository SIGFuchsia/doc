# Kernel Objects

Zircon is an object-based kernel. 

Kernel objects are **ref-counted**. The reference counter increases **both when new handles are created and when a direct pointer reference is acquired**. The reference counter decreases **when handles are detached from the handle table**.

Kernel objects are strongly connected to [rights](https://fuchsia.dev/fuchsia-src/concepts/kernel/rights) and [handles](https://fuchsia.dev/fuchsia-src/concepts/kernel/handles).





## Rights

In Zircon, rights are [32-bit unsigned int](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/system/public/zircon/rights.h;l=10).

The 4 commonly grouped rights are alised as [ZX_RIGHTS_BASIC](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/system/public/zircon/rights.h;l=36)


## Handles

A handle can be thought of as an active session with a specific OS subsystem scoped to a particular resource.



