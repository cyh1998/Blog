## 概 述
之前写Qt一直都是使用的C++，最近用Python写了一个Qt项目，发现两个还是有很多不同之处的。  
这里整理下使用PyQt过程中遇到的一些问题和解决方法。

## 重定向输出
我们可能会需要将 `print()` 的内容输出到控件，比如输出到 `QTextEdit` 上。  
当我们调用 `print()` 时，实际上是调用了 `sys.stdout.write()`。  
最简单的方法，我们只需要直接设置 `sys.stdout` 对象，重写 `write()` 即可，实现如下：  
```
class MainWindow(QMainWindow, Ui_MusicDownloader):
    def __init__(self):
        super(MainWindow, self).__init__()
        self.setupUi(self)
        sys.stdout = self #设置 stdout

    # 重写 write
    def write(self, text):
        self.PrintTextEdit.insertPlainText(text) #将内容输出到 PrintTextEdit 控件
```
当然，我们还可以添加一个新类，利用Qt的信号机制来实现，如下：  
```
# 新增继承 QObject 的类
class Stream(QtCore.QObject):
    textPrint = QtCore.pyqtSignal(str) #创建信号

    # 重写 write
    def write(self, text):
        self.textPrint.emit(str(text)) #发送信号

class MainWindow(QMainWindow, Ui_MusicDownloader):
    def __init__(self):
        super(MainWindow, self).__init__()
        self.setupUi(self)
        # 设置 stdout 并绑定槽函数
        sys.stdout = Stream(textPrint=self.output_print)

    # 重定向输出
    def output_print(self, text):
        self.PrintTextEdit.insertPlainText(text) #将内容输出到 PrintTextEdit 控件
```
此方法在关闭主界面时，会有报错 
```
RuntimeError: wrapped C/C++ object of type Stream has been deleted
```
这是因为我们重设了 `sys.stdout`，当界面关闭后，我们创建的 `Stream` 对象就会被删除。`sys.stdout` 就会因为找不到 `Stream` 而报错。  
我只需要在界面关闭事件中将 `sys.stdout` 设置回去即可：
```
stdout_temp = sys.stdout

# 新增继承 QObject 的类
class Stream(QtCore.QObject):
    # ...省略

class MainWindow(QMainWindow, Ui_MusicDownloader):
    # ...省略

    # 关闭响应
    def closeEvent(self, event):
        sys.stdout = stdout_temp #将stdout设置回去
        super().closeEvent(event)
```
**补充：**  
上文将文本输出到 `QTextEdit` 上使用了 `insertPlainText()`。  
此外，我们还可以通过操作光标来实现文本输出，只是相对复杂一些，实现如下：
```
# 重定向输出
def output_print(self, text):
    cursor = self.PrintTextEdit.textCursor() #获取光标位置副本
    cursor.movePosition(QtGui.QTextCursor.End) #将光标移动到最后
    cursor.insertText(text) #添加文本
    self.PrintTextEdit.setTextCursor(cursor) #设置文本到控件
    self.PrintTextEdit.ensureCursorVisible() #滚动文本编辑确保光标可见
```

## 多线程
在之前 [C++/Qt 多线程](/Qt/Qt%20%E5%A4%9A%E7%BA%BF%E7%A8%8B.md) 的文章中提到，C++可以使用 `QtConcurrent::run` 高级API来实现多线程，并且无需继承任何类，也不需要重写函数。  
然而 `QtConcurrent` 是通过C++模板特性来实现的，显然Python是不支持的，所以我们还是只能使用 `moveToThread` 来实现。  
原理不再赘述，实现如下：
```
class DownloadWork(QtCore.QObject):
    end_sig = QtCore.pyqtSignal()

    def __init__(self):
        super(DownloadWork, self).__init__()

    def download_music(self, mode, id):
        # ...
        self.end_sig.emit()

class MainWindow(QMainWindow, Ui_MusicDownloader):
    begin_sig = QtCore.pyqtSignal()

    def __init__(self):
        super(MainWindow, self).__init__()
        self.setupUi(self)

        # 下载线程初始化
        self.download_thread = QtCore.QThread()
        self.download_work = DownloadWork()
        self.download_work.moveToThread(self.download_thread)
        # 下载线程信号绑定
        self.begin_sig.connect(self.download_work.download_music)
        self.download_work.end_sig.connect(self.download_thread_end)
        # 启动线程
        self.download_thread.start()

    # 下载
    def download_clicked(self):
        self.begin_sig.emit()
```
这里线程内的工作内容建议通过信号去启动，如果使用线程的 `started` 信号来启动工作会无法传参，而使用自定义信号就可以方便传递参数。

## 异步调用
我们知道 `QMetaObject.invokeMethod` 只能调用Qt 元对象系统已知的方法。因此需要使用 `pyqtSlot` 装饰器来将自定义函数修饰为槽函数。例如：
```
@QtCore.pyqtSlot(int, str, result=bool)
def asyn_download(self, number, string):
	# ...
	return True
```
参考：[QMetaObject::invokeMethod: No such method](https://stackoverflow.com/questions/67135015/qmetaobjectinvokemethod-no-such-method-signal-emittersolve)  
有关 `QMetaObject.invokeMethod` 的内容可以参考 [Qt invokeMethod 异步调用](/Qt/Qt%20invokeMethod%20%E5%BC%82%E6%AD%A5%E8%B0%83%E7%94%A8.md)

****
源码地址：[MusicDownloader](https://github.com/cyh1998/MusicDownloader/tree/dev-GUI)
