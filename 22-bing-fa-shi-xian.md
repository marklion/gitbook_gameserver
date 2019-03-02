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

## 2.2.1 添加和摘除fd

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

