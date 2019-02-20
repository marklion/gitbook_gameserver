# 1.2 zinx框架

zinx是一个通用的并发IO处理框架
github路径：https://github.com/marklion/Game_zinx
## 1.2.1 学习使用zinx框架

1. 阅读框架文档
> 一般地，项目的github主页上会列出README文档。

2. 运行文档中的案例并做一定扩展

## 1.2.2 tcp封包处理实现

**问题**：tcp或类似的流式文件无法保证收到的数据按照期望的格式分割。
**举例**：服务器期望接收2个字节的数据作为一个合理请求。客户端发送了两个请求（四个字节）后，由于网络拥塞，服务器收到了1个字节后，recv返回，1秒钟后，数据到来，再次调用recv会收到3个字节。
**常规套路**：
1. 设定报文边界，一般使用Tag Length Value的格式
2. recv数据后，若接收缓冲区当前数据长度小于报文内规定长度，则保留当前缓冲区，下次recv数据后重新处理
3. 若接收缓冲区数据长度大于等于报文内规定长度，则生成请求并保留后续多余的数据等待下次recv数据后重新处理

> zinx框架提供了Aprotocol类专门用于报文/流向请求的转化

**案例**：参照1.1.4中定义的消息结构，实现服务器收到tcp消息后回传**消息内容**。
**思路**：
+ 定义myMessage类继承Amessage类，类内定义成员变量msg_content用于存储消息内容
+ 实现myprotolcol类继承Aprotocol类并重写其中的两个方法
> bool Aprotocol::raw2request(const RawData *pstData, std::list<Request *> &_ReqList);
bool Aprotocol::response2raw(const Response * pstResp, RawData * pstData);

+ 绑定myprotolcol对象到tcp的channel对象（tcp数据将会自动被myprotolcol处理）
+ 定义myrole继承Arole类，重写proc_msg方法，实现回传tcp消息






