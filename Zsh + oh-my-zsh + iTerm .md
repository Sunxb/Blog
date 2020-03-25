---
title: Zsh + oh-my-zsh + iTerm 
date: 2016-05-21 17:50:46
tags: iOS
---

**提前声明这篇文章是我借鉴网上各位大神的博客操作然后写下来的** 


Zsh和bash一样，是一种Unix shell，但大多数Linux发行版本都默认使用bash shell。但Zsh有强大的自动补全参数、文件名、等功能和强大的自定义配置功能。

#### 下面的所有操作以mac 为例:
mac 预装了zsh ,但是很少有人直接切换过来使用此shell ,因为 zsh 的默认配置及其复杂繁琐,让人望而却步,直到有了oh-my-zsh这个开源项目,让zsh配置降到0门槛.而且它完全兼容 bash .


#### 使用curl自动安装

    curl -L https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh | sh

#### 手动安装：
1. 克隆这个项目到本地(前提是你得有装git)
>git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
2. 创建一个zsh的配置文件注意:如果你已经有一个~/.zshrc文件的话，建议你先做备份。使用以下命令
  >cp ~/.zshrc ~/.zshrc.orig

  然后开始创建zsh的配置文件
>cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc

3.  设置zsh为你的默认的shell
>chsh -s /bin/zsh

4.  重启并开始使用你的zsh (打开一个新的终端窗口便可…)

#### 接下来配置适合自己Zsh
1. 主题修改，我比较喜欢前面是$符号，所以选择了steeef这款主题
>$ vim ~/.zshrc
  
  配置文件里找到：
>ZSH_THEME="robbyrussell"

  修改为：
>ZSH_THEME="steeef"

  这里是官方提供的各种主题，有截图参考 [oh-my-zsh-themes](https://github.com/robbyrussell/oh-my-zsh/wiki/themes)

2. 插件的选择，支持git、brew、vi、osx等插件，具体请查看这里[oh-my-zsh-plugins](https://github.com/robbyrussell/oh-my-zsh/wiki/Plugins)

