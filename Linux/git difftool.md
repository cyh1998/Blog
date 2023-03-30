#### 一、vimdiff使用
Vim提供的diff模式可以比较文件差异，即vimdiff。
```
$ vimdiff FILE_LEFT FILE_RIGHT
$ vim -d FILE_LEFT FILE_RIGHT
```

#### 二、git difftool使用vimdiff
**2.1 临时使用vimdiff**  
```
$ git difftool --extcmd vimdiff FILE_NAME
```
**2.2 默认使用vimdiff**  
```
$ git config --global diff.tool vimdiff
```
**2.3 取消二次提示**  
每次使用`git difftool`时会有二次提示，如下设置可以取消
```
$ git config --global difftool.prompt false
```
**2.4 退出整个对比**  
```
$ git config --global difftool.trustExitCode true
```
`:qa`可以退出当前文件对比，`:cq`可以退出整个文件对比

**2.5 常用命令**  
`] c`：跳转到下一个diff处  
`[ c`：跳转到上一个diff处  
`zo`：打开折叠代码  
`zc`：重新折叠代码  
`dp`：将当前差异从当前文件复制到另一个文件中  
`do`：将当前差异从另一个文件复制到当前文件中  
