---
title: "Mac 终端怎么选：系统终端、iTerm2、Ghostty、Warp 的区别与上手方法"
date: 2026-10-24T00:00:00+08:00
lastmod: 2026-10-25T00:00:00+08:00
draft: true
description: "比较系统 Terminal、iTerm2、Ghostty 和 Warp 的使用区别，介绍安装方法、基础操作与适合的场景。"
tags: ["macOS", "终端", "iTerm2", "Ghostty", "Warp", "开发工具"]
categories: ["工具指南"]
---

Mac 自带 Terminal，为什么还有人安装 iTerm2、Ghostty 或 Warp？

它们都能运行命令，但输入、查找输出、管理多个会话的方式各有侧重。有人只想按教程装个工具，有人需要一边看日志一边执行命令，也有人希望直接在终端里使用 AI。

Mac 上还有 [WezTerm](https://wezterm.org/)、[kitty](https://sw.kovidgoyal.net/kitty/) 等选择。这篇先介绍四款：系统 Terminal、iTerm2、Ghostty 和 Warp，看看区别在哪里，以及安装后可以先试什么。

## 先看区别，再决定试哪款

| 终端 | 主要特点 | 适合先试的场景 |
| --- | --- | --- |
| 系统 Terminal | 系统自带，支持标签页和外观配置 | 偶尔执行命令、安装工具 |
| iTerm2 | 分屏、Profiles、搜索与热键窗口 | 同时跑服务、看日志、操作多个项目 |
| Ghostty | 原生界面、GPU 渲染、文本配置 | 喜欢原生界面，用文件管理设置 |
| Warp | 分块输出、文本编辑式输入、内置 AI | 编辑长命令、整理输出、使用 AI 辅助 |

表里列的是选择时值得关注的特点。iTerm2、Ghostty、Warp 都能分屏，iTerm2 也提供 AI Chat，不能仅凭“有没有某个功能”就判断哪款更适合自己。具体操作和官方资料放在下面各节。

![四款 Mac 终端对比示意](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/mac_terminal.png)

## 先跑通一条命令

按 `Command + 空格`，输入“终端”或“Terminal”，打开系统自带的终端应用。

![打开 macOS 系统终端](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/20261025162012.png)

逐条输入下面的命令，每输入一条就按回车：

```sh
cd ~
pwd
printf 'Hello, terminal!\n'
```

![系统终端执行示例命令](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/20261025162204.png)

第一条进入你的用户主目录。第二条显示当前位置，通常类似 `/Users/你的用户名`。第三条打印一行文字。

你已经完成了一次终端操作：输入命令，执行，然后查看结果。换到 iTerm2、Ghostty 或 Warp 的终端模式，仍然可以执行这几条命令。

**这四款软件都提供终端界面。** 它们接收键盘输入、显示程序输出，也负责窗口和标签页等操作。[Apple：终端是什么](https://support.apple.com/zh-cn/guide/terminal/trmld4c92d55/mac)

解释命令的是窗口里运行的 Shell，常见的有 zsh、bash 和 fish。Git、Node.js、SSH 则是你可以从 Shell 中调用的工具。Homebrew 负责安装和管理软件，也是一种命令行工具。

这几层分开之后，很多疑问就容易解释了：换终端通常不需要重装 Node.js；装好 Homebrew，也不会改变终端窗口的样式。

## 偶尔使用，系统终端就能开始

如果你的日常需求是安装工具、运行脚本，或者按教程完成一次配置，先用系统终端即可。

它支持多个窗口和标签页。需要调整字号、背景颜色时，可以使用应用里的描述文件设置；较长的输出也能通过标记和书签辅助定位。[Apple 终端使用手册](https://support.apple.com/zh-cn/guide/terminal/welcome/mac)

第一次使用，先掌握几个动作：

| 想做什么 | 操作 |
| --- | --- |
| 查看当前目录 | 输入 `pwd` |
| 列出当前目录的文件 | 输入 `ls` |
| 返回上一级目录 | 输入 `cd ..` |
| 新建标签页 | `Command + T` |
| 查找保留在终端里的输出 | `Command + F` |
| 调整字体和背景 | 打开“终端 → 设置 → 描述文件” |

路径带空格时，可以用引号包起来，例如 `cd "$HOME/Documents/My Project"`。这里的 `My Project` 要换成已经存在的文件夹。[Apple：执行命令和运行工具](https://support.apple.com/zh-cn/guide/terminal/apdb66b5242-0d18-49fc-9c47-a2498b7c91d5/mac)

复制文字通常用 `Command + C`；请求中断前台程序则用 `Control + C`。例如运行 `sleep 30` 后，按 `Control + C` 可以提前结束等待，回到提示符。

## 同时做几件事，看看 iTerm2 的分屏

从 [iTerm2 官网](https://iterm2.com/downloads.html)下载安装；如果已经装有 Homebrew，也可以执行：

```sh
brew install --cask iterm2
```

开发时常会遇到这样的场景：一个服务正在终端里持续输出日志，你还需要查看项目文件、执行 Git 命令。

可以新开一个标签页，也可以用 iTerm2 把一个窗口分成几块，让不同会话同时出现在眼前。

iTerm2 的默认快捷键是：

| 操作 | 快捷键 |
| --- | --- |
| 左右分屏 | `Command + D` |
| 上下分屏 | `Command + Shift + D` |
| 在分屏间切换 | `Command + Option + 方向键` |
| 临时放大当前分屏，再按恢复 | `Command + Shift + Enter` |

这些是 iTerm2 的默认键位，自定义过快捷键后，以自己的设置为准。[iTerm2 操作说明](https://iterm2.com/documentation-one-page.html)

例如，左边留给持续运行的服务，右边留给日常命令。服务出现异常时，可以一边看输出，一边在另一个会话里检查问题，不必把服务停掉。

默认情况下，新建分屏会启动独立会话。每个会话有自己的当前目录和前台程序，输入不会自动同步到另一边，正在运行的任务也不会复制过去。

这里有个容易混淆的区别：系统 Terminal 的 `Command + D` 分割的是同一个会话的视图，方便对照历史输出；它不会因此启动第二个 Shell。想让两边分别运行命令，可以在系统终端里另开窗口或标签页，或者使用 iTerm2、Ghostty 的独立会话分屏。

![iTerm2 界面示例](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/iterm2.png)

除了分屏，iTerm2 还提供搜索、粘贴历史、热键窗口和 Profiles（配置档案）。经常切换项目或服务器时，可以给不同会话安排不同的外观与配置。[iTerm2 功能介绍](https://iterm2.com/features.html)

先试一次分屏，再按 `Command + F` 搜索输出，就能体会它是否适合自己的工作方式。字体和配色可以在 `Settings → Profiles` 中调整。

后续如果需要识别命令边界、查看命令状态，可以了解 [Shell Integration](https://iterm2.com/documentation-shell-integration.html)。它还提供 [AI Chat](https://iterm2.com/documentation-ai-chat.html)，能否读取会话、执行命令取决于关联方式和权限设置。

## 想换一种界面与配置方式，可以试 Ghostty

从 [Ghostty 官网](https://ghostty.org/download)下载 Mac 安装包，打开后拖入“应用程序”；也可以通过 Homebrew 安装：

```sh
brew install --cask ghostty
```

Ghostty 同样支持标签页和分屏。它在 macOS 上使用原生界面组件，通过 Metal 加速终端画面的绘制；也提供 Linux 版本。[Ghostty 功能介绍](https://ghostty.org/docs/features)

Ghostty 可以直接使用默认设置。想调整外观时，从应用菜单选择 `Settings`，用文本编辑器打开配置文件，再加入下面几项即可。[打开配置文件的说明](https://ghostty.org/docs/config/keybind/reference#open_config)

```ini
# 字号，单位是点
font-size = 14

# 文字与窗口边缘的距离
window-padding-x = 10
window-padding-y = 10
```

这段内容写进配置文件，不能直接粘贴到 Shell 提示符后执行。字号和留白都可以按屏幕大小调整。[Ghostty 配置选项](https://ghostty.org/docs/config/reference)

当前官方文档使用 `config.ghostty` 文件名，也兼容旧的 `config`。Mac 上可以使用这个位置：

```text
~/Library/Application Support/com.mitchellh.ghostty/config.ghostty
```

这里的 `~` 表示当前用户的主目录。用访达查找时，按 `Command + Shift + G` 打开“前往文件夹”，粘贴下面的文件夹路径并回车，再找到里面的 `config.ghostty`：

```text
~/Library/Application Support/com.mitchellh.ghostty/
```

确认文件存在后，也可以在终端执行下面的命令，用系统默认的文本编辑器打开。路径含有空格，所以需要保留双引号：

```sh
open -t "$HOME/Library/Application Support/com.mitchellh.ghostty/config.ghostty"
```

如果已经在 `~/.config/ghostty/` 中维护配置，也可以继续使用。多个位置同时存在时，要留意加载顺序：后加载的同名设置会覆盖前面的值。

保存后，默认按 `Command + Shift + ,` 可以重新加载。部分选项需要新建会话或重启应用才会生效。[Ghostty 配置说明](https://ghostty.org/docs/config)

配置里的 `#` 注释要单独占一行，不要直接接在设置值后面。先调整字号和留白，等有具体需求时再加其他配置。

![Ghostty 字号与窗口留白配置](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/ghostty_config.png)

GPU 加速说的是终端画面的渲染方式。项目编译、网络下载和业务脚本的耗时，还取决于编译器、网络、代码以及机器资源，不能直接由终端是否支持 GPU 加速来判断。

选择 Ghostty 时，可以观察字体是否看得舒服、中文和符号显示是否正常，以及现有命令行程序是否符合自己的操作习惯。这些比单看一张配色截图更贴近日常使用。

## 经常编辑命令和整理输出，可以试 Warp

Warp 既能作为终端运行常规命令，也把 AI 助手放进了同一个应用。它对命令输入和输出展示做了不少调整，初次使用时，这些区别往往比配色更容易注意到。

从 [Warp 官网](https://www.warp.dev/download)下载安装，或使用 Homebrew：

```sh
brew install --cask warp
```

按当前官方说明，首次打开可以跳过注册和登录，先使用终端功能。[Warp 安装说明](https://docs.warp.dev/getting-started/quickstart/installation-and-setup/)

它比较直观的区别是 **Blocks（命令块）**：一条命令和对应的输出会组织在一起。连续执行几条命令后，你可以按块选择、复制和查找内容，不必总在整屏文字里拖动鼠标寻找边界。[Warp 命令块操作](https://docs.warp.dev/terminal/blocks/block-actions/)

安装后，可以在终端模式里逐条执行：

```sh
pwd
ls
printf 'Hello, Warp!\n'
```

观察每条命令对应的输出块，再选中一个块，按 `Command + C` 复制输出。这个小操作就能看出它与传统连续文本界面的区别。

输入区也有不同。你可以像使用文本编辑器一样，点击命令中间移动光标，选中文字后修改；需要输入多行时，用 `Shift + Enter` 换行，再按回车执行。[Warp 上手指南](https://docs.warp.dev/quickstart/)

![Warp 命令块与输入界面](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/wrap.png)

Warp 也支持标签页和[独立会话分屏](https://docs.warp.dev/terminal/windows/split-panes/)。如果你经常修改长命令、从多次执行结果里找内容，可以先试它的输入和输出组织方式。

需要 AI 时，再进入 Agent Mode（智能体模式），用自然语言描述问题，例如“解释这段报错”，并提供对应的输出内容。这类请求与普通 Shell 命令的处理方式不同。Warp 提供免费方案，但 AI 用量有单独的计费规则；普通命令和非 AI 终端功能不消耗 Agent 用量。具体额度和费用以[官方计费说明](https://docs.warp.dev/support-and-community/plans-and-billing/credits)及账号页面为准。

## 换终端前，记住几件小事

**不用同时装齐四款。** 先按自己的需求挑一款，用同一个项目试试进入目录、执行命令、查找输出和打开第二个会话。常用操作是否顺手，比界面截图更能帮你做决定。

安装 iTerm2、Ghostty、Warp 时，如果没有 Homebrew，直接使用各自的官方下载即可。安装前也看一下官网列出的 macOS 版本要求。

**窗口布局、输出记录、运行中的程序是三件事。** 恢复窗口不代表原来的任务仍在执行。远程任务需要在 SSH 断开后继续运行时，可以在服务器上使用 [tmux](https://github.com/tmux/tmux/wiki/Getting-Started)。它依赖服务器继续运行，不能让进程跨越关机或重启。刚入门可以先不用配置它。

## 换完终端，原来的命令怎么找不到了

假设旧终端能运行 `node`，新终端却提示：

```text
command not found: node
```

先别急着重新安装。终端启动的 Shell、登录方式或初始化配置不同，都可能让同一台电脑出现不同的命令环境。

如果使用 zsh 或 bash，可以在新旧两个终端里分别执行下面三条命令。其他 Shell 的语法可能不同，下面以 macOS 常见的 zsh 为例：

```sh
ps -p $$ -o comm=
command -v node
printf '%s\n' "$PATH"
```

先看第一条，比较正在运行的 Shell。再看第二条，确认各自能否找到 `node`，找到的是不是同一个位置。最后比较 `PATH`，检查需要的程序目录有没有包含在内。

如果排查的是其他工具，把 `node` 替换成它的命令名即可。

如果两边的 Shell 或程序路径不同，再检查对应的初始化配置。例如 zsh 的 `.zprofile` 与 `.zshrc` 读取时机不同，版本管理器的初始化代码放在哪个文件里，会影响它是否生效。[zsh 启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)

偶尔使用时，系统终端通常能完成需要的操作。经常同时管理多个会话，可以试 iTerm2；偏好原生界面与文本配置，可以试 Ghostty；关注命令编辑、分块输出和内置 AI，可以试 Warp。用了之后发现原来的终端更顺手，继续用原来的也没问题。

## 参考资料

功能说明于 2026 年 10 月 25 日依据官方文档核对，本文未进行性能测试或排名。菜单、配置和 AI 计费规则可能随版本变化。

- [Apple 终端使用手册](https://support.apple.com/zh-cn/guide/terminal/welcome/mac)
- [iTerm2 官方文档](https://iterm2.com/documentation.html)
- [Ghostty 官方文档](https://ghostty.org/docs)
- [Warp 官方上手指南](https://docs.warp.dev/quickstart/)
- [zsh 启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [tmux 官方入门指南](https://github.com/tmux/tmux/wiki/Getting-Started)

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**

