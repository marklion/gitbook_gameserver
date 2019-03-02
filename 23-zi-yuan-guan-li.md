# 2.3 资源管理

## 2.3.1 role对象的管理

protocol对象有两个主要职责：

1. 解析数据形成请求消息（message对象）
2. 分发消息给合适的处理对象（找到合适的role对象）

> 因为游戏项目中的数据通道是TCP，所以处理请求的role对象可以和protocol对象channel对象可以形成一对一对一的绑定关系，无需再protocol对象中进行消息分发

**问题：**protoco对象需要找到合适的role对象进行消息分发

**举例：** 棋牌游戏服务器监听UDP数据，客户端发送的数据报文中包含两部分：玩家ID和游戏动作（出牌，下注等等）。

**分析：** 此时我们需要通过protocol对象：

1. 将UDP数据解析出发送者（玩家ID）和业务数据（游戏动作）两部分，
2. 根据报文中的发送者信息选择合适的role对象进行后续处理

**解决方式：** 用server对象管理所有的role对象。

+ 创建的role对象要添加到server对象中，失效的role对象要从server对象中摘除
+ server对象对外提供查询函数

#### role对象存储的数据结构

同类型role组成list，按照类型将list存成map

![](/assets/role对象数据结构.png)

> 为什么使用这样的数据结构？

#### 代码实现

```cpp
/*添加role对象，要通过参数指定特征字符串，一般使用类名*/
bool Server::add_role(string szCharacter, Arole * pxRole)
{
    bool bRet = false;

    /*先调用init*/
    if (true == pxRole->init())
    {
        /*map节点如果之前不存在，则新创建*/
        auto itr = m_role_list_map.find(szCharacter);
        if (itr == m_role_list_map.end())
        {
            list<Arole *> RoleList;
            RoleList.push_back(pxRole);
            m_role_list_map[szCharacter] = RoleList;
        }
        else
        {
            (*itr).second.push_back(pxRole);
        }
        bRet = true;
    }

    return bRet;
}

/*摘除role对象*/
void Server::del_role(string szCharacter, Arole * pxRole)
{
    auto itr = m_role_list_map.find(szCharacter);

    if (itr != m_role_list_map.end())
    {
        (*itr).second.remove(pxRole);
        if ((*itr).second.empty())
        {
            m_role_list_map.erase(itr);
        }
        pxRole->fini();
    }
}

/*server的fini函数中，要添加遍历摘除释放role对象的流程*/
void Server::fini()
{
    auto list_itr = m_role_list_map.begin();

    while (list_itr != m_role_list_map.end())
    {
        auto my_pair = *list_itr++;
        auto RoleList = my_pair.second;
        auto key = my_pair.first;
        auto itr = RoleList.begin();

        while (itr != RoleList.end())
        {
            auto pxRole = (*itr++);
            del_role(key, pxRole);
            delete pxRole;
        }
    }

    auto channel_map_itr = m_channel_map.begin();

    while (channel_map_itr != m_channel_map.end())
    {
        auto pxChannel = (*channel_map_itr++).second;
        uninstall_channel(pxChannel);
        delete pxChannel;
    }

    close(m_epollfd);
}
```

## 2.3.2 server类实例化

server类是zinx框架所有功能对外使用的窗口，整个应用程序应该有且只有一个server对象（单例），并在应用程序启动是创建对象。

```cpp
class Server{
public:
    static Server *GetServer()
    {
        if (NULL == Server::pxServer)
        {
            Server::pxServer = new Server();
        }

        return Server::pxServer;
    }
private:
    static Server *pxServer;
    Server();
};
```
