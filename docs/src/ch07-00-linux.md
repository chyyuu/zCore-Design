zCore 可以通过修改编译选项，使得其能够运行原生的 Linux 用户程序。其原理是，Zircon 微内核提供了进程管理和内存管理功能这些关键的功能。因此，只需要在此基础之上补充部分 Linux 宏内核的其它功能（如文件系统），并且对外提供 Linux 的系统调用，就可以运行原生的 Linux 用户程序了。

关于Linux支持的主要功能模块如下所示

| error   | Linux错误编码      |
| ------- | ------------------ |
| fs      | Linux 文件对象     |
| ipc     | Linux 进程间通信   |
| loader  | Linux ELF 程序加载 |
| process | Linux 进程对象     |
| signal  | Linux 信号对象     |
| sync    | Linux同步原语      |
| thread  | Linux 线程对象     |
| time    | Linux 时间对象     |