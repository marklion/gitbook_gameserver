# 3.5 QT实现登陆客户端

为了方便启动游戏，我们使用QT实现登陆客户端

## 3.5.1 需求和设计

**原始需求：** 用户可以选择某个游戏服务器并登陆游戏

**详细需求（设计）：**

+ QT程序启动时应向服务器获取当前游戏服务器信息
+ 用户点击登陆后要启动游戏进程并传递端口号作为参数

## 3.5.2 关键技术

#### http网络访问

`QNetworkAccessManager`,`QNetworkRequest`和`QNetworkReply`这三个类是Qt在网络访问比较常用的类。

**使用步骤：**

1. 创建QNetworkAccessManager对象
2. 创建QNetworkRequest对象，并用setXXX的方法，设置http的请求头或请求数据
3. 调用QNetworkAccessManager对象的post或get方法，将请求发出去。函数返回一个QNetworkReply对象。
4. 将第3步的QNetworkReply对象的finish信号绑定到一个槽函数。
5. 在槽函数中通过QNetworkReply的readAll函数可以读取到服务器的响应数据

#### Json数据解析

`QJsonDocument`, `QJsonArray`和 `QJsonObject`这三个类在解析Json数据时比较常用

**使用步骤**：

1. 使用QJsonDocument创建整体的Json数据对象
2. 调用QJsonDocument的array函数可以返回QJsonArray的对象
3. 遍历QJsonArray对象可以取出QJsonObject对象
4. 操作符[]可以访问指定的key所对应的值

#### 启动外部进程

`QProcess` 用于启动外部程序

`startDetached`成员函数可以启动程序，并且跟被启动的程序脱离父子关系。

## 3.5.3 实现

```cpp
#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>
#include <QtNetwork/QNetworkAccessManager>
#include <QStringList>

#define LOGIN_SERVER_IP "192.168.254.105"

namespace Ui {
class MainWindow;
}

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    explicit MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

private slots:
    void on_NetWorkInfoFinish();
    void on_LogonBut_clicked();

private:
    Ui::MainWindow *ui;
    QNetworkAccessManager acc_mngr;
    QNetworkReply *m_reply = Q_NULLPTR;
    QString szServerIP = LOGIN_SERVER_IP;
    QStringList m_port_list;
};

#endif // MAINWINDOW_H
```

```cpp
#include "mainwindow.h"
#include "ui_mainwindow.h"
#include <QJsonArray>
#include <QJsonDocument>
#include <QJsonObject>
#include <QtNetwork/QNetworkRequest>
#include <QtNetwork/QNetworkReply>
#include <QProcess>
#include <QFile>
#include <QFileInfo>

MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);
    QFile ip_file("server_ip_record.txt");
    ip_file.open(QIODevice::ReadOnly|QIODevice::Text);
    QTextStream in_file(&ip_file);
    in_file >> szServerIP;
    ip_file.close();
    QNetworkRequest req;
    QString url = QString("http://%1:9999/info").arg(szServerIP);
    req.setUrl(url);
    if (Q_NULLPTR != m_reply)
    {
        m_reply->deleteLater();
    }
    m_reply = acc_mngr.get(req);
    connect(m_reply, SIGNAL(finished()), this, SLOT(on_NetWorkInfoFinish()));
}

MainWindow::~MainWindow()
{
    delete ui;
}

void MainWindow::on_NetWorkInfoFinish()
{
    QByteArray reply_data = m_reply->readAll();
    QJsonDocument json_doc = QJsonDocument::fromJson(reply_data);

    if (true == json_doc.isArray())
    {
        QJsonArray json_array = json_doc.array();
        for (int i = 0; i < json_array.size(); i++)
        {
            QJsonObject json_obj = json_array[i].toObject();
            ui->server_zone_item->addItem(json_obj["Name"].toString());
            m_port_list.push_back(json_obj["Port"].toString());
        }
    }
}

void MainWindow::on_LogonBut_clicked()
{
    int server_zone_no = ui->server_zone_item->currentIndex();

    QString port = m_port_list[server_zone_no];
    QStringList args;
    args<<szServerIP<<port;
    QProcess *pxPro = new QProcess(this);
    pxPro->startDetached("client.exe", args);
    qApp->exit();
}
```

