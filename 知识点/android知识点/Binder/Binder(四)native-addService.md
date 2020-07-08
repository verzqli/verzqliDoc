#### frameworks/native/libs/binder/  

- Binder.cpp  ()

- BpBinder.cpp  

- IPCThreadState.cpp  

- ProcessState.cpp  

- IServiceManager.cpp  

- IInterface.cpp  

- Parcel.cpp 

#### frameworks/native/include/binder/ 

- IInterface.h (包括BnInterface, BpInterface)
- Binder.h 也就是BBinder

https://blog.csdn.net/shift_wwx/article/details/100104542

https://blog.csdn.net/zbunix/article/details/8758631

​    BpBinder是client端创建的用于消息发送的代理，而BBinder是server端用于接收消息的通道。

### 1.BpBinder::transact

> frameworks/native/libs/binder/BpBinder.cpp

  ```c++
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{	//这里的code就是前面的ADD_SERVICE_TRANSACTION
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        //调用IPCThreadState的transact
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}

  ```

Binder代理类调用transact()方法，真正工作还是交给IPCThreadState来进行transact工作。先来 看看IPCThreadState::self的过程。

#### 1.1 IPCThreadState::self

> frameworks/native/libs/binder/IPCThreadState.cpp

```c++
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        //初始化创建IPCThreadState
        return new IPCThreadState;
    }

    if (gShutdown) {
        ALOGW("Calling IPCThreadState::self() during shutdown is dangerous, expect a crash.\n");
        return nullptr;
    }

    pthread_mutex_lock(&gTLSMutex);
    //gHaveTLS首次进来为false
    if (!gHaveTLS) {
        //创建线程的TLS
        int key_create_value = pthread_key_create(&gTLS, threadDestructor);
        if (key_create_value != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            ALOGW("IPCThreadState::self() unable to create TLS key, expect a crash: %s\n",
                    strerror(key_create_value));
            return nullptr;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}
```

TLS是指Thread local storage(线程本地储存空间)，每个线程都拥有自己的TLS，并且是私有空间，线程之间不会共享。通过``pthread_getspecific/pthread_setspecific``函数可以获取/设置这些空间中的内容。从线程本地存储空间中获得保存在其中的IPCThreadState对象。

#### 1.2 IPCThreadState的创建

> frameworks/native/libs/binder/IPCThreadState.cpp

```c++
IPCThreadState::IPCThreadState()
    //创建进程的ProcessState
    : mProcess(ProcessState::self()),
      mWorkSource(kUnsetWorkSource),
      mPropagateWorkSource(false),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0),
      mCallRestriction(mProcess->mCallRestriction)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
    mIPCThreadStateBase = IPCThreadStateBase::self();
}

```

每个线程都有一个`IPCThreadState`，每个`IPCThreadState`中都有一个mIn、一个mOut。成员变量mProcess保存了ProcessState变量(每个进程只有一个)。

- mIn 用来接收来自Binder设备的数据，默认大小为256字节；
- mOut用来存储发往Binder设备的数据，默认大小为256字节。

#### 1.2 IPCThreadState::transact

> frameworks/native/libs/binder/IPCThreadState.cpp

```c++
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err;
    flags |= TF_ACCEPT_FDS;
	...
    LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
        (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
	...
	//ONE_WAY标签标识该次binder通信是同步还是异步
    if ((flags & TF_ONE_WAY) == 0) {
        ...
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        ...
    } else {
        err = waitForResponse(nullptr, nullptr);
    }
    return err;
}

```

#### 1.3 IPCThreadState::writeTransactionData

```c++
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;	 // handle = 0
    tr.code = code;				 // code = ADD_SERVICE_TRANSACTION
    tr.flags = binderFlags;      // binderFlags = 0
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;
	// data为记录注册服务信息的Parcel对象
    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);		 //cmd = BC_TRANSACTION 为向binder驱动请求通信的协议符号
    mOut.write(&tr, sizeof(tr)); //写入binder_transaction_data数据

    return NO_ERROR;
}

```

其中handle的值用来标识目的端，注册服务过程的目的端为service manager，此处handle=0所对应的是binder_context_mgr_node对象，正是service manager所对应的binder实体对象。[binder_transaction_data结构体](http://gityuan.com/2015/11/01/binder-driver/#bindertransactiondata)是binder驱动通信的数据结构，该过程最终是把Binder请求码BC_TRANSACTION和binder_transaction_data结构体写入到`mOut`。

transact过程，先写完binder_transaction_data数据，其中Parcel data的重要成员变量：

- mDataSize:保存再data_size，binder_transaction的数据大小；
- mData: 保存在ptr.buffer, binder_transaction的数据的起始地址；
- mObjectsSize:保存在ptr.offsets_size，记录着flat_binder_object结构体的个数；
- mObjects: 保存在offsets, 记录着flat_binder_object结构体在数据偏移量；

接下来执行waitForResponse()方法。

#### 1.4 IPCThreadState::waitForResponse

```c++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        //talkWithDriver 顾名思义，和binder 驱动进行通信
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
      		if (!reply && !acquireResult) goto finish;
                break;
        case BR_DEAD_REPLY:...

        case BR_FAILED_REPLY:...

        case BR_ACQUIRE_RESULT:...
 			goto finish;
        case BR_REPLY:...
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}
```

在waitForResponse过程, 首先收到驱动的回应BR_TRANSACTION_COMPLETE，表示已经连接上；另外，目标进程收到事务后，处理BR_TRANSACTION事务。 然后发送给当前进程，再执行BR_REPLY命令。

#### 1.5 IPCThreadState::talkWithDriver

```java
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }
	//这个就是进程间数据传输的格式
    binder_write_read bwr;

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

  ...
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
       ...
#if defined(__ANDROID__)
     //通过ioctl不停的读写操作，跟Binder Driver进行通信
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
...
    } while (err == -EINTR);
...
    return err;
}
```

[binder_write_read结构体](http://gityuan.com/2015/11/01/binder-driver/#binderwriteread)用来与Binder设备交换数据的结构, 通过ioctl与mDriverFD通信，是真正与Binder驱动进行数据读写交互的过程。 主要是操作mOut和mIn变量。

ioctl()经过系统调用后进入Binder Driver.

ioctl -> binder_ioctl -> binder_ioctl_write_read

###  2.Binder Driver

#### 	2.1binder_ioctl_write_read

```c++
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    struct binder_proc *proc = filp->private_data;
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    //将用户空间bwr结构体拷贝到内核空间(binder_write_read)
    copy_from_user(&bwr, ubuf, sizeof(bwr));
    ...

    if (bwr.write_size > 0) {
        //将数据放入目标进程
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        ...
    }
    if (bwr.read_size > 0) {
        //读取自己队列的数据
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
             bwr.read_size,
             &bwr.read_consumed,
             filp->f_flags & O_NONBLOCK);
        if (!list_empty(&proc->todo))
            wake_up_interruptible(&proc->wait);
        ...
    }

    //将内核空间bwr结构体拷贝到用户空间
    copy_to_user(ubuf, &bwr, sizeof(bwr));
    ...
}   
```

#### 2.2 binder_thread_write

```java
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    uint32_t cmd;
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr < end && thread->return_error == BR_OK) {
        //拷贝用户空间的cmd命令，此时为BC_TRANSACTION
        if (get_user(cmd, (uint32_t __user *)ptr)) -EFAULT;
        ptr += sizeof(uint32_t);
        switch (cmd) {
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;
            //拷贝用户空间的binder_transaction_data
            if (copy_from_user(&tr, ptr, sizeof(tr)))   return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }
        ...
    }
    *consumed = ptr - buffer;
  }
  return 0;
}
```

#### 2.2 binder_transaction

```java
static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){
    struct binder_transaction *t;
   	struct binder_work *tcomplete;
    ...

    if (reply) {
        ...
    }else {
        if (tr->target.handle) {
            ...
        } else {
            // handle=0则找到servicemanager实体
            target_node = binder_context_mgr_node;
        }
        //target_proc为servicemanager进程
        target_proc = target_node->proc;
    }

    if (target_thread) {
        ...
    } else {
        //找到servicemanager进程的todo队列
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }

    t = kzalloc(sizeof(*t), GFP_KERNEL);
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

    //非oneway的通信方式，把当前thread保存到transaction的from字段
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;

    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc; //此次通信目标进程为servicemanager进程
    t->to_thread = target_thread;
    t->code = tr->code;  //此次通信code = ADD_SERVICE_TRANSACTION
    t->flags = tr->flags;  // 此次通信flags = 0
    t->priority = task_nice(current);

    //从servicemanager进程中分配buffer
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));

    t->buffer->allow_user_free = 0;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;

    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL); //引用计数加1
    offp = (binder_size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

    //分别拷贝用户空间的binder_transaction_data中ptr.buffer和ptr.offsets到内核
    //这才是第一次拷贝数据，之前和之后都是拷贝命令
    copy_from_user(t->buffer->data,
        (const void __user *)(uintptr_t)tr->data.ptr.buffer, tr->data_size);
    copy_from_user(offp,
        (const void __user *)(uintptr_t)tr->data.ptr.offsets, tr->offsets_size);

    off_end = (void *)offp + tr->offsets_size;

    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);
        off_min = *offp + sizeof(struct flat_binder_object);
        switch (fp->type) {
            case BINDER_TYPE_BINDER:
            case BINDER_TYPE_WEAK_BINDER: {
              struct binder_ref *ref;
               //根据proc获取当前binder_node，如果为空就创建，
              struct binder_node *node = binder_get_node(proc, fp->binder);
              if (node == NULL) {
                //服务所在进程 创建binder_node实体
                node = binder_new_node(proc, fp->binder, fp->cookie);
                ...
              }
              //servicemanager进程binder_ref
              ref = binder_get_ref_for_node(target_proc, node);
              ...
              //调整type为HANDLE类型
              if (fp->type == BINDER_TYPE_BINDER)
                fp->type = BINDER_TYPE_HANDLE;
              else
                fp->type = BINDER_TYPE_WEAK_HANDLE;
              fp->binder = 0;
              fp->handle = ref->desc; //设置handle值
              fp->cookie = 0;
              binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                       &thread->todo);
            } break;
            case :...
    }

    if (reply) {
        ..
    } else if (!(t->flags & TF_ONE_WAY)) {
        //BC_TRANSACTION 且 非oneway,则设置事务栈信息
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
    } else {
        ...
    }

    //将BINDER_WORK_TRANSACTION添加到目标队列，本次通信的目标队列为target_proc->todo
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);

    //将BINDER_WORK_TRANSACTION_COMPLETE添加到当前线程的todo队列
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);

    //唤醒等待队列，本次通信的目标队列为target_proc->wait
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
}
```



注册服务的过程，传递的是BBinder对象，故writeStrongBinder()过程中localBinder不为空， 从而flat_binder_object.type等于BINDER_TYPE_BINDER。

服务注册过程是在服务所在进程创建binder_node，在servicemanager进程创建binder_ref。 对于同一个binder_node，每个进程只会创建一个binder_ref对象。

向servicemanager的binder_proc->todo添加BINDER_WORK_TRANSACTION事务，接下来进入ServiceManager进程。

#### 2.3 binder_get_node

```c++
static struct binder_node *binder_get_node(struct binder_proc *proc,
             binder_uintptr_t ptr)
{
  struct rb_node *n = proc->nodes.rb_node;
  struct binder_node *node;

  while (n) {
    node = rb_entry(n, struct binder_node, rb_node);

    if (ptr < node->ptr)
      n = n->rb_left;
    else if (ptr > node->ptr)
      n = n->rb_right;
    else
      return node;
  }
  return NULL;
}
```

从binder_proc来根据binder指针ptr值，查询相应的binder_node。

#### 2.4 binder_new_node

```c++
static struct binder_node *binder_new_node(struct binder_proc *proc,
                       binder_uintptr_t ptr,
                       binder_uintptr_t cookie)
{
    struct rb_node **p = &proc->nodes.rb_node;
    struct rb_node *parent = NULL;
    struct binder_node *node;
    ... //红黑树位置查找

    //给新创建的binder_node 分配内核空间
    node = kzalloc(sizeof(*node), GFP_KERNEL);

    // 将新创建的node添加到proc红黑树；
    rb_link_node(&node->rb_node, parent, p);
    rb_insert_color(&node->rb_node, &proc->nodes);
    node->debug_id = ++binder_last_id;
    node->proc = proc;
    node->ptr = ptr;
    node->cookie = cookie;
    node->work.type = BINDER_WORK_NODE; //设置binder_work的type
    INIT_LIST_HEAD(&node->work.entry);
    INIT_LIST_HEAD(&node->async_todo);
    return node;
}
```

#### 2.5 binder_get_ref_for_node

```java
static struct binder_ref *binder_get_ref_for_node(struct binder_proc *proc,
              struct binder_node *node)
{
  struct rb_node *n;
  struct rb_node **p = &proc->refs_by_node.rb_node;
  struct rb_node *parent = NULL;
  struct binder_ref *ref, *new_ref;
  //从refs_by_node红黑树，找到binder_ref则直接返回。
  while (*p) {
    parent = *p;
    ref = rb_entry(parent, struct binder_ref, rb_node_node);

    if (node < ref->node)
      p = &(*p)->rb_left;
    else if (node > ref->node)
      p = &(*p)->rb_right;
    else
      return ref;
  }

  //创建binder_ref
  new_ref = kzalloc_preempt_disabled(sizeof(*ref));

  new_ref->debug_id = ++binder_last_id;
  new_ref->proc = proc; //记录进程信息
  new_ref->node = node; //记录binder节点
  rb_link_node(&new_ref->rb_node_node, parent, p);
  rb_insert_color(&new_ref->rb_node_node, &proc->refs_by_node);

  //计算binder引用的handle值，该值返回给target_proc进程
  new_ref->desc = (node == binder_context_mgr_node) ? 0 : 1;
  //从红黑树最最左边的handle对比，依次递增，直到红黑树遍历结束或者找到更大的handle则结束。
  for (n = rb_first(&proc->refs_by_desc); n != NULL; n = rb_next(n)) {
    //根据binder_ref的成员变量rb_node_desc的地址指针n，来获取binder_ref的首地址
    ref = rb_entry(n, struct binder_ref, rb_node_desc);
    if (ref->desc > new_ref->desc)
      break;
    new_ref->desc = ref->desc + 1;
  }

  // 将新创建的new_ref 插入proc->refs_by_desc红黑树
  p = &proc->refs_by_desc.rb_node;
  while (*p) {
    parent = *p;
    ref = rb_entry(parent, struct binder_ref, rb_node_desc);

    if (new_ref->desc < ref->desc)
      p = &(*p)->rb_left;
    else if (new_ref->desc > ref->desc)
      p = &(*p)->rb_right;
    else
      BUG();
  }
  rb_link_node(&new_ref->rb_node_desc, parent, p);
  rb_insert_color(&new_ref->rb_node_desc, &proc->refs_by_desc);
  if (node) {
    hlist_add_head(&new_ref->node_entry, &node->refs);
  }
  return new_ref;
}
```

handle值计算方法规律：

- 每个进程binder_proc所记录的binder_ref的handle值是从1开始递增的；
- 所有进程binder_proc所记录的handle=0的binder_ref都指向service manager；
- 同一个服务的binder_node在不同进程的binder_ref的handle值可以不同；