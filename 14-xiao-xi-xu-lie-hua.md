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

* test.pb.h文件中定义了pb\_sample::Student类
* test.pb.cc中实现了pb\_sample::Student类中数据序列化和解析的函数
  * `bool ParseFromArray(const void * data, int size)` 函数将长度为size的字节流缓冲区解析成消息对象。
  * `bool ParseFromString(const string & data)` 函数将data这个字符串内容（本质上还是不可阅读的字节流）解析为消息对象。
  * `bool SerializeToArray(void * data, int size) const`函数将消息对象转换成字节流存到data开头长度为size的缓冲区里
  * `bool SerializeToString(string * output) const`函数将消息对象转换成字节流存到output这个字符串中
  * `size_t ByteSizeLong()`函数用于获取序列化后的缓冲区长度

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

编译测试文件\(main.cpp\)和消息类文件\(test.pb.cc\)并测试

```bash
$ g++ -std=c++11 -pthread main.cpp test.pb.cc -lprotobuf
$ ./a.out
08 01 12 03 61 62 63
#测试OK
```

**练习：**

完成以下代码片段，将缓冲区内的数据转换成消息对象s，并打印两个成员变量的值。

```cpp
#include "test.pb.h"
#include <string>
#include <iostream>

using namespace std;

int main()
{
    char aucBuff[] = {0x08, 0x01, 0x12, 0x03, 0x61, 0x62, 0x63};

    /*TODO: 将aucBuff内的数据解析成对象并打印内容*/    

    return 0;
}
```

**小结：**

1. protobuf序列化后的数据时不可阅读流，是保证健壮的情况下占用空间最小的形式（相比xml json）。
2. protobuf使用了Varint技术，使得小的数字不会占用无用空间。详细：[https://developers.google.com/protocol-buffers/docs/encoding](https://developers.google.com/protocol-buffers/docs/encoding)
3. 只要遵守protobuf推荐的消息定义方式，protobuf可以保证消息处理的向前兼容。

## 1.4.2 游戏消息实现

参照之前需求分析时的表格，我们给每种类型的消息起个名字：

| 消息ID | 消息内容 | 消息类名 |
| --- | --- | --- |
| 1 | 玩家ID和玩家姓名 | SyncPid |
| 2 | 聊天内容 | Talk |
| 3 | 新位置 | Position |
| 200 | 玩家ID，聊天内容/初始位置/动作（预留）/新位置 | BroadCast |
| 201 | 玩家ID和玩家姓名 | SyncPid |
| 202 | 周围玩家们的位置 | SyncPlayers |

每个类型的消息详细定义如下（客户端已经定义好了）：

msg.proto

```protobuf
syntax="proto3";
package pb;
//无关选项，用于客户端
option csharp_namespace="Pb";

message SyncPid{
    int32 Pid=1;
    string Username=2;
}

message Player{
    int32 Pid=1;
    Position P=2;
    string Username=3;
}

message SyncPlayers{
        /*嵌套多个子消息类型Player的消息*/
    repeated Player ps=1;
}

message Position{
    float X=1;
    float Y=2;
    float Z=3;
    float V=4;
    int32 BloodValue=5;
}

message MovePackege{
    Position P=1;
    int32 ActionData=2;
}

message BroadCast{
    int32 Pid=1;
    int32 Tp=2;
    /*根据Tp不同，BroadCast消息会包含：
    聊天内容（Content）或初始位置(P)或新位置P*/
    oneof Data {
            string Content=3;
            Position P=4;
            /*ActionData暂时预留*/
        int32 ActionData=5;
        }
    string Username=6;
}


message Talk{
    string Content=1;
}
```

将msg.proto文件放到game目录下，基于该文件生成c++源文件，为了适配我们的Makefile，将其文件名后缀改成cpp

```bash
$ protoc --cpp_out=./ msg.proto
$ ls
game             
GameChannel.h    
GameMessage.h     
GameProtocol.h  
GameRole.h  
Makefile   
msg.pb.h
GameChannel.cpp  
GameMessage.cpp  
GameProtocol.cpp  
GameRole.cpp    
main.cpp    
msg.pb.cpp  
msg.proto
```

#### GameMessage类的实现

+ 属性pxProtoBufMsg用来存消息内容对象：protobuf生成的SyncPid，Position和BroadCast等类都是google::protobuf::Message的子类。
+ 构造函数内，需要根据消息ID不同，创建不同的google::protobuf::Message的子类对象（SyncPid，Position和BroadCast等）
+ 在成员函数ParseBuff2Msg内直接调用pxProtoBufMsg对象的解析方法
+ 在成员函数SerialMsg2Buff内直接调用pxProtoBufMsg对象的序列化方法
+ 在成员函数GetSerialLength内直接调用pxProtoBufMsg对象的获取长度方法

```cpp
#include "GameMessage.h"
#include "msg.pb.h"
#include <iostream>

using namespace std;

/*根据ID不同，创建不同的ProtobufMsg对象*/
GameMessage::GameMessage(int _id):IdMessage(_id)
{
    /*下列宏均需要在GameMessage.h中定义*/
    switch (_id)
    {
        case GAME_MSG_ID_LOGON_SYNCPID:
            pxProtoBufMsg = new pb::SyncPid();
            break;
        case GAME_MSG_ID_TALK_CONTENT:
            pxProtoBufMsg = new pb::Talk();
            break;
        case GAME_MSG_ID_NEW_POSITION:
            pxProtoBufMsg = new pb::Position();
            break;
        case GAME_MSG_ID_BROADCAST:
            pxProtoBufMsg = new pb::BroadCast();
            break;
        case GAME_MSG_ID_LOGOFF_SYNCPID:
            pxProtoBufMsg = new pb::SyncPid();
            break;
        case GAME_MSG_ID_SURROUND_POSITION:
            pxProtoBufMsg = new pb::SyncPlayers();
            break;
        default:
            pxProtoBufMsg = NULL;
            break;
    }
}

/*析构时，要释放pxProtoBufMsg*/
GameMessage::~GameMessage()
{
    if (NULL != pxProtoBufMsg)
    {
        delete pxProtoBufMsg;
        pxProtoBufMsg = NULL;
    }
}

bool GameMessage::ParseBuff2Msg(const unsigned char * pucDataBuff, int iLength)
{
    bool bRet = false;

    if (NULL != pxProtoBufMsg)
    {
        /*调用成员对象的解析函数*/
        bRet = pxProtoBufMsg->ParseFromArray(pucDataBuff, iLength);
    }

    /*打印当前消息内容*/
    PrintDebugInfo();

    return bRet;
}

bool GameMessage::SerialMsg2Buff(unsigned char * pucDataBuff, int iBufLength)
{
    bool bRet = false;

    /*打印当前消息内容*/
    PrintDebugInfo();

    if (NULL != pxProtoBufMsg)
    {
        /*调用成员对象的序列化函数*/
        bRet = pxProtoBufMsg->SerializeToArray(pucDataBuff, iBufLength);
    }

    return bRet;
}

int GameMessage::GetSerialLength()
{
    int iRet = 0;

    if (NULL != pxProtoBufMsg)
    {
        return pxProtoBufMsg->ByteSizeLong();
    }

    return iRet;
}

void GameMessage::PrintDebugInfo()
{
    cout<<"msg ID:"<<Id<<endl;
    cout<<"msg struct:"<<endl;
    if (NULL != pxProtoBufMsg)
    {
        /*该函数可以将protobuf的消息对象转换成易读的字符串*/
        cout<<pxProtoBufMsg->Utf8DebugString()<<endl;
    }
    else
    {
        cout<<"(Not a protobuf msg)"<<endl;
    }
}
```

#### 测试

**测试用例：** 打开客户端并连接到服务器上，发送一条聊天消息。服务器收到后回传一个字符串"OK"。

**打桩：** 在GameRole类中注册处理消息ID为2的消息处理接口，实现回传"OK"的功能

GameRole.cpp

```cpp
class IdProcTalkMsg:public IIdMsgProc{
    /*处理ID为2的消息的函数*/
    virtual bool ProcMsg(IdMsgRole * _pxRole, IdMessage * _pxMsg)
    {
        cout<<"IdProcTalkMsg.ProcMsg is called"<<endl;
        /*构造空的待发送消息，指定ID为2*/
        GameMessage *pxMsg = new GameMessage(GAME_MSG_ID_TALK_CONTENT);
        /*将成员pxProtoBufMsg动态强转成Talk类型*/
        pb::Talk *pxTalkMsg = dynamic_cast<pb::Talk *>(pxMsg->pxProtoBufMsg);
        /*设置回传内容是OK*/
        pxTalkMsg->set_content("OK");
        Response stResp;
        stResp.pxMsg = pxMsg;
        /*发送给消息来源*/
        stResp.pxSender = _pxRole;
        
        Server::GetServer()->send_resp(&stResp);

        return true;
    }
};
```

**测试执行**：

+ 指定服务器地址和端口号运行客户端：`test.exe 127.0.0.1 8899`
+ 输入任意内容并点击发送

![](/assets/测试消息收发.png)

+ 查看调试信息

```bash
++++++++++++++++++++++++++++++
new tcp connection, fd = 5
++++++++++++++++++++++++++++++
GameRole object is added to server
<----------------------------------------->
recv from 5:05 00 00 00 02 00 00 00 0A 03 61 62 63
<----------------------------------------->
#接收的数据成功转换成ID为2的消息，内容正确
msg ID:2
msg struct:
Content: "abc"

IdProcTalkMsg.ProcMsg is called
msg ID:2
msg struct:
Content: "OK"
#发送的消息成功转换成数据
<----------------------------------------->
send to 5:04 00 00 00 02 00 00 00 0A 02 4F 4B
<----------------------------------------->
```

**小结**： 
+ 代理模式：GameMessage类的序列化和解析的功能由其包含的对象实现。调用者无需区分对待具体每种类型消息。
+ 简单工厂：根据不同情况创建不同的子类对象
