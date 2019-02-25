# 1.4 消息序列化

前边，我们已经搭建好了软件架构并实现了基于TCP的数据收发和TCP数据的封包。
接下来，我们看一下游戏业务中的各种消息应该怎样封装或解析。

消息ID|消息内容|发送方向|客户端处理|服务器处理
-|-|-|-|-
1|玩家ID和玩家姓名|S->C|记录自己ID和姓名|无
2|聊天内容|C->S|无|广播给所有玩家
3|新位置|C->S|无|处理玩家位置更新后的信息同步
200|玩家ID，聊天内容/初始位置/动作（预留）/新位置|S->C|根据子类型不通而不同|无
201|玩家ID和玩家姓名|S->C|把该ID的玩家从画面中拿掉|无
202|周围玩家们的位置|S->C|在画面中显示周围的玩家|无

上表中可以看到，消息内容是结构化的，在软件中用结构体或类可以轻松表示，但在网络传输中却不能直接传输这样的结构化数据。

> 为什么？

## 1.4.1 Protobuf技术

**定义：** 一种语言无关的，平台无关的，可扩展的结构化数据序列化的方式。可以用于通信协议和数据存储等场景。

> protocol buffers – a language-neutral, platform-neutral, extensible way of serializing structured data for use in communications protocols, data storage, and more.

**资料：**
+ 项目仓库：https://github.com/protocolbuffers/protobuf
+ 官方文档：https://developers.google.com/protocol-buffers/docs/overview

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

