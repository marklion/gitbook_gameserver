# 3.4 登陆服务器实现

现在使用Nginx+FastCGI的形式实现登陆服务器。

## 3.4.1 需求和设计

**原始需求：** 为客户端提供当前存在的所有服务器名称和对应端口。

**详细需求：**

1. 游戏服务器名称要可配（不能改代码）
2. 按照配置的游戏服务器数量启动若干容器，并收集映射的端口
3. 当用户请求来临时，向其回复当前若干对{服务器,端口号}数据

**设计：**

1. 用文件存储当前开服的游戏服务器名称
2. 简单的容器命令组合成shell脚本实现启动多个容器
3. 简单的命令组合成shell脚本实现查询并记录容器的端口号
4. 使用FastCGI程序处理用户请求，回复内容是：之前记录的服务器名称以及端口号

## 3.4.2 关键技术

服务器按照什么格式向客户端发送数据？

方案1：纯文本（显然不利于解析和扩展）  
方案2：json（结构化格式，很适合）

CJsonObject库可以用来进行Json的编解码。

API|描述
-|-
CJsonObject\(\)|创建空的json对象
CJsonObject\(const std::string& strJson\)|基于json字符串创建json对象（解析）
Add(const std::string& strKey, int32 iValue);|向json对象中添加键值对，值类型可以重载成json对象等所有支持的类型
Add(const std::string& strValue);|向json对象中添加数组元素，可以重载成json对象等所有支持的类型
ToFormattedString()|将json对象转换成字符串

**示例：**

```cpp
#include "CJsonObject.hpp"
#include <iostream>

using namespace std;

int main()
{
    neb::CJsonObject jo;
    jo.Add("key1", 1);
    jo.Add("key2", "this is 2");
    neb::CJsonObject jo_a;
    jo_a.Add(jo);
    jo_a.Add(3);
    cout<<jo_a.ToFormattedString()<<endl;

    return 0;
}
```

编译时将CJsonObject包内的代码一并编译

```bash
$ g++ *.c* 
dev@ubuntu:~/CJsonObject-1.0$ ./a.out 
[{
		"key1":	1,
		"key2":	"this is 2"
	}, 3]
```

## 3.4.2 实现

* 编写游戏服务器名称文件

```bash
$ cat server_name.txt
金戈铁马
折戟沉沙
笑傲江湖
勇创天涯
```

* 编写脚本实现启动容器和查看容器

```bash
$ cat server_manager.sh 
#!/bin/bash

# 先删掉所有game_run镜像的容器
docker rm -f `docker ps -aq -f ancestor=game_run`

# 读取server_name.txt文件，每读到一个名字就创建一个容器
for itr in `cat server_name.txt`
do
    docker run -dP game_run
done
# 查询当前所有容器内打开8899端口的容器的暴露端口，并重定向到server_port.txt文件中
docker ps | grep 8899 | awk -F "->" '{print $1}' | awk -F ":" '{print $2}' > server_port.txt
```

+ 编写FastCGI程序

```cpp
#include "CJsonObject.hpp"
#include <fstream>
#include <string>
#include <fcgi_stdio.h>

using namespace std;

int main()
{
    ifstream name_file;
    ifstream port_file;

    while (FCGI_Accept() >= 0)
    {
        neb::CJsonObject jo;
        string server_name;
        string server_port;
        name_file.open("../server_define.txt");
        port_file.open("../server_ports.txt");
        while (getline(name_file, server_name) && getline(port_file, server_port))
        {
            /*将读取到的数据组织成两对键值对的json对象*/
            neb::CJsonObject subjo;
            subjo.Add("Name", server_name);
            subjo.Add("Port", server_port);
            /*将所有json对象最为数组元素添加到外城json对象中*/
            jo.Add(subjo);
        }
        name_file.close();
        port_file.close();
     
        /*将json对象变成json格式的字符串*/
        string resp = jo.ToFormattedString();
        /*指定本次返回内容为json数据*/
        printf("Content-type: application/json\r\n");
        /*指定json数据的长度并打印空行*/
        printf("Content-length: %d\r\n\r\n", resp.size());
        /*输出json数据*/
        printf("%s\r\n", resp.c_str());
    }

    return 0;
}
```

+ 使用spawn-fcgi启动FastCGI程序并配置nginx。

## 3.4.3 测试

用浏览器访问会得到json字符串。


