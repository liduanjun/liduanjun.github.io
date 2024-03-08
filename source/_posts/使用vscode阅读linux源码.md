---
title: 使用vscode阅读linux源码
date: 2024-03-08 09:31:15
tags:
---

## 准备工具和源码

1. 下载 vscode
```
https://code.visualstudio.com/
```

2. 下载内核源码
```
mkdir -pv ~/ws
cd ~/ws
wget https://mirrors.ustc.edu.cn/kernel.org/linux/kernel/v5.x/linux-5.4.100.tar.xz
tar xf linux-5.4.100.tar.xz
```

3. 系统必备包
```
# 基于 ubuntu 20.04
sudo apt install bear clangd git build-essential bc kmod cpio flex libncurses5-dev libelf-dev libssl-dev bison
```

## 配置工具

1. 编译linux码
```
cd ~/ws/linux-5.4.100
make oldconfig
bear -- make -j12
# 编译完成后检查根目录是否生成了 compile_commands.json 文件
```

2. 配置 vscode

在 vscode 中安装clangd插件，设置clangd的参数为，如下图
```
--compile-commands-dir=${workspaceFolder} --background-index --completion-style=detailed --header-insertion=never -log=info
```

![clangd配置](images/clangd-config.png)
