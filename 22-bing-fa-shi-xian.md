# 2.2 并发实现

选择IO多路复用（epoll）的机制作为并发模型。

> 为什么是epoll, 而不是select或poll？

## 2.2.1 添加和摘除fd

设计成员函数install和uninstall用于实现从epoll实例中