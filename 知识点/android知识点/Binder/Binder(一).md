## 背景知识

> 为了理解Binder我们先澄清一些概念。为什么需要跨进程通信（IPC），怎么做到跨进程通信？为什么是Binder？
>
> Binder 是 Android 系统中进程间通讯（IPC）的一种方式，也是 Android 系统中最重要的特性之一。 Android 中的四大组件 Activity，Service，Broadcast，ContentProvider，不同的 App 等都运行在不同的进程中，它是这些进程间通讯的桥梁。正如其名“粘合剂”一样，它把系统中各个组件粘合到了一起，是各个组件的桥梁。

### 进程隔离

> 进程隔离是为保护操作系统中进程互不干扰而设计的一组不同硬件和软件的技术。这个技术是为了避免进程A写入进程B的情况发生。 进程的隔离实现，使用了虚拟地址空间。进程A的虚拟地址和进程B的虚拟地址不同，这样就防止进程A将数据信息写入进程B。
>
> 操作系统的不同进程之间，数据不共享；对于每个进程来说，它都天真地以为自己独享了整个系统，完全不知道其他进程的存在；因此一个进程需要与另外一个进程通信，需要某种系统机制才能完成。

### 虚拟地址空间

为了理解虚拟地址空间，这里先描述下物理地址和物理内存

物理地址不用多说，指的就是我们的物理存储硬件，如下面所展示的那样，硬件电路通过总线查询对应存储单元的值，用高低电位来表示0和1

```java
————————————————-————————————————
|   |   |   |   |   |   |   |   |   A
————————————————-————————————————
|   |   |   |   |   |   |   |   |   B
————————————————-————————————————
|   |   |   |   |   |   |   |   |   C
————————————————-————————————————
|   |   |   |   |   |   |   |   |   D
————————————————-————————————————
  1   2   3   4   5   6   7   8
```

例如这里A3,B4就是这些存储单元的物理地址，一个8G的内存条就有$2^{33}(8589934592)$个内存单元。

但是一个32位操作系统的电脑，给CPU的地址是32位的，最大寻址能力也就是$2^{32}$,也就是4GB，显然，你插入一个8G内存条对32位系统来说有4G的地址是寻不到的，但是通过内存管理单元(MMU)将一个逻辑地址转化为一个线性地址，也就是虚拟地址。

- 线性地址：也叫虚拟地址，值得注意的是这个地址就是一个32位无符号的整型数，所以虚拟地址空间总共就是4 GB大小。其中0-1GB的地址空间给予内核访问，其它3GB由每个进程自己访问。

- 逻辑地址：包含在机器语言指令中用来指定一个操作数或一条指令的地址。通常用一个段地址和一个偏移量组成

具体访问到哪个物理地址要看MMU将线性地址转换后的物理地址才能知道，所以有可能两个进程的地址同时访问同一个物理地址，这在物理地址很小的情况下是经常会发生的事情。这种情况下可以在执行一个进程时，把另一个进程数据换到磁盘。然后切换进程时再从磁盘挪回来，把被切换出的进程的数据挪到磁盘。在当然，同一时刻，一个物理内存单元只可能由某个进程来访问

MMU有两个硬件电路单元，一个称之为分段单元(segmentation unit)、一个称之为分页单元(paging unit)，一个逻辑地址通过分段单元转化为线性地址，在通过分页单元转化为物理地址。

总结：

- 每个进程以为自己运行在一个4G的连续内存中，它根本不认为这个内存里还有其它进程。但是如果真的这么做，对于内存是一个很大的浪费。实际上是系统管理进程时只给正在运行的进程在真正的内存中分配他当时需要的内存，不需要的部分立刻被回收。

- 因为连续的大内存会造成浪费，所以现在一般采用分页的形式，把虚拟地址空间和物理内存划分为固定大小的小块，建立页表，把进程的虚拟地址空间映射到物理内存的页帧上。页表上保存的就是映射管理，随着进程的运行，按需分配页，长时间未使用的页帧就会被操作系统回收。

- 在内存小的极端情况下会让磁盘内存参与进来。



### 系统调用/内核态/用户态

虽然从逻辑上抽离出用户空间和内核空间；但是不可避免的的是，总有那么一些用户空间需要访问内核的资源；比如应用程序访问文件，网络是很常见的事情，怎么办呢？

> Kernel space can be accessed by user processes only through the use of system calls.

用户空间访问内核空间的唯一方式就是**系统调用**；通过这个统一入口接口，所有的资源访问都是在内核的控制下执行，以免导致对用户程序对系统资源的越权访问，从而保障了系统的安全和稳定。用户软件良莠不齐，要是它们乱搞把系统玩坏了怎么办？因此对于某些特权操作必须交给安全可靠的内核来执行。

当一个任务（进程）执行系统调用而陷入内核代码中执行时，我们就称进程处于内核运行态（或简称为内核态）此时处理器处于特权级最高的（0级）内核代码中执行。当进程在执行用户自己的代码时，则称其处于用户运行态（用户态）。即此时处理器在特权级最低的（3级）用户代码中运行。处理器在特权等级高的时候才能执行那些特权CPU指令。

### 内核模块/驱动

通过系统调用，用户空间可以访问内核空间，那么如果一个用户空间想与另外一个用户空间进行通信怎么办呢？很自然想到的是让操作系统内核添加支持；传统的Linux通信机制，比如Socket，管道等都是内核支持的；但是Binder并不是Linux内核的一部分，它是怎么做到访问内核空间的呢？Linux的动态可加载内核模块（Loadable Kernel Module，LKM）机制解决了这个问题；模块是具有独立功能的程序，它可以被单独编译，但不能独立运行。它在运行时被链接到内核作为内核的一部分在内核空间运行。这样，Android系统可以通过添加一个内核模块运行在内核空间，用户进程之间的通过这个模块作为桥梁，就可以完成通信了。

在Android系统中，这个运行在内核空间的，负责各个用户进程通过Binder通信的内核模块叫做**Binder驱动**;

> 驱动程序一般指的是设备驱动程序（Device Driver），是一种可以使计算机和设备通信的特殊程序。相当于硬件的接口，操作系统只有通过这个接口，才能控制硬件设备的工作；

驱动就是操作硬件的接口，为了支持Binder通信过程，Binder使用了一种“硬件”，因此这个模块被称之为驱动。

### 为什么使用Binder？

Android使用的Linux内核拥有着非常多的跨进程通信机制，比如管道，System V，Socket等；为什么还需要单独搞一个Binder出来呢？主要有两点，性能和安全。在移动设备上，广泛地使用跨进程通信肯定对通信机制本身提出了严格的要求；Binder相对出传统的Socket方式，更加高效；另外，传统的进程通信方式对于通信双方的身份并没有做出严格的验证，只有在上层协议上进行架设；比如Socket通信ip地址是客户端手动填入的，都可以进行伪造；而Binder机制从协议本身就支持对通信双方做身份校检（uid），因而大大提升了安全性。这个也是Android权限模型的基础。

todo：进程间通信的共享内存

引用：[如何理解虚拟地址空间？](https://www.zhihu.com/question/290504400)

