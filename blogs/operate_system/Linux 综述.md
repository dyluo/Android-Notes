---
Linux 综述
---

#### 目录

1. Linux 内核系统结构
2. 系统调用
   - 进程管理
   - 内存管理
   - 文件管理
   - 异常处理和信号处理
   - 进程间通信
   - 网络通信

#### Linux 内核体系结构

Linux 内核可以分为六大块，分别是：

1. 系统调用子系统

   操作系统的功能调用的统一入口。

2. 进程管理子系统

   对执行的程序进行生命周期和资源管理。

3. 内存管理子系统

   对操作系统的内存进行管理、分配、回收、隔离。

4. 文件子系统

   对文件进行管理。

5. 设备子系统

   对输入输出设备进行管理。

6. 网络子系统

   网络协议栈和收发网络包。

![](https://i.loli.net/2019/05/28/5ced1431bbbec82878.jpg)

#### 系统调用

系统调用可以为了几块：进程管理、内存管理、文件管理、网络通信、进程间通信已经信号处理。

##### 进程管理

创建进程的系统调用叫 fork，译为 "分支"。在 Linux 里，要创建一个新的进程，需要一个老的进程调用 fork 来实现，其中老的进程叫做父进程，新的进程叫子进程。当父进程调用 fork 创建进程的时候，子进程将各个子系统为父进程创建的数据结构也全部拷贝里一份，甚至连程序代码也是拷贝过来的。按理说，如果不进行特殊处理，父进程和子进程都按相同的程序代码进行下去，这样就没有什么意义了。

Linux 会根据 fork 系统调用的返回值来决定如何处理，如果当前进程是子进程，就返回 0；如果当前进程是父进程，就返回子进程的进程号。这样就可以根据返回来来区分，如果是父进程，还接着做原来的事情，如果是子进程，需要请求另一个系统调用 execve 来执行另外一个程序，这个时候，子进程和父进程就彻底分道扬镳了，也就产生了一个分支（fork）了。

有时候，父进程要关心子进程的运行情况，父进程可以调用一个系统调用 waitpid，将子进程的进程号作为参数传给它，这样父进程就知道子进程运行完了没有，成功与否。

##### 内存管理

对于进程的内存空间来讲，放程序代码的这部分，称为代码段（Code Segment），放进程运行中产生数据的这部分，称为数据段（Data Segment）。其中局部变量的部分，在当前函数执行的时候起作用，当进入另一个函数时，这个变量就释放了；也有动态分配的，会较长时间保存，指明才销毁的，这部分称为堆（Heap）。堆里面分配内存有两个系统调用：brk 和 mmap。当分配的内存数据比较小的时候，使用 brk，分配内存数量比较大时，使用 mmap。

##### 文件管理

对于文件的操作，有六个系统调用最重要了：open、close、creat、lseek、read、write。对于已经有的文件，可以使用 open 打开这个文件，close 关闭这个文件；对于没有的文件，可以使用 creat 创建文件；打开文件以后，可以使用 lseek 跳到文件的某个位置；可以对文件的内容进行读写，对应的系统调用即 read 和 write。

但是别忘了，Linux 里有一个特点，那就是**一切皆文件**。

启动一个进程，需要一个程序文件，这是一个二进制文件；启动的时候，要加载一些配置文件，例如 yml、properties 等，这是文本文件；启动之后会打印一些日志，如果写到磁盘上，也是文本文件；但是如果把日志打印到交互控制台上，这其实是标准输出 stdout 文件；这个进程的输出可以作为另一个进程的输入，这种方式称为管道，管道也是一个文件。进程可以通过网络和其他进程进行通信，建立的 Socket，也是一个文件；进程需要访问外部设备，设备也是一个文件；文件都被存储在文件夹里面，其实文件夹也是一个文件；进程运行起来，要想看到进程运行的情况，会在 /proc 下面有对应的进程号，还是一系列文件；

每个文件，Linux 都会分配一个文件描述符（File Descriptor），这是一个整数。有了这个文件描述符，就可以使用1调用，查看或者干预进程运行的方方面面。

所以说，文件操作是贯穿始终的，这也是 "一切皆文件" 的优势，就是统一了操作的入口，提供了极大的便利。

##### 异常处理和信号处理

进程的运行不一定是一帆风顺的，很可能遇到各种异常情况，这时候就需要发送一个信号（Signal）给系统，经常遇到的信号有以下几种：

1. 在执行一个程序的时候，键盘输入 "CTRL+C"，这就是中断的信号，正在执行的命令就会中止退出。
2. 非法访问内存
3. 硬件故障
4. 用户进程通过 kill 函数，将一个用户信号发送给另一个进程

对于一些不严重的信号，可以忽略，但是像 SIGKILL（用于终止一个进程的信号）和 SIGSTOP（用于中止一个进程的信号）是不能忽略的，可以执行对于该信号的默认动作。每种信号都定义了默认的动作，例如硬件故障，默认终止；也可以提供信号处理函数，可以通过 sigaction 系统调用，注册一个信号处理函数。

##### 进程间通信

首先就是发个消息，不需要一段很长的数据，这种方式称为消息队列（Message Queue）。可以通过 msgget 创建一个消息队列，msgsnd 将消息发送到消息队列，而消息接收方可以使用 msgrcv 从队列中取消息。

当两个进程间需要交互的信息比较大时，可以使用共享内存的方式，也就是两个进程共享同一块内存。可以通过 shmget 创建一个共享内存块，通过 shmat 将共享内存映射到自己的内存空间，然后就可以读写了。

但是，如果两个进程共同访问同一块内存空间，同时修改同一块数据，就存在数据竞争的问题了。为了让不同的进程能够排他的访问，这就是信号量的机制 Semaphore。

信号量机制比较复杂，但是可以简化为：对于只允许一个进程访问的数据，我们可以将信号量设为 1。当一个进程要访问的时候，先调用 sem_wait，如果这个时候没有进程访问，则占有这个信号量，该进程就可以开始访问了。如果这个时候又有另外一个进程要访问，也会调用 sem_wait。由于前一个进程已经在访问了，所有后面的进程就必须等待上一个进程访问完之后才能访问。当上一个进程访问完毕之后，会调用 sem_post 将信号量释放，于是下一个进程就可以访问这个资源了。

##### 网络通信

不同机器通过网络相互通信，要遵循相同的网络协议，也即 TCP/IP 网络协议栈。Linux 内核里有对于网络协议栈的实现。如何暴露出服务给进程使用呢？

网络服务是通过套接字 Socket 来提供服务的，Socket 可以作为 "插口" 或 "插槽" 讲。虽然我们是写软件程序，但是你可以想象成弄一根网线，一头插在客户端，一头插在服务端，然后进行通信。因此，在通信之前，双方都要建立一个 Socket。

我们可以通过 Socket 系统调用建立一个 Socket，Socket 也是一个文件，也有一个文件描述符，也可以通过读写函数进行通信。

##### 总结

![](https://i.loli.net/2019/05/28/5ced3eae61bc578116.jpg)