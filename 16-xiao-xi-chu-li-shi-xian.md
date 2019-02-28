# 1.6 消息处理实现

* 新客户端连接后，向其发送ID和名称
* 新客户端连接后，向其发送**周围**玩家的位置
* 新客户端连接后，向**周围**玩家发送其位置
* 收到客户端的移动信息后，向**周围**玩家发送其新位置
* 收到客户端的移动信息后，向其发送**周围新**玩家位置
* 收到客户端的聊天信息后，向**所有**玩家发送聊天内容
* 客户端断开时，向**周围**玩家发送其断开的消息

## 1.6.1 消息内容封装

* 发送时机
* **消息内容**
* 发送对象

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

```cpp
GameMessage *GameRole::MakeLogoffSyncIdMsg()
{
    /*与第一个消息的唯一区别就是消息ID不同*/
    GameMessage *pxMsg = new GameMessage(GAME_MSG_ID_LOGOFF_SYNCPID);
    pb::SyncPid *pxSyncPidMsg = dynamic_cast<pb::SyncPid *>(pxMsg->pxProtoBufMsg);

    pxSyncPidMsg->set_pid(iPid);
    pxSyncPidMsg->set_username(szName);

    return pxMsg;
}
```

| 消息ID | 消息内容 | 消息类名 |
| --- | --- | --- |
| 202 | 周围玩家们的位置 | SyncPlayers |

```cpp
/*上线后周围玩家的位置消息，参数是周围玩家list*/
GameMessage *GameRole::MakeSurroudPosition(list < GameRole * > & players)
{
    GameMessage *pxMsg = new GameMessage(GAME_MSG_ID_SURROUND_POSITION);
    pb::SyncPlayers *pxSyncPlayersMsg = dynamic_cast<pb::SyncPlayers *>(pxMsg->pxProtoBufMsg);

    /*遍历玩家list，将每个玩家的信息都填入消息对象*/
    for (auto itr = players.begin(); itr != players.end(); itr++)
    {
        GameRole *pxPlayer = *itr;
        /*多个嵌套消息时，用add_XXX函数创建一个子消息对象并关联到父消息中*/
        pb::Player *pxSubPlayerMsg = pxSyncPlayersMsg->add_ps();
        /*将每个玩家的相关信息填入子消息对象*/
        pxSubPlayerMsg->set_pid(pxPlayer->iPid);
        pxSubPlayerMsg->set_username(pxPlayer->szName);
        pb::Position *pxSubPlayerPos = pxSubPlayerMsg->mutable_p();
        pxSubPlayerPos->set_x((int)(pxPlayer->x));
        pxSubPlayerPos->set_y((int)(pxPlayer->y));
        pxSubPlayerPos->set_z((int)(pxPlayer->z));
        pxSubPlayerPos->set_v((int)(pxPlayer->v));
    }

    return pxMsg;
}
```

## 1.6.2 流程完善

* **发送时机**
* 消息内容
* 发送对象

结合需求，可以看出，服务器发送消息的时机有：

+ 客户端连接后（在init函数中调用）
  - 回传ID和名称（定义函数SendSelfIDName实现）
  - 回传周围玩家位置，向周围玩家发送新玩家位置（定义函数SyncSelfPostion实现）
  
```cpp
void GameRole::SendSelfIDName()
{
    Response stResp;
    stResp.pxSender = this;
    stResp.pxMsg = MakeLogonSyncIdMsg();

    Server::GetServer()->send_resp(&stResp);
}

void GameRole::SyncSelfPostion()
{
    list < GameRole * > Players;

    GetSurroundPlayers(Players);
    
    /*回传周围玩家位置*/
    Response stResp;
    stResp.pxSender = this;
    stResp.pxMsg = MakeSurroudPosition(Players);
    Server::GetServer()->send_resp(&stResp);

    /*遍历周围玩家，发送新玩家位置*/
    Response stResp2Players;
    stResp2Players.pxMsg = MakeBroadCastLogonPosition();
    for (auto itr = Players.begin(); itr != Players.end(); itr++)
    {
        stResp2Players.pxSender = *itr;
        Server::GetServer()->send_resp(&stResp2Players);
    }
}
```

+ 客户端断开后向周围玩家发送下线消息（在fini函数调用）

```cpp
/*客户端断开后执行*/
void GameRole::fini()
{
    cout<<"GameRole object is deled from server"<<endl;
    list < GameRole * > Players;
    GetSurroundPlayers(Players);

    /*构造下线消息并向周围玩家发送*/
    Response stResp;
    stResp.pxMsg = MakeLogoffSyncIdMsg();
    for (auto itr = Players.begin(); itr != Players.end(); itr++)
    {
        stResp.pxSender = *itr;
        Server::GetServer()->send_resp(&stResp);
    }

    g_xGameWorld.GetGrid(x, z)->remove(this);
}
```

+ 收到客户端的移动消息后（注册IdProcMoveMsg，调用update函数）
  - 处理玩家切换网格后的视野变化（定义函数OnExchangeAioGrid）
  - 向周围玩家发送新位置（在函数update中调用）

```cpp
/*Update函数用于处理玩家位置更新，参数是新位置*/
void GameRole::Update(int _newX, int _newY, int _newZ, int _newV)
{
    /*获取新位置所在的网格和旧位置所在的网格*/
    int OldGridNo = g_xGameWorld.GetGridNo(x, z);
    Grid *pxOldGrid = g_xGameWorld.GetGrid(x, z);
    int NewGridNo = g_xGameWorld.GetGridNo(_newX, _newZ);
    Grid *pxNewGrid = g_xGameWorld.GetGrid(_newX, _newZ);

    /*更新坐标*/
    x = _newX;
    y = _newY;
    z = _newZ;
    v = _newV;

    /*如果新旧网格变化了，则要处理视野切换*/
    if (OldGridNo != NewGridNo)
    {
        pxOldGrid->remove(this);
        pxNewGrid->add(this);
        OnExchangeAioGrid(OldGridNo, NewGridNo);
    }

    /*构造发送新位置的消息*/
    Response stResp;
    stResp.pxMsg = MakeBroadCastNewPosition();

    /*发送给当前位置周围的所有玩家*/
    list<GameRole *> players;
    GetSurroundPlayers(players);
    for (auto itr = players.begin(); itr != players.end(); itr++)
    {
        stResp.pxSender = *itr;
        Server::GetServer()->send_resp(&stResp);
    }
}

/*网格切换后的处理函数*/
void GameRole::OnExchangeAioGrid(int _oldGrid, int _newGrid)
{
    list<Grid *> OldGrids;
    list<Grid *> NewGrids;

    /*分别获取新旧的九宫格*/
    g_xGameWorld.GetSurroundGrids(_oldGrid, OldGrids);
    g_xGameWorld.GetSurroundGrids(_newGrid, NewGrids);

    /*遍历旧九宫格查找不在新的九宫格的部分*/
    for (auto itr = OldGrids.begin(); itr != OldGrids.end(); itr++)
    {
        Grid *pxGrid = (*itr);
        auto find_itr = find(NewGrids.begin(), NewGrids.end(), pxGrid);
    
        /*找不到意味着这个格子已经不是当前玩家的周围玩家了*/
        if (find_itr == NewGrids.end())
        {
            /*视野丢失处理*/
            ViewLost(pxGrid);
        }
    }
    /*遍历新的九宫格查找不在旧九宫格的部分*/
    for (auto itr = NewGrids.begin(); itr != NewGrids.end(); itr++)
    {
        Grid *pxGrid = (*itr);
        auto find_itr = find(OldGrids.begin(), OldGrids.end(), pxGrid);
        
        /*找不到意味着这个格子是当前玩家的新邻居*/
        if (find_itr == OldGrids.end())
        {
            /*视野出现处理*/
            ViewAppear(pxGrid);
        }
    }
}

/*视野出现处理，参数是需要视野出现的格子*/
void GameRole::ViewAppear(Grid * pxGrid)
{
    /*构造视野出现的消息，跟玩家上线时的消息相同*/
    Response stResp2Others;
    stResp2Others.pxMsg = MakeBroadCastLogonPosition();
    /*遍历参数指定的格子内所有按键，挨个发送*/
    for (auto itr = pxGrid->players.begin(); itr != pxGrid->players.end(); itr++)
    {
        GameRole *pxPlayer = *itr;
        stResp2Others.pxSender = pxPlayer;
        Server::GetServer()->send_resp(&stResp2Others);
        
        /*将遍历到的每个玩家的信息构造成视野出现的消息*/
        Response stResp;
        stResp.pxMsg = pxPlayer->MakeBroadCastLogonPosition();
        /*发送给当前玩家*/
        stResp.pxSender = this;
        Server::GetServer()->send_resp(&stResp);
    }
}

/*视野丢失的处理，参数需要视野丢失的格子*/
void GameRole::ViewLost(Grid * pxGrid)
{
    /*构造视野消失的消息，跟玩家下线的消息相同*/
    Response stResp2Other;
    stResp2Other.pxMsg = MakeLogoffSyncIdMsg();
    /*遍历参数指定的格子内所有玩家，挨个发送*/
    for (auto itr = pxGrid->players.begin(); itr != pxGrid->players.end(); itr++)
    {
        GameRole *pxPlayer = *itr;
        stResp2Other.pxSender = pxPlayer;
        Server::GetServer()->send_resp(&stResp2Other);
        
        /*将遍历到的每个玩家的信息构造成视野丢失的消息*/
        Response stResp;
        stResp.pxMsg = pxPlayer->MakeLogoffSyncIdMsg();
        /*发送给当前玩家*/
        stResp.pxSender = this;
        Server::GetServer()->send_resp(&stResp);
    }
}
/*玩家移动消息处理对象*/
class IdProcMoveMsg:public IIdMsgProc{
    virtual bool ProcMsg(IdMsgRole * _pxRole, IdMessage * _pxMsg)
    {
        bool bRet = false;

        GameRole *pxCurPlayer = dynamic_cast<GameRole *>(_pxRole);
        GameMessage *pxMoveMsg = dynamic_cast<GameMessage *>(_pxMsg);
        pb::Position *pxNewPos = dynamic_cast<pb::Position *>(pxMoveMsg->pxProtoBufMsg);
        if (NULL != pxNewPos)
        {
            /*取出消息中携带的坐标，调用update函数*/
            pxCurPlayer->Update(pxNewPos->x(), pxNewPos->y(), pxNewPos->z(), pxNewPos->v());
            bRet = true;
        }

        return bRet;
    }
};

bool GameRole::init()
{
    bool bRet = false;

    cout<<"GameRole object is added to server"<<endl;
    
     /*注册消息处理对象*/ 
    bRet |= register_id_func(GAME_MSG_ID_NEW_POSITION, new IdProcMoveMsg());

    if (true == bRet)
    {
        g_xGameWorld.GetGrid(x, z)->add(this);
        
        SendSelfIDName();
        SyncSelfPostion();
    }
    
    return bRet;
}
```

+ 收到聊天信息后，发送给所有玩家（注册IdProcTalkMsg
对象处理）

```cpp
class IdProcTalkMsg:public IIdMsgProc{
    virtual bool ProcMsg(IdMsgRole * _pxRole, IdMessage * _pxMsg)
    {
        bool bRet = false;
        
        GameRole *pxCurPlayer = dynamic_cast<GameRole *>(_pxRole);
        GameMessage *pxRecvMsg = dynamic_cast<GameMessage *>(_pxMsg);
        pb::Talk *pxTalkContent = dynamic_cast<pb::Talk *>(pxRecvMsg->pxProtoBufMsg);

        if (NULL != pxTalkContent)
        {
            /*取出聊天内容并构造成聊天消息*/
            Response stResp;
            stResp.pxMsg = pxCurPlayer->MakeBroadCastTalkContent(pxTalkContent->content());
            
            /*取出所有玩家对象，遍历并发送聊天信息*/
            auto RoleList = Server::GetServer()->GetRoleListByCharacter("GameRole");
            for (auto itr = RoleList->begin(); itr != RoleList->end(); itr++)
            {
                stResp.pxSender = *itr;
                Server::GetServer()->send_resp(&stResp);
            }
            
            bRet = true;
        }

        return bRet;
    }
};
bool GameRole::init()
{
    bool bRet = false;

    cout<<"GameRole object is added to server"<<endl;
    /*注册聊天消息处理对象*/
    bRet = register_id_func(GAME_MSG_ID_TALK_CONTENT, new IdProcTalkMsg());   
    bRet |= register_id_func(GAME_MSG_ID_NEW_POSITION, new IdProcMoveMsg());

    if (true == bRet)
    {
        g_xGameWorld.GetGrid(x, z)->add(this);
        
        SendSelfIDName();
        SyncSelfPostion();
    }
    
    return bRet;
}
```



