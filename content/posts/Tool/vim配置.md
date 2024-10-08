---
author: "李昌"
title: "vim配置"
date: "2021-12-29"
tags: ["vim", "env"]
categories: ["Tool"]
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

" Shorthand notation; fetches https://github.com/junegunn/vim-easy-align Plug 'junegunn/vim-easy-align'
" Any valid git URL is allowed
Plug 'junegunn/vim-github-dashboard'


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

" auto complete
Plug 'Valloric/YouCompleteMe'

" auto complete ()[]{}
Plug 'tmsvg/pear-tree'

Plug 'preservim/nerdtree'

Plug 'vim-airline/vim-airline'

Plug 'vim-airline/vim-airline-themes'

Plug 'preservim/tagbar'

Plug 'rust-lang/rust.vim'

Plug 'maxmx03/dracula.nvim', { 'branch': 'vim' }

Plug 'junegunn/fzf', { 'do': { -> fzf#install() } }

Plug 'junegunn/fzf.vim'


Plug 'NLKNguyen/papercolor-theme'

" Initialize plugin system
call plug#end()

" 设置行号
set nu

" 设置tab
set ts=4
%retab!

" 设置YouCompleteMe不需要预览
set completeopt-=preview

if !isdirectory(&undodir)
                call mkdir(&undodir, 'p', 0700)
endif

set nobackup
set undodir=~/.vim/undodir

" 设置ctrl+n打开目录
map <C-n> :NERDTreeToggle<CR>

" 设置F8打开tag bar
nmap <F8> :TagbarToggle<CR>

" 设置搜索高亮
"set nohlsearch

" 停止搜索高亮的键映
nnoremap <silent> <BS> <BS>:nohlsearch<CR>

set backspace=2


"设置pear_tree自动补全的括号可见
let g:pear_tree_repeatable_expand=0

" 设置主题
set background=dark
colorscheme PaperColor

let g:go_imports_autosave = 1
let g:go_fmt_autosave = 1
let g:go_imports_mode = 'gopls'
let g:go_fmt_command = 'gopls'

let g:ycm_key_list_select_completion = ['<Enter>', '<C-n>']

set shiftwidth=4
set expandtab
set tabstop=4

map <C-\> :tab split<CR>:exec("tag ".expand("<cword>"))<CR>
map <A-]> :vsp <CR>:exec("tag ".expand("<cword>"))<CR>


" Alt-right/left to navigate forward/backward in the tags stack
map <C-Left> <C-T>
map <C-Right> <C-]>
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
