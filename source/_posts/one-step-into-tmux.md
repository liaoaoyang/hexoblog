title: 一些简单的tmux设置
date: 2016-02-16 22:47:00
tags: [tmux]
---

# 概述

如果使用Linux作为服务器的操作系统，通过ssh操作时，会出现一个困难的选择：是否需要打开n个终端窗口？

如果服务器可以直接ssh的话，那么通过复用会话的方式似乎还算是个好选择，或者编写一个expect登陆脚本完成自动登陆操作。

当然，如果有如下需求：

+ 回家之后继续在公司的工作
+ 防止偶发的网络断开引起的重新连接
+ 运行时同时使用监控软件查看运行状态

那么这时候就需要使用终端复用软件了。

# 问题

tmux这个终端复用软件的强大无需多说，众多复杂的设置似乎对我来说并没有必要。而在使用中，自己曾经遇到过这些问题：

+ 默认的前缀按键Ctrl+b比较难按
+ 部分管理的快捷键并不方便（如关闭window，`Ctrl+b` -> `Shift+7` -> `y`）或者并不形象（比如分割窗口）

# 需求

针对上面的问题，需求就变成了：

+ 能快速创建、切换pane（充分利用屏幕空间）
+ 快速切换window（可以直接通过组合数字键等方式切换）
+ 按键要便捷

细化一下，就变成了：

+ 能够通过数字键切换window
+ 能够顾通过方向键切换相邻window
+ 通过两次按键的组合键完成创建、切换、关闭window的操作
+ 通过两次按键的组合键完成创建、切换、关闭pane的操作
+ 通过-完成纵向切分window的操作
+ 通过\完成横向切分window的操作（\与|在一个按键上）
+ 快捷键前缀由`C-b`变为`C-x`

![tmux-multi-pane-effect][1]

# 配置

```
# use C-x instead of C-b
set -g prefix C-x
unbind C-b

# Start windows and panes at 1, not 0
set -g base-index 1
setw -g pane-base-index 1

# split window alt+\-
bind -n M-\ split-window -h
bind -n M-- split-window -v

# swith window by number
bind-key -n M-1 select-window -t 1
bind-key -n M-2 select-window -t 2
bind-key -n M-3 select-window -t 3
bind-key -n M-4 select-window -t 4
bind-key -n M-5 select-window -t 5
bind-key -n M-6 select-window -t 6
bind-key -n M-7 select-window -t 7
bind-key -n M-8 select-window -t 8
bind-key -n M-9 select-window -t 9

# switch window alt+arrow
bind -n M-Up    select-pane -U
bind -n M-Down  select-pane -D
bind -n M-Left  select-pane -L
bind -n M-Right select-pane -R

# close panel alt+k
bind -n M-k confirm-before kill-pane

# create window alt+t
bind -n M-t new-window

# Shift arrow to switch windows
bind -n S-Left  previous-window
bind -n S-Right next-window

# close window alt+e
# e means exit
bind -n M-e confirm-before kill-window

# refresh config
bind -n M-r source-file ~/.tmux.conf \; display "Configuration Reloaded!"

```

其中`M`表示`键盘上的option/alt键`，`S`表示`Shift键`。

如果不想增加关闭前的确认步骤，只需去掉指令中的`confirm-before`。

# 还有一步

使用iTerm2时，为了能够让`M`键（即键盘上的option/alt键）能够完成上述工作，还需要进行设定：

![tmux-iterm2-config][2]

键盘上的两个option/alt键只需使用一个即可。

[1]: https://blog.wislay.com/wp-content/uploads/2016/02/tmux-multi-pane-effect.png

[2]: https://blog.wislay.com/wp-content/uploads/2016/02/tmux-iterm2-config.png




