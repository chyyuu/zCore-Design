# 同时等待多个信号：Port 对象

## Port 对象概述

端口(port)允许线程等待从各种事件对象传递的数据包。这些事件包括port上的显式排队，绑定到端口的其他句柄上的异步等待以及来自IPC的异步消息传递。

## Port系统调用

- port_create ：创建端口
- port_queue：发送数据包到端口
- port_wait：等待数据包到达端口

## 实现 Port 对象框架

> 定义 Port 和 PortPacket 结构体

## 实现事件推送和等待

> 实现 KernelObject::send_signal_to_port 和 Port::wait 函数，并做单元测试
