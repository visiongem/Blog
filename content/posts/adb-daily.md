---
title: "Android 开发最常用的 10 个 adb 命令"
date: 2026-05-29T10:00:00+08:00
draft: false
categories: ["Android"]
tags: ["Android", "博客"]
---

## 什么是 adb

adb（Android Debug Bridge）是 Android SDK 自带的调试工具，用来在电脑和 Android 设备之间通信。

平时在 Android Studio 里点的很多操作，本质底层都是 adb。

下面整理 10 个日常开发真正高频在用的 adb 命令。

---

## 0. 准备

adb 在 Android SDK 的 `platform-tools` 目录下。配进 PATH 之后，终端任意位置都能调。SDK 可以通过 Android Studio 的 SDK Manager 安装，也可以独立下载 [Android SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools)。

多设备场景每条命令都可以加 `-s <serial>` 指定设备：

```bash
adb -s emulator-5554 install app.apk
```

不加默认就是唯一连着的设备；多个设备没指定会报错。

---

## 1. adb devices

```bash
adb devices
```

最基础的：看连了几个设备。`device` 是正常，`unauthorized` 是没在手机上点"信任"，`offline` 是连接挂了。

通常最好先确认设备认到了再干别的。

---

## 2. adb install

```bash
adb install app.apk           # 第一次装
adb install -r app.apk        # 覆盖安装（保留数据）
adb install -t app.apk        # 允许测试包
adb install -d app.apk        # 允许降级
```

最常用是 `-r` 覆盖安装。新版本 Android 装 debug 包需要加 `-t`。

报 `INSTALL_FAILED_VERSION_DOWNGRADE` 时加 `-d`。

---

## 3. adb logcat（最实用的姿势）

```bash
# 全量看
adb logcat

# 按 tag + level（W 以上）
adb logcat MyApp:V *:W

# 按进程名（只看当前 App 的日志）
adb logcat --pid=$(adb shell pidof -s com.your.app)

# 清空再看（看启动日志特别有用）
adb logcat -c && adb logcat

# 存文件
adb logcat -d > log.txt
```

按进程过滤是最实用的——不然各种系统日志会把屏幕刷满。

---

## 4. adb shell pm clear

```bash
adb shell pm clear com.your.app
```

清掉 App 所有数据，等于设置里点"清除数据"。

最常用场景：测引导页、首次启动、登录流程，跑一遍就是干净状态。

---

## 5. adb shell am start / force-stop

```bash
# 杀进程
adb shell am force-stop com.your.app

# 启 Activity
adb shell am start -n com.your.app/.MainActivity

# 带 deep link
adb shell am start -n com.your.app/.DeepLinkActivity \
  -d "myapp://order/123"

# 用系统 action 测 URL
adb shell am start -a android.intent.action.VIEW -d https://example.com
```

调 deep link、测启动流程必备。

`pm clear` + `am force-stop` + `am start` 三连就是"模拟首次启动"。

---

## 6. adb shell input

自动化点击 / 输入：

```bash
adb shell input text "hello"            # 输入文字（不支持中文）
adb shell input tap 500 1000            # 点击坐标
adb shell input swipe 500 1500 500 500  # 滑动
adb shell input keyevent 4              # 按返回键
adb shell input keyevent 66             # 按回车
```

常用 keyevent：

| 数字 | 含义 |
|---|---|
| 3 | HOME |
| 4 | BACK |
| 26 | POWER |
| 66 | ENTER |
| 67 | DEL |

写脚本批量操作很爽。中文输入要用 ADBKeyBoard 之类的第三方输入法配合。

---

## 7. adb shell screencap / screenrecord

```bash
# 截图（直接保存到电脑）
adb exec-out screencap -p > screen.png

# 录屏
adb shell screenrecord /sdcard/demo.mp4
# Ctrl+C 停止后拉出来
adb pull /sdcard/demo.mp4

# 限制时长 + 分辨率
adb shell screenrecord --time-limit 30 --size 720x1280 /sdcard/demo.mp4
```

写 PR 描述、提 bug、做 demo 都用这个。**录屏系统限制最长 3 分钟**。

---

## 8. adb push / pull

```bash
# 推到设备
adb push local.txt /sdcard/Download/

# 从设备拉出来
adb pull /sdcard/Download/some.log

# 拉 App 私有目录里的数据库（需要 debuggable 包）
adb exec-out run-as com.your.app cat databases/app.db > app.db
```

最后那条特别实用：直接拉 debug 包的 Room 数据库出来用 DB Browser 看。

---

## 9. adb reverse

```bash
adb reverse tcp:8080 tcp:8080
```

**让手机能访问自己电脑上的 localhost**——调本地服务端神器。

跑了这条之后，手机里访问 `http://localhost:8080` 就是访问电脑上的 8080。

比改 `10.0.2.2` 或者折腾局域网 IP 方便太多。

部分厂商 ROM 可能不支持。

---

## 10. adb shell dumpsys

万能调试工具。常用几个：

```bash
# 当前最上层 Activity（导航问题排查必备）
adb shell dumpsys activity activities | grep "ResumedActivity"

# 看应用内存
adb shell dumpsys meminfo com.your.app

# 看进程
adb shell dumpsys activity processes | grep com.your.app

# 看电池消耗
adb shell dumpsys batterystats com.your.app

# 看 Window 焦点（键盘 / Dialog 状态用）
adb shell dumpsys window | grep mCurrentFocus



Windows 下可改用 findstr
```

第一条"看最上层 Activity"是排查 Navigation 问题的常备工具。

---

## 番外：值得 alias 的几个

`.zshrc` 里值得保存这几个，敲得最频繁的：

```bash
alias adv='adb devices'
alias alog='adb logcat -c && adb logcat'
alias akill='adb shell am force-stop'
alias aclear='adb shell pm clear'
# 杀掉 + 清数据：arelaunch com.your.app
alias arelaunch='f(){ adb shell am force-stop $1 && adb shell pm clear $1; }; f'
# 看当前最上层 Activity
alias atop='adb shell dumpsys activity activities | grep ResumedActivity'
```

---

## 小结

| 场景 | 命令 |
|---|---|
| 看连了啥 | `adb devices` |
| 装包 | `adb install -r` |
| 看日志 | `adb logcat --pid=$(adb shell pidof -s <pkg>)` |
| 清数据 | `adb shell pm clear <pkg>` |
| 起 / 杀 App | `adb shell am start / force-stop` |
| 自动化输入 | `adb shell input` |
| 录屏 / 截图 | `adb shell screenrecord` / `screencap` |
| 推 / 拉文件 | `adb push / pull` |
| 让手机访问电脑 localhost | `adb reverse` |
| 看系统状态 | `adb shell dumpsys` |

10 条记熟，日常 debug 八成场景都能不离开终端搞定。
