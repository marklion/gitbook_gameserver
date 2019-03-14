# 1.2 zinx框架

zinx是一个通用的并发IO处理框架

github路径：[https://github.com/marklion/Game\_zinx](https://github.com/marklion/Game_zinx)

## 1.2.1 学习使用zinx框架

1. 阅读框架文档

   > 一般地，项目的github主页上会列出README文档。
   
   ![](/assets/zinx框架图.png)
   
2. 安装框架，搭建开发环境
3. 运行文档中的案例并做一定扩展

**小结：** zinx框架用于实现通用的IO事物处理。在zinx框架下实现功能时，应该将功能分解成四个部分（IO，协议，消息，角色）后分别由四个类（AChannel, Aprotocol, Amessage, Arole）的派生类实现。

## 1.2.2 tcp封包处理实现

**问题**：tcp或类似的流式文件无法保证收到的数据按照期望的格式分割。

**举例**：服务器期望接收2个字节的数据作为一个合理请求。客户端发送了两个请求（四个字节）后，由于网络拥塞，服务器收到了1个字节后，recv返回，1秒钟后，数据到来，再次调用recv会收到3个字节。

**常规套路**：  
1. 设定报文边界，一般使用Tag Length Value的格式  
2. recv数据后，若接收缓冲区当前数据长度小于报文内规定长度，则保留当前缓冲区，下次recv数据后重新处理  
3. 若接收缓冲区数据长度大于等于报文内规定长度，则生成请求并保留后续多余的数据等待下次recv数据后重新处理

> zinx框架提供了Aprotocol类专门用于报文/流向请求的转化

### **案例**：

参照1.1.4中定义的消息结构，实现服务器收到tcp消息后将**消息内容反转**并按照原消息结构回传。

> \|消息内容的长度（4个字节，低字节在前）\|消息ID（4个字节，低字节在前）\|消息内容\|

**思路（四个类）**：

* 定义myMessage类继承Amessage类，类内定义成员变量msg\_content用于存储消息内容
* 定义myRole继承Arole类，重写proc\_msg方法，实现反转消息内容并回传tcp消息的功能
* 实现myProtolcol类继承Aprotocol类并重写其中的两个方法

  > bool Aprotocol::raw2request\(const RawData _pstData, std::list&lt;Request _&gt; &\_ReqList\);  
  > bool Aprotocol::response2raw\(const Response _ pstResp, RawData _ pstData\);

* 绑定myProtolcol对象到tcp的channel对象（tcp数据将会自动被myprotolcol处理）

**结构图：**  
![](/assets/设计结构.png)

```cpp
#include "zinx/zinx.h"
#include <string>
#include <algorithm>

using namespace std;

/*定义myMessage类继承Amessage并添加成员变量msg_content存储消息内容*/
class myMessage:public Amessage{
public:
    string msg_content;
};

/*定义myRole类继承Arole，重写proc_msg函数*/
class myRole:public Arole{
public:
    virtual bool init()
    {
        return true;
    }
    virtual void fini()
    {
    }

    /*该函数实现了反转源消息并回传的功能*/
    virtual bool proc_msg(Amessage * pxMsg)
    {
        bool bRet = false;

        /*动态类型转换，记得判断返回值*/
        myMessage *pxMyMsg = dynamic_cast<myMessage *>(pxMsg);
        Response stResp;
        if (NULL != pxMyMsg)
        {
            /*拷贝构造源消息对象，用于回传给发送者*/
            myMessage *pxEchoMsg = new myMessage(*pxMyMsg);
            /*调用通用算法库反转用于回传的消息*/
            reverse(pxEchoMsg->msg_content.begin(), pxEchoMsg->msg_content.end());
            /*拼装发送对象，包含待发送的消息和发送者*/
            stResp.pxMsg = pxEchoMsg;
            stResp.pxSender = this;
            /*调用server的发送函数将请求发出去*/
            bRet = Server::GetServer()->send_resp(&stResp);
        }
    }
};

/*定义myProtocol类继承Aprotocol实现数据到和消息的互相转化*/
class myProtocol:public Aprotocol{
private:
    /*用于暂存当前收到的报文*/
    RawData stCurBuffer;
    /*用于绑定处理消息的角色对象*/
    myRole *pxBindRole = NULL;

    /**********************
    描述：获取小字节序的整数
    参数：pucData是输入数据起始地址
    返回值：转换后的整数
    **********************/
    int GetLittleEndNumber(const unsigned char *pucData)
    {
        int iRet = 0;
        iRet = pucData[0];
        iRet += pucData[1] * 0xff;
        iRet += pucData[2] * 0xffff;
        iRet += pucData[3] * 0xffffff;

        return iRet;
    }

    /**********************
    描述：设置小字节序的整数
    参数：
        _Num是待转换的整数
        pucData是输出数据起始地址
    **********************/
    void SetLittleEndNumber(int _Num, unsigned char *pucData)
    {
        pucData[0] = _Num & 0xff;
        pucData[1] = (_Num >> 8) & 0xff;
        pucData[2] = (_Num >> 16) & 0xff;
        pucData[3] = (_Num >> 24) & 0xff;
    }

    /**********************
    描述：将原始数据转化成消息
    参数：
        pucParseBegin是消息内容字段的起始地址
        iLengthLast是消息内容的长度
    返回值：基于报文创建的消息对象（堆对象）
    **********************/
    myMessage *GetMessageFromRaw(const unsigned char *pucParseBegin, int iLengthLast)
    {
        myMessage *pstRet = NULL;

        pstRet = new myMessage();
        pstRet->msg_content.assign((char *)pucParseBegin, iLengthLast);

        return pstRet;
    }
public:

    /*通过构造函数指定当前协议绑定的处理角色*/
    myProtocol(myRole *_bindRole):pxBindRole(_bindRole)
    {
    }

    /*析构时，还需要从server中摘除并销毁绑定的角色对象*/
    virtual ~myProtocol()
    {
        if (NULL != pxBindRole)
        {
            Server::GetServer()->del_role("myRole", pxBindRole);
            delete pxBindRole;
        }
    }

    /*封包的核心逻辑*/
    virtual bool raw2request(const RawData * pstData, std :: list < Request * > & _ReqList)
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
            /*若缓冲区内的未处理数据长度小于报文中定义的消息长度时（不完整），退出循环，等待下次数据到来后重新处理*/
            if (stCurBuffer.iLength - iCurPos < 8 + iNewMsgLength)
                break;
            /*按照报文中获取的长度构造消息对象*/
            myMessage *pstNewMsg = GetMessageFromRaw(pucParseBegin + 8, iNewMsgLength);
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
    virtual bool response2raw(const Response * pstResp, RawData * pstData)
    {
        bool bRet = false;
        myMessage *pxMsg = dynamic_cast<myMessage *>(pstResp->pxMsg);

        if (NULL != pxMsg)
        {
            int iMsgContenLen = pxMsg->msg_content.size();
            /*申请一条报文长度的临时空间*/
            unsigned char *pucTemp = (unsigned char *)calloc(1UL, iMsgContenLen + 8);
            if (NULL != pucTemp)
            {
                /*将长度设置到最开始的字段*/
                SetLittleEndNumber(iMsgContenLen, pucTemp);
                /*将待发送的消息放到第8个位置*/
                pxMsg->msg_content.copy((char *)(pucTemp + 8), iMsgContenLen, 0);
                /*将临时空间内的数据设置到输出参数中后释放临时空间*/
                pstData->SetData(pucTemp, iMsgContenLen + 8);
                free(pucTemp);
                bRet = true;
            }
        }


        return bRet;
    }
};

//定义myTcpLst处理tcp连接
class myTcpLst:public TcpListenChannel{
public:
    //调用父类构造函数指定监听端口为6789
    myTcpLst():TcpListenChannel(6789)
    {
    }

    //重写TcpAfterConnection，创建TcpDataChannel对象，并将myRole对象，myProtocol对象和TcpDataChannel这三者进行绑定
    virtual bool TcpAfterConnection(int _iDataFd, struct sockaddr_in * pstClientAddr)
    {
        myRole *role = new myRole();
        /*将新创建的myRole对象添加到server方便管理*/
        Server::GetServer()->add_role("myRole", role);
        /*创建框架自带的tcp数据处理对象*/
        TcpDataChannel *pxTcpData = new TcpDataChannel(_iDataFd, new myProtocol(role));
        /*将tcp数据处理对象添加为myRole对象的出口通道*/
        role->SetChannel(pxTcpData);
        //将TcpDataChannel对象添加到server实例中
        return Server::GetServer()->install_channel(pxTcpData);
    }
};

int main()
{
    Server *pxServer = Server::GetServer();
    pxServer->init();

    pxServer->install_channel(new myTcpLst());
    pxServer->run();

    return 0;
}
```

**测试：**

* 发送一帧完整的请求报文

```bash
<----------------------------------------->
recv from 5:03 00 00 00 00 00 00 00 12 34 56
<----------------------------------------->
<----------------------------------------->
send to 5:03 00 00 00 00 00 00 00 56 34 12
<----------------------------------------->
#测试OK
```

* 发送两帧完整的请求报文

```bash
<----------------------------------------->
recv from 5:03 00 00 00 00 00 00 00 12 34 56 03 00 00 00 00 00 00 00 12 34 56
<----------------------------------------->
<----------------------------------------->
send to 5:03 00 00 00 00 00 00 00 56 34 12
<----------------------------------------->
<----------------------------------------->
send to 5:03 00 00 00 00 00 00 00 56 34 12
<----------------------------------------->
#测试OK
```

* 发送两个半帧请求报文

```bash
<----------------------------------------->
recv from 5:03 00 00 00 00 00 00 00 12 34   #这条报文少一个字节，下一条报文补上
<----------------------------------------->
<----------------------------------------->
recv from 5:56
<----------------------------------------->
<----------------------------------------->
send to 5:03 00 00 00 00 00 00 00 56 34 12
<----------------------------------------->
#测试OK
```

* 发送一帧半和半帧请求报文

```bash
<----------------------------------------->
recv from 5:03 00 00 00 00 00 00 00 12 34 56 03  #这条报文多一个字节，是下一条报文的首个字节
<----------------------------------------->
<----------------------------------------->
send to 5:03 00 00 00 00 00 00 00 56 34 12
<----------------------------------------->
<----------------------------------------->
recv from 5:00 00 00 00 00 00 00 12 34 56
<----------------------------------------->
<----------------------------------------->
send to 5:03 00 00 00 00 00 00 00 56 34 12
<----------------------------------------->
#测试OK
```

**小结：**
1. zinx框架的用法浓缩为：继承四个类。
2. TCP等流式数据的封包思路：缓存和滑窗
3. 面向对象思想的设计理念：
  + 封装：将过程拆解成小段，由多种角色承担。
  + 继承：派生类无需实现全部功能，细节已经由基类实现。
  + 多态：通过派生类实现业务，派生类对于框架来说等同于基类。

**作业：**基于本章的例子，添加功能：若消息ID是0则回传原消息，若消息ID是1则回传反转后消息。

