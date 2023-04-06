## 一、文件下载
使用 `requests` 模块实现文件的下载
```
import requests

# url:文件下载的网络地址
# dst:下载路径以及文件名
response = requests.get(url)
if response.ok:
    with(open(dst, 'wb')) as f:
        f.write(response.content)
```
当下载的文件较大时，可以使用分段下载来实现，避免文件过大而卡死
```
import requests

# url:文件下载的网络地址
# dst:下载路径以及文件名
response = requests.get(url, stream=True)
if response.ok:
    with(open(dst, 'wb')) as f:
        for chunk in response.iter_content(chunk_size=1024):
            if chunk:
                f.write(chunk)
```
即在 `requests.get()` 中设置参数 `stream` 为 `True`，开启分段下载。`chunk_size` 为每次分段的大小  

## 二、断点续传
思路：获取下载文件的总大小，再获取已经下载部分文件的大小，从而设置请求头，请求下载文件剩下的部分
```
import requests
from urllib.request import urlopen

# url:文件下载的网络地址
# dst:下载路径以及文件名
file_size = int(urlopen(url).info().get('Content-Length', -1)) #获取下载文件的总大小

if os.path.exists(dst): 
    first_byte = os.path.getsize(dst) #获取已经下载部分文件的大小
else:
    first_byte = 0

header = {"Range": "bytes=%s-%s" % (first_byte, file_size)} #设置下载头

req = requests.get(url, headers=header, stream=True) #请求下载文件剩下的部分
with(open(dst, 'ab')) as f:
    for chunk in req.iter_content(chunk_size=1024):
        if chunk:
            f.write(chunk)
```

## 三、进度条显示
使用 `tqdm` 模块实现显示进度条  
安装
```
pip install tqdm
```

简单的使用
```
from tqdm import tqdm
import time

pbar = tqdm(total=100)
for i in range(10):
    time.sleep(0.1)
    pbar.update(10)
pbar.close()
```

### 实现断点续传以及下载进度条显示的整体代码
```
# -*- coding: utf-8 -*-
import os
import requests
from urllib.request import urlopen
from tqdm import tqdm

def download_from_url(url, dst):
    response = requests.get(url)
    if response.ok:
        print(dst.split('\\')[-1] + ' 获取成功')
        file_size = int(urlopen(url).info().get('Content-Length', -1)) #获取下载文件大小
        if os.path.exists(dst): #判断下载文件是否存在
            first_byte = os.path.getsize(dst) #获取已经下载部分的文件大小
        else:
            first_byte = 0
        if first_byte >= file_size:
            return file_size
        header = {"Range": "bytes=%s-%s" % (first_byte, file_size)} #设置下载头
        pbar = tqdm( #设置显示进度条
            total=file_size, initial=0,
            unit='B', unit_scale=True, desc=dst.split('\\')[-1])
        req = requests.get(url, headers=header, stream=True) #分段下载
        with(open(dst, 'ab')) as f:
            for chunk in req.iter_content(chunk_size=1024):
                if chunk:
                    f.write(chunk)
                    pbar.update(1024) #更新进度条
        pbar.close()
        return file_size
    else:
        print(dst.split('\\')[-1] + ' 获取失败')
        return 0


if __name__ == '__main__':

    # ...数据处理...

    for apk in apk_url:
        url = apk['download_url']
        path = fail_path + '\\' + apk['name'] + '.apk'
        # 下载apk
        if download_from_url(url,path):
            print(apk['name'] + '下载成功')
        else:
            print(apk['name'] + '下载失败')
```
对于 `tdqm` 中的参数：  
`total` ：期待的迭代数，默认为None，则尽可能的迭代下去。  
`initial` ：初始计数器值，默认值为0。  
`unit` ：用来定义每个迭代单元的字符串。默认为"it"，表示每个迭代；在下载或解压时，设为"B"，代表每个“块”。  
`unit_scale` ：设置为1或者True，如果迭代数过大，它就会自动在后面加上M、k等字符来表示迭代进度等。默认为False。比如，在下载进度条的例子中，如果为False，数据大小是按照字节显示，设为True之后转换为Kb、Mb。  
`desc` ：进度条的前缀。  

更多 `tdqm` 的参数可以参考 [https://www.cnblogs.com/wanghui-garcia/p/11514579.html](https://www.cnblogs.com/wanghui-garcia/p/11514579.html)








