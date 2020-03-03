---
layout: post
title: 配置 vim + ctags + taglist 打造开发环境
date: 2020-03-03 00:00:00 +0300
stickie: false
tags: [Linux, vim, ctags, taglist] # add tag
---

# Ctags 安装配置

## mac OS 下

```bash
brew install ctags
```

## CentOS 下

```bash
yum install ctags
```

## 解决 mac OS 下 ctags illegal option -- R 的问题

产生这个问题的主要原因是，mac OS 本身会自己安装一个 ctags，而下载安装完后直接执行

```bash
ctags -R *
```

实际上是调用的默认安装的 ctags，而不是我们使用 brew 新安装的 ctags，因此不支持 -R 参数。

解决方案就是修改 .zshrc 文件来指定使用我们新安装的 ctags，在 .zshrc 文件中添加如下配置

```txt
alias ctags="`brew --prefix`/bin/ctags"
```

然后

```bash
source .zshrc
```

即可解决上述的问题



# Taglist 安装配置

## 下载 Taglist

在 https://www.vim.org/scripts/script.php?script_id=273 网站下载 taglist 压缩包。

## 解压

解压缩到 $HOME/.vim 或是 $HOME/vimfiles 或是 $VIM/vimfiles 目录下。解压后，在相应的目录下会多出两个文件

- plugin/taglist.vim
- doc/taglist.txt

## 处理 help 文件

进入到对应目录的 doc 目录下，打开 vim ，运行 ":helptags ." 来对 help 文件进行处理。


## 编辑 .vimrc 文件

在 $HOME/.vimrc 文件中添加如下基本配置

```txt
let Tlist_Show_One_File=1
let Tlist_Exit_OnlyWindow=1   " 如果taglist 窗口是最后一个窗口则关闭 vim
let Tlist_Ctags_Cmd="/usr/bin/ctags"   " 指向 ctags 可执行文件所在的路径
```

然后重启 vim，执行 :Tlist 即可打开和关闭 taglist 窗口。

通过 ctrl-ww 快捷键来在 taglist 窗口和编辑窗口之间进行切换。


# 配置 vim 使用 Dracula 主题

因为我使用 vim + vundle 来管理插件，因此只需要在 .vimrc 文件中添加

```txt
Plugin 'dracula/vim', { 'name': 'dracula' }
```

然后启动 vim 执行

```txt
:PluginInstall
```

即可。
