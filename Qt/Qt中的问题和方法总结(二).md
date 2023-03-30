#### 1. 修改QDockWidget的背景颜色
通过样式表为`QDockWidget`添加背景颜色，直接添加`background-color`的效果：  

![效果](https://upload-images.jianshu.io/upload_images/22192996-530d11247211b7da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，`QDockWidget`包括其内部的子组件，都被修改了样式，因为我们没有指定`background-color`添加的对象，添加样式对象:
```
QDockWidget {
	background-color: rgb(255, 255, 255);
}
```
这样`QDockWidget`内部的子组件不会被修改样式，运行后，我们发现，当此窗口**悬停**时，背景变成了白色，而**停靠**主界面边缘时，设置的背景颜色没有失效：

![停靠时](https://upload-images.jianshu.io/upload_images/22192996-524948383adc3d77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![悬停时](https://upload-images.jianshu.io/upload_images/22192996-f2671b0d4d3a6bd7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个问题，我们需要在样式表中添加窗口停靠时的样式`QDockWidget > QWidget{ }`：
```
QDockWidget {
	background-color: rgb(255, 255, 255);
}
QDockWidget > QWidget { 
    background-color: rgb(255, 255, 255);
}
```
#### 2. 取消QDialog窗口的帮助(?)按钮
当我们创建一个`QDialog`并显示时，我们会发现`QDialog`会有个`?`的帮助按钮，通过`setWindowFlags()`可以取消这个按钮
```
//构造函数
Dialog::Dialog(QWidget *parent) :
    QDialog(parent),
    ui(new Ui::Dialog)
{
    ui->setupUi(this);
    //setWindowFlags 设置只保留关闭按钮
    setWindowFlags(Qt::Dialog | Qt::WindowCloseButtonHint); //Qt::Dialog不能省略
}
```
也可以直接使用`new`的方式创建`QDialog`，并指定窗口的`WindowFlags`
```
void MainWindow::on_pushButton_clicked()
{
    QDialog *w = new QDialog(this, Qt::WindowCloseButtonHint);
    w->exec();
}
```
若在此`QDialog`中添加布局，程序运行后，会报错
```
QWindowsWindow::setGeometry: Unable to set geometry 120x30+737+410 (frame: 136x46+729+402) on QWidgetWindow/"QDialogClassWindow" on "\\.\DISPLAY1". Resulting geometry: 244x221+737+410 (frame: 260x237+729+402) margins: 8, 8, 8, 8 minimum size: 244x221 MINMAXINFO maxSize=0,0 maxpos=0,0 mintrack=260,237 maxtrack=0,0)
```
将`WindowFlags`修改为
```
//Qt::CustomizeWindowHint即为固定QDialog的窗口大小
QDialog *w = new QDialog(this, Qt::CustomizeWindowHint | Qt::WindowCloseButtonHint);
```
当然，Qt提供了多种`WindowFlags`用于设置窗口，具体可以参考文档。
#### 3. 移除布局中添加的Widget
我们通常会使用`addWidget()`将自定义的Widget添加到布局中，当需要移除此Widget时，可以使用`removeWidget()`
```
//m_myLayout为QGridLayout类型的成员变量
//m_myWidget为继承于QWidget的自定义Widget
//添加Widget
m_myLayout->addWidget(m_myWidget, 0, 0);

//移除Widget
m_myLayout->removeWidget(m_myWidget);
```
运行程序后，发现只使用`removeWidget()`，自定义的Widget还是会保留在窗口中，如图：

![removeWidget](https://upload-images.jianshu.io/upload_images/22192996-0907cf3dcc44b704.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为`addWidget()`默认将自定义的Widget的父节点设为当前窗口，所以我们需要将自定义的Widget的父节点设为空，就可以彻底移除
```
m_myLayout->removeWidget(m_myWidget);
m_myWidget->setParent(nullptr); //将父节点设为空
```