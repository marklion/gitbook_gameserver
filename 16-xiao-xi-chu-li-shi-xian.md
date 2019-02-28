# 1.6 消息处理实现

+ 新客户端连接后，向其发送ID和名称
+ 新客户端连接后，向其发送**周围**玩家的位置
+ 新客户端连接后，向**周围**玩家发送其位置
+ 收到客户端的移动信息后，向**周围**玩家发送其新位置
+ 收到客户端的移动信息后，向其发送**周围新**玩家位置
+ 收到客户端的聊天信息后，向**所有**玩家发送聊天内容
+ 客户端断开时，向**周围**玩家发送其断开的消息

## 1.6.1 消息内容封装

+ 发送时机
+ **消息内容**
+ 发送对象：AOI实现

结合需求和之前的消息类型表格，定义函数用于生成待发送的消息

| 消息ID | 消息内容 | 消息类名 |
| --- | --- | --- |
| 1 | 玩家ID和玩家姓名 | SyncPid |

```cpp
GameMessage *GameRole::MakeLogonSyncIdMsg()
{
    GameMessage *pxMsg = new GameMessage(GAME_MSG_ID_LOGON_SYNCPID);
    pb::SyncPid *pxSyncPidMsg = dynamic_cast<pb::SyncPid *>(pxMsg->pxProtoBufMsg);

    pxSyncPidMsg->set_pid(iPid);
    pxSyncPidMsg->set_username(szName);

    return pxMsg;
}
```

| 消息ID | 消息内容 | 消息类名 |
| --- | --- | --- |
| 200 | 玩家ID，聊天内容/初始位置/动作（预留）/新位置 | BroadCast |

```cpp
/*广播聊天消息，参数是消息内容*/
GameMessage *GameRole::MakeBroadCastTalkContent(string szContent)
{
    GameMessage *pxMsg = new GameMessage(GAME_MSG_ID_BROADCAST);
    pb::BroadCast *pxBCTalk = dynamic_cast<pb::BroadCast *>(pxMsg->pxProtoBufMsg);

    pxBCTalk->set_pid(iPid);
    pxBCTalk->set_username(szName);
    /*设定tp=1代表这是一个聊天信息*/
    pxBCTalk->set_tp(1);
    pxBCTalk->set_content(szContent);

    return pxMsg;
}

/*广播玩家登陆后所处位置*/
GameMessage *GameRole::MakeBroadCastLogonPosition()
{
    GameMessage *pxMsg = new GameMessage(GAME_MSG_ID_BROADCAST);
    pb::BroadCast *pxBCLogonPos = dynamic_cast<pb::BroadCast *>(pxMsg->pxProtoBufMsg);

    pxBCLogonPos->set_pid(iPid);
    pxBCLogonPos->set_username(szName);
    /*设定tp=2代表这是玩家位置消息*/
    pxBCLogonPos->set_tp(2);
    /*位置数据是通过消息嵌套表示，mutable代表创建嵌套对象并关联到父消息中*/
    pb::Position *pxSubPos = pxBCLogonPos->mutable_p();
    /*向子消息中逐个填入坐标*/
    pxSubPos->set_x((int)x);
    pxSubPos->set_y((int)y);
    pxSubPos->set_z((int)z);
    pxSubPos->set_v((int)v);

    return pxMsg;
}

/*广播玩家移动后的新位置消息*/
GameMessage *GameRole::MakeBroadCastNewPosition()
{
    GameMessage *pxMsg = new GameMessage(GAME_MSG_ID_BROADCAST);
    pb::BroadCast *pxBCNewPos = dynamic_cast<pb::BroadCast *>(pxMsg->pxProtoBufMsg);

    pxBCNewPos->set_pid(iPid);
    pxBCNewPos->set_username(szName);
    /*设定tp=4代表这是玩家新位置消息*/
    pxBCNewPos->set_tp(4);
    pb::Position *pxSubPos = pxBCNewPos->mutable_p();
    pxSubPos->set_x((int)x);
    pxSubPos->set_y((int)y);
    pxSubPos->set_z((int)z);
    pxSubPos->set_v((int)v);

    return pxMsg;
}
```

| 消息ID | 消息内容 | 消息类名 |
| --- | --- | --- |
| 201 | 玩家ID和玩家姓名 | SyncPid |

| 消息ID | 消息内容 | 消息类名 |
| --- | --- | --- |
| 202 | 周围玩家们的位置 | SyncPlayers |

+ 新客户端连接后，向其发送ID和名称
  - GameRole的构造函数中，需要对ID和名称进行赋值，使用全局变量递增，保证ID唯一
  - 创建函数SendSelfIDName用于回传ID和名称
  - init函数内调用函数SendSelfIDName
+ 新客户端连接后，向其发送**周围**玩家的位置
+ 新客户端连接后，向**周围**玩家发送其位置
  - GameRole的构造函数中，要对玩家坐标进行赋值
  - 根据客户端实际情况创建单例游戏世界对象，后续通过该对象完成所有**周围**的功能
  - 定义函数GetSurroundPlayers用于获取周围玩家
  - 定义函数SyncSelfPostion用于向自己发送周围玩家的位置和向周围玩家发送自己的位置
  - init函数内调用SyncSelfPostion
+ 收到客户端的移动信息后，向**周围**玩家发送其新位置
+ 收到客户端的移动信息后，向其发送**周围新**玩家位置
  - 定义IdProcMoveMsg类用于处理玩家移动的消息
  - 定义OnExchangeAioGrid用于处理玩家切换网格时的视野更新
  - 定义Update函数用于向和周围玩家同步位置（别人的位置告诉自己，自己的位置告诉别人）
  - init函数内注册IdProcMoveMsg
+ 收到客户端的聊天信息后，向**所有**玩家发送聊天内容
  - 定义IdProcTalkMsg类用于处理聊天信息
  - 从server实例中获取所有玩家，并循环发送聊天内容
  - init函数内注册IdProcTalkMsg
+ 客户端断开时，向**周围**玩家发送其断开的消息
  - fini函数中向周围玩家发送下线消息



