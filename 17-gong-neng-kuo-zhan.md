# 1.7 功能扩展

当前游戏已经可用，我们接下来尝试扩展两个功能：

+ 出生点随机
+ 玩家昵称随机

## 1.7.1 出生点随机

**设计**： GameRole对象创建时，随机生成合理范围内的坐标。生成随机数的方法: std::default_random_engine(详细资料：http://www.cplusplus.com/reference/random/default_random_engine/)

+ 构造函数参数用于指定种子（一般使用当前时间）
+ ()操作符返回随机数（无符号整形）

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

