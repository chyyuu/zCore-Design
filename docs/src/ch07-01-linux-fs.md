

## Linux文件相关的数据结构

这里主要是与文件相关的各种数据结构，包括常规文件、管道、输入输出设备文件等。

| [FcntlFlags](struct.FcntlFlags.html)           | fcntl flags                    |
| ---------------------------------------------- | ------------------------------ |
| [File](struct.File.html)                       | file implement struct          |
| [FileDesc](struct.FileDesc.html)               | file descriptor wrapper        |
| [FileFlags](struct.FileFlags.html)             | file operate flags             |
| [MemBuf](struct.MemBuf.html)                   | memory buffer for device       |
| [OpenOptions](struct.OpenOptions.html)         | file open options struct       |
| [Pipe](struct.Pipe.html)                       | pipe struct                    |
| [PipeData](struct.PipeData.html)               | Pipe inner data                |
| [Pseudo](struct.Pseudo.html)                   | Pseudo INode struct            |
| [RandomINode](struct.RandomINode.html)         | random INode struct            |
| [RandomINodeData](struct.RandomINodeData.html) | random INode data struct       |
| [STDIN](struct.STDIN.html)                     | STDIN global reference         |
| [STDOUT](struct.STDOUT.html)                   | STDOUT global reference        |
| [Stdin](struct.Stdin.html)                     | Stdin struct, for Stdin buffer |
| [Stdout](struct.Stdout.html)                   | Stdout struct, empty now       |



### 通用文件接口trait

主要涉及如下相关功能

- 常规文件与目录的读写
- 对Socket的支持
- 对轮询Epoll的支持

```rust
pub trait FileLike: KernelObject {
    /// read to buffer
    async fn read(&self, buf: &mut [u8]) -> LxResult<usize>;
    /// write from buffer
    fn write(&self, buf: &[u8]) -> LxResult<usize>;
    /// read to buffer at given offset
    async fn read_at(&self, offset: u64, buf: &mut [u8]) -> LxResult<usize>;
    /// write from buffer at given offset
    fn write_at(&self, offset: u64, buf: &[u8]) -> LxResult<usize>;
    /// wait for some event on a file descriptor
    fn poll(&self) -> LxResult<PollStatus>;
    /// wait for some event on a file descriptor use async
    async fn async_poll(&self) -> LxResult<PollStatus>;
    /// manipulates the underlying device parameters of special files
    fn ioctl(&self, request: usize, arg1: usize, arg2: usize, arg3: usize) -> LxResult<usize>;
    /// manipulate file descriptor
    fn fcntl(&self, cmd: usize, arg: usize) -> LxResult<usize>;
}
```

