- ### binder_proc

```c++
struct binder_proc { 
struct hlist_node proc_node; // 根据proc_node，可以获取该进程在"全局哈希表binder_procs(统计了所有的binder proc进程)"中的位置 
struct rb_root threads; // binder_proc进程内用于处理用户请求的线程组成的红黑树(关联binder_thread->rb_node) 
struct rb_root nodes; // binder_proc进程内的binder实体组成的红黑树(关联binder_node->rb_node) 
struct rb_root refs_by_desc; // binder_proc进程内的binder引用组成的红黑树，该引用以句柄来排序(关联binder_ref->rb_node_desc) 
struct rb_root refs_by_node; // binder_proc进程内的binder引用组成的红黑树，该引用以它对应的binder实体的地址来排序(关联binder_ref->rb_node)
int pid; // 进程id 
struct vm_area_struct *vma; // 进程的内核虚拟内存 
struct mm_struct *vma_vm_mm; 
struct task_struct *tsk; // 进程控制结构体(每一个进程都由task_struct 数据结构来定义)。 
struct files_struct *files; // 保存了进程打开的所有文件表数据 
struct hlist_node deferred_work_node; int deferred_work; void *buffer; // 该进程映射的物理内存在内核空间中的起始位置 
ptrdiff_t user_buffer_offset; // 内核虚拟地址与进程虚拟地址之间的差值 // 内存管理的相关变量 
struct list_head buffers; // 和binder_buffer->entry关联到同一链表，从而对Binder内存进行管理 
struct rb_root free_buffers; // 空闲内存，和binder_buffer->rb_node关联。 
struct rb_root allocated_buffers; // 已分配内存，和binder_buffer->rb_node关联。 
size_t free_async_space; struct page **pages; // 映射内存的page页数组，page是描述物理内存的结构体 
size_t buffer_size; // 映射内存的大小 
uint32_t buffer_free; 
struct list_head todo; // 该进程的待处理事件队列。
 wait_queue_head_t wait; // 等待队列。 
struct binder_stats stats; 
struct list_head delivered_death; 
int max_threads; // 最大线程数。定义threads中可包含的最大进程数。 int requested_threads; 
int requested_threads_started; 
int ready_threads; 
long default_priority; // 默认优先级。 
struct dentry *debugfs_entry;
};c
```



```java
　进程打开了设备文件 /dev/binder 后，还必须调用mmap将它映射到进程地址空间来，实际上是请求Binder驱动程序为它分配一个内核缓冲区，以便可以用来在进程间传递数据。
 * Binder驱动程序分配的内核缓冲区的大小保存在成员变量 buffer_size 中。这些内核缓冲区有两个地址，其中一个是内核空间地址，另一个是用户空间地址。内核空间地址是在Binder
 * 驱动程序内部使用的，保存在成员变量 buffer 中，而用户空间地址是在应用程序进程内部使用的，保存在成员变量 vma 中。这两个地址相差一个固定的值，保存在成员变量 
 * user_buffer_offset 中。这样，给定一个用户空间地址或者内核空间地址，Binder驱动程序就可以计算出另一个地址。
 * 　　成员变量 buffer 指向的是一块大的内核缓冲区，Binder驱动程序为了方便的对它进行管理，会将它划分才若干个小块。这些小块的内核缓冲区就是使用 binder_buffer 来描述的，
 * 它们保存在一个列表中，按照地址值从小到大的顺序来排列。成员变量 buffers 指向的便是该列表的头部。列表中的小块内核缓冲区有正在使用的，即已经分配了的物理页面；有空闲的，即还没有
 * 分配的物理页面，它们分别组织在两个红黑树中，其中，前者保存在成员变量 allocated_buffers 所描述的红黑树中，而后者保存在成员变量 free_buffers 所描述的红黑树中。此外，成员变量
 * buffer_free 保存了空闲内核缓冲区的大小，而成员变量 free_async_space 保存了当前可以用来保存异步事务数据的内核缓冲区的大小。
 * 　　前面已提到，每一个使用了Binder进程间通信机制的进程都有一个线程池，用来处理进程间通信请求，这个Binder线程池是有Binder驱动程序来维护的。结构体 binder_proc 的成员变量
 * threads 是一个红黑树的根节点，它以线程ID作为关键字来组织一个进程的Binder线程池。进程可以调用函数 ioctl 将一个线程注册到Binder驱动程序中，同时，当前进程没有足够的空闲线程
 * 来处理进程间通信请求时，Binder驱动程序也可以主动要求进程注册更多的线程到Binder线程池中。Binder驱动程序最多可以主动注册请求进程注册的线程的数量保存在成员变量 max_threads 
 * 中，而成员变量 ready_threads 表示当前进程的空闲Binder线程数。
 * 注意：
 * 　　 成员变量 max_threads 并不表示Binder线程池中的最大线程数，进程本身可以主动注册任意数目的线程到Binder线程池中。Binder驱动程序每一次主动请求进程注册一个线程时，都会
 * 将成员变量 request_threads 的值加1；而当进程响应这个请求后，Binder驱动程序就会将成员变量 requested_threads 的值减1，而且将成员变量 requested_threads_started 的
 * 值加1，表示Binder驱动程序已经主动请求进程注册了多少个线程到Binder线程池中。
 * 　　当进程收到一个进程间通信请求时，Binder驱动程序就将该请求封装成一个工作项，并且加入到进程的待处理工作项队列中，这个队列使用成员变量 todo 来描述。Binder线程池中的空闲
 * Binder线程会睡眠在由成员变量 wait 所描述的一个等待队列中，当他们的宿主进程的待处理工作项队列增加了新的工作项后，Binder驱动程序就会唤醒这些线程，以便它们可以去处理新的
 * 工作项。成员变量 default_priority 的值被初始化为进程的优先级。档一个线程处理一个工作项时，它的线程优先级有可能被设置为其宿主进程的优先级，即设置为成员变量 default_priority
 * 的值，这是由于线程是代表其宿主进程来处理一个工作项的。线程处理一个工作项时的优先级还会收到其他因素的影响，后面在详述。
 * 　　一个进程内部包含了一系列的Binder实体对象和Binder引用对象，进程使用三个红黑树来组织它们，其中，成员变量 nodes 是来组织Binder实体对象的，它以Binder实体对象的成员变量ptr
 * 作为关键字；而成员变量 refs_by_desc 和 refs_by_node 所描述的红黑树是用来组织Binder引用对象的，前者以Binder引用对象的成员变量desc为关键字，而后者以Binder引用对象的成员
 * 变量 node 作为关键字。
 *　　 defered_work_node是一个hash列表，用来保存进程可以延迟的工作项。这些延迟的工作项有三种类型，如下所示
```

- ### binder_write_read

```
struct binder_write_read {
	binder_size_t		write_size;	/* bytes to write */
	binder_size_t		write_consumed;	/* bytes consumed(消耗) by driver */
	binder_uintptr_t	write_buffer;
	binder_size_t		read_size;	/* bytes to read */
	binder_size_t		read_consumed;	/* bytes consumed by driver */
	binder_uintptr_t	read_buffer;
};
```

- ### svcinfo

```c++

struct svcinfo
{
    struct svcinfo *next; //指针，指向下一个svcinfo的结构体指针
    uint32_t handle; //句柄
    struct binder_death death; //结构体的binder_death对象
    int allow_isolated;//是否允许隔离
    uint32_t dumpsys_priority;//dumpsys 的优先级
    size_t len;//服务的字符个数
    uint16_t name[0];//服务名称，是char *的指针类型
};


```

