---
layout: post
title: iTerm使用指南
description: Mac iTerm terminal guide, including shortcut, profile.
category: blog
---

## Shortcut

| Key                    | Description             |
| ---------------------- | ----------------------- |
| option + cmd + i       | 显示／隐藏              |
| cmd + ,                | 显示Preference          |
| cmd + t                | 新建Tab                 |
| cmd + w                | 关闭Tab                 |
| cmd + i                | 显示当前Tab的Preference |
| cmd + f                | 在当前Tab中搜索         |
| cmd + d                | 垂直分屏                |
| cmd + shift + d        | 水平分屏                |
| cmd + +/-              | 扩大／缩小窗口          |
|                        |                         |
| cmd + /                | 定位到光标位置          |
| cmd + 点击文件／文件夹 | 打开文件／文件夹        |
| control + u            | 清空当前行              |



## Profile

### Login shell

```shell
#!/usr/bin/expect

set PORT [lindex $argv 0]
set USER [lindex $argv 1]
set HOST [lindex $argv 2]
set LOGIN_PASSWORD [lindex $argv 3]
set timeout 30

spawn ssh -p $PORT $USER@$HOST
expect {
        "(yes/no)?"
        {send "yes\n";exp_continue}
        "*password:*"
        {send "$LOGIN_PASSWORD\n"}
}
interact
```

> Send text at start: ~/.ssh/iterm2login.sh PORT USER HOST LOGIN_PASSWORD