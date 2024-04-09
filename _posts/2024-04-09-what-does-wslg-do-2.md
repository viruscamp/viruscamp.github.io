---
title: 'WSLg 做了什么(2)'
date: 2024-04-09
permalink: /posts/2024/04/what-does-wslg-do-2/
tags:
  - arch
  - wsl
  - wslg
  - namespace
---

在[前一篇](2024-03-08-what-does-wslg-do.md)的基础上, 更深入了解一些细节.  

先安装一个最简化的 WSL 发行版 [miniwsl](https://github.com/0xbadfca11/miniwsl), 里面基本上只有 busybox 没有 systemd 也没有其他后台服务.  

[知乎](https://zhuanlan.zhihu.com/p/691590843)

- miniwsl 的进程树  
```
{init(miniwsl)} /init
├─ {init} plan9 --control-socket 5 --log-level 4 --server-fd
└─ {SessionLeader} /init
   └─ {Relay(n)} /init
      └─ -sh
```

- X11 相关变量与文件  
  `DISPLAY=:0` 对应的文件是 `/tmp/.X11-unix/X0`  
```sh
$ stat  /tmp/.X11-unix /mnt/wslg/.X11-unix
  File: /tmp/.X11-unix
  Size: 60              Blocks: 0          IO Block: 4096   directory
Device: 8eh/142d        Inode: 2           Links: 2
Access: (0777/drwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2024-04-09 09:47:08.820608974 +0000
Modify: 2024-04-09 03:30:21.580162432 +0000
Change: 2024-04-09 03:30:21.580162432 +0000
  File: /mnt/wslg/.X11-unix
  Size: 60              Blocks: 0          IO Block: 4096   directory
Device: 8eh/142d        Inode: 2           Links: 2
Access: (0777/drwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2024-04-09 09:47:08.820608974 +0000
Modify: 2024-04-09 03:30:21.580162432 +0000
Change: 2024-04-09 03:30:21.580162432 +0000
```
目录`/tmp`应该是由`wsl --system`系统发行版创建并挂载.  

当启用 systemd 后, 一般`tmp.mount` 会重新创建 `/tmp` 导致 `/tmp/.X11-unix/X0` 不存在, 必须重新创建目录或文件链接以保证 `/tmp/.X11-unix/X0 -> /mnt/wslg/.X11-unix/X0`.  

Ubuntu@WSL 禁用了 `tmp.mount`, 方法是只有 `/usr/share/systemd/tmp.mount` 而没有 `/usr/lib/systemd/system/tmp.mount`.  

Arch 默认就有 `/usr/lib/systemd/system/tmp.mount` 一直启用 `tmp.mount`. 可以用在 WSL2 中禁用:  
```sh
$ sudo systemctl mask tmp.mount
Created symlink /etc/systemd/system/tmp.mount → /dev/null.
``` 

- Wayland pulseaudio 相关变量与文件  

```sh
$ env | grep -E "XDG|PULSE|WAYLAND"
XDG_RUNTIME_DIR=/mnt/wslg/runtime-dir
WAYLAND_DISPLAY=wayland-0
PULSE_SERVER=unix:/mnt/wslg/PulseServer

$ ls -al $XDG_RUNTIME_DIR/wayland*  $XDG_RUNTIME_DIR/pulse/*
srwxrwxrwx    1 1000     1000             0 Apr  9 03:30 /mnt/wslg/runtime-dir/pulse/native
-rw-------    1 1000     1000             3 Apr  9 03:30 /mnt/wslg/runtime-dir/pulse/pid
srwxrwxrwx    1 1000     1000             0 Apr  9 03:30 /mnt/wslg/runtime-dir/wayland-0
-rw-rw----    1 1000     1000             0 Apr  9 03:30 /mnt/wslg/runtime-dir/wayland-0.lock
```

没有启用 systemd 的 WSL2 发行版, 目录`/mnt/wslg/runtime-dir`应该是由`wsl --system`系统发行版创建并挂载, `XDG_RUNTIME_DIR` 是由 进程树中 `{Relay(n)} /init` 这个进程设置的.  

启用 systemd 后 `XDG_RUNTIME_DIR` 是由 `logind` 创建并设置的. 创建 `/run/user/$(id -u)` 目录并设置到 `XDG_RUNTIME_DIR`, 必须创建上述4个文件的链接.  

Ubuntu@WSL 之前能正常创建上述4个文件的链接, 但近日的 24.04 没有创建 wayland-0 等的链接.  
