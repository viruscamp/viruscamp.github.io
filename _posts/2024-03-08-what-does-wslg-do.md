---
title: 'WSLg 做了什么'
date: 2024-03-08
permalink: /posts/2024/03/what-does-wslg-do/
tags:
  - arch
  - wsl
  - wslg
---

[知乎](https://zhuanlan.zhihu.com/p/686018563)

WSL2/WSLg 的系统架构[^1]就不谈了, 大家都知道 WSL2 发行版基本就是一个 Hyper-V 虚拟机里的 Linux , 加上与宿主 Windows 的互联互通, 与 WSLg 系统的互联互通.  
今天研究下 WSL2 发行版比 Hyper-V 里安装的 Linux 多了些什么, 才能做到上述的互联互通.  

# 用户发行版与系统发行版
## 用户发行版: 用户安装的 Linux
```sh
PS> wsl
$ uname -a
Linux 5.15.133.1-microsoft-standard-WSL2 #1 SMP Thu Oct 5 21:02:42 UTC 2023 x86_64 GNU/Linux
$ cat /etc/os-release
NAME="Arch Linux"
```

## 系统发行版: WSLg 系统
```sh
PS> wsl --system
wslg$ uname -a
Linux 5.15.133.1-microsoft-standard-WSL2 #1 SMP Thu Oct 5 21:02:42 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
wslg$ cat /etc/os-release
NAME="Common Base Linux Mariner"
VERSION="2.0.20230630"
```

# kernel 内核修改
进入用户发行版, `ls /boot` 是看不见任何文件的, 实际的启动内核在 `C:\Program Files\WSL\tools`.  

```sh
$ ls -l "/mnt/c/Program Files/WSL/tools"
总计 18172
-r-xr-xr-x 1 user1 user1  2105816 12月 1日 08:34 init
-r-xr-xr-x 1 user1 user1  2106368 12月 1日 08:34 initrd.img
-r-xr-xr-x 1 user1 user1 14387168 10月 5日 21:23 kernel
```
这个内核除了精简了不需要的驱动以外, 只加了一个驱动[^2] `drivers/gpu/dxgkrnl`, 这个驱动会生成 `/dev/dxg`.  
这个内核应该是 WSLg 系统和所有已安装的用户发行版共享的.  

# `/init` 启动程序
内核启动后, 会复制一份 `/init` 并启动.  

```sh
$ md5sum "/mnt/c/Program Files/WSL/tools/init"
22443cb1e29a18cc768895e49bcf3e85  /mnt/c/Program Files/WSL/tools/init
$ md5sum /init
22443cb1e29a18cc768895e49bcf3e85  /init
```

- 没有启用 systemd 的系统  
```
/init #wsl init
├─ plan9 --control-socket 5 --log-level 4 --server-fd
└─ /init
   └─ /init
      └─ -bash -- user1
```

1. 启动网络文件服务器  
    可以看见一个进程 `plan9 --control-socket 5 --log-level 4 --server-fd`, 这是从 WSL1 时代就有的, 用于宿主 Windows 文件互通的网络文件系统[^3], 叫做 `Plan 9 file server`.   

2. 链接与挂载 WSLg 内的文件  
    - 设备文件(`/dev/dxg`) 这个是个跨系统硬链接
    - socket 文件(`/tmp/.X11-unix/X0`) 注意一下, `/tmp/.X11-unix`是目录的跨系统硬链接
    - 驱动文件(`/usr/lib/wsl/drivers`)
    - 其他(`/mnt/wslg/*`)

    ```sh
    $ ls -lda /tmp/.X11-unix $XDG_RUNTIME_DIR/wayland* $XDG_RUNTIME_DIR/pulse/*
    srwxrwxrwx 1 user1 user1  0 Mar  6 12:37 /mnt/wslg/runtime-dir/pulse/native
    -rw------- 1 user1 user1  3 Mar  6 12:37 /mnt/wslg/runtime-dir/pulse/pid
    srwxrwxrwx 1 user1 user1  0 Mar  6 12:37 /mnt/wslg/runtime-dir/wayland-0
    -rw-rw---- 1 user1 user1  0 Mar  6 12:37 /mnt/wslg/runtime-dir/wayland-0.lock
    drwxrwxrwx 2 root      root      60 Mar  6 12:37 /tmp/.X11-unix

    $ stat /tmp/.X11-unix  /mnt/wslg/.X11-unix
    File: /tmp/.X11-unix
    Size: 60              Blocks: 0          IO Block: 4096   directory
    Device: 0,94    Inode: 2           Links: 2
    Access: (0777/drwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
    Access: 2024-03-06 12:37:37.810283280 +0800
    Modify: 2024-03-06 12:37:37.954905785 +0800
    Change: 2024-03-06 12:37:37.954905785 +0800
    Birth: -
    File: /mnt/wslg/.X11-unix
    Size: 60              Blocks: 0          IO Block: 4096   directory
    Device: 0,94    Inode: 2           Links: 2
    Access: (0777/drwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
    Access: 2024-03-06 12:37:37.810283280 +0800
    Modify: 2024-03-06 12:37:37.954905785 +0800
    Change: 2024-03-06 12:37:37.954905785 +0800
    Birth: -
    ```

3. 设置环境变量
    ```sh
    $ env | grep -E "DISPLAY|PULSE|XDG"
    DISPLAY=:0
    WAYLAND_DISPLAY=wayland-0
    PULSE_SERVER=unix:/mnt/wslg/PulseServer
    XDG_RUNTIME_DIR=/mnt/wslg/runtime-dir
    ```

- 启用 systemd 的系统  
    ```
    /sbin/init #systemd init
    │  ├─ /init #wsl init
    │  ├─ ├─ plan9 --control-socket 5 --log-level 4 --server-fd
    │  │  └─ /init
    │  │     └─ -bash -- user1
    │  └─ login -- user1
    │     └─ -bash
    ├─ /usr/lib/systemd/systemd-journald
    ├─ #systemd mount
    ├─ #systemd tmpfiles
    ```

    对于启用 systemd 的系统, 其后系统服务可能会挂载 `/tmp`和`/run/user/1000`, 并重新设置 `XDG_RUNTIME_DIR`, 使得`/init`创建的部分链接失效, 需要正确设置 systemd 来重新创建.  
    ```sh
    $ env | grep -E "DISPLAY|PULSE|XDG"
    DISPLAY=:0
    WAYLAND_DISPLAY=wayland-0
    PULSE_SERVER=unix:/mnt/wslg/PulseServer
    XDG_RUNTIME_DIR=/run/user/1000/

    $ ls -lda /tmp/.X11-unix $XDG_RUNTIME_DIR/wayland* $XDG_RUNTIME_DIR/pulse/*
    lrwxrwxrwx 1 user1 user1 34 Mar  6 12:49 /run/user/1000//pulse/native -> /mnt/wslg/runtime-dir/pulse/native
    lrwxrwxrwx 1 user1 user1 31 Mar  6 12:49 /run/user/1000//pulse/pid -> /mnt/wslg/runtime-dir/pulse/pid
    lrwxrwxrwx 1 user1 user1 31 Mar  6 12:49 /run/user/1000//wayland-0 -> /mnt/wslg/runtime-dir/wayland-0
    lrwxrwxrwx 1 user1 user1 36 Mar  6 12:49 /run/user/1000//wayland-0.lock -> /mnt/wslg/runtime-dir/wayland-0.lock
    lrwxrwxrwx 1 root      root      19 Mar  6 12:49 /tmp/.X11-unix -> /mnt/wslg/.X11-unix
    ```

# 设备硬链接可能导致的问题  
研究发现, WSL2 的许多设备文件(以 `/dev/dri/card0` 为例), 是同一个文件在多个系统内的硬链接.  

```sh
wslg$ stat /dev/dri/card0
  File: /dev/dri/card0
  Size: 0               Blocks: 0          IO Block: 4096   character special file
Device: 5h/5d   Inode: 94          Links: 1     Device type: e2,0
Access: (0660/crw-rw----)  Uid: (    0/    root)   Gid: (   44/ UNKNOWN)
Access: 2024-03-08 18:31:17.232796176 +0800
Modify: 2024-03-08 18:31:17.232796176 +0800
Change: 2024-03-08 18:31:17.232796176 +0800
 Birth: -

Arch$ stat /dev/dri/card0
  File: /dev/dri/card0
  Size: 0               Blocks: 0          IO Block: 4096   character special file
Device: 0,5     Inode: 94          Links: 1     Device type: 226,0
Access: (0660/crw-rw----)  Uid: (    0/    root)   Gid: (   44/ UNKNOWN)
Access: 2024-03-08 18:31:17.232796176 +0800
Modify: 2024-03-08 18:31:17.232796176 +0800
Change: 2024-03-08 18:31:17.232796176 +0800
 Birth: -

Ubuntu$ stat /dev/dri/card0
  File: /dev/dri/card0
  Size: 0               Blocks: 0          IO Block: 4096   character special file
Device: 0,5     Inode: 94          Links: 1     Device type: 226,0
Access: (0660/crw-rw----)  Uid: (    0/    root)   Gid: (   44/   video)
Access: 2024-03-08 18:31:17.232796176 +0800
Modify: 2024-03-08 18:31:17.232796176 +0800
Change: 2024-03-08 18:31:17.232796176 +0800
 Birth: -
```

这很可能会导致设备文件用户组混乱的问题, 其中一个系统会突然无法使用 GPU 设备[^4].  
1. 多个系统的同名 group 的 GID 不太可能相同  
    ```sh
    Arch$ cat /etc/group | grep video
    video:x:110:user1

    Ubuntu$ cat /etc/group | grep video
    video:x:44:user1
    ```
2. 后一个启动的用户发行版会更改设备文件的用户组  
    比如 Ubuntu 就把 `/dev/dri/card0` 改为属于 gid=44, 在先启动的 Arch 看来, `/dev/dri/card0` 就不属于 Arch 的 video, 那么普通用户就无法使用 GPU.  

# References 参考链接
[^1]: [WSLg Architecture](https://devblogs.microsoft.com/commandline/wslg-architecture/)  
[^2]: [DirectX is coming to the Windows Subsystem for Linux](https://devblogs.microsoft.com/directx/directx-heart-linux/)  
[^3]: [What is this weird process I see in WSL called 'plan9'?](https://superuser.com/questions/1749690/what-is-this-weird-process-i-see-in-wsl-called-plan9)  
[^4]: [With two distros in WSL2, group of /dev/dri/card0 may be changed by the last started distro, and cause vainfo failed in the first distro ](https://github.com/microsoft/wslg/issues/1208)  
