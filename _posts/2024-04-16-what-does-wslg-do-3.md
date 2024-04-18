---
title: 'WSLg 做了什么(3)'
date: 2024-04-09
permalink: /posts/2024/04/what-does-wslg-do-3/
tags:
  - arch
  - wsl
  - wslg
  - miniwsl
---

## WSL2 without systemd
In [miniwsl](https://github.com/0xbadfca11/miniwsl)
1. files  

  ```
  $ ls -l /mnt/wslg/.X11-unix
  srwxrwxrwx    1 1000     1000             0 Apr 16 11:49 X0

  $ ls -l /mnt/wslg/runtime-dir
  drwx------    3 1000     1000            60 Apr 16 11:49 dbus-1
  drwx------    2 1000     1000            80 Apr 16 11:49 pulse
  srwxrwxrwx    1 1000     1000             0 Apr 16 11:49 wayland-0
  -rw-rw----    1 1000     1000             0 Apr 16 11:49 wayland-0.lock

  $ ls -l /mnt/wslg/runtime-dir/pulse
  srwxrwxrwx    1 1000     1000             0 Apr 16 11:49 native
  -rw-------    1 1000     1000             3 Apr 16 11:49 pid

  $ ls -l /mnt/wslg/PulseServer
  srwxrwxrwx    1 1000     1000             0 Apr 16 11:49 /mnt/wslg/PulseServer
  ```

2. mount  

  ```
  mount -o bind,ro /mnt/wslg/.X11-unix /tmp/.X11-unix
  ```

3. env vars  

  ```
  DISPLAY=:0
  XDG_RUNTIME_DIR=/mnt/wslg/runtime-dir
  WAYLAND_DISPLAY=wayland-0
  PULSE_SERVER=unix:/mnt/wslg/PulseServer
  ```

## WSL2 with systemd  
The above files, mounts and env vars without systemd, also exist in the distro like Ubuntu-22.04.  

And there are more:  
1. files  

  ```
  $ ls -l /mnt/wslg/run/user/1000
  drwxr-xr-x 3 user1 user1  60 Apr 16 19:30 dbus-1
  drwxr-xr-x 2 user1 user1  80 Apr 16 19:22 pulse
  lrwxrwxrwx 1 user1 user1  31 Apr 16 19:22 wayland-0 -> /mnt/wslg/runtime-dir/wayland-0
  lrwxrwxrwx 1 user1 user1  36 Apr 16 19:22 wayland-0.lock -> /mnt/wslg/runtime-dir/wayland-0.lock

  $ ls -l /mnt/wslg/run/user/1000/pulse
  lrwxrwxrwx 1 user1 user1 34 Apr 16 19:22 native -> /mnt/wslg/runtime-dir/pulse/native
  lrwxrwxrwx 1 user1 user1 31 Apr 16 19:22 pid -> /mnt/wslg/runtime-dir/pulse/pid
  ```

2. mount  

  ```
  mount -o bind /mnt/wslg/run/user /run/user
  ```

3. env vars  

  ```
  XDG_RUNTIME_DIR=/run/user/1000
  ```

4. `/tmp` and `/run/user/1000` are not mounted as tmpfs in Ubuntu-22.04  

  ```
  $ mountpoint /tmp
  /tmp is not a mountpoint

  $ mountpoint /run/user/1000
  /run/user/1000 is not a mountpoint
  ```

  We can see why `/run/user/1000` is not mounted as tmpfs:  

  ```
  $ systemctl status user-runtime-dir@1000
  systemd-user-runtime-dir[407]: Will mount /run/user/1000 owned by 1000:1000
  systemd-user-runtime-dir[407]: /run/user/1000 is already a mount point
  systemd[1]: Finished User Runtime Directory /run/user/1000.
  ```

  It's a bug in systemd-249 for Ubuntu-22.04 in `/lib/systemd/systemd-user-runtime-dir`:  

  ```sh
  $ sudo mkdir /run/user/33
  $ sudo chgrp www-data /run/user/33
  $ sudo chown www-data /run/user/33

  $ ls -l /run/user
  total 0
  drwxr-xr-x 6 user1     user1     200 Apr 18 19:01 1000
  drwxr-xr-x 2 www-data  www-data   40 Apr 18 19:04 33

  $ sudo SYSTEMD_LOG_LEVEL=debug /lib/systemd/systemd-user-runtime-dir start 33
  Will mount /run/user/33 owned by 33:33
  /run/user/33 is already a mount point

  $ sudo SYSTEMD_LOG_LEVEL=debug /lib/systemd/systemd-user-runtime-dir start 34
  Will mount /run/user/34 owned by 34:34
  Mounting tmpfs (tmpfs) on /run/user/34 (MS_NOSUID|MS_NODEV "mode=0700,uid=34,gid=34,size=1661825024,nr_inodes=405719")...

  $ ls -l /run/user
  total 0
  drwxr-xr-x 6 user1     user1     200 Apr 18 19:01 1000
  drwxr-xr-x 2 www-data  www-data   40 Apr 18 19:04 33
  drwx------ 2 backup    backup     40 Apr 18 19:09 34

  $ findmnt | grep run/user
  │ └─/mnt/wslg/run/user/34                               tmpfs                   tmpfs         rw,nosuid,nodev,relatime,size=1622876k,nr_inodes=405719,mode=700,uid=34,gid=34
  │ └─/run/user                                           none                    tmpfs         rw,nosuid,nodev,noexec,noatime,mode=755
  │   └─/run/user                                         none[/run/user]         tmpfs         rw,relatime
  │     └─/run/user/34                                    tmpfs                   tmpfs         rw,nosuid,nodev,relatime,size=1622876k,nr_inodes=405719,mode=700,uid=34,gid=34
  ```


## WSL2 with systemd, but something goes wrong  
1. In arch there is `/lib/systemd/system/tmp.mount`,  
  which do `mount -t tmpfs tmpfs /tmp`  

  ```
  $ findmnt /tmp
  TARGET SOURCE FSTYPE OPTIONS
  /tmp   tmpfs  tmpfs  rw,nosuid,nodev,size=8114388k,nr_inodes=1048576
  ```

  So, no `/tmp/.X11-unix/X0`, cause `Error: Can't open display: :0`.    

  Solutions:  
  1.1 disable `tmp.mount` by:  

    ```sh
    $ sudo systemctl mask tmp.mount
    Created symlink /etc/systemd/system/tmp.mount → /dev/null.
    ```

  1.2 or create links after `/tmp` mounted.

2. In arch and Ubuntu-24.04, there are `/usr/lib/systemd/system/user-runtime-dir@.service`,  
  which do `mount -t tmpfs tmpfs /run/user/1000`  

  ```
  $ findmnt /run/user/1000
  TARGET         SOURCE FSTYPE OPTIONS
  /run/user/1000 tmpfs  tmpfs  rw,nosuid,nodev,relatime,size=1622876k,nr_inodes=405719,mode=700,uid=1000,gid=1000
  ```

  So, no `/run/user/1000/wayland-0` nor some other, cause `Failed to create wl_display`.  

  Solution is creating links after `/run/user/1000/` mounted.  
