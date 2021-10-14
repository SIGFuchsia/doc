# MMU (以 ARM64 为例)

- ISA: ARMv8.0-A



## Basic Configuration

- [有效地址位数: 39 bits](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/arch/arm64/include/arch/arm64/mmu.h;l=56)
- 粒度: 4K page size


## Memory Layout


```
+----------------------+  <- 0x0000_0000_0100_0000 (USER_ASPACE_BASE)
|                      |
|  user address space  |  }  0x0000_ffff_fe00_0000 (USER_ASPACE_SIZE)
|                      |
+----------------------+
|         ...          |
+----------------------+  <- 0xffff_0000_0000_0000 (KERNEL_ASPACE_BASE)
|                      |
| kernel address space |  }  0x0001_0000_0000_0000 (KERNEL_ASPACE_SIZE)
|                      |
+----------------------+
```

Concluded from [kernel_aspace.h](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/arch/arm64/include/arch/kernel_aspace.h).


<!-- ## `ttbr0_el1`

`ttbr0_el1` 寄存器在 Fuchsia 中起到两个作用:

1. 在 Setup MMU 时，`ttbr0_el1` 保存了 trampoline 的页表地址
    - 参考 [zircon/kernel/arch/arm64/start.S](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/arch/arm64/start.S;l=115)
2. 保存用户态的页表地址
    - 参考 [ContextSwitch](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/arch/arm64/mmu.cc;l=1715) 中有关 `ttbr0_el1` 的设置部分
 -->
