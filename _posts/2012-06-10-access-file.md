---
title: Linux accessing file
layout: post
tags: file mapping memory
category: linux
---

*本文是ULK Ch16 Accessing files的笔记*.

本文讨论Linux是如何访问文件的. 这个话题, 我一直最感兴趣的是文件的内存映射.

---

#内存映射

"[Linux process address space](http://xanpeng.github.com/2012/06/06/process-address-space/)"中介绍了虚存区(线性区, vma), 一个vma可以和磁盘文件系统的普通文件的某一部分或者块设备文件相关联. 这就意味着内核把对vma中页内某个字节的访问转换成对文件中相应字节的操作, 这种技术称为内存映射(memory mapping).

有两种内存映射:  
- 共享型: vma页上的任何写操作都会修改磁盘上的文件. 并且如果有其他进程映射了同一文件, 这种修改也是可见的.  
- 私有型: 这种映射的目的只是为了读文件, 而非写文件. 出于这种目的, 私有映射的效率要比共享映射的效率更高. 但对私有映射页的任何写操作都会使内核停止映射该文件中的页, 因此写操作既不会改变磁盘上的文件, 对访问相同文件的其他进程也不可见.

进程使用mmap()系统调用创建一个新的内存映射, 指定MAP_SHARED或MAP_PRIVATE标志. 使用munmap()系统调用撤销内存映射.

##内存映射的数据结构

内存映射用下列数据结构的组合来表示:  
- 对应文件的inode.  
- 对应文件的address_space.  
- 不同进程对一个文件进行不同映射所使用的文件对象file.  
- 对文件进行每一不同映射所使用的vm_area_struct.  
- 对文件进行映射的vma所分配的每个页框对应的page结构.  

![](https://github.com/xanpeng/xanpeng.github.com/raw/master/images/mmap_data_structure.png)

##创建内存映射

传入mmap()的参数有:  
- file descriptor, 标志文件.  
- 文件内的偏移量.  
- 要映射的文件部分长度.  
- 一组标志, 如MAP_SHARED,MAP_PRIVATE.  
- 一组权限, 如PROT_READ,PROT_WRITE,PROT_EXEC.  
- 一个可选的虚拟地址, 内核把该地址作为新vma应该从哪里开始的一个线索.

mmap()返回新vma中第一个单元位置的虚拟地址. 出于兼容目的, mmap()在系统调用表中对应有两个服务例程, 它们都调用do_mmap_pgoff(), 这个函数的步骤:  
1. 检查.  
2. 调用get_upmapped_area(), 为文件的内存映射分配一个合适的vma.  
3. 权限等标志位设置.  
4. 用文件对象的地址初始化vma的vm_file字段, 增加文件的引用计数. 对映射的文件调用mmap方法, 将文件对象地址和vma描述符地址作为参数. 对于大多数文件系统, 该方法由generic_file_mmap()实现.  
5. 增加inode.i_writecount字段的值, 这个字段是写进程的引用计数器.  

##撤销内存映射

传入munmap()的参数:  
- 要删除的vma的第一个单元的地址.   
- 要删除的vma区间的长度.

---

#其他方式

其他访问文件的模式还有:  
- 规范模式: 不是O_SYNC和O_DIRECT方式, 而且文件内容由系统调用read()和write()存取. read()阻塞调用进程, 直到数据被拷贝到用户地址空间, write()在数据被拷贝到page cache后结束.  
- 同步模式: 以O_SYNC方式打开文件, 或稍后由fcntl()设置O_SYNC. 这个标志只会影响写操作, 它阻塞调用进程, 直到数据被有效地写入磁盘.  
- 直接IO模式: 以O_DIRECT模式打开文件. 任何读写操作都将数据在用户态地址空间与磁盘间直接传送, 而不通过page cache.  
- 异步模式: 数据传输请求不阻塞调用进程, 而是在后台执行, 同时应用程序继续它的正常运行.

读文件是基于页的, 内核总是一次传送几个完整的数据页. 如果进程发出read()来读取一些字节, 而这些数据还不在RAM中, 那么内核就要分配一个新页框, 并用文件的适当部分来填充, 把该页加入page cache, 最后将所请求的字节拷贝到进程地址空间中.
