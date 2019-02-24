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
    bool ParseBuff2Msg(const unsigned char *pucDataBuff, int iLength);
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
#include "GameProtocol.h"

using namespace std;

/**********************
描述：获取小字节序的整数
参数：pucData是输入数据起始地址
返回值：转换后的整数
**********************/
int GameProtocol::GetLittleEndNumber(const unsigned char * pucData)
{
    int iRet = 0;
    iRet = pucData[0];
    iRet += pucData[1];
    iRet += pucData[2];
    iRet += pucData[3];

    return iRet;
}

/**********************
描述：设置小字节序的整数
参数：
    _Num是待转换的整数
    pucData是输出数据起始地址
    **********************/
void GameProtocol::SetLittleEndNumber(int _Num, unsigned char * pucData)
{
    pucData[0] = _Num & 0xff;
    pucData[1] = (_Num >> 8) & 0xff;
    pucData[2] = (_Num >> 16) & 0xff;
    pucData[3] = (_Num >> 24) & 0xff;
}

/**********************
描述：将原始数据转化成消息
参数：
    _MsgId是当前消息ID
    pucParseBegin是消息内容字段的起始地址
    iLengthLast是消息内容的长度
返回值：基于报文创建的消息对象（堆对象）
**********************/
GameMessage *GameProtocol::GetMessageFromRaw(int _MsgId, const unsigned char * pucParseBegin, int iLengthLast)
{
    GameMessage *pstRet = NULL;

    pstRet = new GameMessage(_MsgId);
    pstRet->ParseBuff2Msg(pucParseBegin, iLengthLast);

    return pstRet;
}

/*通过构造函数指定当前协议绑定的处理角色*/
GameProtocol::GameProtocol(GameRole *_bindRole):pxBindRole(_bindRole)
{
}

/*析构时，还需要从server中摘除并销毁绑定的角色对象*/
GameProtocol::~GameProtocol()
{
    if (NULL != pxBindRole)
    {
        Server::GetServer()->del_role("GameRole", pxBindRole);
        delete pxBindRole;
    }
}

/*封包的核心逻辑*/
bool GameProtocol::raw2request(const RawData * pstData, std :: list < Request * > & _ReqList)
{
    bool bRet = false;
    int iCurPos = 0;//记录当前数据处理的游标

    /*先将新收到的数据追击到之前缓存的对象中*/
    stCurBuffer.AppendData(pstData);

    /*循环处理缓存对象中的数据，当数据长度小于8时退出*/
    while (8 < (stCurBuffer.iLength - iCurPos))
    {
        unsigned char *pucParseBegin = stCurBuffer.pucData + iCurPos;
        /*获取后续消息内容的长度*/
        int iNewMsgLength = GetLittleEndNumber(pucParseBegin);
        int iNewMsgID = GetLittleEndNumber(pucParseBegin + 4);
        /*若缓冲区内的未处理数据长度小于报文中定义的消息长度时（不完整），退出循环，等待下次数据到来后重新处理*/
        if (stCurBuffer.iLength - iCurPos < 8 + iNewMsgLength)
            break;
        /*按照报文中获取的长度构造消息对象*/
        GameMessage *pstNewMsg = GetMessageFromRaw(iNewMsgID, pucParseBegin + 8, iNewMsgLength);
        /*构造Request对象并添加到输出参数的list中*/
        Request *pstNewReq = new Request();
        pstNewReq->pxMsg = pstNewMsg;
        pstNewReq->pxProcessor = pxBindRole;//指定处理该消息的角色是绑定角色
        _ReqList.push_back(pstNewReq);
        /*向后移动游标，循环处理后续数据*/
        iCurPos += iNewMsgLength + 8;
        bRet = true;
    }
    /*循环退出意味着数据正好处理完或数据不完整，用缓冲区中剩余未处理的数据替换掉原缓冲区数据*/
    stCurBuffer.SetData(stCurBuffer.pucData + iCurPos, stCurBuffer.iLength - iCurPos);

    return bRet;
}

/*和接收数据的逻辑正好相反，用来执行消息到原始数据的转换*/
bool GameProtocol::response2raw(const Response * pstResp, RawData * pstData)
{
    bool bRet = false;
    GameMessage *pxMsg = dynamic_cast<GameMessage *>(pstResp->pxMsg);

    if (NULL != pxMsg)
    {
        /*创建临时缓冲区存放消息内容序列化后的数据*/
        unsigned char aucBuffer[2048] = {0};
        /*从第8个字节开始存放消息内容序列化的数据*/
        int iMsgContenLen = pxMsg->SerialMsg2Buff(aucBuffer + 8, sizeof(aucBuffer) - 8);
        /*将长度设置到最开始的字段*/
        SetLittleEndNumber(iMsgContenLen, aucBuffer);
        /*将类型ID设置到第4个位置*/
        SetLittleEndNumber(pxMsg->Id, aucBuffer + 4);
        /*将临时空间内的数据设置到输出参数中*/
        bRet = pstData->SetData(aucBuffer, iMsgContenLen + 8);
    }

    return bRet;
}
```

## 1.3.5 GameChannel类设计

GameChannel继承TcpListenChannel，重写TcpAfterConnection方法，实现通道，协议，角色这三者的绑定和添加到server实例。
GameChannel.h文件

```cpp
#ifndef _GAME_CHANNEL_H_
#define _GAME_CHANNEL_H_

#include <zinx/zinx.h>

class GameChannel:public TcpListenChannel{
public:
    GameChannel();
    virtual bool TcpAfterConnection(int _iDataFd, struct sockaddr_in * pstClientAddr);
};

#endif
```

GameChannel.cpp文件
```cpp
#include "GameChannel.h"
#include "GameRole.h"
#include "GameProtocol.h"

//调用父类构造函数指定监听端口为8899
GameChannel::GameChannel():TcpListenChannel(8899)
{
}

//重写TcpAfterConnection，创建TcpDataChannel对象，并将GameRole对象，GameProtocol对象和TcpDataChannel这三者进行绑定
bool GameChannel::TcpAfterConnection(int _iDataFd, struct sockaddr_in * pstClientAddr)
{
    GameRole *role = new GameRole();
    /*将新创建的GameRole对象添加到server方便管理*/
    Server::GetServer()->add_role("GameRole", role);
    /*创建框架自带的tcp数据处理对象*/
    TcpDataChannel *pxTcpData = new TcpDataChannel(_iDataFd, new GameProtocol(role));
    /*将tcp数据处理对象添加为myRole对象的出口通道*/
    role->SetChannel(pxTcpData);
    //将TcpDataChannel对象添加到server实例中
    return Server::GetServer()->install_channel(pxTcpData);
}
```

## 1.3.6 架构测试

现在已经完成了数据收发和成帧的功能，未完成的功能有：消息内容解析和序列化，消息处理。合理的开发模式提倡测试驱动开发，所以，即便还有很多功能没有完成，但仍然可以测试已有架构是否可以跑通，已实现功能是否正常。

**测试方法：**

+ 为待实现函数打桩

GameMessage.cpp文件

```cpp
#include "GameMessage.h"
#include <iostream>

using namespace std;

/*调用父类构造函数*/
GameMessage::GameMessage(int _id):IdMessage(_id)
{
}

/*析构时，要释放pxProtoBufMsg*/
GameMessage::~GameMessage()
{
    if (NULL != pxProtoBufMsg)
    {
        delete pxProtoBufMsg;
        pxProtoBufMsg = NULL;
    }
}

bool GameMessage::ParseBuff2Msg(const unsigned char * pucDataBuff, int iLength)
{
    bool bRet = false;

    /*仅仅打印一条提示信息，不做真正的报文解析*/
    cout<<"ParseBuff2Msg is called"<<endl;
    bRet = true;

    return bRet;
}

int GameMessage::SerialMsg2Buff(unsigned char * pucDataBuff, int iBufLength)
{
    int iRet = 0;

    /*写死，产生的消息内容为0x11 0x22*/
    pucDataBuff[0] = 0x11;
    pucDataBuff[1] = 0x22;
    iRet = 2;

    cout<<"SerialMsg2Buff is called"<<endl;

    return iRet;
}
```

GameRole.cpp文件

```cpp
#include "GameRole.h"
#include <iostream>
#include "GameMessage.h"

using namespace std;

class IdProc0Msg:public IIdMsgProc{
    virtual bool ProcMsg(IdMsgRole * _pxRole, IdMessage * _pxMsg)
    {
        cout<<"IdProc0Msg.ProcMsg is called"<<endl;
        Response stResp;

        /*构造一条ID为1的报文，回复给对应客户端*/
        stResp.pxMsg = new GameMessage(1);
        stResp.pxSender = _pxRole;
        return Server::GetServer()->send_resp(&stResp);
    }
};

bool GameRole::init()
{
    bool bRet = false;

    cout<<"GameRole object is added to server"<<endl;
    bRet = register_id_func(0, new IdProc0Msg());    

    return bRet;
}

void GameRole::fini()
{
    cout<<"GameRole object is deled from server"<<endl;
}
```

+ 模拟真实场景发送报文


