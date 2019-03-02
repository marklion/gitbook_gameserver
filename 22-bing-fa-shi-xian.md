# 2.2 并发实现

选择IO多路复用（epoll）的机制作为并发模型。

> 为什么是epoll, 而不是select或poll？

## 2.2.1 server成员变量及其数据结构

server类包含成员变量`int m_epollfd`表示epoll实例，包含成员变量`std::map<int, Achannel*>m_channel_map`用于存储。

![](/assets/epoll数据结构.png)

+ 内核态的epoll树已经已经可用记录Channel对象指针，为什么还需要m_channel_map来存储Channel对象？
+ m_channel_map为什么设计成map结构？

```cpp
/*init函数内对epollfd进行初始化*/
bool Server::init()
{
    bool bRet = false;

    int iFd = epoll_create(1);
    if (0 <= iFd)
    {
        m_epollfd = iFd;
        bRet = true;
    }

    return bRet;
}

/*fini函数要摘除并释放所有channel对象*/
void Server::fini()
{
    auto channel_map_itr = m_channel_map.begin();

    /*map容器遍历删除，注意迭代器的操作*/
    while (channel_map_itr != m_channel_map.end())
    {
        auto pxChannel = (*channel_map_itr++).second;
        uninstall_channel(pxChannel);
        delete pxChannel;
    }
    /*及时关掉epollfd*/
    close(m_epollfd);
}
```

## 2.2.2 添加和摘除fd

设计成员函数install和uninstall用于实现从epoll实例中添加和摘除fd。

```cpp
/*添加channel对象*/
bool Server::install_channel(Achannel * pxChannel)
{
    struct epoll_event stEvent;
    bool bRet = false;

    /*先调用channel对象的init函数*/
    if (true == pxChannel->init())
    {
        /*构造epoll_event结构体，使用ptr存储channel对象指针*/
        stEvent.events = pxChannel->GetEvent();
        stEvent.data.ptr = (void *)pxChannel;
        
        /*调用epoll_ctl将fd添加到epoll中*/
        if (0 == epoll_ctl(m_epollfd, EPOLL_CTL_ADD, pxChannel->GetFd(), &stEvent))
        {
            /*epoll添加成功后将channel对象添加到map中*/
            m_channel_map[pxChannel->GetFd()] = pxChannel;
            bRet = true;
        }
    }

    return bRet;  
}

/*摘除channel对象*/
void Server::uninstall_channel(Achannel * pxChannel)
{
    epoll_ctl(m_epollfd, EPOLL_CTL_DEL, pxChannel->GetFd(), NULL);
    m_channel_map.erase(pxChannel->GetFd());
    /*摘掉channel对象后要调用fini函数*/
    pxChannel->fini();
}
```

## 2.2.3 并发处理流程

确定并发模型为epoll和其对应的数据结构后，其处理流程是固定的：

#### 处理并行输入

1. epoll_wait
2. 遍历ready的fd
3. 取出channel对象
4. 调用channel对象的readFd函数得到数据
5. 调用channel对象绑定的protocol对象进行数据处理得到request对象
6. 调用request对象的成员对象role的proc_msg函数对request的msg对象进行处理。

![](/assets/主循环时序.png)

```cpp
bool Server::run()
{
    int iEpollRet = -1;

    if (0 > m_epollfd)
    {
        return false;
    }

    /*主循环*/
    while (true != m_need_exit)
    {
        /*设定一次epoll_wait最多处理100个并发数据*/
        struct epoll_event atmpEvent[100];
        iEpollRet = epoll_wait(m_epollfd, atmpEvent, 100, -1);
        if (-1 == iEpollRet)
        {
            /*若epoll_wait被信号打断，则忽略失败，继续epoll_wait*/
            if (EINTR == errno)
            {
                continue;
            }
            else
            {
                break;
            }
        }
        /*遍历本次的并发数据*/
        for (int i = 0; i < iEpollRet; i++)
        {
            Request *pstReq;
            RawData stRD;
            /*取出channel对象*/
            Achannel *pxchannel = static_cast<Achannel *>(atmpEvent[i].data.ptr);
            
            /*调用channel对象的readFd函数，本次epoll事件类型作为输入参数，读取到的数据stRD作为输出参数*/
            if (true == pxchannel->readFd(atmpEvent[i].events, &stRD))
            {
                Aprotocol *pxProtocol = pxchannel->GetProtocol();
                list<Request *> ReqList;
                /*调用protocol对象的数据处理函数，得到若干request对象*/
                if ((NULL != pxProtocol) && (true == pxProtocol->raw2request(&stRD, ReqList)))
                {
                    auto itr = ReqList.begin();
                    /*遍历处理所有request对象*/
                    while (itr != ReqList.end())
                    {
                        pstReq = (*itr);
                        (void)handle(pstReq);
                        ReqList.erase(itr++);
                        delete pstReq;
                    }
                }
            }
            /*如果读取数据不成功，则认为fd异常，要摘除并释放channel对象*/
            else
            {
                uninstall_channel(pxchannel);
                delete pxchannel;
                /*这里要及时break，后续的并发数据由下次epoll_wait重新处理*/
                break;
            }
        }
    }

    return true;
}

/*处理request对象*/
bool Server::handle(Request * pstReq)
{
    /*调用role对象的proc_msg函数处理message对象*/
    return pstReq->pxProcessor->proc_msg(pstReq->pxMsg);
}
```

#### 输出数据

![](/assets/发送数据时序图.png)

```cpp
bool Server::send_resp(Response * pstResp)
{
    bool bRet = false;

    if ((NULL != pstResp) && (NULL != pstResp->pxMsg) && (NULL != pstResp->pxSender))
    {
        Achannel *pxDestChannel = pstResp->pxSender->GetChannel();
        if (NULL != pxDestChannel)
        {
            RawData stRD;
            if (true == pxDestChannel->GetProtocol()->response2raw(pstResp, &stRD))
            {
                bRet = pxDestChannel->writeFd(&stRD);
            }
        }
    }

    return bRet;
}
```