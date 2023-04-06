## 概 述
Qt中提供了 `QSerialPort` 类来实现串口相关的操作，本文介绍Qt中如何使用 `QSerialPort` 类来实现串口通信。  
`.pro` 文件中添加
```
QT += serialport
```
头文件
```
#include <QSerialPort>
#include <QSerialPortInfo> //用于获取串口数据
```

## 实 现
**一、获取串口信息**  
使用 `QSerialPortInfo::availablePorts()`，获取系统上可用的串口列表。即QSerialPortInfo对象列表：`QList<QSerialPortInfo>`，列表中的每个QSerialPortInfo对象表示一个串行端口，可以查询端口名称、系统位置、描述等信息。
```
//遍历返回的QSerialPortInfo对象列表
foreach (const QSerialPortInfo &info, QSerialPortInfo::availablePorts()) {
    //将串口名称添加到界面下拉框中
    ui->serialName->addItem(info.portName()); 
}
```

**二、设置串口信息**  
打开串口前，先对串口的相关配置进行设置，例如连接的串口名称、波特率、数据位等等。
```
m_serialPort->setPortName(ui->serialName->currentText()); //设置串口名称，使用界面下拉框中的串口
m_serialPort->setBaudRate(QSerialPort::Baud115200); //设置波特率
m_serialPort->setDataBits(QSerialPort::Data8); //设置数据位
m_serialPort->setStopBits(QSerialPort::OneStop); //设置停止位
m_serialPort->setParity(QSerialPort::NoParity); //设置有无校验位
m_serialPort->setFlowControl(QSerialPort::NoFlowControl); //设置流控制
```

**三、打开串口**  
```
void MainWindow::on_serialOpenBtn_clicked()
{
    if (!m_serialPort->isOpen()) {
        if (m_serialPort->open(QIODevice::ReadWrite)) { //以读写方式打开串口
            m_serialPort->clear(); //每次打开串口，先清空一次缓冲区数据
            connect(m_serialPort, SIGNAL(readyRead()), SLOT(onReadyRead())); //连接数据读取信号槽
            qDebug() << "打开成功";
        } else {
            qDebug() << "打开失败";
        }
    } else {
        m_serialPort->clear();
        m_serialPort->close();
        qDebug() << "关闭";
    }
}
```

**四、数据交互**  
```
//发送数据
void MainWindow::on_sendData_Btn_clicked()
{
    QByteArray data;
    
    //根据于下位机定义的协议，自定义数据内容
    //...

    qint64 writeLen = m_serialPort->write(data);
    if (writeLen == -1) {
        qDebug() << "数据发送失败";
    } else {
        qDebug() << "数据发送成功";
    }
}

//读取数据
//在打开串口时，连接的槽函数
void MainWindow::onReadyRead()
{
    //简单的读取所有数据
    QByteArray data = m_serialPort->readAll();
    qDebug() << "Data:" << data;

    //根据自定义协议，处理数据
    //...
}
```
**注：** 对于复杂的通信协议，为了避免丢包等一系列问题造成的数据丢失，简单的 `readAll()` 并不能满足要求，Qt提供了以下的函数：
```
//返回等待读取的传入字节数。
bytesAvailable()
//从设备读取maxSize字节数据（此方法的读取，不会将数据从缓冲区取走）
peek(char *data, qint64 maxSize)
//在设备上启动新的读取事务
startTransaction()
//完成读取事务
commitTransaction()
//回滚读取事务
rollbackTransaction()
```
简单的示例：
```
void MainWindow::onReadyRead()
{
    MSG_HEAD msgHead; //自定义的协议消息头
    qint64 bytes;
    uint32_t flag; //自定义协议中的标志位
    
    while (true) {
        bytes = m_serialPort->bytesAvailable();
        if (bytes >= qint64(sizeof flag)) { //flag为自定义协议中的标志位，用于判断是否为有效的协议数据
            //peek flag长度的数据，但不从缓冲区中取出
            m_serialPort->peek(reinterpret_cast<char *>(&flag), sizeof flag);
            if (flag == FLAG) { //判断标志位，
                m_serialPort->startTransaction(); //开启事务
                m_serialPort->read(reinterpret_cast<char *>(&msgHead), sizeof(MSG_HEAD)); //读取协议消息头
                bytes -= sizeof(MSG_HEAD); //减去协议消息头长度

                if (bytes >= msgHead.msgLen) { //判断剩余数据长度是否大于协议消息体长度

                    handleAddrData(m_serialPort->read(ackHead.msgLen)); //读取消息体长度的数据

                    //数据处理
                    //...

                    m_serialPort->commitTransaction(); //提交读取事务
                    continue;
                }
                m_serialPort->rollbackTransaction(); //数据长度不足，回滚读取事务
            }
        }
        break;
    }
}
```

**五、析构**  
清理缓冲区并关闭串口
```
if (m_serialPort->isOpen()) {
    m_serialPort->clear();
    m_serialPort->close();
}
delete m_serialPort;
```