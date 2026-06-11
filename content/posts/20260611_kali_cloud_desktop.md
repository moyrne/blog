---
title: "20260611_kali_cloud_desktop"
date: 2026-06-11T18:44:33+08:00
draft: true
comments: false
images:
---

# kali-2026.1 云桌面踩坑记录

## 输入法

1. 安装 fcitx5

~~~shell
sudo apt install fcitx5 fcitx5-pinyin fcitx5-frontend-gtk2 fcitx5-frontend-gtk3 fcitx5-frontend-qt5
~~~

2. 配置 ~/.xprofile

~~~
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export INPUT_METHOD=fcitx
export SDL_IM_MODULE=fcitx

# 启动 Fcitx5 守护进程（如果还没运行）
if ! pgrep -x fcitx5 > /dev/null; then
    fcitx5 -d
fi
~~~

## QXL

1. 显存尽可能的调大

~~~
ram_size 524288
vga_mem 131072
~~~

2. 关闭 显示合成

操作路径: `菜单` -> `设置(Settings)` -> `窗口管理器微调(Window Manager Tweaks)` -> `合成器(Compositor)` -> `取消勾选"启用显示合成"(Enable display compositing)`

3. 安装 spice-vdagent, guest-agent

~~~shell
sudo apt install spice-vdagent qemu-guest-agent xserver-xorg-video-qxl
~~~
