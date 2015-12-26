title: 在Mac上简易设置Oh-My-Zsh的BulletTrain主题
date: 2015-12-26 23:00:30
tags: Oh-My-Zsh
---
# 起因

近期有了一台新的MacBook，自然少不了基本的装机过程，其中Homebrew和Oh-My-Zsh作为生产力工具算是必装的软件。

不过Oh-My-Zsh的默认主题看久了仍然还是觉得有些枯燥，动了想要更换主题的念头，于是有了后面的步骤。

# 步骤

## 安装Vim

常规的Homebrew安装和Oh-My-Zsh不需多言，而这次选定的Bullet Train主题却需要Vim的Powerline插件支持。而Powerline需要正常显示则需要安装[已Patch了特殊字符的字体][1]，如果使用iTerm2还需要设定显示字体为已Patch的字体……

由于Powerline需要Python支持，那么安装vim可以按照如下方式进行安装：

```
brew install python
brew install vim --with-python --with-ruby --with-perl
brew install macvim --env-std --override-system-vim
```

## 安装已Patch的字体

针对一些譬如手写对勾和叉以及git分支之类的符号，需要安装已Patch的字体，按照说明安装即可。

由于个人比较喜欢Monaco字体，对Monaco字体进行了Patch。

## 安装powerline

由于一直使用[Vundler][2]管理vim插件，通过Vundler安装这一插件十分方便，在`~/.vimrc`中添加：

```
Bundle 'Lokaltog/powerline', {'rtp': 'powerline/bindings/vim/'}
```

为了启用这一插件的美化效果，则需要在`~/.vimrc`中添加如下配置：

```
set laststatus=2
let g:Powerline_symbols = 'fancy'
set guifont=Monaco\ for\ Powerline
```

之后在vim中执行`:BundleInstall`即可，还没有使用Vundler的可以一试。

具体参考了[setup-vim-powerline-and-iterm2-on-mac-os-x][3]

## 设置iTerm2

iTerm2个人一直在使用，需要将显示字体设定为已Patch的字体。

![iterm2-set-display-patch-font][4]

vim能够显示三角、分支等特殊字符即说明已然设置完毕。

![fancy-vim][5]

## 安装Bullet Train for oh-my-zsh

Oh-My-Zsh的主题安装一直都是很简便，直接wget对应的插件到`~/.oh-my-zsh/themes`即可，启用则是在`~/.zshrc`中设定`ZSH_THEME="bullet-train"`即可。

![default-theme-effect][6]

## 定制显示颜色

默认的显示颜色感觉略微的不和谐，好在这一主题可以通过在`~/.zshrc`中设置颜色等属性完成设定。

首先这里要保证iTerm2使用的是`xterm-256color`终端方式（在iTerm2的`Preference->Profiles->Terminal`中可以查看），后续显示使用的颜色会设定成这256色中一种。

定制颜色主要分为前景色，即字体的显示颜色，以及背景色。

这一主题的箭头标部分主要显示的是时间、目录、当前目录git信息，所以主要设定的是这三个部分的颜色以及参数：

```
BULLETTRAIN_TIME_BG=105
BULLETTRAIN_DIR_BG=026
BULLETTRAIN_GIT_BG=231
BULLETTRAIN_GIT_DIRTY=" %F{red}✘%F{black}"
BULLETTRAIN_GIT_CLEAN=" %F{green}✔%F{black}"
BULLETTRAIN_GIT_UNTRACKED=" %F{208}✭"
```

阅读主题源码后了解到对于颜色直接对属性值赋予[256色对应的颜色值][7]即可。

颜色与数值的对应关系可以参考下图：

![256-colors][8]

# 最后

历经这一过程，终于完成了一些简单的修改，工作的时候可能也会更愉悦吧😊

![theme-effect][9]

以上。

[1]: https://github.com/Lokaltog/powerline-fonts
[2]: https://github.com/VundleVim/Vundle.vim
[3]: https://coderwall.com/p/yiot4q/setup-vim-powerline-and-iterm2-on-mac-os-x
[4]: http://blog.wislay.com/wp-content/uploads/2015/12/oh-my-zsh-with-bullet-train-iterm2.png
[5]: http://blog.wislay.com/wp-content/uploads/2015/12/oh-my-zsh-with-bullet-train-vim.png
[6]: http://blog.wislay.com/wp-content/uploads/2015/12/oh-my-zsh-with-bullet-train-default.png
[7]: https://en.wikipedia.org/wiki/File:Xterm_256color_chart.svg
[8]: http://blog.wislay.com/wp-content/uploads/2015/12/Xterm_256color_chart.png
[9]: http://blog.wislay.com/wp-content/uploads/2015/12/oh-my-zsh-with-bullet-train-sample.png
