## 概述

zCore在硬件支持方面，主要包括 x86_64、ARM64和RISC-V 64 架构。在应用兼容性支持方面，zCore 主要支持 Zircon 用户程序和Linux 用户程序。



### kernel_hal 模块分析

由于 zCore 的设计初期就已经考虑到了针对不同平台架构设计兼容抽象层 kernel_hal ，所以主要的工作在于为 kernel_hal_bare 增加 RiscV 架构。除此之外，我注意到中断跳转等工作主要在 trapframe 模块中实现，这个模块定义了中断帧、用户中断上下文等结构体，也是与硬件平台相关的。因为这个模块同时还是 rCore 的依赖模块，而 rCore 的 RiscV 平台移植已经完成，所以 trapframe 模块对 RiscV 平台的支持已经较为完善。

kernel_hal 模块中的 dummy.rs 文件中包含了大部分平台相关的，需要进行移植的函数。经过整理，可能需要重写的函数如下：

- 线程相关

| 接口名称        | 功能描述                         |
| --------------- | -------------------------------- |
| Thread::spawn   | 将一个线程加入调度器             |
| Thread::set_tid | 设置当前执行线程的 id            |
| Thread::get_tid | 获取当前执行线程的 id            |
| context_run     | 根据指定的线程的上下文执行该线程 |

Thread 结构体内定义的相关接口主要用于将线程加入调度，并进行简单的管理。需要将目标线程包装为 Future ，然后通过 executor 库的相关接口，将该 Future 加入调度。

传统的线程调度是通过线程调度器进行调度，比如以前在 rCore 中叫做 scheduler 。executor 和 scheduler 的区别在于，executor 是进行协程调度，scheduler 是进行线程调度。前面提到，在 zCore 中，在创建新需要将线程包装为 Future ，而 Future 是可以被作为协程执行的。所以 zCore 中的线程，本质上是协程。在进行协程切换时，由于要实现线程切换的效果，还需要额外增加页表切换的步骤。

将线程包装为协程的好处是，可以利用 Rust 语言中的异步机制。Rust 语言的无栈协程可以减少线程之间切换所需要时间，同时能减少多线程下的内存开销。在 executor 中，所有协程可以共用一个栈，因此空间开销变小了。在 Linux 中，一般每个线程需要至少 8K 的内核栈，而一个 Future 占用的空间大约是几百 Bytes ，只需要预留空间保存 yield 的时候的变量就行了，并不需要保存整个栈。

使用 Future 还有一个好处，标准化了阻塞-唤醒这套机制，Future、Poll、Waker 都包含在了内核里面，避免了死锁或者丢失唤醒的问题。

context_run 主要用于进行上下文切换，只是简单的调用了 trapframe 模块中的内容。将在后面的部分进行详细的分析。

- 页表相关

| 接口名称               | 功能描述                                       |
| ---------------------- | ---------------------------------------------- |
| PageTable::current     | 获取当前页表                                   |
| PageTable::new         | 创建一个新的页表                               |
| PageTable::map         | 将指定的物理页帧映射到指定的虚拟地址           |
| PageTable::unmap       | 解除指定虚拟地址的映射                         |
| PageTable::protect     | 修改指定虚拟地址的标志位                       |
| PageTable::query       | 查询指定虚拟地址对应的物理页帧                 |
| PageTable::table_phys  | 查询根页表的物理地址                           |
| PageTable::map_many    | 将多个物理页帧映射到指定的连续的虚拟地址       |
| PageTable::map_cont    | 将多个连续的物理页帧映射到指定的连续的虚拟地址 |
| PageTable::unmap_count | 接触某个虚拟地址起始的连续的一片范围的映射     |

PageTable 结构体提供的接口主要用于对页表进行一些基本操作。由于页表与 CPU 的平台架构相关性很高，而且非常底层，所以会需要使用到许多汇编指令，这里可以复用 rCore 中的 riscv 模块。rCore 中的 riscv 模块将常用的 riscv 指令和指令组合包装成为函数的形式，可以更加方便的使用。同时，还提供了一些底层的页表相关的操作，因此在实现 PageTable 结构体的接口时，可以使用这些 riscv 模块中的页表操作接口简化实现。

- 物理内存相关

| 接口名称                    | 功能描述                             |
| --------------------------- | ------------------------------------ |
| PhysFrame::alloc            | 分配一个物理页帧                     |
| PhysFrame::alloc_contiguous | 分配一片连续的物理内存               |
| PhysFrame::zero_frame_addr  | 获取“零页”的物理地址                 |
| PhysFrame::drop             | 物理页帧的析构函数，用于回收该页帧   |
| pmem_read                   | 将指定物理页帧的内容读入缓冲区       |
| pmem_write                  | 将缓冲区的内容写入指定页帧           |
| frame_copy                  | 将指定物理页帧的内容复制到另一个页帧 |
| frame_zero_in_range         | 将指定物理内存区域置为零             |
| frame_flush                 | 将指定物理页帧的缓存刷入可持久化设备 |

物理页帧的管理过程基本不涉及硬件，因此可以复用 x86_64 架构的代码，只需要进行简单的修改即可。为了防止内存泄露，所以为物理页帧实现了析构函数。在 Rust 语言中，需要为结构体实现 Drop trait ，才能达到析构的效果。pmem_read/write 用于读写物理地址，只需要将传入的物理地址转换为虚拟地址，然后将其作为指针进行正常的读写就可以了。在 Zircon 中，默认内存的初始状态为全零，并且需要提供零页，所以 HAL 层也设计提供了一个内容为全零的物理页帧。

- 串口相关

| 接口名称            | 功能描述                     |
| ------------------- | ---------------------------- |
| serial_set_callback | 设置串口获得输入后的回调函数 |
| serial_read         | 从串口读入字符               |
| serial_write        | 向串口输出字符               |

通过 RustSBI 完成对串口的写操作，每次产生键盘中断，都会将键盘字符存入 vector 中，同时还会检察是否满足触发回调函数的条件，

- 时钟相关

| 接口名称   | 功能描述                                       |
| ---------- | ---------------------------------------------- |
| timer_now  | 获取当前 CPU 时间                              |
| timer_set  | 设置一个计时器，当截止时间到达时会触发回调函数 |
| timer_tick | 处理时钟中断                                   |

timer_set 会将 deadline 和回调函数加入计时器。每次 timer_tick 的时候，会通知计时器当前的系统时间。如果系统时间超过 deadline ，则会触发相应的回调函数。

- 中断请求相关

| 接口名称                                 | 功能描述                         |
| ---------------------------------------- | -------------------------------- |
| InterruptManager::handle                 | 处理中断                         |
| InterruptManager::add_handle             | 将中断处理方式注册到中断请求表中 |
| InterruptManager::remove_handle          | 将中断处理方式从中断请求表中删除 |
| InterruptManager::hal_irq_allocate_block | 为中断请求分配一片连续的位置     |
| InterruptManager::free_block             | 释放将中断请求的位置             |
| InterruptManager::overwrite_handler      | 重新指定中断请求的处理方式       |
| InterruptManager::enable                 | 开启中断                         |
| InterruptManager::disable                | 关闭中断                         |
| InterruptManager::configure              | 修改中断处理的配置               |
| InterruptManager::is_valid               | 检查中断号是否有效               |

初步移植只计划支持简单的 riscv64 静态链接程序，不会设计到过多的 irq ，只需要处理 syscall 等简单的中断，所以这部分内容基本不需要处理。

### 中断处理模块分析

前面说到，在 zCore 中，有两套中断处理机制，一套用于处理来自内核态的中断，一套用于处理来自用户态的中断。这种处理方式看起来会比较神奇，但是这种写法可以使得内核态与用户态的切换变得更加自然。

我们先来回顾一下，以往在 rCore 中的用户进程创建过程。为了能够进入用户态，需要强行在内核栈上创建一个 trapframe ，然后通过 sret 跳转进入用户态。但是在 zCore 中，内核态进入用户态的过程，变成了一个函数调用。在内核中创建用户进程，会调用 new_thread 函数。在这个函数中会通过调用 kernel_hal::conntext_run 进入到用户程序。

kernel_hal 模块中的 conntext_run 函数的实现，就是调用了 trapframe 模块中的 run_user 函数。run_user 函数功能就是对内核态到用户态的上下文切换。因此，该函数对上层的表现为一个函数的形式，本质上是针对不同的硬件平台架构，通过调用函数的过程进行上下文切换，并且将内核执行过程中的上下文保存到内核栈中。示例代码如下：

```rust
loop {
    let mut cx = thread.wait_for_run().await;
    if thread.state() == ThreadState::Dying {
        break;
    }
    kernel_hal::context_run(&mut cx);
    let trap_num = riscv::register::scause::read().bits();
    match trap_num {
        _ if kernel_hal_bare::arch::is_page_fault(trap_num) => {
            ...
        }
        _ if kernel_hal_bare::arch::is_timer_intr(trap_num) => {
            ...
        }
        _ if kernel_hal_bare::arch::is_syscall(trap_num) => handle_syscall(&thread, &mut cx).await,
        _ => {
            panic!("unhandled trap {:?}", riscv::register::scause::read().cause());
        }
    }
    thread.end_running(cx);
}
```

首先，通过 thread.wait_for_run() 获得将要运行的线程的上下文，然后将该上下文传给 context_run 函数。context_run 函数会使用这个上下文启动该线程。这里传入的上下文是可变的，这是因为线程启动运行会导致上下文的变化。当用户态线程因发生中断、异常、主动发起系统调用而进入内核态时，在内核的视角来看，就是 context_run 函数执行完毕并返回。返回之后，通过读取该线程变化后的上下文中的中断信息，即可进行相应的中断处理。

### trapframe 模块分析

trapframe 模块的工作原理为，实现一个名为 trap_entry 的函数。在编写操作系统时，需要进行如下初始化，将 trap_entry 的地址写入 stvec 寄存器，模式设置为 direct 。当产生中断时，会由 cpu 自动跳转至 stvec 寄存器中的地址，即 trap_entry 函数。

trap_entry 函数首先会判断中断来源，如果中断来自于用户态，则会保存用户栈的指针，然后将 sp 设置为内核栈。如果中断来自于内核态，则 sscratch 的数值为 0 ，这时候我们可以在当前的内核栈继续进行工作。具体做法如下：

```asm
trap_entry:
    csrrw sp, sscratch, sp
    bnez sp, trap_from_user
trap_from_kernel:
    csrr sp, sscratch
    addi sp, sp, -34 * XLENB
trap_from_user:
    ...
```

如果系统进入了用户态，则 sscratch 寄存器的数值会被设置为内核栈的地址，如果系统正处于内核态，则 sscratch 寄存器的值会被设置为 0 。因此可以通过 sscrtch 的数值，判断中断是来源于内核态还是来源于用户态，并且将栈设为合适的值。由于 sscratch 寄存器是控制状态寄存器，并不能直接通过常规的命令来对其进行读写。所以，首先通过 csrrw 指令将 sp 和 sscratch 的数值进行交换，这样即完成了 sp 数值的保存，还将 sscratch 的数值读入了通用寄存器 sp ，从而可以更加轻易的对其进行操作。接下来通过 bnez 指令判断 sp ，即原 sscratch 的值，如果不为 0 ，则表示中断来源于用户态。由于中断来源于用户态时，sscratch 的值为内核栈地址。由于前面已经通过 csrrw 指令交换了 sp 和 sscratch 的数值，则此时用户栈的地址被存入 sscratch 寄存器，而 sp 被设置为了内核栈。如果 bnez 指令判断 sp 的数值为 0 ，则中断来源是内核态，此时需要通过 csrr 指令将 sscratch 寄存器内的数值，即原内核栈地址写回 sp ，并开辟一段空间，用于保存中断帧。`34 * XLENB` 大小的空间，就是用于保存 32 个通用寄存器和 sstatus、sepc 寄存器。

在完成了栈的切换之后，trap_entry 函数会保存中断现场，然后再次根据中断来源进行不同的处理。如果中断来自内核态，则会通过 j trap_handler 指令跳转到用于处理内核中断的函数。在 trapframe 模块中，trap_handler 函数内部是 unimplemented ，标记为 weak 链接。因此需要 OS 为其实现一个名为 trap_handler 的函数，该函数需要能够对不同类型的中断进行相应的处理。不同的硬件平台架构有不同的处理方式，因此这部分内容应该 kernel_hal_bare 模块中进行实现。但是如果中断来自用户态，则会需要通过 ret 回到内核态。实例代码如下：

```asm
beqz t1, end_trap_from_user
end_trap_from_kernel:
    mv a0, sp               # first arg is TrapFrame
    la ra, trap_return      # set return address
    j trap_handler
end_trap_from_user:
    # load callee-saved registers
    ...
    addi sp, sp, 14 * XLENB
    ret
```

这里的做法和前面类似，仍然需要根据中断来源进行不同的处理。如果中断来源于内核态，则会将 a0 寄存器设置为 sp 的值，ra 寄存器设置为 trap_return 函数的地址，然后调到 trap_handler 函数，处理来自内核态的中断。trap_handler 的定义为：`extern "C" fn trap_handler(tf: &mut TrapFrame)` ，extern "C" 表示 使用 C 语言的规范，在此情况下，a0 寄存器的值即为函数的第一个参数，此时 a0 刚刚保存的 sp 的顶端存放着 trapframe ，因此可以根据 trapframe 内的数据对中断进行相应的处理。

如果中断来源于用户态，前面说到，内核进入用户态的过程，是通过调用 kernel_hal::context_run 函数，因此是遵循了函数调用的规范。需要先根据规范将被调用者保存的寄存器恢复，然后释放这部分栈的空间，最后再通过 ret 回到内核态，进行用户态中断的处理。

使用两套中断处理机制，分别处理来自内核态的中断和来自用户态的中断的这种做法，其实是有一些冗余的，尤其是这两部分的代码在很多地方是重复的。之所以写成两套是为了 libOS 服务。由于 libOS 完全运行在用户态，所以实际上是没有内核态的。用户程序的系统调用会被替换为函数调用，而中断会以类似信号的形式通知内核进行处理。

### 其余部分（TODO）

除此之外，在进行 sys_fork 的时候，需要拷贝相应的寄存器和页表等数据，这部分内容也与平台相关所以可能需要进行相应的适配。

syscall sepc+4