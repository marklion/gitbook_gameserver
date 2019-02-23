# 1.2 zinx框架

zinx是一个通用的并发IO处理框架

github路径：https://github.com/marklion/Game_zinx
## 1.2.1 学习使用zinx框架

1. 阅读框架文档
> 一般地，项目的github主页上会列出README文档。

2. 运行文档中的案例并做一定扩展

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

> |消息内容的长度（4个字节，低字节在前）|消息ID（4个字节，低字节在前）|消息内容|

**思路（四个类）**：

+ 定义myMessage类继承Amessage类，类内定义成员变量msg_content用于存储消息内容
+ 定义myRole继承Arole类，重写proc_msg方法，实现反转消息内容并回传tcp消息的功能
+ 实现myProtolcol类继承Aprotocol类并重写其中的两个方法
> bool Aprotocol::raw2request(const RawData *pstData, std::list<Request *> &_ReqList);
bool Aprotocol::response2raw(const Response * pstResp, RawData * pstData);

+ 绑定myProtolcol对象到tcp的channel对象（tcp数据将会自动被myprotolcol处理）

```c++
#include "zinx/zinx.h"
#include <string>
#include <algorithm>

using namespace std;

class myMessage:public Amessage{
public:
    string msg_content;
};

class myRole:public Arole{
public:
    virtual bool init()
    {
        return true;
    }
    virtual void fini()
    {
    }
    virtual bool proc_msg(Amessage * pxMsg)
    {
        bool bRet = false;
        myMessage *pxMyMsg = dynamic_cast<myMessage *>(pxMsg);
        Response stResp;
        if (NULL != pxMyMsg)
        {
            myMessage *pxEchoMsg = new myMessage(*pxMyMsg);
            reverse(pxEchoMsg->msg_content.begin(), pxEchoMsg->msg_content.end());
            stResp.pxMsg = pxEchoMsg;
            stResp.pxSender = this;
            bRet = Server::GetServer()->send_resp(&stResp);
        }
    }
};

class myProtocol:public Aprotocol{
private:
    RawData stCurBuffer;
    myRole *pxBindRole = NULL;
    int GetLittleEndNumber(const unsigned char *pucData)
    {
        int iRet = 0;
        iRet = pucData[0];
        iRet += pucData[1];
        iRet += pucData[2];
        iRet += pucData[3];

        return iRet;
    }
    void SetLittleEndNumber(int _Num, unsigned char *pucData)
    {
        pucData[0] = _Num & 0xff;
        pucData[1] = (_Num >> 8) & 0xff;
        pucData[2] = (_Num >> 16) & 0xff;
        pucData[3] = (_Num >> 24) & 0xff;
    }
    myMessage *GetMessageFromRaw(const unsigned char *pucParseBegin, int iLengthLast)
    {
        myMessage *pstRet = NULL;

        pstRet = new myMessage();
        pstRet->msg_content.assign((char *)pucParseBegin, iLengthLast);

        return pstRet;
    }
public:
    myProtocol(myRole *_bindRole):pxBindRole(_bindRole)
    {
    }
    virtual ~myProtocol()
    {
        if (NULL != pxBindRole)
        {
            delete pxBindRole;
        }
    }
    virtual bool raw2request(const RawData * pstData, std :: list < Request * > & _ReqList)
    {
        bool bRet = false;
        int iCurPos = 0;

        stCurBuffer.AppendData(pstData);
        iCurPos = 0;
        while (8 < (stCurBuffer.iLength - iCurPos))
        {
            unsigned char *pucParseBegin = stCurBuffer.pucData + iCurPos;
            int iNewMsgLength = GetLittleEndNumber(pucParseBegin);
            if (stCurBuffer.iLength - iCurPos < 8 + iNewMsgLength)
                break;
            
            myMessage *pstNewMsg = GetMessageFromRaw(pucParseBegin + 8, iNewMsgLength);
            Request *pstNewReq = new Request();
            pstNewReq->pxMsg = pstNewMsg;
            pstNewReq->pxProcessor = pxBindRole;
            _ReqList.push_back(pstNewReq);
            iCurPos += iNewMsgLength + 8;
            bRet = true;
        }
        stCurBuffer.SetData(stCurBuffer.pucData + iCurPos, stCurBuffer.iLength - iCurPos);
        
        return bRet;
    }
    virtual bool response2raw(const Response * pstResp, RawData * pstData)
    {
        bool bRet = false;
        myMessage *pxMsg = dynamic_cast<myMessage *>(pstResp->pxMsg);

        if (NULL != pxMsg)
        {
            int iMsgContenLen = pxMsg->msg_content.size();
            unsigned char *pucTemp = (unsigned char *)calloc(1UL, iMsgContenLen + 8);
            if (NULL != pucTemp)
            {
                SetLittleEndNumber(iMsgContenLen, pucTemp);
                pxMsg->msg_content.copy((char *)(pucTemp + 8), iMsgContenLen, 0);
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
    
    //重写TcpAfterConnection
    virtual bool TcpAfterConnection(int _iDataFd, struct sockaddr_in * pstClientAddr)
    {
        myRole *role = new myRole();
        Server::GetServer()->add_role("myRole", role);
        TcpDataChannel *pxTcpData = new TcpDataChannel(_iDataFd, new myProtocol(role));
        role->SetChannel(pxTcpData);
        //创建TcpDataChannel对象并添加到server实例中
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





