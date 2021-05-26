# zCore 文档

- [概述](README.md)
  
- [zCore 整体结构和设计模式](zcore-intro.md)
  
- [内核对象](ch01-00-object.md)
    - [内核对象](ch01-01-kernel-object.md)
    - [对象管理器：Process 对象](ch01-02-process-object.md)
    - [对象传送器：Channel 对象](ch01-03-channel-object.md)

- [任务管理](ch02-00-task.md)
    - [任务管理](ch02-01-zircon-task.md)
    - [作业管理：Job 对象](ch02-02-job-object.md)
    - [进程管理：Process对象](ch02-02-process-object.md)
    - [线程管理：Thread 对象](ch02-03-thread-object.md)
    - [异步协程管理：Future对象](ch02-04-async.md)
    
- [内存管理](ch03-00-memory.md)
    - [Zircon 内存管理模型](ch03-01-zircon-memory.md)
    - [物理内存：VMO 对象](ch03-02-vmo.md)
    - [虚拟内存：VMAR 对象](ch03-04-vmar.md)
    
- [信号和等待](ch05-00-signal-and-waiting.md)
  
    - [等待内核对象的信号](ch05-01-wait-signal.md)
    - [同时等待多个信号：Port 对象](ch05-02-port-object.md)
    - [实现更多：EventPair, Timer 对象](ch05-03-more-signal-objects.md)
- [用户态同步互斥：Futex 对象](ch05-04-futex-object.md)
  
- [硬件抽象层](ch06-00-hal.md)
    - [ zCore硬件抽象层](ch06-01-zcore-hal.md)
    - [ UNIX硬件抽象层](ch06-02-zcore-hal-unix.md)
    - [物理硬件抽象层](ch06-03-zcore-hal-baremetal.md)
    
- [Fuchsia支持](ch04-00-userspace.md)
  
    - [Zircon 用户程序](ch04-01-user-program.md)
    - [Zircon上下文切换](ch04-02-context-switch.md)
    - [Zircon系统调用](ch04-03-syscall.md)
    
- [Linux支持](ch07-00-linux.md)
  
    - [Linux任务管理](ch07-01-linux-proc.md)
    - [Linux文件管理](ch07-01-linux-fs.md)
    - [Linux支持管理](ch07-01-linux-other.md)