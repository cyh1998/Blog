## 前  言
工作之余我自己会剪一些游戏GMV(game music video)，因此经常需要去找一些音频素材。而网盘资源里的素材文件，大部分都是英文标题，使用起来很不方便。所以我就做了一个一键批量翻译文件名的工具。

## 实  现
主要是调用了百度翻译API，支持将指定目录下的所有音频文件标题一键翻译成中文。功能也比较简单，这里就不再赘述，有兴趣的可以访问 [GitHub源码](https://github.com/cyh1998/FileTrans)。  
百度翻译API相关内容可以访问 [官网链接](http://api.fanyi.baidu.com/product/11)。

## 总  结
这里总结下开发遇到的一些问题和代码优化  
**1. `exit` 未定义**  
代码里使用了 `exit()` 来退出程序，打包为 `exe` 执行后，会报错：
```
NameError: name 'exit' is not defined
```
*解决：* 将 `exit()` 改为 `sys.exit()` 即可。

**2. 检查字符串为空**  
代码里想要判断字符串是否为空或全空格，所以一开始实现的代码：
```
def check_string(str):
    return len(str) == 0 or str.isspace()
```
看着比较臃肿，改进如下：
```
def check_string(str):
    return not str.strip()
```
用 `strip()` 去除空格后，直接返回判断即可。

**3. 使用 `f-strings`**  
在调用API时，需要拼接url地址，例如：
```
myurl = api_url + '?appid=' + appid + '&q=' + urllib.parse.quote(q) + '&from=' + fromLang + '&to=' + toLang + '&salt=' + str(salt) + '&sign=' + sign
```
可以使用 `f-strings` 风格去拼接字符串，修改如下：
```
myurl = f"{api_url}?appid={appid}&q={urllib.parse.quote(q)}&from={fromLang}&to={toLang}&salt={str(salt)}&sign={sign}"
```
