Job & Process
=============

本文讨论的核心问题：

1. `Job 和 Process 的不同`
2. `Job 存在的意义`

## 相关链接

- [Job](https://fuchsia.dev/fuchsia-src/reference/kernel_objects/job)
- [Process](https://fuchsia.dev/fuchsia-src/reference/kernel_objects/process)


## Synposis

1. A job is a group of processes and possibly other (child) jobs, every process belongs to a single job.
2. Used to track privileges to perform kernel operations, and track and limit basic resource consumption.
    - Privileges to perform syscalls
    - Memory & CPU usage
3. All jobs form a tree.
4. Every job except the root job belongs to a single (parent) job.



## Job as an Object

### Source Code

- [fuchsia/zircon/kernel/object/include/object/job_dispatcher.h](https://cs.opensource.google/fuchsia/fuchsia/+/main:/zircon/kernel/object/include/object/job_dispatcher.h)
- [fuchsia/zircon/kernel/object/job_dispatcher.cc](https://cs.opensource.google/fuchsia/fuchsia/+/main:zircon/kernel/object/job_dispatcher.cc)

### Member Objects

Job as an object contains the following:

- a reference to a parent job
- a set of child jobs (each of which has this job as its parent)
- a set of member processes
- a set of policies [:warning: not implemented]

All the above elements are highlight

```c++ linenums="216" hl_lines="1 30 31 33"
  const fbl::RefPtr<JobDispatcher> parent_;
  const uint32_t max_height_;

  // The user-friendly job name. For debug purposes only. That
  // is, there is no mechanism to mint a handle to a job via this name.
  fbl::Name<ZX_MAX_NAME_LEN> name_;

  // The common |get_lock()| protects all members below.
  State state_ TA_GUARDED(get_lock());
  int64_t return_code_ TA_GUARDED(get_lock());
  // TODO(cpu): The OOM kill system is incomplete, see fxbug.dev/32577 for details.
  bool kill_on_oom_ TA_GUARDED(get_lock());

  template <typename Ptr, typename Tag>
  using SizedDoublyLinkedList = fbl::DoublyLinkedList<Ptr, Tag, fbl::SizeOrder::Constant,
                                                      fbl::DefaultDoublyLinkedListTraits<Ptr, Tag>>;

  using RawJobList = SizedDoublyLinkedList<JobDispatcher*, RawListTag>;
  using JobList = fbl::TaggedSinglyLinkedList<fbl::RefPtr<JobDispatcher>, ListTag>;

  using RawProcessList =
      SizedDoublyLinkedList<ProcessDispatcher*, ProcessDispatcher::RawJobListTag>;
  using ProcessList =
      fbl::TaggedSinglyLinkedList<fbl::RefPtr<ProcessDispatcher>, ProcessDispatcher::JobListTag>;

  // Access to the pointers in these lists, especially any promotions to
  // RefPtr, must be handled very carefully, because the children can die
  // even when |lock_| is held. See ForEachChildInLocked() for more details
  // and for a safe way to enumerate them.
  RawJobList jobs_ TA_GUARDED(get_lock());
  RawProcessList procs_ TA_GUARDED(get_lock());

  JobPolicy policy_ TA_GUARDED(get_lock());
```

> 236 行和 238 的 `using` 关键字是 C++11 的 type alias 关键字，等同于 `typedef`。

## Possible Operations on Jobs






## Security Concerns

- OOM kill is **incomplete**