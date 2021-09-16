# Handle management and rights.

- When is a handle bound to a process? 

  When allocated, the owner `process_id_` is `ZX_KOID_INVALID`. We can see the source code in `//zircon/kernel/lib/syscalls/priv.h`. Kazoo will help do this automatically.

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
  
    zx_status_t dup(Handle* source, zx_rights_t rights) {
      h_ = Handle::Dup(source, rights);
      return h_ ? ZX_OK : ZX_ERR_NO_MEMORY;
    }
  
    zx_status_t transfer(HandleOwner&& source) {
      h_.swap(source);
      return ZX_OK;
    }
  
    // These methods are called by the kazoo-generated wrapper_* functions
    // (syscall-kernel-wrappers.inc).  See KernelWrapperGenerator::syscall.
  
    bool begin_copyout(ProcessDispatcher* current_process, user_out_ptr<zx_handle_t> out) const {
      if (h_)
        return out.copy_to_user(current_process->handle_table().MapHandleToValue(h_));
      return false;
    }
  
    void finish_copyout(ProcessDispatcher* current_process) {
      if (h_)
        current_process->handle_table().AddHandle(ktl::move(h_));
    }
  
   private:
    HandleOwner h_;
  };
  ```

- How does handle or right transfer? Is there a time when the handle belongs to both processes? For a handle, if we want to duplicate or transfer it (not create a default one), can we add additional rights?

  First, handles with transfer right can be transfered via `zx_channel_write` (gives the handle from the calling process to the kernel) and `zx_channel_read/zx_channel_call` (binds the handle to the calling process). The channel write function is in `//zircon/kernel/lib/syscalls/channel.cc`. In fact, this kind of input arguments remind me of buffer over-read. We check `RemoveUserHandles()` and `msg_put_handles()` to check if the kernel gives the protection. When we read from a channel, the message includes the reference of the original owner and the handle (in `zx_handle_t` state).

  `zx_channel_write()` attempts to write a message of `num_bytes` bytes and `num_handles` handles to the channel specified by `handle`. The pointers `handles` and `bytes` may be NULL if their respective sizes are zero.

  On success, all `num_handles` of the handles in the `handles` array are attached to the message and will become available to the reader of that message from the opposite end of the channel.

  All handles are discarded and no longer available to the caller, on success or failure.

  The pointer of `zx_handle_disposition` is `UserHandles` in the function. In the structure, handle is the user provided integer, operation is `ZX_HANDLE_OP_MOVE` or `ZX_HANDLE_OP_DUPLICATE`, type is used to perform validation of the object type that the caller expects handle to be, rights are the desired rights. **Handle will be transferred with capability rights which can be `ZX_RIGHT_SAME_RIGHTS` or a reduced set of rights, or `ZX_RIGHT_NONE`.**

  ```c++
  template <typename UserHandles>
  static zx_status_t channel_write(zx_handle_t handle_value, uint32_t options,
                                   user_in_ptr<const void> user_bytes, uint32_t num_bytes,
                                   UserHandles user_handles, uint32_t num_handles) {
  
    auto up = ProcessDispatcher::GetCurrent();
  
    auto cleanup = fit::defer([&]() { RemoveUserHandles(user_handles, num_handles, up); });
  
    fbl::RefPtr<ChannelDispatcher> channel;
    zx_status_t status =
        up->handle_table().GetDispatcherWithRights(handle_value, ZX_RIGHT_WRITE, &channel);
  
    MessagePacketPtr msg;
    if ((options & ZX_CHANNEL_WRITE_USE_IOVEC) != 0) {
      status = MessagePacket::Create(user_bytes.reinterpret<const zx_channel_iovec_t>(), num_bytes,
                                     num_handles, &msg);
    } else {
      status =
          MessagePacket::Create(user_bytes.reinterpret<const char>(), num_bytes, num_handles, &msg);
    }
  
    if (num_handles > 0u) {
      status = msg_put_handles(up, msg.get(), user_handles, num_handles,
                               static_cast<Dispatcher*>(channel.get()));
    }
  
    cleanup.cancel();
  
    status = channel->Write(up->get_koid(), ktl::move(msg));
  
    return ZX_OK;
  }
  ```

  ```c++
  typedef struct zx_handle_disposition {
      zx_handle_op_t operation;
      zx_handle_t handle;
      zx_obj_type_t type;
      zx_rights_t rights;
      zx_status_t result;
  } zx_handle_disposition_t;
  
  // In msg_put_handles() -> get_handle_for_message_locked() -> handle_checks_locked()
  // , the handle's rights must be reduced or keep same.
  static zx_status_t handle_checks_locked(const Handle* handle, const Dispatcher* channel,
                                          zx_handle_op_t operation, zx_rights_t desired_rights,
                                          zx_obj_type_t type) {
    if (!handle)
      return ZX_ERR_BAD_HANDLE;
    if (!handle->HasRights(ZX_RIGHT_TRANSFER))
      return ZX_ERR_ACCESS_DENIED;
    if (handle->dispatcher().get() == channel)
      return ZX_ERR_NOT_SUPPORTED;
    if (type != ZX_OBJ_TYPE_NONE && handle->dispatcher()->get_type() != type)
      return ZX_ERR_WRONG_TYPE;
    if (operation != ZX_HANDLE_OP_MOVE && operation != ZX_HANDLE_OP_DUPLICATE)
      return ZX_ERR_INVALID_ARGS;
    if (desired_rights != ZX_RIGHT_SAME_RIGHTS) {
      if ((handle->rights() & desired_rights) != desired_rights) {
        return ZX_ERR_INVALID_ARGS;
      }
    }
    if ((operation == ZX_HANDLE_OP_DUPLICATE) && !handle->HasRights(ZX_RIGHT_DUPLICATE))
      return ZX_ERR_ACCESS_DENIED;
    return ZX_OK;
  }
  ```

  For the process with the handle pointing to another end of the channel, the `channel_read` is simpler in rights checking.

  ```c++
  template <typename HandleInfoT>
  static zx_status_t channel_read(zx_handle_t handle_value, uint32_t options,
                                  user_out_ptr<void> bytes, user_out_ptr<HandleInfoT> handles,
                                  uint32_t num_bytes, uint32_t num_handles,
                                  user_out_ptr<uint32_t> actual_bytes,
                                  user_out_ptr<uint32_t> actual_handles) {
    auto up = ProcessDispatcher::GetCurrent();
  
    fbl::RefPtr<ChannelDispatcher> channel;
    zx_status_t result =
        up->handle_table().GetDispatcherWithRights(handle_value, ZX_RIGHT_READ, &channel);
  
    // Currently MAY_DISCARD is the only allowable option.
    if (options & ~ZX_CHANNEL_READ_MAY_DISCARD)
      return ZX_ERR_NOT_SUPPORTED;
  
    MessagePacketPtr msg;
    result = channel->Read(up->get_koid(), &num_bytes, &num_handles, &msg,
                           options & ZX_CHANNEL_READ_MAY_DISCARD);
  
    // On ZX_ERR_BUFFER_TOO_SMALL, Read() gives us the size of the next message (which remains
    // unconsumed, unless |options| has ZX_CHANNEL_READ_MAY_DISCARD set).
    if (actual_bytes) {
      zx_status_t status = actual_bytes.copy_to_user(num_bytes);
    }
  
    if (actual_handles) {
      zx_status_t status = actual_handles.copy_to_user(num_handles);
    }
  
    if (num_bytes > 0u) {
      if (msg->CopyDataTo(bytes.reinterpret<char>()) != ZX_OK)
        return ZX_ERR_INVALID_ARGS;
    }
  
    // The documented public API states that that writing to the handles buffer
    // must happen after writing to the data buffer.
    if (num_handles > 0u) {
      zx_status_t status = msg_get_handles(up, msg.get(), handles, num_handles);
    }
  
    record_recv_msg_sz(num_bytes);
    return result;
  }
  ```

  <font color='red'>For the second question, the answer tends to be `NO`. But, currently, I am not so sure.</font>

  For the third question, the answer is no. At least, we can see the transferable handles must go through rights checking.
