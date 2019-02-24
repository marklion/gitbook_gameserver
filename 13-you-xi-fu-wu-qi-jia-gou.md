# 1.3 游戏服务器架构

第一章我们已经分析过游戏服务器的需求并得出结论：

> 服务器需求
1. 接收客户端连接，处理客户端断开
2. 按照上表处理客户端传来的各类消息
3. 按照上表向客户端发送各类消息

所以，接下来我们将需求和zinx框架结合起来设计一下服务器的软件架构。
当前该游戏只有两个场景：**上下线和玩（聊天跟玩本质相同）**，所以我们从这两个场景分别入手。

## 1.3.1 上下线

简单讲：收到TCP连接代表登陆，连接断开代表下线。
套用到之前四个类的设计思路，我们用TCPDataCHannel对象维护和客户端的两连接，绑定协议对象和角色对象，实现数据处理。继承TCPLstChannel类实现连接建立后的相关处理对象实例化。

```Bash
game/
├── GameChannel.cpp
├── GameChannel.h
├── GameMessage.cpp
├── GameMessage.h
├── GameProtocol.cpp
├── GameProtocol.h
├── GameRole.cpp
├── GameRole.h
├── main.cpp
└── Makefile

```



