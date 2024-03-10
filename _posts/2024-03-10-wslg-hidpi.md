---
title: 'WSLg 较好的 HiDPI 方案'
date: 2024-03-10
permalink: /posts/2024/03/wslg-hidpi/
tags:
  - wsl
  - wslg
  - hidpi
  - qt
  - gtk
  - wayland
---

[知乎](https://zhuanlan.zhihu.com/p/686248176)

- 渲染流程  
  程序-->weston  
  程序-->xwayland-->weston  

- 最好的方案是程序自身知道 HiDPI 的缩放比例, 可以自行放大字体, 自行选择高分辨率的图片, 会更清晰  

- 稍差的方案是由渲染器来缩放, 对 WSLg 是 weston  

以下方案不涉及多显示器  

环境变量前加 `export` 然后放在 `~/.bash_profile`  

# Weston Scale  
  **输出时强制放大 很可能模糊**  
  WSLg 的 Weston Wayland 合成器自带缩放  
  修改或创建 `%USERPROFILE%\.wslgconfig` 比如 `C:\Users\User1\.wslgconfig`  
  下面的配置是 150% 缩放[^3], 详情可看WSLg配置文档[^2]    
  ```
  [system-distro-env]
  ;hi-dpi
  WESTON_RDP_HI_DPI_SCALING=true
  WESTON_RDP_FRACTIONAL_HI_DPI_SCALING=false
  ;100 to 500
  WESTON_RDP_DEBUG_DESKTOP_SCALING_FACTOR=150
  ```
  **如果用了此方法 下面的方法都别用**  

# Qt 5/6  
  - 首选 wayland 其次 x11[^1]  
  - 效果比较好, 唯一的问题是标题栏无法缩放  
  ```
  QT_QPA_PLATFORM="wayland;xcb"
  QT_AUTO_SCREEN_SCALE_FACTOR=0
  QT_ENABLE_HIGHDPI_SCALING=0
  QT_SCALE_FACTOR=1.5
  ```

# GTK 3/4  
  - Gtk3 和 Gtk4 都是首选 wayland 的, 而 GDK_SCALE 在 wayland 中是无效的  
  - GDK_SCALE 控件字体全部按比例扩大, 但只支持整数
  - GDK_DPI_SCALE 似乎只改了字体大小, 控件是被字体撑大的, 只有图标的按钮就没有扩大, 比如窗口关闭按钮, 而且标题栏无法缩放[^4]  
  - GDK_DPI_SCALE 在 1.5 及以下效果还行, 2 的效果比较差  
  ```
  GDK_DPI_SCALE=1.5
  ```

# XWayland
  针对 GTK2 和 xcalc 之类, 暂时没有什么太好的方案  

# References 参考链接
[^1]: [HiDPI - ArchWiki](https://wiki.archlinux.org/title/HiDPI)  
[^2]: [WSLg Configuration Options for Debugging](https://github.com/microsoft/wslg/wiki/WSLg-Configuration-Options-for-Debugging)  
[^3]: [WSLg 高分屏上显示过小的问题](https://www.yijianhao.cn/archives/wslg%E9%AB%98%E5%88%86%E5%B1%8F%E4%B8%8A%E6%98%BE%E7%A4%BA%E8%BF%87%E5%B0%8F%E7%9A%84%E9%97%AE%E9%A2%98)  
[^4]: [【Linux】关于 X11 GUI 转发和 WSLg 的坑](https://juejin.cn/post/7006357005954711588)  
