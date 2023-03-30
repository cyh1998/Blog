#### 创建
```
import os

# path：文件的目录(绝对地址)
os.mkdir(path)
os.makedirs(path)  #创建多级目录
```
#### 删除
```
import os
import shutil

# path：文件的目录(绝对地址)
os.remove(path) #删除文件
os.removedirs(path) #删除文件夹，但是文件夹必须为空。
os.rmdir(path) #删除目录，必须是个空目录
shutil.rmtree(path) #删除非空目录
```
#### 移动
```
import shutil

# oldpath：文件的旧目录(绝对地址)
# newpath：文件的新目录(绝对地址)
shutil.move(oldpath,newpath) #移动文件
```
#### 重命名
```
import os

# oldname：旧文件名(绝对地址)
# newname：新文件名(绝对地址)
os.rename(oldname,newname) #重命名文件
```
#### 查看编码格式
```
import chardet

chardet.detect(str) #返回字符串的编码格式信息，str必须为字符串
```
#### =>遇到由于编码问题，无法获取`os.popen()`的返回值的解决办法：
```
import os

res = os.popen(cmd)
readstr = res._stream.buffer.read() #获取cmd返回结果
strcoding = chardet.detect(readstr) #查看编码格式

readstr.decode(encoding = strcoding['encoding']) #根据编码格式解码
```
可根据 `strcoding['encoding']` 的编码格式，来解码处理字符串