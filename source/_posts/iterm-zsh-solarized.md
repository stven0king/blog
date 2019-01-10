---
title: 【可视化教程】iTerm2+oh-my-zsh+solarized配色方案
date: 2018-03-07 18:16:00
tags: [Terminal]
categories: MAC
description: "【可视化教程】iTerm2+oh-my-zsh+solarized配色方案,自己Mac的terminal配色。"
---

# 【可视化教程】iTerm2+oh-my-zsh+solarized配色方案

## 自己Mac的terminal配色

<center>![terminal配色](http://upload-images.jianshu.io/upload_images/1319879-0571ea9cc6e1eae3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>


## 修改Mac的terminal配色

```sh
#enables colorin the terminal bash shell
export export CLICOLOR=1
#sets up thecolor scheme for list
export export LSCOLORS=gxfxcxdxbxegedabagacad
#sets up theprompt color (currently a green similar to linux terminal)
#export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;36m\]\w\[\033[00m\]\$ '
#enables colorfor iTerm 
export TERM=xterm-color
```
其中`export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;36m\]\w\[\033[00m\]\$ '`是注释掉的。

## 开工前的准备
- [iTerm](https://link.jianshu.com/?t=http://www.iterm2.com/) Mac最好用的终端，点击链接下载最新的版本。
- [Solarized Dark](https://link.jianshu.com/?t=http://ethanschoonover.com/solarized) 现在配色资源。
- [oh-my-zsh](https://link.jianshu.com/?t=https://github.com/robbyrussell/oh-my-zsh)可使用`curl -L https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh | sh`进行安装
- [Monaco Powerline](https://link.jianshu.com/?t=https://github.com/mneorr/powerline-fonts/blob/bfcb152306902c09b62be6e4a5eec7763e46d62d/Monaco/Monaco%20for%20Powerline.otf)解决部分字符的乱码问题。

## iTerm

下载&解压，移动到`Application`文件中。

## Solarized Dark

下载解压`Solarized Dark`，点击目录`solarized\iterm2-colors-solarized`的`Solarized Dark.itermcolors`和 `Solarized Light.itermcolors`进行安装。

<center>![iTerm的配色方案.jpg](http://upload-images.jianshu.io/upload_images/1319879-c42c443703510c22.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>


## oh-my-zsh安装agnoster主题

通过命令行安装：`curl -L https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh | sh`。
卸载oh-my-zsh命令：`uninstall_oh_my_zsh`。

```
vim ~/.zshrc    //进入.zshrc文件，将ZSH_THEME后面字段改为agnoster
```
<center>![安装agnoster主题.jpg](http://upload-images.jianshu.io/upload_images/1319879-6b73c3e18c5f5d1e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>




## Monaco Powerline

下载并安装。
<center>![ITerm修改字体.jpg](http://upload-images.jianshu.io/upload_images/1319879-470e09ba88519562.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>


## 设置语法高亮 -- [zsh-syntax-highlighting](https://link.jianshu.com/?t=https://github.com/zsh-users/zsh-syntax-highlighting)

直接使用homebrew安装zsh-syntax-highlighting插件
```
brew install zsh-syntax-highlighting
```
然后在根目录下.zshrc中插入下面内容：
```
source /usr/local/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

## 隐藏用户名信息
一般终端每一行前都会有`xxx@xxxdeMacbook-Pro:`我们可以将其隐藏掉。
进入oh-my-zsh的agnoster主题，编辑agnoster.zsh-theme文件：
`vim ~/.oh-my-zsh/themes/agnoster.zsh-theme`

<center>![隐藏用户信息.jpg](http://upload-images.jianshu.io/upload_images/1319879-635328c0ce8a09f0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>


参考文章：

[最漂亮iTerm2+oh-my-zsh配色](https://www.jianshu.com/p/246b844f4449)

[mac－改造你的terminal](https://www.jianshu.com/p/bb1c97269b11)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>