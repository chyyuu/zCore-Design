# Event和EventPair对象

## Event 对象

Event是可通知用户的对象，为用户空间保留的8个信号位（*ZX_USER_SIGNAL_0*至*ZX_USER_SIGNAL_7*）可以用于设置，清除和等待事件。

## Event系统调用

- event_create：创建event
- object_signal：设置或清除对象上的用户信号

## EventPair 对象

Event pair是相互连接的用户可通知的一对对象，为用户空间保留的8个信号位（*ZX_USER_SIGNAL_0*至*ZX_USER_SIGNAL_7*）可以用于设置或清除本地或Event Pair相对端点上的信号。

## 系统调用

- eventpair_create：创建一对相互连接的事件
- object_signal_peer：在相对端点上设置或清除用户信号


