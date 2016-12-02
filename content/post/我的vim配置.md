+++
title = "我的vim配置"
date = "2016-05-15T05:25:32+08:00"
tags = ["vim"]
author = "xiaojiong"

+++

github地址:https://github.com/mixiaojiong/tool
有好多开源的vim配置，比如k-vim, spf13.
一直想定制一款自己的vim配置。本配置严重借鉴，k-vim和spf13的配置。
### .vimrc

```
let mapleader = ','
" Vundle
filetype off
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

" install Vundle bundles
if filereadable(expand("~/.vimrc.bundles"))
	source ~/.vimrc.bundles
endif

call vundle#end()

" 文件
filetype on
filetype indent on                  " 针对不同的文件类型采用不同的缩进格式
filetype plugin on                  " 允许插件
filetype plugin indent on           " 启动自动补全
set encoding=utf-8                  " 设置编码自动识别, 中文引号显示
set fileencoding=utf-8
set fileencodings=ucs-bom,utf-8,cp936,gb18030,big5,euc-jp,euc-kr,latin1
" 外观
set background=dark                 " 主题背景
colorscheme solarized               " 颜色主题
syntax on                           " 语法高亮
set laststatus=2
set showcmd                         " 显示命令
set number                          " 显示行号
set ruler                           " 打开状态栏标尺
set cursorline                      " 突出显示当前行
set showmatch                      " 高亮显示匹配的括号
set matchtime=3                    " 匹配括号高亮的时间(单位：0.1s
set scrolloff=10                    " 光标到屏幕底端保留 10 行 (光标位于屏底看着非常不舒服的)
set ruler                           " 显示右下角提示栏
set showmode                        " 显示左下角状态栏
set listchars=tab:▸\ ,trail:▫
set wildmenu                        " 使用菜单形式展示列表
set wildmode=longest,list,full
" 基本配置
set autoread                        " 自动加载文件变化
set wrap                            " 设定折行
set textwidth=78                    " N个字符折行
set history=1000                    " 历史记录1000
set nocompatible                    " 禁用VI相关
set backspace=indent,eol,start      " 启用backspace删除
set clipboard+=unnamed              " 共享剪切板
set showcmd                         " 输入的命令显示出来
set iskeyword+=_,$,@,%,#,-          " 带有例如以下符号的单词不要被换行切割
"set mouse=a                         " 鼠标可用
set confirm                         " 未保存或者仅仅读时，弹出确认
set nobackup                        " 不生成备份文件
set noswapfile                      " 不生成 swap 文件
set bufhidden=hide                  " 当 buffer 被丢弃的时候隐藏
set noerrorbells                    " 不发出警告声
" 搜索
set ignorecase smartcase            " 搜索忽略大写和小写，但有大写字母时仍保持大写和小写敏感
set hlsearch                        " 高亮搜索
set incsearch                       " 增量式搜索,逐字符高亮

" 对齐
set autoindent                      " 继承前一行的缩进方式。特别适用于多行凝视
set smartindent                     " 智能折叠
set smarttab                        " 开启新行时使用智能 tab 缩进
set expandtab                       " 空格取代Tab
set tabstop=4                       " Tab 键的宽度
set shiftwidth=4                    " 行交错宽度
set softtabstop=4                   " 敲入tab键时实际占有的列数

" 自动
autocmd! bufwritepost .vimrc source %
autocmd BufNewFile *.sh,*.py,*.php,*.c exec ":call AutoSetFileHead()"
autocmd FileType c,cpp,java,go,php,javascript,puppet,python,rust,twig,xml,yml,perl autocmd BufWritePre <buffer> :call <SID>StripTrailingWhitespaces()

" 修改默认键
" 同物理行上线直接跳
nnoremap k gk
nnoremap gk k
nnoremap j gj
nnoremap gj j
" F2 注释
map <F2>a :DoxAuthor<CR>
map <F2>f :Dox<CR>
map <F2>b :DoxBlock<CR>
map <F2>c O/** */<Left><Left><CR>
" 常用分屏
map <silent> <F3> <Esc>:He<CR>
map <silent> <F4> <Esc>:Ve<CR>
map <silent> <F5> <Esc>:Te<CR>
" F6 换行开关
nnoremap <F6> :set wrap! wrap?<CR>
" F7 复制粘贴网页代码前开启
set pastetoggle=<F7>
" F8 显示可打印字符开关
nnoremap <F8> :set list! list?<CR>
" F9 显示行号
nnoremap <F9> :set nu! nu?<CR>
" 分屏窗口移动
map <C-j> <C-W>j
map <C-k> <C-W>k
map <C-h> <C-W>h
map <C-l> <C-W>l
" 行首，行尾
noremap H ^
noremap L $
" 快速输入命令
nnoremap ; :
" 快速搜索
map <space> /
" 快速切换tabe
map <leader>gp gT
map <leader>gn gt
" normal模式下切换到确切的tab
noremap <leader>1 1gt
noremap <leader>2 2gt
noremap <leader>3 3gt
noremap <leader>4 4gt
noremap <leader>5 5gt
noremap <leader>6 6gt
noremap <leader>7 7gt
noremap <leader>8 8gt
noremap <leader>9 9gt
noremap <leader>0 :tablast<CR>

"==========================================
" FileType Settings  文件类型设置
"==========================================

" 具体编辑文件类型的一般设置，比如不要 tab 等
autocmd FileType python set tabstop=4 shiftwidth=4 expandtab ai
autocmd FileType ruby,javascript,html,css,xml set tabstop=4 shiftwidth=4 expandtab ai
autocmd BufRead,BufNewFile *.md,*.mkd,*.markdown set filetype=markdown.mkd
autocmd BufRead,BufNewFile *.part set filetype=html
au BufEnter ~/code/letv/sso/* setlocal tags+=~/code/letv/sso/tags

" 自定义函数 {
    " 定义函数AutoSetFileHead，自动插入文件头
    function! AutoSetFileHead()
        "如果文件类型为.sh文件
        if &filetype == 'sh'
            call setline(1, "\#!/bin/bash")
        endif

        "如果文件类型为python
        if &filetype == 'python'
            call setline(1, "\#!/usr/bin/env python")
            call append(1, "\# encoding: utf-8")
        endif

        "如果文件类型为.php文件
        if &filetype == 'php'
            call setline(1, "<?php")
        endif

        "如果文件类型为.c文件
        if &filetype == 'c'
            call setline(1, "#include <stdio.h>")
            call append(1, "\#include <stdlib.h>")
            call setline(3, "")
            call setline(4, "int main(void) {")
        endif

        normal G
        normal o
        normal o
    endfunc

    " 保存文件时删除多余空格
    fun! <SID>StripTrailingWhitespaces()
        let l = line(".")
        let c = col(".")
        %s/\s\+$//e
        call cursor(l, c)
    endfun
" }

```

### 插件.vimrc.bundles

```
Plugin 'VundleVim/Vundle.vim'
Plugin 'altercation/vim-colors-solarized'               " 颜色 
Plugin 'scrooloose/nerdtree'                            " 文件显示
Plugin 'scrooloose/syntastic'                           " 语法检查
Plugin 'Xuyuanp/nerdtree-git-plugin'                    " 文件显示git
Plugin 'jistr/vim-nerdtree-tabs'                        " 文件标签管理
Plugin 'terryma/vim-expand-region'                      " 快速范围选中
Plugin 'vim-scripts/DoxygenToolkit.vim'                 " 注释
Plugin 'vim-airline/vim-airline'                        " airline
Plugin 'vim-airline/vim-airline-themes'                 " airline主题
Plugin 'tpope/vim-fugitive'                             " git
Plugin 'airblade/vim-gitgutter'                         " git
Plugin 'junegunn/vim-easy-align'						" 快速对齐
Plugin 'fatih/vim-go'									" golang
Plugin 'evanmiller/nginx-vim-syntax'					" nginx配置文件
Plugin 'Lokaltog/vim-easymotion'						" 强大的跳转
Plugin 'majutsushi/tagbar'                              " 代码导航
Plugin 'vim-scripts/matchit.zip'                        " html标签匹配
Plugin 'valloric/MatchTagAlways'                        " html标签高亮匹配
Plugin 'maksimr/vim-jsbeautify'                         " 格式化代码

" 插件相关
" syntastic {
    " dependence
    let g:syntastic_error_symbol='>>'
    let g:syntastic_warning_symbol='>'
    let g:syntastic_check_on_open=1
    let g:syntastic_check_on_wq=0
    let g:syntastic_enable_highlighting=1

    "set statusline+=%#warningmsg#
    "set statusline+=%{SyntasticStatuslineFlag()}
    "set statusline+=%*

    "let g:syntastic_always_populate_loc_list = 1
    "let g:syntastic_auto_loc_list = 1
    "let g:syntastic_check_on_open = 1
    "let g:syntastic_check_on_wq = 0
" }

" DoxygenToolkit {
    let g:DoxygenToolkit_briefTag_pre="@synopsis  "
	let g:DoxygenToolkit_paramTag_pre="@param "
	let g:DoxygenToolkit_returnTag="@return   "
	let g:DoxygenToolkit_authorName="zhaorui, zhaorui3@le.com"
" }

" MatchTagAlways {
    let g:mta_use_matchparen_group = 1
    let g:mta_filetypes = {
        \ 'html' : 1,
        \ 'xhtml' : 1,
        \ 'xml' : 1,
        \ 'php' : 1,
        \ 'jinja' : 1,
        \}
" }

" jsbeautify {
	map <F6> :call JsBeautify()<cr>
	" or
	autocmd FileType javascript noremap <buffer>  <F6> :call JsBeautify()<cr>
	" for json
	autocmd FileType json noremap <buffer> <F6> :call JsonBeautify()<cr>
	" for jsx
	autocmd FileType jsx noremap <buffer> <F6> :call JsxBeautify()<cr>
	" for html
	autocmd FileType html noremap <buffer> <F6> :call HtmlBeautify()<cr>
	" for css or scss
	autocmd FileType css noremap <buffer> <F6> :call CSSBeautify()<cr>
" }

" easymotion {
    let g:EasyMotion_smartcase = 1
    "let g:EasyMotion_startofline = 0 " keep cursor colum when JK motion
    map <Leader><leader>h <Plug>(easymotion-linebackward)
    map <Leader><Leader>j <Plug>(easymotion-j)
    map <Leader><Leader>k <Plug>(easymotion-k)
    map <Leader><leader>l <Plug>(easymotion-lineforward)
    " 重复上一次操作, 类似repeat插件, 很强大
    map <Leader><leader>. <Plug>(easymotion-repeat)
" }

" easyalign {
    vmap <Leader>a <Plug>(EasyAlign)
    nmap <Leader>a <Plug>(EasyAlign)
    if !exists('g:easy_align_delimiters')
    let g:easy_align_delimiters = {}
    endif
    let g:easy_align_delimiters['#'] = { 'pattern': '#', 'ignore_groups': ['String'] }
" }
" expandregion {
    vmap v <Plug>(expand_region_expand)
    vmap V <Plug>(expand_region_shrink)
" }

" fugitive {
    " :Gdiff  :Gstatus :Gvsplit
    nnoremap <leader>ge :Gdiff<CR>
    " not ready to open
    " <leader>gb maps to :Gblame<CR>
    " <leader>gs maps to :Gstatus<CR>
    " <leader>gd maps to :Gdiff<CR>  和现有冲突
    " <leader>gl maps to :Glog<CR>
    " <leader>gc maps to :Gcommit<CR>
    " <leader>gp maps to :Git push<CR>
" }

" gitgutter {
    " 同git diff,实时展示文件中修改的行
    " 只是不喜欢除了行号多一列, 默认关闭,gs时打开
    let g:gitgutter_map_keys = 0
    let g:gitgutter_enabled = 0
    let g:gitgutter_highlight_lines = 1
    nnoremap <leader>gs :GitGutterToggle<CR>
" }

" nerdtree nerdtreetabs {
    map <Leader>e <plug>NERDTreeTabsToggle<CR>
    let NERDTreeHighlightCursorline = 1
    let NERDTreeIgnore=[ '\.pyc$', '\.pyo$', '\.obj$', '\.o$', '\.so$', '\.egg$', '^\.git$', '^\.svn$', '^\.hg$' ]
    " 只有一个窗口的时候默认关闭
    autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTreeType") && b:NERDTreeType == "primary") | q | end
    " s/v 分屏打开文件
    let g:NERDTreeMapOpenSplit = 's'
    let g:NERDTreeMapOpenVSplit = 'v'
    let g:nerdtree_tabs_open_on_gui_startup = 1
    "let g:nerdtree_tabs_open_on_console_startup = 1
    let g:nerdtree_tabs_smart_startup_focus = 2
    let g:nerdtree_tabs_autoclose = 1
" }

" airline {
    if !exists('g:airline_symbols')
        let g:airline_symbols = {}
    endif
	let g:airline_theme = 'solarized'
    let g:airline_left_sep = '▶'
    let g:airline_left_alt_sep = '❯'
    let g:airline_right_sep = '◀'
    let g:airline_right_alt_sep = '❮'
    let g:airline_symbols.linenr = '¶'
    let g:airline_symbols.branch = '⎇'
    " 是否打开tabline
    let g:airline#extensions#tabline#enabled = 1
" }

" solarized {
    let g:solarized_termtrans=1
    let g:solarized_contrast="normal"
    let g:solarized_visibility="normal"
    "let g:solarized_termcolors=256
" }

" TagBar {
    "autocmd VimEnter * TagbarToggle
	if isdirectory(expand("~/.vim/bundle/tagbar/"))
		nnoremap<leader>] :TagbarToggle<CR>
	endif
" }


```

### 效果图
![vim](http://ww4.sinaimg.cn/large/68faff51gw1f3xfxxzab4j21kw0y2wof.jpg)
