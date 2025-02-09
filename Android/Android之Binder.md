Binder是基于内存映射mmap实现的一种进程间的通信方式，

# mmap

将一个文件或者其他对象映射到进程的地址空间，实现文件磁盘地址与进程虚拟地址空间的一段地址的一一对应关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存。

### 内存管理

### Binder中的mmap

#### 一次拷贝的 write/read

read 和 write 每次调用都会发生一次数据拷贝。以 read 系统调用为例：当用户进程读取磁盘文件时，它会调用 read。这时，文件数据会从磁盘读取到内核空间的一个缓冲区，再拷贝到用户空间。

> “拷贝”通常指的是将数据从一个内存区域复制到另一个内存区域，这个过程会消耗 CPU 资源。读取/写入文件、设备的时候，通常会采用一个叫 DMA（Direct Memory Access）的硬件技术。它允许设备（比如硬盘或网络接口）直接读取和写入主内存，无需经过 CPU，从而避免了传统意义上的“拷贝”。
> 因此，我们不把 DMA 算作“拷贝”。因为它不涉及 CPU，可以直接从一个地方传输数据到另一个地方。

### Binder 进程通信一次拷贝的秘密

1. 服务端进程通过mmap分别将用户空间和内核空间中一段虚拟内存映射到同一块物理内存上；
2. 这段物理内存作为缓冲区，客户端发送事务数据时，内核会通过 copy_from_user 将数据放进缓冲区，然后通知服务端。服务端只需要通过用户空间相应的虚拟地址，就能直接访问数据
   ![[Binder的一次拷贝.webp]]
3. [ProcessState初始化](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/native/libs/binder/ProcessState.cpp;l=485;bpv=1;bpt=0)
   1. mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE,opened.value(), 0);

### 参考链接

1. [mmap实现原理解析](https://blog.csdn.net/weixin_42937012/article/details/121269058)
2. [图解 Binder：内存管理 - 掘金](https://juejin.cn/post/7244734179828203579)
3. [认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)
4. [Android源码分析 - Binder概述 - 掘金](https://juejin.cn/post/7059601252367204365)
5. [Choreographer原理 - Gityuan博客 | 袁辉辉的技术博客](http://gityuan.com/2017/02/25/choreographer/)
