C++/qt项目中实现使用回调触发函数的机制，因为是公司项目，所以无法展示全部的代码，所以文章主要讲述回调函数的机制和实现。

##### 项目需求
项目内核(动态库)检测当前服务端的socket连接情况，当连接断开时，通知客户端(qt实现)弹出强制退出对话框。
需要使用回调函数的方式，在连接断开时，内核触发调用客户端弹出强制退出对话框的函数。

#### 基本概念和机制
首先要理解什么是回调函数，简单来讲，就是使用一个函数指针来调用函数，这就是回调函数。
通常，回调函数用于将调用者与被调用者分开，或者说用于层间协作，从而降低耦合度。回调函数写在上层(调用者)，低层(被调用者)通过一个函数指针来保存这个函数，在某个特殊事件的触发时，低层(被调用者)通过该函数指针调用上层(调用者)那个函数。

#### 实现步骤
(1)  在调用模块中定义一个回调函数，即特殊情况发生是需要执行的函数；  
(2)  在被调用模块中定义函数指针，并定义一个注册回调函数，用于将函数指针指向传递进来的函数；  
(3)  在调用模块中，调用注册回调函数，将需要执行的函数指针作为参数传递给注册回调函数；  
(4)  当特殊情况发生时，使用函数指针调用回调函数对事件进行处理。  

#### 具体实现和关键代码
**声明函数指针**  
在内核的`.h`文件中声明函数指针类型(函数指针类型要和客户端中需要执行的函数一致)，并在类中定义一个函数指针变量。因为客户端需要执行的函数是弹出对话框，所以没有函数参数，返回值为`void`。
```
typedef void (*CallBack)(); //声明函数指针类型

class Core{
public:
//......
private:
    CallBack m_callbackfun; //函数指针变量
}
```
**声明注册回调函数**  
在内核的同一`.h`文件中声明一个注册回调函数，参数为刚才声明的函数指针。
```
typedef void (*CallBack)(); //声明函数指针类型

class Core{
public:
    virtual void RegiseterExitCallback(CallBack cbfun); //注册回调函数
private:
    CallBack m_callbackfun; //函数指针变量
}
```
**定义注册回调函数**  
在内核的`.h`文件对应的`.cpp`文件中定义注册回调函数，将函数指针指向传递进来的函数
```
void Core::RegiseterExitCallback(CallBack cbfun) {
    m_callbackfun = cbfun;
}
```
**声明定义回调函数**  
在qt客户端，定义回调函数，并调用注册回调函数
```
//MainUi.h
class MainUi{
//......
public slots:
	void ShowExitPrompt(); //声明回调函数
}

//MainUi.cpp
//显示退出提示框
void MainUi::ShowExitPrompt()
{
	QDialog* ExitPromptDialog = qobject_cast<QDialog*>(m_ExitPromptUi->m_Widget);
	if(ExitPromptDialog) ExitPromptDialog->exec();
}

//main.cpp
int main(int argc, char *argv[]){
//......
RegiseterExitCallback(QApp_MVPExitEvent); //调用注册回调函数

}

//注册掉线回调事件
static void QApp_MVPExitEvent() {
	QMetaObject::invokeMethod(g_pQApp->m_pMainUi, "ShowExitPrompt"); //利用qt对象调用槽函数
}
```
**利用函数指针调用回调函数**  
在内核中，在检测到socket连接断开时，直接利用函数指针调用回调函数
```
if (recv_len <= 0){ //连接断开
    socket_close(m_Socket); //关闭socket连接
    m_Socket = INVALID_SOCKET;
    if(m_callbackfun) m_callbackfun(); //调用回调函数
    break;
}
```
最后还需要注意，由于内核生成为动态库，所以qt客户端是调用不到内核中的函数的。所以需要在声明函数时，用`__declspec(dllexport)`修饰，表明将此函数从动态库中导出，这样qt客户端就可以调用到注册回调函数！