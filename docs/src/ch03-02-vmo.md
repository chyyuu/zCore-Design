# 物理内存：VMO 对象

## VMO 简介

虚拟内存对象（Virtual Memory Object，VMO）表示虚拟内存的连续区域，可以将其映射到多个地址空间中。VMO用于内核空间和用户空间，来表示分页的虚存和物理内存。这是用户态进程和内核访问内存所需的基本管理结构。

VMO通过[vmo_create](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_create.md)系统调用创建，可以使用[vmo_read](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_read.md)和[vmo_write](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_write)对它们执行基本I/O操作。 并使用[vmo_set_size](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_set_size.md)设置VMO的大小， 相反，通过[vmo_get_size](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_get_size.md)可获取VMO当前大小。VMO的大小将由内核向上舍入至下一页面边界的大小。

通过[vmo_read](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_read.md)或[vmo_write](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_write.md)，或通过[vmar_map](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_map.md)写入VMO的映射，页面被按需提交（或分配）到VMO上。 通过使用带*ZX_VMO_OP_COMMIT*和*ZX_VMO_OP_DECOMMIT*的标志调用[vmo_op_range](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_op_range.md)，可以手动从VMO提交和解除页面，但这被视为低层次的操作。 [vmo_op_range](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_op_range.md)也可用于对VMO所拥有的页面进行缓存和锁操作。

具有涉及缓存策略的特殊使用方法的进程可以使用[vmo_set_cache_policy](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmo_set_cache_policy.md)来更改给定VMO的策略。 此用法通常适用于设备驱动程序。

## 系统调用



### VMO相关系统调用

- [`zx_vmo_create()`](https://fuchsia.dev/docs/reference/syscalls/vmo_create) -创建一个新的vmo
- [`zx_vmo_read()`](https://fuchsia.dev/docs/reference/syscalls/vmo_read) -从vmo读取
- [`zx_vmo_write()`](https://fuchsia.dev/docs/reference/syscalls/vmo_write) -写入vmo
- [`zx_vmo_get_size()`](https://fuchsia.dev/docs/reference/syscalls/vmo_get_size) -获取vmo的大小
- [`zx_vmo_set_size()`](https://fuchsia.dev/docs/reference/syscalls/vmo_set_size) -调整vmo的大小
- [`zx_vmo_op_range()`](https://fuchsia.dev/docs/reference/syscalls/vmo_op_range) -在一定范围的vmo上执行操作
- [`zx_vmo_set_cache_policy()`](https://fuchsia.dev/docs/reference/syscalls/vmo_set_cache_policy) -为vmo设置的页面设置Cache策略
- [`zx_vmar_map()`](https://fuchsia.dev/docs/reference/syscalls/vmar_map) -将VMO映射到进程
- [`zx_vmar_unmap()`](https://fuchsia.dev/docs/reference/syscalls/vmar_unmap) -从进程中取消内存映射

## 实现 VMO 对象框架

> 实现 VmObject 结构，其中定义 VmObjectTrait 接口，并提供三个具体实现 Paged, Physical, Slice

### 关键数据结构

#### VmObject

```rust
/// A Virtual Memory Object (VMO) represents a contiguous region of virtual memory
/// that may be mapped into multiple address spaces.
pub struct VmObject {
    base: KObjectBase,
    _counter: CountHelper,
    resizable: bool,
    trait_: Arc<dyn VMObjectTrait>,
    inner: Mutex<VmObjectInner>,
}
struct VmObjectInner {
    parent: Weak<VmObject>,
    children: Vec<Weak<VmObject>>,
    mapping_count: usize,
    content_size: usize,
}
```



## HAL：用文件模拟物理内存

> 初步介绍 mmap，引出用文件模拟物理内存的思想
>
> 创建文件并用 mmap 线性映射到进程地址空间
>
> 实现 pmem_read, pmem_write

## 实现物理内存 VMO

> 用 HAL 实现 VmObjectPhysical 的方法，并做单元测试

## 实现切片 VMO

> 实现 VmObjectSlice，并做单元测试

# 物理内存：按页分配的 VMO

## 简介

> 说明一下：Zircon 的官方实现中为了高效支持写时复制，使用了复杂精巧的树状数据结构，但它同时也引入了复杂性和各种 Bug。
> 我们在这里只实现一个简单版本，完整实现留给读者自行探索。
>
> 介绍 commit 操作的意义和作用

## HAL：物理内存管理

> 在 HAL 中实现 PhysFrame 和最简单的分配器

## 辅助结构：BlockRange 迭代器

> 实现 BlockRange

## 实现按页分配的 VMO

> 实现 for_each_page, commit, read, write 函数

## VMO 复制

> 实现 create_child 函数