---
title: 'Wayland 环境下的窗口装饰'
date: 2024-03-16
permalink: /posts/2024/03/wayland-window-decorations/
tags:
  - wayland
  - x11
  - hidpi
  - qt
  - gtk
---

[知乎](https://zhuanlan.zhihu.com/p/687399748)

在设置 WSLg 下 HiDPI 的时, 界面正常的放大后, 标题栏等窗口装饰还是很小, 最后还是没有完全解决, 但了解了这些.  

# 窗口装饰 window decorations  
- 标题栏, 包括: 窗口名称 最小化 最大化 关闭, 有移动功能  
- 窗口边框, 有改变窗口大小功能  

# X11 下的窗口装饰  
- X Server 不提供窗口装饰
- 通常由 Window Manager 窗口管理器(比如 openbox, i3, kwin) 绘制处理  
- 客户程序(X clients) 可以要求 WM 不绘制窗口装饰[^1], 但通常客户端会自己绘制[^2]  

# Wayland 下的窗口装饰  
## Wayland Compositor 不提供窗口装饰  
- 相当于 X Server  
- 最简的 wayland 程序没有窗口装饰  

## 客户程序自绘窗口装饰  
- 通用的 libdecor 库  
    一个插件化的系统, `LIBDECOR_PLUGIN_DIR` 指定插件目录, 但不能指定明确的插件, 有如下插件:  
	- `/usr/lib/libdecor/plugins-1/libdecor-cairo.so`  
	- `/usr/lib/libdecor/plugins-1/libdecor-gtk.so`  

- Gtk3, Gtk4 由程序自绘(功能由控件库提供)  
    - 普通的标题栏, 使用`Gtk.Window.set_title`等函数  
    - `Gtk.HeaderBar`[^2] 提供的多功能标题栏  
    - 上述两种均受 `GDK_SCALE` 和 `GDK_DPI_SCALE` 影响  

- Qt5, Qt6 由控件库自绘  
    - 普通的标题栏, 插件化  
      默认插件为: `/usr/lib/qt6/plugins/wayland-decoration-client/libbradient.so`  
      **不受`QT_SCALE_FACTOR`影响**  
      本人已修改官方插件使之支持HiDPI: [qt-wayland-decorations-bradient-mkii](https://github.com/viruscamp/qt-wayland-decorations-bradient-mkii)  
      用法: `QT_WAYLAND_DECORATION=bradient-mkii featherpad`  
    - 也可以完全自绘[^3], 但目前没有方便的控件  
      `window.setWindowFlags(Qt::FramelessWindowHint)`  

## X11 程序在 Wayland 中  
- Gtk2 Qt4 xterm 等 X11 程序, 由 xwayland 作为 X Server  
- Wayland Compositor 内嵌一个 Window Manager, 只用于 X11 程序, 负责绘制此类程序的窗口装饰  
  参见 [xwayland/window-manager.c](https://github.com/microsoft/weston-mirror/blob/working/xwayland/window-manager.c#L1264),
  应该是可以创建[主题](https://github.com/microsoft/weston-mirror/blob/working/shared/cairo-util.c#L385),
  但目前写死了大小,颜色,字体和[图标](https://github.com/microsoft/weston-mirror/blob/working/shared/frame.c#L376).  
- 使用 wayland 缩放时适配 HiDPI  
- 客户程序自行处理 HiDPI 时, 不会缩放  

# References 参考链接  
[^1]: [Gtk.Window.set_decorated ](https://docs.gtk.org/gtk3/method.Window.set_decorated.html)  
[^2]: [Gtk.HeaderBar ](https://docs.gtk.org/gtk3/class.HeaderBar.html)  
[^3]: [Custom client-side window decorations in Qt 5.15](https://www.qt.io/blog/custom-window-decorations)  
