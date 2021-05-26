

## 同步相关原语

主要是`Event`事件通知和`Semaphore`信号量。在信号量方面，支持计数信号量和互斥信号量两种类型。主要的数据结构包括：

| [Event](struct.Event.html)                   | event bus Event flags                                        |
| -------------------------------------------- | ------------------------------------------------------------ |
| [EventBus](struct.EventBus.html)             | event bus struct                                             |
| [Semaphore](struct.Semaphore.html)           | A counting, blocking, semaphore.                             |
| [SemaphoreGuard](struct.SemaphoreGuard.html) | An RAII guard which will release a resource acquired from a semaphore when dropped. |



## Singal信号机制

这是为了支持Linux中的信号机制。主要的数据结构包括：

| [MachineContext](struct.MachineContext.html)       | 用于保存回复现场的上下文信息           |
| -------------------------------------------------- | -------------------------------------- |
| [SigInfo](struct.SigInfo.html)                     | 信号编码                               |
| [SignalAction](struct.SignalAction.html)           | 信号行为                               |
| [SignalActionFlags](struct.SignalActionFlags.html) | 信号行为标记                           |
| [SignalFrame](struct.SignalFrame.html)             | 内核构造的用户信号处理函数的内核返回帧 |
| [SignalStack](struct.SignalStack.html)             | 用户信号处理函数的栈                   |
| [SignalStackFlags](struct.SignalStackFlags.html)   | 用户信号处理函数的栈标记               |
| [SignalUserContext](struct.SignalUserContext.html) | 用户信号处理函数的用户执行环境         |
| [Sigset](struct.Sigset.html)                       | 信号的集合                             |

## 时间相关信息

主要的数据结构包括：

| [RUsage](struct.RUsage.html)     | RUsage for sys_getrusage() ignore other fields for now |
| -------------------------------- | ------------------------------------------------------ |
| [TimeSpec](struct.TimeSpec.html) | TimeSpec struct for clock_gettime, similar to Timespec |
| [TimeVal](struct.TimeVal.html)   | TimeVal struct for gettimeofday                        |
| [Tms](struct.Tms.html)           | Tms for times()                                        |

## IPC相关机制

主要的数据结构包括：

| [IpcPerm](struct.IpcPerm.html)             | structure specifies the access permissions on the semaphore set |
| ------------------------------------------ | ------------------------------------------------------------ |
| [SemArray](struct.SemArray.html)           | A System V semaphore set                                     |
| [SemProc](struct.SemProc.html)             | Semaphore table in a process                                 |
| [SemidDs](struct.SemidDs.html)             | semid data structure                                         |
| [ShmGuard](struct.ShmGuard.html)           | shared memory buffer and data                                |
| [ShmIdentifier](struct.ShmIdentifier.html) | shared memory Identifier for process                         |
| [ShmProc](struct.ShmProc.html)             | Shared_memory table in a process                             |
| [ShmidDs](struct.ShmidDs.html)             | shmid data structure                                         |

## LinuxLoader用户态加载机制

LinuxLoader是用于zCore的LibOS模式下加载执行Linux应用程序。主要的数据结构包括：

```rust
/// Linux ELF Program Loader.
pub struct LinuxElfLoader {
    /// syscall entry
    pub syscall_entry: usize,
    /// stack page number
    pub stack_pages: usize,
    /// root inode of LinuxElfLoader
    pub root_inode: Arc<dyn INode>,
}
```

主要的方法是`load`，它完成对Linux应用程序的ELF解析，分配内存空间和堆栈等，建立用户执行环境，然后把控制权交给代表Linux应用程序的入口函数执行。