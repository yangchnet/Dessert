---
author: "李昌"
title: "vim配置"
date: "2021-12-29"
tags: ["Linux", "vim"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

## 1. 安装插件系统
> 使用的是[vim-plug](https://github.com/junegunn/vim-plug)
```sh
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

## 2. 安装插件
打开`~/.vimrc`, 在其中写入：
```
" Specify a directory for plugins
" - For Neovim: stdpath('data') . '/plugged'
" - Avoid using standard Vim directory names like 'plugin'
call plug#begin('~/.vim/plugged')

" Make sure you use single quotes

" Shorthand notation; fetches https://github.com/junegunn/vim-easy-align
Plug 'junegunn/vim-easy-align'

" Any valid git URL is allowed
Plug 'https://github.com/junegunn/vim-github-dashboard.git'

" Multiple Plug commands can be written in a single line using | separators
Plug 'SirVer/ultisnips' | Plug 'honza/vim-snippets'

" On-demand loading
Plug 'scrooloose/nerdtree', { 'on':  'NERDTreeToggle' }
Plug 'tpope/vim-fireplace', { 'for': 'clojure' }

" Using a non-default branch
Plug 'rdnetto/YCM-Generator', { 'branch': 'stable' }

" Using a tagged release; wildcard allowed (requires git 1.9.2 or above)
Plug 'fatih/vim-go', { 'tag': '*' }

" Plugin options
Plug 'nsf/gocode', { 'tag': 'v.20150303', 'rtp': 'vim' }

" Plugin outside ~/.vim/plugged with post-update hook
Plug 'junegunn/fzf', { 'dir': '~/.fzf', 'do': './install --all' }

" Unmanaged plugin (manually installed and updated)
Plug '~/my-prototype-plugin'

" Initialize plugin system
call plug#end()
```

然后在命令模式下执行：`PlugInstall`  
等待一段时间，一些插件将会被安装到vim上

## 3. 使用主题
> 这里选择[molokai](https://github.com/tomasr/molokai)主题

下载主题文件
```sh
git clone https://github.com/tomasr/molokai.git

cd molokai && mv colors/molokai.vim ~/.vim/colors/
```

使用主题  
在`~/.vimrc`文件中写入：
```
:colorscheme molokai
```

重新进入vim，主题已经生效

## 4. 自动补全
> 这里使用的是[YouCompleteMe](https://github.com/ycm-core/YouCompleteMe)
在vim plug中添加插件
```
Plug 'Valloric/YouCompleteMe'
```

vim运行`:PlugInstall`

添加完成后还需要手动编译
```sh
cd ~/.vim/plugged/YouCompleteMe && python3 install.py --all --verbose
```

`--all`代表要安装所有语言的自动补全功能，如果你只需要某个语言的(如C语言)，可以使用`--clang`参数代替`--all`

使用`YouCompleteMe`时会跳出函数文档，可以设置不显示：
```
set completeopt-=preview
```

## 5. 其他配置
1. 括号自动补全
```
Plug 'tmsvg/pear-tree'
```

2. 显示行号
```
set nu
```

3. tab替换为4个空格
```
set ts=4
%retab!
```
