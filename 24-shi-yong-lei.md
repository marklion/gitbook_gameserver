# 2.4 实用类

之前的功能都实现完之后，zinx此时是一个高度抽象的框架（仅仅提供了并发IO和资源管理的功能）。开发者要想使用框架，必须要根据实际业务，继承相应的抽象类，重写函数实现具体业务功能。

成熟的框架一般会提供一些比较通用的实用类帮助开发者快速构建应用程序。比如处理TCP收发的Achannel子类，封装拥有ID的消息的Amessage子类，处理ID类消息的Arole子类等。

## 2.4.1 TCP数据处理

创建TCP服务器的模式一般是：创建监听XXX端口的TCP监听套接字，有客户端连接后则创建数据套接字进行数据通信。若客户端关闭连接，则读取数据套接字会返回0。

关键点：

+ 需要两个类用来处理监听和数据套接字
+ 创建监听channel对象只需要指定端口号
+ 创建数据channel对象只需要指定数据套接字
+ 监听channel类可以提供虚函数描述accept之后进行的操作，子类重写该函数用于创建自己需要的数据channel对象
+ 数据channel的fd需要分别相应有数据的事件和对端关闭的事件

```cpp
bool TcpDataChannel::readFd(uint32_t _event, RawData * pstData)
{
    bool bRet = false;

    /*先判断是否有数据*/
    if (0 != (_event & EPOLLIN))
    {
        /*有数据来则调用读取数据函数*/
        bRet = TcpProcDataIn(pstData);
        /*如果读取数据失败，也当做对端关闭处理*/
        if (false == bRet)
        {
            TcpProcHup();
        }
    }
    
    /*再判断是否对端关闭*/
    if (0 != (_event & (EPOLLHUP|EPOLLERR)))
    {
        bRet = false;
        TcpProcHup();
    }

    return bRet;
}

bool TcpDataChannel::TcpProcDataIn(RawData * pstData)
{
    bool bRet = false;
    ssize_t iReadLen = -1;
    unsigned char aucBuff[1024] = {0};

    /*循环读取数据直到读不到数据为止*/
    while (0 < (iReadLen = recv(m_fd, aucBuff, sizeof(aucBuff), MSG_DONTWAIT)))
    {
        unsigned char *pucTempData = (unsigned char *)calloc(1UL, iReadLen + pstData->iLength);
        if (NULL != pucTempData)
        {
            bRet = true;
            /*追加循环读取到的数据*/
            memcpy(pucTempData, pstData->pucData, pstData->iLength);
            memcpy(pucTempData + pstData->iLength, aucBuff, iReadLen);
            pstData->iLength += iReadLen;
            if (NULL != pstData->pucData)
            {
                free(pstData->pucData);
            }
            pstData->pucData = pucTempData;
        }
    }

    std::cout<<"<----------------------------------------->"<<std::endl;
    std::cout<<"recv from "<<m_fd<<":"<<Achannel::Convert2Printable(pstData)<<std::endl;
    std::cout<<"<----------------------------------------->"<<std::endl;

    return bRet;
}

void TcpDataChannel::TcpProcHup()
{
    /*对端关闭后要及时关闭fd并置为-1*/
    std::cout<<"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"<<std::endl;
    std::cout<<m_fd<<" is hangup"<<std::endl;
    std::cout<<"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"<<std::endl;
    close(m_fd);
    m_fd = -1;
}
```
 ## 2.4.2 ID类消息处理类
 
 一般地，待处理的消息都会按数字或标志位进行分类，为方便开发者对消息类型区分处理，设计IdMsgRole类用于处理用数字作为ID的消息。
 
+ 提供注册和去注册函数用于指定消息类型对应的处理函数
+ 使用map存储注册的处理函数对象
+ 使用该role进行ID类消息处理时，本质上就是从map中取出对应的处理函数再进行处理 

```cpp
bool IdMsgRole::register_id_func(int _id, IIdMsgProc *Iproc)
{
    m_IdMsgMap[_id] = Iproc;

    return true;
}

void IdMsgRole::unregister_id_func(int _id)
{
    m_IdMsgMap.erase(_id);
}

bool IdMsgRole::proc_msg(Amessage * pxMsg)
{
    bool bRet = false;
    IdMessage *pxIdMsg = dynamic_cast<IdMessage *>(pxMsg);

    if (NULL != pxIdMsg)
    {
        IIdMsgProc *pxIproc = m_IdMsgMap[pxIdMsg->Id];
        if (NULL != pxIproc)
        {
            bRet = pxIproc->ProcMsg(this, pxIdMsg);
        }
    }

    return bRet;
}

IdMsgRole::~IdMsgRole()
{
    auto itr = m_IdMsgMap.begin();
    while (itr != m_IdMsgMap.end())
    {
        int Id = (*itr).first;
        auto pxIdproc = (*itr++).second;
        unregister_id_func(Id);
        delete pxIdproc;
    }
}
```

