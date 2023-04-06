## 一、自动解包
### 1.apktool的安装何使用
安装  
<https://ibotpeaches.github.io/Apktool/install/>  

解包命令
```
apktool d name.apk
```
说明：需要java环境；apktool需要在环境变量中

### 2.Python调用cmd
有两种方法：`os.system()` 和 `os.popen()`  
`os.system()` 的返回值是脚本的退出状态码，0(成功)，1(失败)
```
import os

cmd = 'ipconfig'
res = os.system(cmd)
print(res)
```
`os.popen()` 返回脚本执行的输出内容作为返回值
```
import os

s = "ipconfig"
res = os.popen(s)
print(res.read()) #输出返回的字符串
```

## 二、检索xml文件
### 获取当前目录下的所有文件
两个方法 `os.walk()` 和 `os.listdir()`  
`os.walk()` 返回三个值，分别是 当前目录路径、当前路径下所有子目录、当前路径下所有非目录子文件
```
import os

for root, dirs, files in os.walk(filePath):  
    # root 当前目录路径  
    # dirs 当前路径下所有子目录  
    # files 当前路径下所有非目录子文件
```
`os.listdir()` 返回指定的文件夹包含的文件或文件夹的名字的列表
```
import os

for file in os.listdir(path):
```

###  =>不递归遍历文件目录
```
import os

def readfile(path):
    files = os.listdir(path)
    file_list = []
    for file in files:  # 遍历文件夹
        if not os.path.isdir(os.path.join(path,file)): #判断路径是否为目录
            file_list.append(path + '\\' + file)
    return file_list
```
注：`os.path.isdir()` 的参数必须为绝对路径，否注判断的结果都会是 `false`。所以需要用 `os.path.join(path,file)` 做一下地址拼接，`os.path.isfile()` 同理！！！

### 读取xml文件
```
 #-*-coding:utf-8-*-
import xml.etree.ElementTree as ET

#读取xml文件
def readxml(filepath):
    result_list = []
    root = ET.parse(filepath).getroot() #获取文件定位到根节点
    walkData(root,result_list)
    return result_list

#遍历xml所有的节点
def walkData(root_node,result_list):
    for root_in in root_node.attrib.values(): #获取每个节点的attrib属性的值
        result_list.append(root_in)

    #遍历每个子节点
    children_node = root_node.getchildren()
    if len(children_node) == 0:
        return
    for child in children_node:
        walkData(child,result_list)
    return
```
`ET.parse()` ：载入数据  
`.getroot()` ：获取根节点  
`.getchildren()` ：获取子节点  
`.attrib` ：元素的属性字典  

### 检索字符串
这一步就很简单了，在获取到xml中各个属性值之后，判断下是否存在某个字符串
```
#搜索关键字符
def findstr(list_data,find_str):
    for i in list_data:
        if i.find(find_str) != -1:
            return True
    return False
```