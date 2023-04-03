## 概  述
以前一直是在win下开发，最近也是开始尝试在Linux下写代码，整理了下 `vim` 的相关配置，能让 `vim` 使用的舒服一些。

## 实  现
`vim` 的相关设置在 `~/.vimrc`，文件不存在的话，可以自行创建并添加需要的设置。
**一、中文设置**  
如果使用 `vim` 出现中文乱码的情况，添加如下设置
```
" 中文设置                                                            
set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936
set termencoding=utf-8
set encoding=utf-8
```
保存退出后，重新打开文件即可。

**二、个性设置**  
`vim` 可以添加很多个性化设置，这里列下我自己使用的
```
set wrap            " 长行自动折行
set nocompatible    " 关闭vi兼容
set autoindent      " 自动对齐
set smartindent     " 智能对齐方式
set ai!             " 自动缩进  
set cindent         " 以c/c++的模式缩进
set number          " 显示行号
set cursorline      " 突出显示当前行
set tabstop=4       " 设置Tab长度
set softtabstop=4   " 设置退格键长度
set shiftwidth=4    " 设置当行之间交错时使用4个空格
set showmatch       " 括号匹配
syntax on           " 语法高亮

" 括号自动补全
inoremap ( ()<ESC>i
inoremap [ []<ESC>i
inoremap { {<CR>}<ESC>kA<CR>
```

**三、配色设置**  
`vim` 支持不同的显示配色风格，默认支持的配色风格可以在 `/usr/share/vim/vimXX/colors` ( `/vimXX` 对应 `vim` 的版本)目录下。`.vim` 后缀的文件就是配色文件。  
我们可以在vim模式下，使用如下命令来临时使用配色：
```
:colorscheme default
```
也可以在 `.vimrc` 中添加如下设置来全局使用配色：
```
colorscheme default
```
`default` 可以替换为任意已有配色的文件名。  
此外，我们还可以使用其他自定义配色，例如 `molokai`、`gruvbox` 等等。  
以 `molokai` 为例，先下载好配色文件(这里贴下 [github地址](https://github.com/tomasr/molokai))，复制到上文提到的目录下即可通过 `colorscheme` 来设置。

*说明：*  
1. 一些配色支持深色/浅色模式，只需将背景设置为适当的值：
```
set background=dark    " Setting dark mode
set background=light   " Setting light mode
```
2. 开启真彩色设置
```
set termguicolors
```

**四、代码补全**  
使用 `vim` 来写C++的时候，可以使用 `omnicppcomplete` 来实现代码补全。具体实现如下：  
*1. `ctags` 安装*  
我们需要通过 `ctags` 来生成C++提示文件，安装命令：
```
sudo apt-get install ctags
```

*2. 生成提示文件*  
这里需要cpp源码来生成，vim官网有提供针对 `ctags` 修改过的源码下载以及生成文档，[地址](https://www.vim.org/scripts/script.php?script_id=2358)。  
下载解压后，执行
```
ctags -R --c++-kinds=+pl --fields=+iaS --extra=+q --language-force=C++ ./cpp_src
```
最后的 `./cpp_src` 就是源码解压后的路径，具体的参数说明可以参考官方文档。  
执行后，在当前目录下会生成一个名为 `tags` 的文件，后面会用到它。

*3. `omnicppcomplete` 安装*  
在vim官网下载 `omnicppcomplete-0.41.zip`，[下载地址](https://www.vim.org/scripts/script.php?script_id=1520)。解压到 `~/.vim` 目录下，如果没有 `.vim` 文件夹就自行创建一下。

*4. `.vimrc` 文件设置*  
在 `~/.vim` 目录下创建一个 `tags` 目录并将【*2*】生成的文件移动到 `~/.vim/tags` 目录下(这里建议改名为cpp，方便识别)
```
mv tags ~/.vim/tags/cpp
```
在 `~/.vimrc` 文件中添加如下设置：
```
set nocp
filetype plugin indent on

" configure tags - add additional tags here or comment out not-used ones
set tags+=~/.vim/tags/cpp
" build tags of your own project with Ctrl-F12
map <C-F12> :!ctags -R --sort=yes --c++-kinds=+pl --fields=+iaS --extra=+q .<CR>

" OmniCppComplete
let OmniCpp_NamespaceSearch = 1 
let OmniCpp_GlobalScopeSearch = 1 
let OmniCpp_ShowAccess = 1 
let OmniCpp_ShowPrototypeInAbbr = 1 " show function parameters
let OmniCpp_MayCompleteDot = 1 " autocomplete after .
let OmniCpp_MayCompleteArrow = 1 " autocomplete after ->
let OmniCpp_MayCompleteScope = 1 " autocomplete after ::
let OmniCpp_DefaultNamespaces = ["std", "_GLIBCXX_STD"]
" automatically open and close the popup menu / preview window
au CursorMovedI,InsertLeave * if pumvisible() == 0|silent! pclose|endif
set completeopt=menuone,menu,longest,preview
```
**说明：**  
- `set nocp` ：vim在不兼容模式下工作，即不兼容vi  
- `filetype plugin indent on` ：开启文件类型识别  
如果不设置，vim可能会无法识别cpp文件而无法提示补全  
- `set tags+=~/.vim/tags/cpp` ：设置提示文件路径  
也可以是其他路径，只要确保和提交文件所在目录一致即可  
- `map <C-F12> :!ctags -R --sort=yes --c++-kinds=+pl --fields=+iaS --extra=+q .<CR>` ：
正常补全只能提示C++原有的内容，需要想要提示自己写的相关类成员变量、成员函数等等。就需要添加此行设置。  
在vim模式下，通过`Ctrl+F12`就可以构建自己项目中的提示标签，快捷键也可以在`map <XX> `中自行设置。

剩下的就是 `OmniCppComplete` 自身的设置，这里不再赘述。
如果还有其他问题，可以参考官方 [WIKI](https://vim.fandom.com/wiki/C%2B%2B_code_completion)。

最后，使用vim编辑CPP文件即可提示代码补全。
