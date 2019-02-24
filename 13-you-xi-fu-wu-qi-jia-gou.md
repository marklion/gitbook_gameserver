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

Makefile文件内，编译链接所有cpp文件为game可执行文件。

```Makefile
game:*.cpp
	g++ -std=c++11 $^ -lzinx -o $@
```

## 1.3.2 GameMessage类设计

GameMessage类继承IdMessage，用来封装各种类型的消息，消息不同，其承载的内容也不同。这里选择protobuf技术，用来解析和封装消息内容（序列化和反序列化）。关于protobuf的使用，我们后边会详细介绍

GameMessage.h文件

```cpp
#ifndef _GAME_MESSAGE_H_
#define _GAME_MESSAGE_H_

#include <zinx/zinx.h>
#include <google/protobuf/message.h>

class GameMessage:public IdMessage{
public:
    /*新增属性pxProtoBufMsg用于存放消息内容*/
    google::protobuf::Message *pxProtoBufMsg = NULL;

    /*构造函数要指定当前消息类型编号*/
    GameMessage(int _id);
    /*析构时，要释放pxProtoBufMsg*/
    virtual ~GameMessage();
    
    /*新增两个成员函数，用于数据和请求直接的转换*/
    bool ParseBuff2Msg(unsigned char *pucDataBuff, int iLength);
    int SerialMsg2Buff(unsigned char *pucDataBuff, int iBufLength);
};

#endif
```

## 1.3.3 GameRole类设计

GameRole类继承IdMsgRole，用来处理各种类型的消息。需要重写init函数和fini函数，实现多个消息类型和对应的处理函数的注册，玩家上线后的操作和玩家下线前的操作。

GameRole.h文件

```cpp
#ifndef _GAME_ROLE_H_
#define _GAME_ROLE_H_

#include <zinx/zinx.h>
#include <string>

class GameRole:public IdMsgRole{
public:
    /*记录玩家的唯一ID*/
    int iPid = 0;
    /*玩家的所在位置*/
    float x = 0;
    float y = 0;
    float z = 0;
    float v = 0;
    /*玩家的昵称*/
    std::string szName;
    
    /*init函数实现消息处理函数的注册和玩家上线后的数据同步*/
    virtual bool init();
    /*fini函数实现玩家退出前的善后*/
    virtual void fini();
};

#endif
```

## 1.3.4 GameProtocol类设计

GameProtocol类继承Aprotocol，与上一章的功能基本没有差别。需要增加：

+ 创建Amessage对象时要调用GameMessage的解析函数
+ 构造原始数据时要调用GameMessage的序列化函数

GameProtocol.h文件

```cpp
#ifndef _GAME_PROTOCOL_H_
#define _GAME_PROTOCOL_H_

#include <zinx/zinx.h>
#include "GameMessage.h"
#include "GameRole.h"

class GameProtocol:public Aprotocol{
private:
    /*用于暂存当前收到的报文*/
    RawData stCurBuffer;
    /*用于绑定处理消息的角色对象*/
    GameRole *pxBindRole = NULL;

    /*获取小字节序的整数*/
    int GetLittleEndNumber(const unsigned char *pucData);

    /*设置小字节序的整数*/
    void SetLittleEndNumber(int _Num, unsigned char *pucData);

    /*将原始数据转化成消息*/
    GameMessage *GetMessageFromRaw(int _MsgId, const unsigned char *pucParseBegin, int iLengthLast);
public:

    /*通过构造函数指定当前协议绑定的处理角色*/
    GameProtocol(GameRole *_bindRole);

    /*析构时，还需要从server中摘除并销毁绑定的角色对象*/
    virtual ~GameProtocol();

    /*封包的核心逻辑*/
    virtual bool raw2request(const RawData * pstData, std :: list < Request * > & _ReqList);
    
    /*和接收数据的逻辑正好相反，用来执行消息到原始数据的转换*/
    virtual bool response2raw(const Response * pstResp, RawData * pstData);
};

#endif
```

GameProtocol.cpp文件

```cpp

```

## 1.3.5 GameChannel类设计

GameChannel继承TcpListenChannel，重写TcpAfterConnection方法，实现通道，协议，角色这三者的绑定和添加到server实例。



