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

## miniwsl 分析
- miniwsl 的进程树  
```
1 {init(miniwsl)} /init
├─ 4 {init} plan9 --control-socket 5 --log-level 4 --server-fd
└─ 7 {SessionLeader} /init
   └─ 8 {Relay(n)} /init
      └─ 9 -sh
```

- miniwsl 的环境变量  
通过 `cat /proc/8/environ`({Relay(n)} /init) 和 `cat /proc/9/environ`(sh):  
```sh
$ cat /proc/8/environ
WSL2_CROSS_DISTRO=/wsl

$ cat /proc/9/environ
DISPLAY=:0
XDG_RUNTIME_DIR=/mnt/wslg/runtime-dir
WAYLAND_DISPLAY=wayland-0
PULSE_SERVER=unix:/mnt/wslg/PulseServer
WSL_INTEROP=/run/WSL/8_interop
```
我们知道重要的环境变量是 `/init` 的 Relay 进程在启动 `sh` 时设置给 `sh` 的  

- miniwsl 的文件系统  
暂时推测 wsl-system 挂载 WSL 子系统对应的`ext4.vhdx`为子系统的`/`,  
然后创建临时文件系统(tmpfs) `/mnt/wslg/` 及 `/mnt/wslg/runtime-dir/`, `/mnt/wslg/.X11-unix/`,  
还有里面 X, Wayland, Pulseaudio 对应的 socket 文件,  
将之挂载到子系统 `mount --bind /mnt/wslg/ /{miniwsl-sub-distro}/mnt/wslg/`.  
启动子系统后, 子系统大概会 `mount -o bind,ro /tmp/.X11-unix /mnt/wslg/.X11-unix`.  

## 环境变量与对应文件  
- X11 相关变量与文件  
  `DISPLAY=:0` 对应的文件是 `/tmp/.X11-unix/X0`  

目录`/tmp`应该是由`wsl --system`系统发行版创建并挂载.  

当启用 systemd 后, 一般`tmp.mount` 会重新创建 `/tmp` 导致 `/tmp/.X11-unix/X0` 不存在, 必须重新创建目录或文件链接以保证 `/tmp/.X11-unix/X0 -> /mnt/wslg/.X11-unix/X0`.  

Ubuntu@WSL 禁用了 `tmp.mount`, 方法是只有 `/usr/share/systemd/tmp.mount` 而没有 `/usr/lib/systemd/system/tmp.mount`.  

Arch 默认就有 `/usr/lib/systemd/system/tmp.mount` 一直启用 `tmp.mount`. 可以用在 WSL2 中禁用:  
```sh
$ sudo systemctl mask tmp.mount
Created symlink /etc/systemd/system/tmp.mount → /dev/null.
```

修复方法
1. 禁用 `tmp.mount`
2. 启用 `tmp.mount`, 目录链接 `ln -sf /mnt/wslg/.X11-unix /tmp/.X11-unix`
3. 启用 `tmp.mount`, 文件链接 `ln -sf /mnt/wslg/.X11-unix/X0 /tmp/.X11-unix/X0`
4. 启用 `tmp.mount`, 挂载目录 `mount -o bind,ro /tmp/.X11-unix /mnt/wslg/.X11-unix`
5. 启用 `tmp.mount`, 挂载文件 `mount -o bind,ro /tmp/.X11-unix/X0 /mnt/wslg/.X11-unix/X0`

创建链接可以使用 tmpfiles.d 来实现
```
$ cat /etc/tmpfiles.d/wslg.conf
# See tmpfiles.d(5) for details
# link WSLg display files after system started

# Type Path              Mode UID  GID  Age Argument
#L+     /tmp/.X11-unix    -    -    -    -   /mnt/wslg/.X11-unix
L+     /tmp/.X11-unix/X0 -    -    -    -   /mnt/wslg/.X11-unix/X0
```

挂载的方法只能写一个 systemd service 

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

保持 `XDG_RUNTIME_DIR=/mnt/wslg/runtime-dir` 是不可接受的, 那么只能创建链接.  

创建链接可以使用 user-tmpfiles.d 来实现, 创建文件 `~/.config/user-tmpfiles.d/wslg.conf` 或 `/usr/share/user-tmpfiles.d/wslg.conf`, 内容为:  
```
# Type Path              Mode UID  GID  Age Argument
L+     %t/wayland-0      -    -    -    -   /mnt/wslg/runtime-dir/wayland-0
L+     %t/wayland-0.lock -    -    -    -   /mnt/wslg/runtime-dir/wayland-0.lock
L+     %t/pulse/native   -    -    -    -   /mnt/wslg/runtime-dir/pulse/native
L+     %t/pulse/pid      -    -    -    -   /mnt/wslg/runtime-dir/pulse/pid
L+     %t/dbus-1         -    -    -    -   /mnt/wslg/runtime-dir/dbus-1
```
然后运行 `systemctl --user enable systemd-tmpfiles-setup.service  systemd-tmpfiles-clean.timer`.
