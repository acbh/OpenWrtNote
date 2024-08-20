**vscode remote 连接 openwrt失败, 清除`~/.ssh/known_hosts`旧的密钥信息后仍然报错**

**VS Code 的 Remote-SSH 插件连接到远程服务器时遇到了问题。具体问题可能是由于远程服务器上未安装 `bash`，或者 `bash` 不能正确运行。**

`待解决`

**vim删除所有行**

普通模式下`ggdG`

**查看文件夹总大小**

`du -sh openwrt/`

**git修改最近一次commit信息**

```shell
git commit --amend
git push --force
```

**linux 安装 deb 文件**

`sudo dpkg -i filename.deb`

**查看已安装的软件包的列表**

`dpkg -l | grep filename`

卸载对应的文件

`sudo dpkg --remove filename`

命令行打开sublime

`subl`

**openwrt无法使用rm命令**

`rm -rf /tmp/luci-*`

**clash配置**

跑路云

```bash
https://app.nloli.xyz/sub?target=clash&udp=true&config=https%3A%2F%2Fraw.githubusercontent.com%2FHynoR%2FACL4SSR%2Fmaster%2FClash%2Fconfig%2FACL4SSR_Online.ini&exclude=GAME&emoji=true&filename=Paoluz_Cat4SSR&new_name=true&url=https://rss.paoluz.xyz/link/FmmDlKwZW79aR6Aq?sub=1
```

**ssh连接报错解决方式**

```shell
错误信息
Please contact your system administrator.
Add correct host key in /root/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /root/.ssh/known_hosts:1
  remove with:
  ssh-keygen -f "/root/.ssh/known_hosts" -R "192.168.1.1"
ED25519 host key for 192.168.1.1 has changed and you have requested strict checking.
Host key verification failed.
```

原因分析：重新烧录固件之后`./ssh/known_hosts`文件要更新


```shell
删除旧的主机密钥记录，重新连接
sudo ssh-keygen -f "/root/.ssh/known_hosts" -R "192.168.1.1"
sudo ssh root@192.168.1.1
```

**在tmux中复制文本**

选中文本后不松手按`y`

前提是安装`yank.tmux`

**查看所有被挂起的tmux**

`tmux list-sessions`

**tmux配置文件报错**

`no server running on /tmp/tmux-1000/default`

这里需要先开启一个tmux窗口，然后再执行：

`tmux source .tmux.conf`

祖传tmux配置如下:

```shell
set-option -g status-keys vi
setw -g mode-keys vi

setw -g monitor-activity on

# setw -g c0-change-trigger 10
# setw -g c0-change-interval 100

# setw -g c0-change-interval 50
# setw -g c0-change-trigger  75


set-window-option -g automatic-rename on
set-option -g set-titles on
set -g history-limit 100000

#set-window-option -g utf8 on

# set command prefix
set-option -g prefix C-a
unbind-key C-b
bind-key C-a send-prefix

bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D

bind < resize-pane -L 7
bind > resize-pane -R 7
bind - resize-pane -D 7
bind + resize-pane -U 7


bind-key -n M-l next-window
bind-key -n M-h previous-window



set -g status-interval 1
# status bar
set -g status-bg black
set -g status-fg blue


#set -g status-utf8 on
set -g status-justify centre
set -g status-bg default
set -g status-left " #[fg=green]#S@#H #[default]"
set -g status-left-length 20


# mouse support
# for tmux 2.1
# set -g mouse-utf8 on
set -g mouse on
#
# for previous version
#set -g mode-mouse on
#set -g mouse-resize-pane on
#set -g mouse-select-pane on
#set -g mouse-select-window on


#set -g status-right-length 25
set -g status-right "#[fg=green]%H:%M:%S #[fg=magenta]%a %m-%d #[default]"

# fix for tmux 1.9
bind '"' split-window -vc "#{pane_current_path}"
bind '%' split-window -hc "#{pane_current_path}"
bind 'c' new-window -c "#{pane_current_path}"

# run-shell "powerline-daemon -q"

# set tmux vim highlight
set -g default-terminal "xterm-256color"

# vim: ft=conf
```



