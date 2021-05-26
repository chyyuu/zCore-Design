# 信号和等待

用户进程/线程和zCore中的内核对象之间需要有并发通知的机制，从而便于不同内核对象之间实现松耦合的并发执行。为此zCore提出了`event`等通知机制。用户进程可以对多个信号bit位进行设置、清除和等待，来进行某种通知的收发。目前的主要结构包括：

| [Event](struct.Event.html)               | 用于并发编程的可通知的事件     |
| ---------------------------------------- | ------------------------------ |
| [EventPair](struct.EventPair.html)       | 用于并发编程的可通知的事件对   |
| [Futex](struct.Futex.html)               | 创建用户态同步互斥的机制       |
| [PacketSignal](struct.PacketSignal.html) | 信号类的数据包                 |
| [Port](struct.Port.html)                 | 事件的组合                     |
| [PortPacket](struct.PortPacket.html)     | 发给Port的数据包               |
| [Timer](struct.Timer.html)               | 在将来某个时刻发出信号的定时器 |