# zCore 的用户态运行支持

libos 版 zCore（简称uzCore） 的开发与裸机版 zCore （简称bzCore）同步进行,两个版本的 zCore 共用除了HAL 层之外的所有代码。为了支持 uzCore  的正常运行,zCore 在地址空间划分方面对 Zircon /Linux的原有设计进行了一定的修改,并为此对 Fuchsia 的源码进行了简单的修改、重新编译;另外,uzCore 需要的硬件相关层(HAL)将完全由宿主 OS 提供支持,一个合理的 HAL 层接口划分也是为支持 uzCore做出的重要考虑。




## 修改 VDSO

VDSO 是由内核提供、并只读映射到用户态的动态链接库，以函数接口形式提供系统调用接口。原始的 VDSO 中将会最终使用 syscall 指令从用户态进入内核态。但在 uzCore 环境下,内核和用户程序都运行在用户态,因此需要将 syscall 指令修改为函数调用,也就是将 sysall 指令修改为 call 指令。为此我们修改了 VDSO 汇编代码，将其中的 syscall 替换为 call，提供给 uzCore 使用。在 uzCore 内核初始化环节中，向其中填入 call 指令要跳转的目标地址,重定向到内核中处理 syscall 的特定函数,从而实现模拟系统调用的效果。

## 调整地址空间范围

在 uzCore 中,使用 mmap 来模拟页表,所有进程共用一个 64 位地址空间。因此,从地址空间范围这一角度来说,运行在 uzCore 上的用户程序所在的用户进程地址空间无法像 Zircon 要求的一样大。对于这一点,我们在为每一个用户进程设置地址空间时,手动进行分配,规定每一个用户进程地址空间的大小为 0x100_0000_0000,从 0x2_0000_0000 开始依次排布。0x0 开始至 0x2_0000_0000 规定为 uzCore 内核所在地址空间,不用于 mmap。图 3.3给出了 uzCore 在运行时若干个用户进程的地址空间分布。

与 uzCore 兼容,zCore 对于用户进程的地址空间划分也遵循同样的设计,但在裸机环境下,一定程度上摆脱了限制,能够将不同用户地址空间分隔在不同的页表中。如图 3.4所示,zCore 中将三个用户进程的地址空间在不同的页表中映射,但是为了兼容 uzCore 的运行,每一个用户进程地址空间中用户程序能够真正访问到的部分都仅有 0x100_0000_0000 大小。


## LibOS源代码分析记录

### zCore on riscv64的LibOS支持

* LibOS unix模式的入口在linux-loader main.rs:main()

初始化包括kernel_hal_unix，Host文件系统，其中载入elf应用程序的过程与zcore bare模式一样；

重点工作应该在kernel_hal_unix中的**内核态与用户态互相切换**的处理。

kernel_hal_unix初始化主要包括了，构建Segmentation Fault时SIGSEGV信号的处理函数，当代码尝试使用fs寄存器时会触发信号；

* 为什么要注册这个信号处理函数呢？

根据wrj的说明：由于 macOS 用户程序无法修改 fs 寄存器，当运行相关指令时会访问非法内存地址触发Segmentation Fault。故实现段错误信号处理函数，并在其中动态修改用户程序指令，将 fs 改为 gs

kernel_hal_unix还构造了**进入用户态**所需的run_fncall() -> syscall_fn_return()；

而用户程序需要调用syscall_fn_entry()来**返回内核态**；

Linux-x86_64平台运行时，用户态和内核态之间的切换运用了 fs base 寄存器；

* Linux 和 macOS 下如何分别通过系统调用设置 fsbase / gsbase 。

这个转换过程调用到了trapframe库，x86_64和aarch64有对应实现，而riscv则需要自己手动实现；

* 关于fs寄存器

查找了下，fs寄存器一般会用于寻址TLS，每个线程有它自己的fs base地址；

fs寄存器被glibc定义为存放tls信息，结构体tcbhead_t就是用来描述tls；

进入用户态前，将内核栈指针保存在内核 glibc 的 TLS 区域中。

可参考一个运行时程序的代码转换工具：[https://github.com/DynamoRIO/dynamorio/issues/1568#issuecomment-239819506](https://github.com/DynamoRIO/dynamorio/issues/1568#issuecomment-239819506?fileGuid=VMAPV7ERl7HbpNqg)

* **LibOS内核态与用户态的切换**

Linux x86_64中，fs寄存器是用户态程序无法设置的，只能通过系统调用进行设置；

例如clone系统调用，通过arch_prctl来设置fs寄存器；指向的struct pthread，glibc中，其中的首个结构是tcbhead_t

计算tls结构体偏移：

经过试验，x86_64平台，int型：4节，指针类型：8节，无符号长整型：8节；

riscv64平台，int型： 4节，指针类型：8节，无符号长整型：8节；

计算tls偏移量时，注意下，在musl中，aarch64和riscv64架构有#define TLS_ABOVE_TP，而x86_64无此定义

* 关于Linux user mode (UML)

"No, UML works only on x86 and x86_64."

[https://sourceforge.net/p/user-mode-linux/mailman/message/32782012/](https://sourceforge.net/p/user-mode-linux/mailman/message/32782012/?fileGuid=VMAPV7ERl7HbpNqg)

