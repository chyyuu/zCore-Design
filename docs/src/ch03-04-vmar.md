# 虚拟内存：VMAR 对象

## VMAR 简介

VMAR（VM_Address_Region）是虚拟内存地址空间的连续区域。内核和用户空间使用VMAR来表示地址空间的分配。

每个进程启动时都带有一个包含整个地址空间的VMAR（根VMAR）（请参阅[process_create](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/process_create.md)）。 每个VMAR可以在逻辑上划分为任意数量的非重叠部分，每个部分代表虚拟内存的一个映射或间隙的子VMAR。 使用[vmar_allocate](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_allocate.md)创建子VMAR，以及通过使用[vmar_map](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_map.md)创建虚拟内存映射。

VMAR具有允许映射权限的分层权限模型。 例如，根VMAR允许读取，写入和可执行映射。 可以创建仅允许读写映射的子VMAR，其中创建允许可执行映射的子项是非法的。

使用vmar_allocate创建VMAR时，其父VMAR会保留对它的引用，因此，即使关闭子VMAR的所有句柄，则子对象及其后代也将在地址空间中保持存活状态。 为了将子节点与地址空间断开连接，必须在子节点的句柄上调用[vmar_destroy](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_destroy.md)。

默认情况下，地址空间的所有分配都是随机的，并在VMAR创建时，调用者可以选择使用哪种随机化算法。 默认的分配器尝试在VMAR的整个空间上分配，使用*ZX_VM_COMPACT*选项的备用分配器会试图在VMAR内保持分配尽量簇聚在一起，但也是在空间范围内的随机位置。 建议开发者使用默认的分配器。

VMAR可选地支持固定偏移映射模式（也称为特定映射），此模式可用于创建守卫(guard)页面或确保映射的相对位置。 每个VMAR都可能具有*ZX_VM_CAN_MAP_SPECIFIC*权限，无论其父VMAR是否具有该权限。



## VMAR相关系统调用

- [vmar_allocate](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_allocate.md) —— 创建新的子VMAR
- [vmar_map](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_map.md) —— 将VMO映射到进程
- [vmar_unmap](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_unmap.md) —— 从进程中取消映射的内存区域
- [vmar_protect](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_protect.md) —— 调整内存访问权限
- [vmar_destroy](https://github.com/zhangpf/fuchsia-docs-zh-CN/blob/master/zircon/docs/syscalls/vmar_destroy.md) —— 销毁VMAR及其所有子VMAR

## 实现 VMAR 对象框架

### 关键数据结构

#### VmAddressRange
```rust
/// Virtual Memory Address Regions
pub struct VmAddressRegion {
    flags: VmarFlags,
    base: KObjectBase,
    _counter: CountHelper,
    addr: VirtAddr,
    size: usize,
    parent: Option<Arc<VmAddressRegion>>,
    page_table: Arc<Mutex<dyn PageTableTrait>>,
    /// If inner is None, this region is destroyed, all operations are invalid.
    inner: Mutex<Option<VmarInner>>,
}
```

#### VmMapping

```rust
/// Virtual Memory Mapping
pub struct VmMapping {
    /// The permission limitation of the vmar
    permissions: MMUFlags,
    vmo: Arc<VmObject>,
    page_table: Arc<Mutex<dyn PageTableTrait>>,
    inner: Mutex<VmMappingInner>,
}
```

### 关键函数

实现 create_child, map, unmap, destroy 函数，并做单元测试验证地址空间分配。



## HAL：用 mmap 模拟页表

> 实现页表接口 map, unmap, protect

## 实现内存映射

> 用 HAL 实现上面 VMAR 留空的部分，并做单元测试验证内存映射
