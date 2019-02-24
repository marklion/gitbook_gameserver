# 1.3 游戏服务器架构

第一章我们已经分析过游戏服务器的需求并得出结论：

> 服务器需求  
> 1. 接收客户端连接，处理客户端断开  
> 2. 按照上表处理客户端传来的各类消息  
> 3. 按照上表向客户端发送各类消息

所以，接下来我们将需求和zinx框架结合起来设计一下服务器的软件架构。 

## 1.3.1 目录结构

> zinx框架的使用套路：四个类

在创建game目录并创建四对文件分别实现四个类。

+ GameChannel继承TcpListenChannel，实现基于TCP连接的玩家上下线管理
+ GameMessage继承IdMessage，实现不同类型消息的封装
+ GameProtocol继承Aprotocol，实现TCP封包或GameMessage的分发
+ GameRole继承IdMsgRole，实现各种类型消息的处理

```bash
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

main.cpp文件是服务器程序的入口，用于执行server实例的run函数。

```cpp
#include <zinx/zinx.h>
#include "GameChannel.h"

using namespace std;

int main()
{
    Server *pxServer = Server::GetServer();

    pxServer->init();
    /*创建tcp监听对象并添加到server实例中*/
    pxServer->install_channel(new GameChannel());
    pxServer->run();

    return 0;
}
```

## 1.3.1 上下线

简单讲：收到TCP连接代表登陆，连接断开代表下线。  
套用到之前四个类的设计思路，我们用TCPDataCHannel对象维护和客户端的两连接，绑定协议对象和角色对象，实现数据处理。继承TCPLstChannel类实现连接建立后的相关处理对象实例化。

```bash
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



