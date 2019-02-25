# 1.4 消息序列化

前边，我们已经搭建好了软件架构并实现了基于TCP的数据收发和TCP数据的封包。  
接下来，我们看一下游戏业务中的各种消息应该怎样封装或解析。

| 消息ID | 消息内容 | 发送方向 | 客户端处理 | 服务器处理 |
| --- | --- | --- | --- | --- |
| 1 | 玩家ID和玩家姓名 | S-&gt;C | 记录自己ID和姓名 | 无 |
| 2 | 聊天内容 | C-&gt;S | 无 | 广播给所有玩家 |
| 3 | 新位置 | C-&gt;S | 无 | 处理玩家位置更新后的信息同步 |
| 200 | 玩家ID，聊天内容/初始位置/动作（预留）/新位置 | S-&gt;C | 根据子类型不通而不同 | 无 |
| 201 | 玩家ID和玩家姓名 | S-&gt;C | 把该ID的玩家从画面中拿掉 | 无 |
| 202 | 周围玩家们的位置 | S-&gt;C | 在画面中显示周围的玩家 | 无 |

上表中可以看到，消息内容是结构化的，在软件中用结构体或类可以轻松表示，但在网络传输中却不能直接传输这样的结构化数据。

> 为什么？

## 1.4.1 Protobuf技术

**定义：** 一种语言无关的，平台无关的，可扩展的结构化数据序列化的方式。可以用于通信协议和数据存储等场景。

> protocol buffers – a language-neutral, platform-neutral, extensible way of serializing structured data for use in communications protocols, data storage, and more.

**资料：**

* 项目仓库：[https://github.com/protocolbuffers/protobuf](https://github.com/protocolbuffers/protobuf)
* 官方文档\(科学上网\)：[https://developers.google.com/protocol-buffers/docs/overview](https://developers.google.com/protocol-buffers/docs/overview)

**原理**：将结构体的成员转换成TLV（Tag Length Value）单元后合并成整段不可阅读流（binary stream）。

例如：有一个结构体的定义和实例如下

```cpp
struct Student{
    int No;
    string Name;
};

Student s = {1,"abc"};
```

通过protobuf将s编码（encode）后的数据的二进制展示和解释为：  
`08 01 12 03 61 62 63`

* 08 01表示s.No,08大概代表整数类型，01是数的值
* 12 03 61 62 63表示s.Name,12大概代表字符串类型，03代表字符串长度，61 62 63代表"abc"

**使用方式：**

protobuf是不限语言的，所以我们需要将消息结构定义成protobuf规定格式的配置文件中，进而用protobuf生成目标语言的代码。

#### 第一步，安装protobuf

参考项目仓库的README，执行安装：[https://github.com/protocolbuffers/protobuf/blob/master/src/README.md](https://github.com/protocolbuffers/protobuf/blob/master/src/README.md)

```bash
# 安装依赖
$ sudo apt-get install autoconf automake libtool curl make g++ unzip
```

访问[https://github.com/protocolbuffers/protobuf/releases/latest](https://github.com/protocolbuffers/protobuf/releases/latest)  
下载某个发布版本`protobuf-cpp-[VERSION].tar.gz`

```bash
$ ./configure
$ make
$ make check
$ sudo make install
$ sudo ldconfig # refresh shared library cache.
```

可以敲出protoc命令则意味着安装成功

#### 第二步 编写proto文件

创建test.proto文件用于定义消息结构，protobuf支持的数据类型和C语言的数据类型相似，风格和结构体的风格也类似。

* `message{}`关键字用于定义一个消息类型，大括号内放置消息包含的成员。
* 消息结构的成员定义方法：`[repeated] 数据类型 = 成员编号;`,repeated 代表该成员可以有多个；不写repeated代表该成员只有一个。

创建test.proto文件

```protobuf
//指定当前proto文件的语法是3系列版本
syntax="proto3";
//指定生成代码后，相关结构定义所在的命名空间
package pb_sample;
//消息定义
message Student {
    //数字1 2 是消息成员的序号，要求按顺序编排，若后续新增成员则序号依次递增
    int32 No = 1;
    string Name = 2;
}
```

**扩展**：

* protobuf支持的数据类型包括：数字型（int32 double等）和字符串型（string bytes等），详细：[https://developers.google.com/protocol-buffers/docs/proto3\#scalar](https://developers.google.com/protocol-buffers/docs/proto3#scalar)
* protobuf支持定义更复杂的消息结构：
  * 消息类型直接可以嵌套
    ```protobuf
    message A {
      int32 no=1;
    }
    message B {
      string content=1;
      //消息类型B中包含消息类型A的一个实体
      A sub_message=2;
    }
    ```
  * `Oneof`关键字用于指定消息包含多种数据类型之一。
    ```protobuf
    message B {
      //消息类型B中要么包含字符串content，要么包含一个子类型A的sub_message
      Oneof data {
          string content=1;
          A sub_message=2;
      }
    }
    ```

#### 第三步 生成代码

命令protoc可以基于我们编写的proto文件生成各种语言的原文件。在这里我们生成c++文件。

```bash
# 参数1 指定我们要生成的c++文件放到哪里
# 参数2 指定proto文件
$ protoc --cpp_out=./ test.proto
# 执行成功后会生成两个新文件
$ ls
test.pb.cc  test.pb.h  test.proto
```

* test.pb.h文件中定义了pb_sample::Student类
* test.pb.cc中实现了pb_sample::Student类中数据序列化和解析的函数

#### 第四步 编译并测试

**测试用例：** 创建Student对象并设置其No=1，Name="abc".序列化该对象后打印字节流。

main.cpp

```cpp
#include <cstdio>
#include "test.pb.h"
#include <string>

using namespace std;

int main()
{
    /*创建消息对象s*/
    pb_sample::Student s;
    
    /*调用set函数设置消息内容*/
    s.set_no(1);
    s.set_name("abc");

    string out;
    
    /*将s序列化成字节流，并打印出来*/
    s.SerializeToString(&out);

    for (int i = 0; i < out.size(); i++)
    {
        printf("%02x ", out[i]);
    }
    puts("");

    return 0;
}
```

**编译并测试：**

查看protobuf的编译选项

```bash
$ pkg-config --cflags protobuf
-pthread
```

查看protobuf的链接选项

```bash
$ pkg-config --libs  protobuf
-lprotobuf -pthread
```

编译测试文件(main.cpp)和消息类文件(test.pb.cc)并测试

```bash
$ g++ -std=c++11 -pthread main.cpp test.pb.cc -lprotobuf
$ ./a.out
08 01 12 03 61 62 63
#测试OK
```

**小结：**

1. protobuf序列化后的数据时不可阅读流，是保证健壮的情况下占用空间最小的形式（相比xml json）。
2. protobuf使用了Varint技术，使得小的数字不会占用无用空间。详细：[https://developers.google.com/protocol-buffers/docs/encoding](https://developers.google.com/protocol-buffers/docs/encoding)
3. 只要遵守protobuf推荐的消息定义方式，protobuf可以保证消息处理的向前兼容。

## 1.4.2



