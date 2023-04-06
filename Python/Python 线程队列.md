### 利用Python中的 `threading` 模块实现多线程； `queue` 模块实现任务队列  
Python 的 `queue` 模块中提供了同步的、线程安全的队列类，包括FIFO(先入先出)队列Queue，LIFO(后入先出)队列LifoQueue，和优先级队列PriorityQueue。    
对于有大量重复的任务，可以将所有任务放到队列中，然后开启多个线程，从任务队列中取任务，直至队列中任务全部完成。   
这里通过继承 `threading.Thread` 定义子类创建多线程，重写 `init` 方法和 `run` 方法，demo如下：  
```
# -*- coding:utf-8 -*-
import threading
import queue

# 自定义定义子类
class automatic_cap_work(threading.Thread):
    def __init__(self,name,queue):
        threading.Thread.__init__(self)
        self.name = name
        self.queue = queue
        self.start()

    def run(self): # 把要执行的代码写到run函数里面 线程在创建后会运行run函数
        # 一直执行下一个任务
        while True:
            # 如果队列为空，则退出线程
            if self.queue.empty():
                break
            # 获取队列中的信息
            apk_info = self.queue.get()
            # 运行需要执行的函数
            # ...
            # 任务完成
            self.queue.task_done()

# 任务队列
apk_queue = queue.Queue()
# 加入100个任务队列
for i in range(100):
    apk_queue.put(i)
# 开N个线程
for i in range(5):
    threadName = 'Thread' + str(i)
    Worker(threadName, queue)
# 所有线程执行完毕后关闭
apk_queue.join()
```