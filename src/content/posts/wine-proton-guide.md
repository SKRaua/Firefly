---
title: Wine 与 Proton 实践笔记：游戏、反作弊与多程序协同
published: 2026-11-13
description: 在 Fedora + KDE Wayland + AMD 显卡上折腾 Wine/Proton 的一些经验，包括反作弊、显示驱动、D3D9 卡顿和多程序共用同一个 wineserver。
tags: [Linux, Wine, Proton, DXVK]
category: Linux
draft: false
pinned: false
comment: true
lang: zh-CN
---

这篇文章记录的是我在 Fedora、KDE Wayland 和 AMD 显卡环境下，用 Wine 和 Proton 跑游戏时遇到的一些真实问题。不是教程，更像一份带着报错原文的备忘录，方便以后回头查。

## 几个容易混淆的概念

Wine 在 Linux 上实现 Windows 的 API，让 Windows 程序能直接运行。它本身不是模拟器，性能损失主要来自图形翻译。

Proton 是 Valve 基于 Wine 做的一套发行版，额外打包了 DXVK、protonfixes 和 Steam Runtime 容器。它把很多“开箱即用”的适配工作做完了，代价是多了一层容器。

DXVK 负责把 D3D9、D3D10、D3D11 翻译成 Vulkan，是游戏能不能跑、跑得顺不顺的关键。绝大多数 Windows 游戏的性能都取决于它，而不是 Wine 本身。

prefix 可以理解成一个独立的假 C 盘，里面有自己的 `drive_c`、注册表和字体。每个 prefix 对应一个 wineserver 后台进程，负责窗口、同步和进程管理。只有同一个 prefix 里的程序才共享同一个 wineserver。

容器是 Proton 和 umu 用 Steam Runtime 做出来的隔离环境。它让游戏在不同发行版上表现一致，但也会把游戏和宿主隔开，外部的程序不容易插手进去。

## Wine 从哪里来

纯 Wine 和 Proton 的差别，在动手之前最好先想清楚。

我主要用两种：一种是第三方打包的纯 Wine 构建，带了一些针对反作弊和 Wayland 的补丁，布局是标准的 `bin/wine`，不套容器；另一种是 Proton 构建，直接放进 Lutris 或 Steam 的兼容工具目录。

纯 Wine 的好处是可控，prefix、wineserver、显示驱动都由自己决定，多个程序可以共用一个环境。Proton 的好处是省心，protonfixes 会按游戏自动补环境变量，DXVK 也会自动部署。

对于普通 Steam 游戏，Proton 基本不用折腾。对于需要反作弊或者需要多个程序协同的场景，纯 Wine 更合适。

## 反作弊：进程得从 steam.exe 启动

不少游戏的检测逻辑会检查进程是不是从 `steam.exe` 拉起来的。在纯 Wine 构建里，如果启动日志里出现类似 `wine: launching from steam.exe stub!` 的字样，说明它走了内置的 steam 桩，反作弊这一关就过了。

用 Proton 或 umu 时，则要靠 `GAMEID` 和 `UMU_USE_STEAM=1` 让 protonfixes 注入对应的环境变量。

这一点很容易被忽略。游戏启动就退出、日志里又看不到明显错误时，先确认反作弊那一步有没有过。

## 显示驱动：Wayland 不一定更好

为了“原生 Wayland”，给游戏加了 DLL 顶替，把 `winex11.drv` 禁用、强制 `winewayland.drv`。结果游戏启动后立刻退出，游戏目录里的驱动错误日志只有一行：

```
initDriver Failed: Error [4,643,0].
```

原因是 DXVK 在 winewayland 驱动下创建 Vulkan surface 的路径还不完整，驱动初始化直接失败。

把顶替去掉、回到默认的 winex11（也就是走 XWayland）之后，游戏恢复正常。

这里还有一个更隐蔽的坑：同一个 prefix 里，winex11 和 winewayland 只能存在一个，由 wineserver 决定。如果先启动了用 winex11 的游戏，再启动一个要求 winewayland 的程序，后者会因为驱动冲突而创建窗口失败，报一个 `HwndWrapper` 的 0x80004005。想要多个程序共存，它们必须使用同一种显示驱动。

## 性能相关的几个开关

原神这类 D3D11 游戏，DXVK 要开，VKD3D 要关，因为用不到 D3D12。D3D 附加运行库是给 D3D9 的 d3dx 用的，也不需要开。NVAPI 和 DLSS 在 AMD 显卡上没有意义。

同步方式上，现在更推荐让 Wine 用 NTSync。它需要内核 6.11 以上和 `/dev/ntsync`，比 Esync 和 Fsync 都更接近原生。Lutris 里的 Esync、Fsync 开关保持关闭，Wine 会自己检测并使用 NTSync，启动日志里能看到 `ntsync: up and running.`。

如果遇到 XWayland 下的画面卡顿，可以加一个环境变量：

```yaml
system:
  env:
    DXVK_CONFIG: "dxgi.syncInterval = 0"
```

## 单服务多程序：让游戏和辅助程序共用环境

有时候需要不止一个程序：游戏本身，加上一个辅助工具，比如自动化脚本、截图工具或者带界面的小助手。

这类工具要跟游戏交互，前提是它们跑在同一个 prefix、同一个 wineserver 里。不同 prefix 或者不同容器之间是看不到彼此的，更别说注入或者读内存。

在纯 Wine 下，这件事相对直接：先用同一个 wine、同一个 prefix 启动游戏，再用同样的配置启动辅助程序，两者自然共享 wineserver。可以用 `ps` 确认只有一个 wineserver 在跑，多个程序都挂在它下面。

但这里有个控制台相关的坑。一些 .NET 桌面程序在启动时会主动分配一个控制台窗口，如果它发现自己已经处在一个控制台宿主里（比如从 cmd、bat 或者 wineconsole 启动），就会走另一条初始化路径，然后在 Wine 下报一个 `IOException: 无效的函数`，直接崩掉。

解决办法是让它像双击那样启动，不经过控制台。Lutris 图形界面直接启动，或者用 Wine 的 `start` 命令走 ShellExecute，都可以。

还有一点，这类程序经常依赖 .NET 的全球化功能。Wine 下如果找不到 ICU，启动会失败，加上 `DOTNET_SYSTEM_GLOBALIZATION_USENLS=1` 让它改用 Windows 的 NLS 就能解决。

## Galgame：D3D9 和 Present 等待

老一些的 Galgame 大多用 D3D9。我在一款游戏里遇到过点击按钮后整个画面卡死的情况，抓栈能看到 `xcb_wait_for_special_event`。

原因在 DXVK 的 Present 等待上。默认的 present 模式会等垂直同步或者 Present 事件，在 XWayland 下这个等待有时候回不来，界面就卡住了。

解决方式是给这个游戏单独加：

```yaml
system:
  env:
    DXVK_CONFIG: "d3d9.presentInterval = 0"
```

这样 DXVK 会改用 `VK_PRESENT_MODE_IMMEDIATE_KHR`，不再等待，按钮点击恢复正常。

需要注意的是，不要再同时加 `d3d9.maxFrameRate = 60`。限帧会重新引入 Present 等待，之前修好的卡顿会回来。

另外有些 Galgame 本身带 Steam DRM，在 Wine 下起不来。这跟 Wine 配置无关，换一个不带 DRM 的启动方式就行。

## Lutris 里两个容易放错的地方

Lutris 的配置里，环境变量和 DLL 顶替是两回事，放错位置不会报错，但也不会生效。

环境变量放在 `system.env` 下面，比如 `DXVK_CONFIG`、`DOTNET_SYSTEM_GLOBALIZATION_USENLS`。

DLL 顶替放在 `wine.overrides` 下面，比如 `d3d9`、`dxgi`，或者 `winex11.drv`、`winewayland.drv`。Lutris 会把它汇总成 `WINEDLLOVERRIDES`，所以不要自己把 `WINEDLLOVERRIDES` 写进环境变量，那样会被覆盖。

一个常见的配置大概是这样：

```yaml
system:
  env:
    DXVK_CONFIG: "d3d9.presentInterval = 0"

wine:
  overrides:
    d3d9: n,b
    dxgi: n,b
```

## 一些踩过的坑


. NET 程序从命令行启动崩溃、报 `无效的函数`，是因为它被控制台宿主接管了，换成图形界面或 `start` 启动。

Galgame 点按钮卡死，是 DXVK 的 Present 等待，单独加 `presentInterval = 0`。

Wine 下界面文字变方块，是缺中文字体，把字体复制到 prefix 的 `drive_c/windows/fonts` 下。

程序启动报 ICU 相关错误，加 `DOTNET_SYSTEM_GLOBALIZATION_USENLS=1`。

容器里的游戏，外部程序进不去，`nsenter` 普通用户也没有权限。需要多程序协同的时候，一开始就应该选纯 Wine 而不是 Proton 容器。

## 最后

Proton 适合“游戏能跑就行”的场景，省心，适配多。纯 Wine 适合需要控制细节的场景，prefix、wineserver、显示驱动都在自己手里，也能让多个程序共享同一个环境。

两者不是对立的。先用 Proton 把游戏跑起来，遇到反作弊、注入或者多程序协同的需求，再换到纯 Wine，是比较省事的顺序。
