# 1.7 功能扩展

当前游戏已经可用，我们接下来尝试扩展几个功能：

* 出生点随机
* 玩家昵称随机
* 守护进程和进程监控
* 日志管理

## 1.7.1 出生点随机

**设计**： GameRole对象创建时，随机生成合理范围内的坐标。生成随机数的方法: std::default\_random\_engine\(详细资料：[http://www.cplusplus.com/reference/random/default\_random\_engine/](http://www.cplusplus.com/reference/random/default_random_engine/)\)

* 构造函数参数用于指定种子（一般使用当前时间）
* \(\)操作符返回随机数（无符号整形）

**编码**：修改GameRole.cpp文件

```cpp
/*创建随机数引擎作为全局变量，指定当前时间作为种子*/
static default_random_engine g_Random_engine(time(NULL));

GameRole::GameRole()
{
    iPid = g_GamePlayerId++;
    szName = "abc";
    /*()操作符获取随机数据并设定0~19的范围*/
    x = 160 + g_Random_engine() % 20;
    z = 134 + g_Random_engine() % 20;
}
```

**测试**：多个玩家登陆后位置随机

## 1.7.2 随机昵称

**设计：** 在文件中存储一定量的常用姓和名，GameRole创建时随机组合姓名

* 设计数据结构存储随机姓名池，进程启动时构造

![](/assets/名字结构.png)

* 生成随机名称：取第随机个姓，取第随机个名

![](/assets/生成名字.png)

* GameRole释放前，要把姓名释放回姓名池

**编码：**

实现随机数生成池

```cpp
#define RANDOM_FIRST_NAME "random_first.txt"
#define RANDOM_SECOND_NAME "random_last.txt"
/*定义数据结构*/
struct FirstName {
    std::string szFirstName;
    std::vector<std::string> vecLastName;
};

class RandomName{
public:
    RandomName();
    ~RandomName();
    void LoadFile();
    std::string GetName();
    void ReleaseName(std::string szName);
private:
    std::vector<FirstName *> m_names;
};

void RandomName::LoadFile()
{
    ifstream fFirstName;
    ifstream fSecondName;
    string tmpFirst;
    string tmpSecond;

    fFirstName.open(RANDOM_FIRST_NAME);
    fSecondName.open(RANDOM_SECOND_NAME);

    if (fFirstName.is_open() && fSecondName.is_open())
    {
        while (getline(fFirstName, tmpFirst))
        {
            FirstName *pstFirst = new FirstName();
            pstFirst->szFirstName = tmpFirst;
            m_names.push_back(pstFirst);
            while (getline(fSecondName, tmpSecond))
            {
                pstFirst->vecLastName.push_back(tmpSecond);
            }
            fSecondName.clear(ios::goodbit);
            fSecondName.seekg(ios::beg);
        }

        fFirstName.close();
        fSecondName.close();
    }
}

string RandomName::GetName()
{
    string szRet;

    if (0 < m_names.size())
    {
        int iRandFirst = g_Random_engine() % m_names.size();
        FirstName *pstFirst = m_names[iRandFirst];
        int iRandSecond = g_Random_engine() % pstFirst->vecLastName.size();

        szRet = pstFirst->szFirstName + " " + pstFirst->vecLastName[iRandSecond];

        pstFirst->vecLastName.erase(pstFirst->vecLastName.begin() + iRandSecond);
        if (0 >= pstFirst->vecLastName.size())
        {
            m_names.erase(m_names.begin() + iRandFirst);
            delete pstFirst;
        }
    }
    else
    {
        szRet = "Not Support";
    }

    return szRet;
}

void RandomName::ReleaseName(std :: string szName)
{
    int iSpace = szName.find(" ");
    string szFirstName = szName.substr(0, iSpace);
    string szSecondName = szName.substr(iSpace + 1, szName.size());

    auto itr = m_names.begin();
    for (; itr != m_names.end(); itr++)
    {
        if ((*itr)->szFirstName == szFirstName)
        {
            break;
        }
    }

    FirstName *pstFirst = NULL;
    if (m_names.end() != itr)
    {
        pstFirst = (*itr);
    }
    else
    {
        pstFirst = new FirstName();
        pstFirst->szFirstName = szFirstName;
        m_names.push_back(pstFirst);
    }
    pstFirst->vecLastName.push_back(szSecondName);
}
RandomName::RandomName()
{
}

RandomName::~RandomName()
{
    auto itr = m_names.begin();
    while (itr != m_names.end())
    {
        auto pData = (*itr);
        delete pData;
        itr = m_names.erase(itr);
    }
}
```

调用姓名池

```cpp
/*创建全局变量存储姓名池*/
RandomName g_xRandModule;

GameRole::GameRole()
{
    iPid = g_GamePlayerId++;
    /*获取随机姓名*/
    szName = g_xRandModule.GetName();
    x = 160 + g_Random_engine() % 20;
    z = 134 + g_Random_engine() % 20;
}

GameRole::~GameRole()
{
    /*释放回姓名池*/
    g_xRandModule.ReleaseName(szName);
}

int main()
{
    Server *pxServer = Server::GetServer();
    /*载入文件内容*/
    g_xRandModule.LoadFile();
    pxServer->init();
    pxServer->install_channel(new GameChannel());
    pxServer->run();

    return 0;
}
```

## 1.7.3 守护进程和进程监控

**设计：**

* 判断启动参数，若为daemon则按照守护进程启动
* 启动守护进程时，创建子进程用于游戏业务；父进程用于接收子进程退出状态并重启子进程。



