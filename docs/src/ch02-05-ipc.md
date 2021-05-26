





## IPC机制

目前zCore支持的基本IPC机制如下所示：

| [Channel](struct.Channel.html)             | 双向进程间通讯                                               |
| ------------------------------------------ | ------------------------------------------------------------ |
| [Fifo](struct.Fifo.html)                   | 先进先出进程间队列                                           |
| [Socket](struct.Socket.html)               | 双向流式IPC传输                                              |



### Channel机制

Channel是消息的双向传输，包含一定数量的字节数据和一定数量的句柄

channel维护了一个有序的消息队列，以便在任一方向上传递由一些数据和句柄组成的消息。调用*zx_channel_write()* 会将一条消息加入队列，而调用*zx_channel_read()* 会使一条消息出列（如果队列中消息存在的话）。线程可以被阻塞并挂起，直到通过*zx_object_wait_one()* 或其他等待机制获得待处理消息。

另外，调用*zx_channel_call()* 会在channel的一个方向上写入消息，并等待另一端的响应，而后从channel中读取响应消息。在call模式中，通过消息的前4个字节可以识别相应的响应，它被称为事务ID。内核为使用 *zx_channel_call()* 写入的消息提供不同的事务ID（始终将高位bit设置为1）。

通过channel发送消息的过程包括两个步骤。其一，将数据原子地写入channel，并将消息中所有句柄的所有权移动到此channel中。该操作将始终持有句柄：在调用结束时，所有句柄都已在channel中或全部被丢弃（在发生错误的情况下）。其二，和第一个操作类似：在读取通道之后，下一个要读取的消息中的所有句柄将原子地移动到进程的句柄表中，所以要么保留在channel中，要么被丢弃（仅当调用者指定*ZX_CHANNEL_READ_MAY_DISCARD*选项时）。

与许多其他内核对象类型不同的是，channel不可复制。因此，只有一个与channel端点(endpoint)相关联的句柄。

由于这些属性（channel的消息以原子方式移动其句柄内容，并且channel不可复制），内核只需强制要求channel和handle不可被写入到自己处的简单规则，就可以避免复杂的垃圾回收，生命周期管理或循环检测等操作。

### Channel 系统调用

- channel_call：以同步的方式发送消息并返回响应
- channel_create：创建新的channel
- channel_read：从channel中读取消息
- channel_write：向channel写入消息
- object_wait_one：等待某个对象上的信号



### Socket机制

socket是双向的流式传输。与channel的不同之处在于，socket仅能传输数据（而不移动句柄）。通过*zx_socket_write*写入数据到socket的一端，并通过*zx_socket_read*从相对端点读取。创建时，socket的两端都是可写和可读的。 通过zx_socket_write的ZX_SOCKET_SHUTDOWN_READ和ZX_SOCKET_SHUTDOWN_WRITE选项，可以关闭socket的一端以禁用从这一端读取和/或写入数据。

### Socket系统调用

- [socket_accept](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/socket_accept.md)：通过socket接收另一个socket对象
- socket_create：创建新socket
- socket_read：从socket中读取数据
- socket_share：通过socket发送另一个socket对象
- socket_write：写入数据到socket

### Fifo机制

FIFO类似于管道的机制，其读写操作比socket或channel更高效，但对元素和缓冲区的大小有严格的限制。

### Fifo系统调用

- fifo_create：创建fifo
- fifo_read]：从fifo中读取数据
- fifo_write：写入数据到fifo中

