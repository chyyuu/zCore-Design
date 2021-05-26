## 概述

Linux进程模块主要完成对Linux进程的创建、执行、资源操作等方面的管理

## Linux进程相关数据结构

```rust
/// Linux specific process information.
pub struct LinuxProcess {
    /// The root INode of file system
    root_inode: Arc<dyn INode>,
    /// Parent process
    parent: Weak<Process>,
    /// Inner
    inner: Mutex<LinuxProcessInner>,
}
/// Linux process mut inner data
#[derive(Default)]
struct LinuxProcessInner {
    /// Execute path
    execute_path: String,
    /// Current Working Directory
    ///
    /// Omit leading '/'.
    current_working_directory: String,
    /// file open number limit
    file_limit: RLimit,
    /// Opened files
    files: HashMap<FileDesc, Arc<dyn FileLike>>,
    /// Semaphore
    semaphores: SemProc,
    /// Share Memory
    shm_identifiers: ShmProc,
    /// Futexes
    futexes: HashMap<VirtAddr, Arc<Futex>>,
    /// Child processes
    children: HashMap<KoID, Arc<Process>>,
    /// Signal actions
    signal_actions: SignalActions,
}
```

## Linux进程相关方法

- add_file\add_file_at：把文件添加到文件描述表中
- change_directory：改变工作目录
- close_file：关闭文件
- current_working_directory：获取当前工作目录
- execute_path：获取执行路径
- file_limit：获取/设置文件限制
- get_file：根据`fd`得到 `File`
- get_file_like：根据`fd`得到 `FileLike` 
- get_files：获取所有文件
- get_futex：获取futex对象
- lookup_inode/lookup_inode_at：从进程中查找inode
- new: 创建新进程
- parent：获取父进程
- remove_cloexec_files：关闭设置了FD_CLOEXEC的文件
- root_inode：获取进程的根inode
- semaphores_add：插入一个 `SemArray` 并返回它的 ID
- semaphores_add_undo：增加一个`undo`操作
- semaphores_get：根据`id`获取对应的semaphore set 
- semaphores_remove：根据`id`删除对应的semaphore set 
- set_execute_path：设置执行路径
- set_signal_action：设置信号的行为
- shm_add：插入一个 shared memory并返回它的ID
- shm_get：根据`id`获取shared memory
- shm_get_id：根据shared memory的地址得到其`id`
- shm_pop：根据`id`弹出(删除)一个shared memory
- shm_set：设置一个shared memory的虚拟地址
- signal_action：获取信号的行为

## Linux线程相关数据结构

```rust
/// Linux specific thread information.
pub struct LinuxThread {
    /// Kernel performs futex wake when thread exits.
    /// Ref: <http://man7.org/linux/man-pages/man2/set_tid_address.2.html>
    clear_child_tid: UserOutPtr<i32>,
    /// Signal mask
    pub signal_mask: Sigset,
    /// signal alternate stack
    pub signal_alternate_stack: SignalStack,
}
```

