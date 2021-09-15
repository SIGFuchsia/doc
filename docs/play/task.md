# Program execution.

- Job.

  **Firstly, in front of the JobDispatcher, the author says.**

  ```c++
  // This class implements the Job object kernel interface. Each Job has a parent
  // Job and zero or more child Jobs and zero or more Child processes. This
  // creates a DAG (tree) that connects every living task in the system.
  // This is critically important because of the bottoms up refcount nature of
  // the system in which the scheduler keeps alive the thread and the thread keeps
  // alive the process, so without the Job it would not be possible to enumerate
  // or control the tasks in the system for which there are no outstanding handles.
  //
  // The second important job of the Job is to apply policies that cannot otherwise
  // be easily enforced by capabilities, for example kernel object creation.
  //
  // The third one is to support exception propagation from the leaf tasks to
  // the root tasks.
  //
  // Obviously there is a special case for the 'root' Job which its parent is null
  // and in the current implementation will call platform_halt() when its process
  // and job count reaches zero. The root job is not exposed to user mode, instead
  // the single child Job of the root job is given to the userboot process.
  ```

  - Create a Job. Get current process. Lookup the current process handle table to check if the reference has `ZX_RIGHT_MANAGE_JOB`. Get the pointer to the parent Job. Create a new Job with a kernel handle implementation and the rights. And then make the integer handle for user space. Here, the option is not used by the constructor of JobDispatcher and **must be 0**. The `RefPtr` versus `shared_ptr` in C++ is in https://chunminchang.github.io/blog/post/refptr-v-s-shared-ptr. And `handle->HasRights(desired_rights)` is for rights checking.

    ```c++
    zx_status_t sys_job_create(zx_handle_t parent_job, uint32_t options, user_out_handle* out) {
    
      auto up = ProcessDispatcher::GetCurrent();
    
      fbl::RefPtr<JobDispatcher> parent;
      zx_status_t status =
          up->handle_table().GetDispatcherWithRights(parent_job, ZX_RIGHT_MANAGE_JOB, &parent);
    
      KernelHandle<JobDispatcher> handle;
      zx_rights_t rights;
      status = JobDispatcher::Create(options, ktl::move(parent), &handle, &rights);
      if (status == ZX_OK)
        status = out->make(ktl::move(handle), rights);
      return status;
    }
    ```

    Here, I am interested in `JobDispatcher::Create` and `out->make`. We can see the Job tree has a max height. The constructor of `new_handle` and `default_rights` are interesting. In fact, the `default_rights()` is in `//zircon/kernel/object/include/object/dispatcher.h`. It returns `def_rights` which is defined in the template. For JobDispatcher, `ZX_DEFAULT_JOB_RIGHTS` is the right.

    ```c++
    zx_status_t JobDispatcher::Create(uint32_t flags, const fbl::RefPtr<JobDispatcher>& parent,
                                      KernelHandle<JobDispatcher>* handle, zx_rights_t* rights) {
      if (parent != nullptr && parent->max_height() == 0) {
        return ZX_ERR_OUT_OF_RANGE;
      }
    
      fbl::AllocChecker ac;
      KernelHandle new_handle(
          fbl::AdoptRef(new (&ac) JobDispatcher(flags, parent, parent->GetPolicy())));
      if (!ac.check())
        return ZX_ERR_NO_MEMORY;
    
      if (!parent->AddChildJob(new_handle.dispatcher())) {
        return ZX_ERR_BAD_STATE;
      }
    
      *rights = default_rights();
      *handle = ktl::move(new_handle);
      return ZX_OK;
    }
    ```

    For the KernelHandle (a representation of handle in kernel space), is just a class, with a reference pointer (`fbl::RefPtr<T> dispatcher_`) to the object. The author says as below. Here, the handler is only a kernel object with a reference. Only before it is given to the process will it have rights and owner.

    ```c++
    // A minimal wrapper around a Dispatcher which is owned by the kernel.
    //
    // Intended usage when creating new a Dispatcher object is:
    //   1. Create a KernelHandle on the stack (cannot fail)
    //   2. Move the RefPtr<Dispatcher> into the KernelHandle (cannot fail)
    //   3. When ready to give the handle to a process, upgrade the KernelHandle
    //      to a full HandleOwner via UpgradeToHandleOwner() or
    //      user_out_handle::make() (can fail)
    //
    // This sequence ensures that the Dispatcher's on_zero_handles() method is
    // called even if errors occur during or before HandleOwner creation, which
    // is necessary to break circular references for some Dispatcher types.
    //
    // This class is thread-unsafe and must be externally synchronized if used
    // across multiple threads.
    ```

    For the handle's owner, use `user_out_handle::make(ktl::move(handle), rights)` as an example. The definition is in `//zircon/kernel/lib/syscalls/priv.h`. It uses the KernelHandle or Dispatcher reference along with rights to `Make()` a Handle.

    ```c++
    // This is the type of handle result parameters in system call
    // implementation functions (sys_*).  kazoo recognizes return values of
    // type zx_handle_t and converts them into user_out_handle* instead of into
    // user_out_ptr<zx_handle_t>.  System call implementation functions use the
    // make, dup, or transfer method to turn a Dispatcher pointer or another
    // handle into a handle received by the user.
    class user_out_handle final {
     public:
      zx_status_t make(fbl::RefPtr<Dispatcher> dispatcher, zx_rights_t rights) {
        h_ = Handle::Make(ktl::move(dispatcher), rights);
        return h_ ? ZX_OK : ZX_ERR_NO_MEMORY;
      }
    
      // Note that if this call fails to allocate the Handle, the underlying
      // Dispatcher's on_zero_handles() will be called.
      zx_status_t make(KernelHandle<Dispatcher> handle, zx_rights_t rights) {
        h_ = Handle::Make(ktl::move(handle), rights);
        return h_ ? ZX_OK : ZX_ERR_NO_MEMORY;
      }
      
      ...
      private:
        HandleOwner h_;
    }
    ```

    A Handle is how a specific process refers to a specific Dispatcher. HandleOwner wraps a Handle in a `unique_ptr` that has single ownership of the Handle and deletes it whenever it falls out of scope (`HandleOwner = ktl::unique_ptr<Handle, HandleDestroyer>`). Handles should only be created by `Make` or `Dup(licate)`.

    In `//zircon/kernel/object/include/object/handle.h`, the code is.

    ```c++
    HandleOwner Handle::Make(KernelHandle<Dispatcher> kernel_handle, zx_rights_t rights) {
      uint32_t base_value;
      void* addr = gHandleTableArena.Alloc(kernel_handle.dispatcher(), "new", &base_value);
      if (unlikely(!addr))
        return nullptr;
      kcounter_add(handle_count_made, 1);
      kcounter_add(handle_count_live, 1);
      return HandleOwner(new (addr) Handle(kernel_handle.release(), rights, base_value));
    }
    
    Handle::Handle(fbl::RefPtr<Dispatcher> dispatcher, zx_rights_t rights, uint32_t base_value)
        : process_id_(ZX_KOID_INVALID),
          dispatcher_(ktl::move(dispatcher)),
          rights_(rights),
          base_value_(base_value) {}
    ```

    When a JobDispatcher is constructed and added to its parent Job, the new job is after the parent's next-youngest child, or us if we have none. The handle can **only get a RefPtr** of JobDispatcher, which makes sense. So we use `handle.dispatcher.get` to get the pointer of the Job, which is `JobDispatcher*`.

     ```c++
    JobDispatcher::JobDispatcher(uint32_t /*flags*/, fbl::RefPtr<JobDispatcher> parent,
                                 JobPolicy policy)
        : SoloDispatcher(ZX_JOB_NO_PROCESSES | ZX_JOB_NO_JOBS | ZX_JOB_NO_CHILDREN),
          parent_(ktl::move(parent)),
          max_height_(parent_ ? parent_->max_height() - 1 : kRootJobMaxHeight),
          state_(State::READY),
          return_code_(0),
          kill_on_oom_(false),
          policy_(policy),
          exceptionate_(ZX_EXCEPTION_CHANNEL_TYPE_JOB),
          debug_exceptionate_(ZX_EXCEPTION_CHANNEL_TYPE_JOB_DEBUGGER) {
      kcounter_add(dispatcher_job_create_count, 1);
    }
     ```

- Process.

  First, the definition of ProcessDispatcher is in `//zircon/kernel/object/include/object/process_dispatcher.h`.

  - Create a process.

    ```c++
    zx_status_t sys_process_create(zx_handle_t job_handle, user_in_ptr<const char> _name,
                                   size_t name_len, uint32_t options, user_out_handle* proc_handle,
                                   user_out_handle* vmar_handle) {
      // some basic things
    
      // copy out the name
      char buf[ZX_MAX_NAME_LEN];
      ktl::string_view sp;
      result = copy_user_string(_name, name_len, buf, sizeof(buf), &sp);
    
      fbl::RefPtr<JobDispatcher> job;
      auto status =
          up->handle_table().GetDispatcherWithRights(job_handle, ZX_RIGHT_MANAGE_PROCESS, &job);
    
      // create a new process dispatcher
      KernelHandle<ProcessDispatcher> new_process_handle;
      KernelHandle<VmAddressRegionDispatcher> new_vmar_handle;
      zx_rights_t proc_rights, vmar_rights;
      result = ProcessDispatcher::Create(ktl::move(job), sp, options, &new_process_handle, &proc_rights,
                                         &new_vmar_handle, &vmar_rights);
    
      result = proc_handle->make(ktl::move(new_process_handle), proc_rights);
      if (result == ZX_OK)
        result = vmar_handle->make(ktl::move(new_vmar_handle), vmar_rights);
      return result;
    }
    ```

    In this process creation, `ProcessDispatcher::Create` is the core function.

    In the creation of ProcessDispatcher.

    ```c++
    zx_status_t ProcessDispatcher::Create(fbl::RefPtr<JobDispatcher> job, ktl::string_view name,
                                          uint32_t flags, KernelHandle<ProcessDispatcher>* handle,
                                          zx_rights_t* rights,
                                          KernelHandle<VmAddressRegionDispatcher>* root_vmar_handle,
                                          zx_rights_t* root_vmar_rights) {
      fbl::AllocChecker ac;
      KernelHandle new_handle(fbl::AdoptRef(new (&ac) ProcessDispatcher(job, name, flags)));
    
      zx_status_t result = new_handle.dispatcher()->Initialize();
    
      // Create a dispatcher for the root VMAR.
      KernelHandle<VmAddressRegionDispatcher> new_vmar_handle;
      result = VmAddressRegionDispatcher::Create(new_handle.dispatcher()->aspace()->RootVmar(),
                                                 ARCH_MMU_FLAG_PERM_USER, &new_vmar_handle,
                                                 root_vmar_rights);
    
      // Only now that the process has been fully created and initialized can we register it with its
      // parent job. We don't want anyone to see it in a partially initalized state.
      if (!job->AddChildProcess(new_handle.dispatcher())) {
        return ZX_ERR_BAD_STATE;
      }
    
      *rights = default_rights();
      *handle = ktl::move(new_handle);
      *root_vmar_handle = ktl::move(new_vmar_handle);
    
      return ZX_OK;
    }
    ```

    Here, we are interested in 2 functionalities. What initialization should the process do (`dispatcher()->Initialize()`)? How does the `VmAddressRegionDispatcher::Create` work and how it relates to the new process?

    Here comes the initialization step. The calling tree is `ProcessDiapatcher::Initialize`->`VmAspace::Create`->`VmAspace::Init`. Specifically, the constructor of VmAspace is a initialization list as below.

    ```c++
    VmAspace::VmAspace(vaddr_t base, size_t size, uint32_t flags, const char* name)
        : base_(base),
          size_(size),
          flags_(flags),
          root_vmar_(nullptr),
          aslr_prng_(nullptr, 0),
          arch_aspace_(base, size, arch_aspace_flags_from_flags(flags)) {
      DEBUG_ASSERT(size != 0);
      DEBUG_ASSERT(base + size - 1 >= base);
    
      Rename(name);
    }
    ```

    ```c++
    // Fun 1
    zx_status_t ProcessDispatcher::Initialize() {
      LTRACE_ENTRY_OBJ;
    
      Guard<Mutex> guard{get_lock()};
    
      // create an address space for this process, named after the process's koid.
      char aspace_name[ZX_MAX_NAME_LEN];
      snprintf(aspace_name, sizeof(aspace_name), "proc:%" PRIu64, get_koid());
      aspace_ = VmAspace::Create(VmAspace::TYPE_USER, aspace_name);
      if (!aspace_) {
        return ZX_ERR_NO_MEMORY;
      }
    
      return ZX_OK;
    }
    
    // Fun 2
    fbl::RefPtr<VmAspace> VmAspace::Create(uint32_t flags, const char* name) {
      LTRACEF("flags 0x%x, name '%s'\n", flags, name);
    
      vaddr_t base;
      size_t size;
      switch (flags & TYPE_MASK) {
        case TYPE_USER:
          base = USER_ASPACE_BASE;
          size = USER_ASPACE_SIZE;
          break;
        case TYPE_KERNEL:
          base = KERNEL_ASPACE_BASE;
          size = KERNEL_ASPACE_SIZE;
          break;
        case TYPE_LOW_KERNEL:
          base = 0;
          size = USER_ASPACE_BASE + USER_ASPACE_SIZE;
          break;
        case TYPE_GUEST_PHYS:
          base = GUEST_PHYSICAL_ASPACE_BASE;
          size = GUEST_PHYSICAL_ASPACE_SIZE;
          break;
        default:
          panic("Invalid aspace type");
      }
    
      fbl::AllocChecker ac;
      auto aspace = fbl::AdoptRef(new (&ac) VmAspace(base, size, flags, name));
      if (!ac.check()) {
        return nullptr;
      }
    
      // initialize the arch specific component to our address space
      zx_status_t status = aspace->Init();
      if (status != ZX_OK) {
        status = aspace->Destroy();
        DEBUG_ASSERT(status == ZX_OK);
        return nullptr;
      }
    
      // add it to the global list
      {
        Guard<Mutex> guard{&aspace_list_lock};
        aspaces.push_back(aspace.get());
      }
    
      // return a ref pointer to the aspace
      return aspace;
    }
    ```

    The third function in the calling tree is worth mentioning. For the architectural specific part, `arch_aspace_.Init`, if we taken X86 as an example, we can find the function `X86ArchVmAspace::Init` in `//zircon/kernel/arch/x86/mmu.cc`. Zircon uses **ASLR** to mitigate spatial memory corruption. For a process, the root VMAR covers the entire address space.

    ```c++
    zx_status_t VmAspace::Init() {
      canary_.Assert();
    
      // initialize the architecturally specific part
      zx_status_t status = arch_aspace_.Init();
    
      InitializeAslr();
    
      if (likely(!root_vmar_)) {
        return VmAddressRegion::CreateRoot(*this, VMAR_FLAG_CAN_MAP_SPECIFIC, &root_vmar_);
      }
      return ZX_OK;
    }
    ```

    ```c++
    /*
     * Fill in the high level x86 arch aspace structure and allocating a top level page table.
     */
    zx_status_t X86ArchVmAspace::Init() {
      static_assert(sizeof(cpu_mask_t) == sizeof(active_cpus_), "err");
      canary_.Assert();
    
      LTRACEF("aspace %p, base %#" PRIxPTR ", size 0x%zx, mmu_flags 0x%x\n", this, base_, size_,
              flags_);
    
      if (flags_ & ARCH_ASPACE_FLAG_KERNEL) {
        X86PageTableMmu* mmu = new (&page_table_storage_.mmu) X86PageTableMmu();
        pt_ = mmu;
    
        zx_status_t status = mmu->InitKernel(this, test_page_alloc_func_);
        if (status != ZX_OK) {
          return status;
        }
        LTRACEF("kernel aspace: pt phys %#" PRIxPTR ", virt %p\n", pt_->phys(), pt_->virt());
      } else if (flags_ & ARCH_ASPACE_FLAG_GUEST) {
        X86PageTableEpt* ept = new (&page_table_storage_.ept) X86PageTableEpt();
        pt_ = ept;
    
        zx_status_t status = ept->Init(this, test_page_alloc_func_);
        if (status != ZX_OK) {
          return status;
        }
        LTRACEF("guest paspace: pt phys %#" PRIxPTR ", virt %p\n", pt_->phys(), pt_->virt());
      } else {
        X86PageTableMmu* mmu = new (&page_table_storage_.mmu) X86PageTableMmu();
        pt_ = mmu;
    
        zx_status_t status = mmu->Init(this, test_page_alloc_func_);
        if (status != ZX_OK) {
          return status;
        }
    
        status = mmu->AliasKernelMappings();
        if (status != ZX_OK) {
          return status;
        }
    
        LTRACEF("user aspace: pt phys %#" PRIxPTR ", virt %p\n", pt_->phys(), pt_->virt());
      }
      ktl::atomic_init(&active_cpus_, 0);
    
      return ZX_OK;
    }
    ```

  - Start a process.

    The first argument (`arg1`) is a handle, which will be transferred from the process of the caller to the process being started, and an appropriate handle value will be placed in `arg1` for the newly started thread. `zx_process_start()` is the only way to transfer a handle into a process that doesn't involve the process making some system call using a handle it already has. A process with no handles can make the few system calls that don't require a handle.

    So, the main question is, can we give an example to specify what `arg1` and `arg2` are. According to the tests in Fuchsia, most `arg1s` are Events defined in Zircon. <font color='red'>Here, currently I have no idea why we give the new process an Event handle and how to use this to, let's say, get the handle of its parent job and create another job or process.</font>

    ```c++
    zx_status_t sys_process_start(zx_handle_t process_handle, zx_handle_t thread_handle, zx_vaddr_t pc,
                                  zx_vaddr_t sp, zx_handle_t arg_handle_value, uintptr_t arg2) {
    
      auto up = ProcessDispatcher::GetCurrent();
    
      // get process dispatcher
      fbl::RefPtr<ProcessDispatcher> process;
      zx_status_t status =
          up->handle_table().GetDispatcherWithRights(process_handle, ZX_RIGHT_WRITE, &process);
    
      // get thread_dispatcher
      fbl::RefPtr<ThreadDispatcher> thread;
      status = up->handle_table().GetDispatcherWithRights(thread_handle, ZX_RIGHT_WRITE, &thread);
    
      HandleOwner arg_handle = up->handle_table().RemoveHandle(arg_handle_value);
    
      // test that the thread belongs to the starting process
      if (thread->process() != process.get())
        return ZX_ERR_ACCESS_DENIED;
    
      zx_handle_t arg_nhv = ZX_HANDLE_INVALID;
      if (arg_handle) {
        arg_nhv = process->handle_table().MapHandleToValue(arg_handle);
        process->handle_table().AddHandle(ktl::move(arg_handle));
      }
    
      status =
          thread->Start(ThreadDispatcher::EntryState{pc, sp, static_cast<uintptr_t>(arg_nhv), arg2},
                        /* initial_thread */ true);
    
      return ZX_OK;
    }
    ```

- Thread.

  To begin with, Zircon Threads are detached (not need to wait for join to release resource), by POSIX API may needs join.

  - Create and start.

    Thread creation is similar. And the `options` must be 0 as well. The difference lies in the Processes and Jobs are listed in Jobs as a fat-wide tree structure. So, we only need to dive into `ThreadDispatcher::Create` and `handle.dispatcher()->Initialize()`.

    ```c++
    zx_status_t sys_thread_create(zx_handle_t process_handle, user_in_ptr<const char> _name,
                                  size_t name_len, uint32_t options, user_out_handle* out) {
    
      // copy out the name
      char buf[ZX_MAX_NAME_LEN];
      ktl::string_view sp;
      zx_status_t result = copy_user_string(_name, name_len, buf, sizeof(buf), &sp);
    
      // convert process handle to process dispatcher
      auto up = ProcessDispatcher::GetCurrent();
    
      fbl::RefPtr<ProcessDispatcher> process;
      result =
          up->handle_table().GetDispatcherWithRights(process_handle, ZX_RIGHT_MANAGE_THREAD, &process);
    
      // create the thread dispatcher
      KernelHandle<ThreadDispatcher> handle;
      zx_rights_t thread_rights;
      result = ThreadDispatcher::Create(ktl::move(process), options, sp, &handle, &thread_rights);
      if (result != ZX_OK)
        return result;
    
      result = handle.dispatcher()->Initialize();
    
      return out->make(ktl::move(handle), thread_rights);
    }
    ```

    In Thread creation, first we need to know how Process and Thread are related. Then, we must know the definition of `user_thread` and `core_thread`. Thread knows its parent's Process and the Process can access all threads from the first one by one. `user_thread` is the dispatcher and `core_thread` is for kernel and scheduler. In the creation, the pointer `t` is nullptr. Further reading about `core_thread` will be in scheduling.

    ```c++
    zx_status_t ThreadDispatcher::Create(fbl::RefPtr<ProcessDispatcher> process, uint32_t flags,
                                         ktl::string_view name,
                                         KernelHandle<ThreadDispatcher>* out_handle,
                                         zx_rights_t* out_rights) {
      // Create the user-mode thread and attach it to the process and lower level thread.
      fbl::AllocChecker ac;
      auto user_thread = fbl::AdoptRef(new (&ac) ThreadDispatcher(process, flags));
    
      // Create the lower level thread and attach it to the scheduler.
      Thread* core_thread =
          Thread::Create(name.data(), StartRoutine, user_thread.get(), DEFAULT_PRIORITY);
    
      // We haven't yet compeleted initialization of |user_thread|, and
      // references to it haven't possibly escaped this thread. We can
      // safely set |core_thread_| outside the lock.
      [&user_thread,
       &core_thread]() TA_NO_THREAD_SAFETY_ANALYSIS { user_thread->core_thread_ = core_thread; }();
    
      // The syscall layer will call Initialize(), which used to be called here.
    
      *out_rights = default_rights();
      *out_handle = KernelHandle(ktl::move(user_thread));
      return ZX_OK;
    }
    ```

    ```c++
    Thread* Thread::CreateEtc(Thread* t, const char* name, thread_start_routine entry, void* arg,
                              int priority, thread_trampoline_routine alt_trampoline) {
      unsigned int flags = 0;
    
      if (!t) {
        t = static_cast<Thread*>(memalign(alignof(Thread), sizeof(Thread)));
        if (!t) {
          return nullptr;
        }
        flags |= THREAD_FLAG_FREE_STRUCT;
      }
    
      init_thread_struct(t, name);
    
      t->task_state_.Init(entry, arg);
      Scheduler::InitializeThread(t, priority);
    
      zx_status_t status = t->stack_.Init();
      if (status != ZX_OK) {
        if (flags & THREAD_FLAG_FREE_STRUCT) {
          free(t);
        }
        return nullptr;
      }
    
      // save whether or not we need to free the thread struct and/or stack
      t->flags_ = flags;
    
      if (likely(alt_trampoline == nullptr)) {
        alt_trampoline = &Thread::Trampoline;
      }
    
      // set up the initial stack frame
      arch_thread_initialize(t, (vaddr_t)alt_trampoline);
    
      // add it to the global thread list
      {
        Guard<MonitoredSpinLock, IrqSave> guard{ThreadLock::Get(), SOURCE_TAG};
        thread_list->push_front(t);
      }
    
      kcounter_add(thread_create_count, 1);
      return t;
    }
    ```

    When starting a Thread, it finally calls `Start`. We can see, the initial thread must be started by the Process, and others must be started by a Thread. Of-course, `arg1` and `arg2` are directly sent to the entry function.

    ```c++
    zx_status_t sys_thread_start(zx_handle_t handle, zx_vaddr_t thread_entry, zx_vaddr_t stack,
                                 uintptr_t arg1, uintptr_t arg2) {
    
      auto up = ProcessDispatcher::GetCurrent();
    
      fbl::RefPtr<ThreadDispatcher> thread;
      zx_status_t status =
          up->handle_table().GetDispatcherWithRights(handle, ZX_RIGHT_MANAGE_THREAD, &thread);
    
      return thread->Start(ThreadDispatcher::EntryState{thread_entry, stack, arg1, arg2},
                           /* initial_thread= */ false);
    }
    ```

  - Access thread state system calls are for debugging and can be accessed only when the thread is suspended or has an exception.
