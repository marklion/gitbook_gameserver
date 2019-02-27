# 1.5 AOI设计与实现

前边，我们已经完成了四个类中的三个了，接下来就是要实现GameRole类了。于此的同时，游戏相关的核心消息处理逻辑都是要在该类中实现的。

**需求回首：** 

+ 新客户端连接后，向其发送ID和名称
+ 新客户端连接后，向其发送**周围**玩家的位置
+ 新客户端连接后，向**周围**玩家发送其位置
+ 收到客户端的移动信息后，向**周围**玩家发送其新位置
+ 收到客户端的移动信息后，向其发送**周围新**玩家位置
+ 收到客户端的聊天信息后，向**所有**玩家发送聊天内容
+ 客户端断开时，向**周围**玩家发送其断开的消息

关键字：周围。

以上所列出的需求，基本都是这样的套路：发送XXX消息给XXX。

+ 发送：`bool Server::send_resp(Response * pstResp)`
+ XXX消息：1.4节已经搞定
+ 给XXX：怎样表示周围玩家？

## 1.5.1 AOI算法简介

**定义：** 获取感兴趣的区域（Area Of Interest）的算法。

**解决的问题：** 形成周围的概念。在多人游戏中，各个游戏客户端之间需要通过服务器向彼此更新自身状态。但对于当玩家来说，我们不需要获取“太远”的玩家的信息，所以，在服务器端，我们通过AOI算法可以获取到某个客户端“周围”的玩家，进而只在该小范围内同步信息。

**网格法AOI**：

+ 参考游戏世界的坐标，创建一个边界相同的矩形。
+ 选取适当的颗粒度，将矩形分割成几×几的网格。
+ 每个客户端都要按照实际坐标添加到某个格子里。
+ 客户端所在格子的周围八个格子内的玩家就是周围玩家。

**举例：** 世界坐标是X[20,200]，Y[50,230]，划分成6×6的网格为：

![](/assets/游戏网格.png)

1. 已知玩家坐标（x，y），该玩家在几号网格？
> 网格编号=(x-x轴起始坐标)/x轴网格宽度 + (y-y轴起始坐标)/y轴宽度*x轴网格数量
> x轴网格宽度=(x轴结束坐标-x轴起始坐标)/x轴网格数量；y轴的计算方式相同
2. 已知玩家在n号网格，周围的格子(包括自己)有哪些？

![](/assets/周围网格.png)

## 1.5.2 AOI算法实现

+ 网格类用于存放网格内的玩家：封装一个list用于添加和删除玩家
+ 世界地图类用于构造和表示所有网格
  + 属性：x和y轴的起始结束坐标，x和y轴的网格数
  + 网格表示：封装一个vector存放所有的网格对象，网格序号按照vector存储序号表示
  + 主要函数：根据坐标获取网格，根据网格号获取周围网格

World类和Grid类声明

```cpp

/*网格类*/
class Grid{
public:
    /*构造时指定网格编号*/
    Grid(int _GridNo);
    int GridNo = 0;
    
    /*list类型的成员变量用于记录当前格子内的所有玩家*/
    std::list<GameRole *> players;
    /*提供添加和删除函数*/
    void add(GameRole *_pxPlayer);
    void remove(GameRole *_pxPlayer);
};

/*游戏世界类*/
class World{
private:
    /*vector类型成员变量用来存储当前所有的格子对象*/
    std::vector<Grid *> m_Grids;
    /*记录游戏世界的重要参数*/
    int MinX = 0;
    int MaxX = 0;
    int MinY = 0;
    int MaxY = 0;
    int Xcnt = 0;
    int Ycnt = 0;
    /*获取格子X和Y方向的宽度函数*/
    int Xwidth();
    int Ywidth();
public:
    /*构造世界对象时要指定关键参数*/
    World(int _minX, int _maxX, int _minY, int _maxY, int _Xcnt, int _Ycnt);
    ~World();
    /*根据玩家坐标获取所在格子号或格子对象*/
    int GetGridNo(int _x, int _y);
    Grid *GetGrid(int _x, int _y);
    /*获取所有周围格子对象（包括自己）*/
    void GetSurroundGrids(int GridNo, std::list<Grid *> &Grids);
};
```

成员函数实现：

```cpp

Grid::Grid(int _GridNo):GridNo(_GridNo)
{
}

/*直接调用list添加对象*/
void Grid::add(GameRole * _pxPlayer)
{
    players.push_back(_pxPlayer);
}

/*直接调用list删除对象*/
void Grid::remove(GameRole * _pxPlayer)
{
    players.remove(_pxPlayer);
}

/*构造游戏世界，通过参数指定边界和分隔方式*/
World::World(int _minX, int _maxX, int _minY, int _maxY, int _Xcnt, int _Ycnt):
    MinX(_minX), MaxX(_maxX), MinY(_minY), MaxY(_maxY), Xcnt(_Xcnt), Ycnt(_Ycnt)
{
    /*创建所有格子对象并按顺序添加到vector容器中*/
    for (int i = 0; i < Xcnt * Ycnt; i++)
    {
        m_Grids.push_back(new Grid(i));
    }
}

World::~World()
{
    /*析构时应该删掉所有格子对象*/
    auto itr = m_Grids.begin();
    while (itr != m_Grids.end())
    {
        delete (*itr);
        itr = m_Grids.erase(itr);
    }
}

int World::Xwidth()
{
    return (MaxX - MinX) / Xcnt;
}

int World::Ywidth()
{
    return (MaxY - MinY) / Ycnt;
}

int World::GetGridNo(int _x, int _y)
{
    /*网格编号=(x-x轴起始坐标)/x轴网格宽度 + (y-y轴起始坐标)/y轴宽度*x轴网格数量*/
    return (_x - MinX) / Xwidth() + (_y - MinY) / Ywidth() * Xcnt;
}

Grid *World::GetGrid(int _x, int _y)
{
    return m_Grids[GetGridNo(_x, _y)];
}

/*获取周围格子，获取到的对象会存到第二个参数指定的list中*/
void World::GetSurroundGrids(int GridNo, list < Grid * > &Grids)
{
    /*先记录自己*/
    Grids.push_back(m_Grids[GridNo]);

    /*计算当前格子横着数和竖着数分别是几*/
    int Xno = GridNo % Xcnt;
    int Yno = GridNo / Xcnt;

    /*从九宫格的左上开始依次判断每个格子是否是周围格子，若是则记录*/
    if (Xno > 0 && Yno > 0)
    {
        Grids.push_back(m_Grids[GridNo-1-Xcnt]);        
    }
    if (Yno > 0)
    {
        Grids.push_back(m_Grids[GridNo-Xcnt]);
    }
    if ((Xno < Xcnt - 1) && (Yno > 0))
    {
        Grids.push_back(m_Grids[GridNo+1-Xcnt]);
    }
    if (Xno > 0)
    {
        Grids.push_back(m_Grids[GridNo-1]);
    }
    if (Xno < Xcnt - 1)
    {
        Grids.push_back(m_Grids[GridNo+1]);
    }
    if ((Xno > 0) && (Yno < Ycnt - 1))
    {
        Grids.push_back(m_Grids[GridNo-1+Xcnt]);
    }
    if (Yno < Ycnt - 1)
    {
        Grids.push_back(m_Grids[GridNo+Xcnt]);
    }
    if ((Xno < Xcnt - 1) && (Yno < Ycnt - 1))
    {
        Grids.push_back(m_Grids[GridNo+1+Xcnt]);
    }
}
```

**测试：**

参照1.5.1的例子，获取三个玩家坐标对应的格子编号；分别获取25,33和17号格子的周围格子。

```cpp
using namespace std;

int main()
{
    /*按照上例特征创建游戏世界对象*/
    World *pxWorld = new World(20,200,50,230,6,6);
    
    /*分别获取三个玩家所属的格子号*/
    cout<<"Player1："<<pxWorld->GetGridNo(60, 107)<<endl;
    cout<<"Player2："<<pxWorld->GetGridNo(91, 118)<<endl;
    cout<<"Player3："<<pxWorld->GetGridNo(147, 133)<<endl;

    /*获取25号格子周围的格子*/
    list < Grid * > Grids;
    pxWorld->GetSurroundGrids(25, Grids);
    cout<<"25 surround:"<<endl;
    for (auto itr = Grids.begin(); itr != Grids.end(); itr++)
    {
        cout<<(*itr)->GridNo<<endl;
    }
    Grids.clear();
    
    /*获取33号格子周围的格子*/
    pxWorld->GetSurroundGrids(33, Grids);
    cout<<"33 surround:"<<endl;
    for (auto itr = Grids.begin(); itr != Grids.end(); itr++)
    {
        cout<<(*itr)->GridNo<<endl;
    }
    Grids.clear();
    
    /*获取17号格子周围的格子*/
    pxWorld->GetSurroundGrids(17, Grids);
    cout<<"17 surround:"<<endl;
    for (auto itr = Grids.begin(); itr != Grids.end(); itr++)
    {
        cout<<(*itr)->GridNo<<endl;
    }
    Grids.clear();
    
    return 0;
}
```
