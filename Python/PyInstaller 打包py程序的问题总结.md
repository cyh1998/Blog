## 概述
最近用PyInstaller实现了将py程序打包为exe可执行文件，这里总结一下打包过程中遇到的问题和解决方法。网上有关PyInstaller安装和使用的教程很多，这里不再赘述。

## 问题与解决方法
**一、找不到文件**  
```
FileNotFoundError: [Errno 2] No such file or directory: 'file_path'
```
以上报错，表示某个文件没有找到，通过报错信息提示的 `file_path` 找到缺少的文件，将其拷贝到 `file_path` 对应的目录即可。  
**注：** 一般情况下，缺少的文件都会在py环境安装路径对应的 `Lib\site-packages` 中找到，这里推荐使用 `Everything` 来快速找到文件。

**二、找不到模块**  
```
ImportError: No module named 'module_name'
```
解决方法如下：  
1. 升级PyInstaller版本  
一般情况下，PyInstaller会帮我们将程序所引用的所有模块自动打包到exe可执行文件所在的目录，如果打包后报错提示找不到模块，可以尝试升级PyInstaller版本。

2. 手动引入模块  
使用 `--hidden-import` 命令添加模块，即 `pyinstaller your_script_path --hidden-import module_name` 

3. 直接拷贝模块  
如果以上的方法都无法解决文件，可以参考问题一的解决方法，用 `Everything` 找到缺少的模块，直接拷贝到exe可执行文件所在的目录。需要注意的是，缺少的模块可能是文件夹也可能是单一文件。例如 `textwrap3.py`、`xarray` 等等

## 补充
上文中，我们提到使用 `--hidden-import` 来手动引入模块，但是当需要手动引入的模块很多时，会在PyInstaller命令后面跟上非常多的 `--hidden-import xxx`。很容易遗漏或者写错。  
我们可以先不添加引入模块，直接执行打包  
```
pyinstaller your_script_path
```
在程序目录下会生成一个与程序同名的 `.spec` 文件，我们修改 `.spec` 文件中 `hiddenimports` 的内容，添加需要手动引入的模块，如下：
```
# ...
a = Analysis(['your_script_path'],
             pathex=[''],
             binaries=[],
             datas=[],
             hiddenimports=['module_name1','module_name2','module_name3', ...],
             hookspath=[],
             hooksconfig={},
             runtime_hooks=[],
             excludes=[],
             win_no_prefer_redirects=False,
             win_private_assemblies=False,
             cipher=block_cipher,
             noarchive=False)
# ...
```
最后执行 `.spec` 文件  
```shell
pyinstaller your_script_path.spec
```
这样可以清楚的看到手动引入的模块，不用每次添加大量的 `--hidden-import` 。当然，重新打包程序则会覆盖 `.spec` 文件，记得保留和替换。
