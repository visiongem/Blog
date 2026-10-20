---
title: "Flutter 学习笔记（零）：搭建环境，让一套代码运行在多个平台"
date: 2026-10-20T09:00:00+08:00
draft: false
categories: ["Flutter"]
tags: ["Flutter", "Dart", "Android", "VS Code"]
---

又又又准备开始学习 Flutter。

本文以 **macOS + VS Code** 为开发环境，重点搭建 Android、iOS 客户端开发环境。Web 和桌面部分作为可选参考，当前先学习移动端跨平台开发。

我原本做 Android 开发，文中也整理了 Android、iOS 原生概念与 Flutter 的对照。

具体 SDK 版本和系统要求以[官方安装文档](https://docs.flutter.dev/install)为准，避免过一段时间照着旧版本号安装反而踩坑。

![flutter0_1](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter0_1.png)

## 一、Flutter 有什么特色？

Flutter 是使用 Dart 开发跨平台应用的 UI 工具包，可以面向 Android、iOS、Web 和桌面复用界面与业务代码。跨平台不意味着所有代码都能原样通用：权限、系统能力、平台交互和部分插件仍然需要分别处理。

### 一套代码，多端运行

Flutter 的核心价值是：**使用同一套 Dart 代码编写界面和业务逻辑，分别构建并运行到多个平台。**

| 平台类别 | 运行目标 |
| --- | --- |
| 移动端 | Android、iOS |
| Web | 浏览器 |
| 桌面端 | Windows、macOS、Linux |

例如，`flutter_demo` 的页面代码写在 `lib/` 中，可以选择 Android 模拟器运行，也可以选择 iOS 模拟器或 Chrome 查看同一份页面，不需要为每个平台重新实现一遍基础界面。各平台支持的系统版本见[官方支持平台列表](https://docs.flutter.dev/reference/supported-platforms)。

共享的是源代码，各平台仍生成各自的构建产物：Android 应用包不能直接安装到 iPhone，Web 也需要单独构建和部署。实际项目还要适配屏幕大小、触摸与键盘鼠标操作，并检查插件是否支持目标平台。

### 开发体验与界面组织

另外值得先认识三个特点：

- **声明式 UI**：根据状态描述界面。如果写过 Jetpack Compose，理解这种方式会更快。
- **组件组合**：文字、布局、间距和交互通常由 Widget 组合而成，方便拆分与复用。
- **热重载**：在调试时快速查看代码变化，通常可以保留当前状态，适合反复调整 UI。

Flutter 的普通界面由自身的渲染体系绘制，并非把每个 Widget 都映射成 Android View。需要嵌入原生控件时，可以使用 Platform Views；需要调用系统能力时，通常先找插件，必要时再通过平台通道连接 Android 的 Kotlin/Java 或 iOS 的 Swift/Objective-C 代码。具体分层可参考[官方架构说明](https://docs.flutter.dev/resources/architectural-overview)。

## 二、先分清工具各自负责什么

| 工具 | 作用 |
| --- | --- |
| Flutter SDK | 提供框架和命令行工具，内含匹配的 Dart SDK |
| VS Code / Android Studio + Flutter 扩展 | 编写 Dart、运行和调试 Flutter |
| Android SDK、JDK、Gradle | 支撑 Android 构建；Android Studio 可管理 SDK 和模拟器 |
| Xcode | 提供 iOS 构建、签名及模拟器工具，仅能在 macOS 使用 |
| CocoaPods | 管理使用它的 iOS/macOS 原生插件依赖 |

安装 Flutter 扩展和安装 Flutter SDK 是两步。日常可以选择 VS Code 或 Android Studio；我先用 VS Code，最后也在 Android Studio 中跑通了 Android Demo。

## 三、手动下载 SDK，再让 VS Code 找到它

这次我先安装了 VS Code 的 Flutter 扩展，它会同时安装 Dart 扩展。随后通过 `Flutter: New Project` 创建项目，扩展提示下载或定位 Flutter SDK。我尝试在线下载，但遇到错误，可能与当时的网络有关，于是改为从官网手动下载，解压后回到扩展提示中选择 SDK 路径。

扩展操作参考[官方 VS Code 安装指南](https://docs.flutter.dev/install/with-vs-code)，下面记录我的手动安装过程。

### 1. 下载并解压

从[官方手动安装页面](https://docs.flutter.dev/install/manual)选择 macOS 对应芯片架构的 stable SDK。我的电脑是 Apple Silicon，使用 ARM64 版本，解压后的实际目录是：

```text
/Users/nia/flutterSdk/flutter
```

这个目录下应该能看到 `bin/`、`packages/` 等内容。SDK 放在固定、当前用户可写的位置即可，不必放进某个业务项目。

![flutter0_bin](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter0_bin.png)

### 2. 配置终端 PATH

在 `~/.zprofile` 中加入下面这行，路径换成自己的 SDK 目录：

```bash
export PATH="$HOME/flutterSdk/flutter/bin:$PATH"
```

打开新终端，或重新加载配置：

```bash
source ~/.zprofile
which flutter
flutter --version
```

我的 `which flutter` 应指向 `/Users/nia/flutterSdk/flutter/bin/flutter`。Flutter 自带匹配的 Dart SDK，不需要再单独下载安装 Dart。

### 3. 配置 VS Code 的 SDK 路径

#### 我实际用的方式：在提示中定位 SDK

创建项目时，扩展发现没有可用的 Flutter SDK，会提示下载或定位 SDK。我的下载失败后，提示中仍有定位 SDK 的入口，所以不需要先去设置页面寻找按钮。

1. 完成手动下载并解压。
2. 回到 VS Code 的 SDK 提示，选择 **Locate SDK**（定位 SDK）。按钮文字可能随语言或扩展版本变化。
3. 在目录选择窗口中选择 `/Users/nia/flutterSdk/flutter`，确认即可。
4. 如果提示已经收起，可以先从右下角通知中心查看原提示。

这里选择的是包含 `bin/`、`packages/` 的 **Flutter SDK 根目录**，不是下载的压缩包，也不是 `flutter_demo` 项目目录。

这个入口只在扩展需要定位 SDK 时出现，不是一直固定显示的设置按钮。如果扩展已经从 PATH 自动找到了 SDK，就可能不再弹出提示。

#### 备用方式：直接编辑用户设置 JSON

找不到原提示时，也可以明确指定 SDK 路径。这是 Dart/Flutter 扩展支持的配置项，见 [Dart Code 设置说明](https://dartcode.org/docs/settings/#dartfluttersdkpath)。操作如下：

1. 按 `Cmd + Shift + P` 打开命令面板。
2. 输入并选择 **Preferences: Open User Settings (JSON)**，中文界面对应“首选项：打开用户设置(JSON)”。注意选择用户设置，不是默认设置。
3. 在文件最外层的 `{}` 内添加 `dart.flutterSdkPath`。
4. 按 `Cmd + S` 保存。

例如，文件中原本已有字号设置，可以这样添加：

```json
{
  "editor.fontSize": 14,
  "dart.flutterSdkPath": "/Users/nia/flutterSdk/flutter"
}
```

保留已有设置，在最外层 `{}` 内用英文逗号分隔配置项；若原来是空对象，只添加 `dart.flutterSdkPath` 即可。路径换成自己的实际位置。

**编辑器配置填写 SDK 根目录，终端 PATH 加入的是根目录下的 `bin`。** 两者不要填反，也不要填写成 `bin/cache/dart-sdk`，那是 Dart SDK 的位置。

保存后执行 **Developer: Reload Window**（开发人员：重新加载窗口），或重启 VS Code，再重新打开集成终端检查。用户设置适用于多个项目；若放在项目的 `.vscode/settings.json` 中则对该工作区生效，个人绝对路径不宜直接作为团队公共配置。

SDK 定位优先级见 [Dart Code SDK 查找规则](https://dartcode.org/docs/sdk-locating/)。显式配置的 SDK 路径与终端 PATH 最好指向同一份 SDK，避免编辑器和终端使用不同版本。

### 4. 执行环境检查

```bash
flutter doctor -v
```

这条命令会分别检查 Flutter、Android、Xcode、设备和网络。Flutter 一项通过，只说明 SDK 本身可用，不代表所有平台的环境都齐了。接下来按具体报错逐项处理。

## 四、复用 Android 环境

已有 Android 开发经验，这一步主要是检查现有工具链是否满足当前 Flutter 项目要求。

### 1. 检查 Android SDK

打开 Android Studio 的 SDK Manager，按[官方 Android 配置清单](https://docs.flutter.dev/platform-integration/android/setup)安装所需平台和工具，重点检查：

- Android SDK Platform 和 Build-Tools；
- Android SDK Command-line Tools；
- Android SDK Platform-Tools；
- 使用模拟器时需要的 Android Emulator 和系统镜像；
- 官方清单或项目构建要求的 CMake、NDK 等组件。

Android 平台版本按当前官方要求和项目配置选择，不要为了运行示例随意降低 `compileSdk` 或修改 Gradle 版本。

如果 Flutter 没有找到 SDK，先在 SDK Manager 确认实际路径。macOS 常见路径如下，但应以自己机器为准：

```bash
flutter config --android-sdk "$HOME/Library/Android/sdk"
```

### 2. 我遇到的 Command-line Tools 和许可问题

![flutter0_android_questions](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter0_android_questions.png)

第一次执行检查时，Android 部分出现：

```text
cmdline-tools component is missing.
Android license status unknown.
```

处理顺序是先补工具，再接受许可：

1. 打开 Android Studio → **Tools → SDK Manager**。
2. 进入 **SDK Tools**，勾选 **Android SDK Command-line Tools (latest)**。
3. 点击 Apply，等待安装完成，再执行下面的命令。

```bash
flutter doctor --android-licenses
flutter doctor -v
```

阅读并接受相关许可。若提示找不到 `sdkmanager`，先检查 Command-line Tools；若出现 Java 版本错误，查看 `flutter doctor -v` 显示的实际 Java 路径，再核对 JDK、Gradle、Android Gradle Plugin 的兼容关系。

原生 Android 工程能编译，不代表 Flutter 一定在使用同一套 JDK。排错时先确认工具实际使用的路径。

### 3. 准备设备

模拟器可以继续通过 Android Studio 的 Device Manager 创建和启动，系统镜像选择与主机架构匹配的版本。

真机则开启开发者选项、USB 调试，连接后在手机上授权。然后执行：

```bash
adb devices
flutter devices
```

`adb devices` 若显示 `unauthorized`，先处理手机授权；`flutter devices` 则应能列出准备运行的 Android 设备。

### 4. Multiple adb binaries found：电脑里有两份 adb

后续 Android 检查已经通过，但内部还有一条黄色提示：

```text
Multiple adb binaries found.
```

我的电脑同时安装了 Android SDK 中的 adb 和 Homebrew 的 android-platform-tools。先检查终端实际调用哪份：

```bash
which -a adb
adb version
```

如果希望统一使用 Android SDK 的版本，可以在 `~/.zshrc` 中加入：

```bash
export PATH="$HOME/Library/Android/sdk/platform-tools:$PATH"
```

执行 `source ~/.zshrc`，再检查 `which adb`。这能让终端优先使用 SDK 版本，但不保证 doctor 不再发现另一份 adb。如果设备连接正常，不必急着卸载；只有确认不再需要 Homebrew 那份时，才考虑移除重复安装。

遇到 Java/Gradle 错误时，查看 `flutter doctor -v` 显示的 Java 路径，再核对 JDK、Gradle 和 AGP 的兼容关系。我的环境使用 Android Studio 自带的 JDK。

## 五、准备 iOS、Web 和桌面环境

Flutter SDK 和 VS Code 是公共工具，各目标平台还需要自己的构建环境。**运行目标和开发电脑的操作系统是两回事**：支持跨平台，不代表任意一台电脑都能构建所有平台。

| 目标平台 | 开发主机 | 额外工具 |
| --- | --- | --- |
| Android | macOS、Windows、Linux | Android SDK、JDK、模拟器或真机 |
| iOS | macOS | 完整 Xcode、iOS 平台支持、模拟器或真机 |
| Web | macOS、Windows、Linux | 调试浏览器，例如 Chrome |
| macOS 桌面 | macOS | Xcode 等工具 |
| Windows 桌面 | Windows | Visual Studio 桌面 C++ 开发工具，与 VS Code 是不同软件 |
| Linux 桌面 | Linux | 对应发行版的编译工具和桌面开发依赖 |

完整要求见[官方各平台配置入口](https://docs.flutter.dev/platform-integration)。可以先跑通一个目标，再逐步添加其他平台。

### 1. iOS：Xcode、模拟器和 CocoaPods

我的第一次检查出现了两个问题：

```text
Unable to get list of installed Simulator runtimes.
CocoaPods not installed.
```

它们需要分别处理：前者与 Xcode 模拟器环境有关，后者与原生依赖管理工具有关。

![flutter0_xcode](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter0_xcode.png)

#### 配置完整 Xcode 和模拟器运行时

先安装并打开完整 Xcode，完成首次启动所需的组件安装。仅运行 `xcode-select --install` 安装命令行工具，不足以构建 iOS 应用。

如果 Xcode 安装在默认位置，可以按[官方 iOS 环境文档](https://docs.flutter.dev/platform-integration/ios/setup)执行：

```bash
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch
sudo xcodebuild -license
xcodebuild -downloadPlatform iOS
```

这些命令分别选择 Xcode、初始化工具、阅读并接受许可，以及下载 iOS 平台支持。也可从 Xcode 设置的 Components/Platforms 入口检查和安装模拟器运行时，名称随版本可能不同。

遇到“无法获取运行时列表”，先执行下面的命令查看具体错误：

```bash
xcode-select -p
xcrun simctl list runtimes
```

它不一定只是“没下载模拟器”：Xcode 路径错误、首次初始化未完成或模拟器服务异常也需要排查。先完成上述配置，再重新检查，避免盲目反复下载。

#### 安装 CocoaPods：先解决 Ruby 版本问题

我尝试用 gem 安装 CocoaPods 时遇到：

```text
ffi requires Ruby version >= 3.0, < 4.1.dev.
The current ruby version is 2.6.10.210.
```

这是 Ruby 与依赖版本不兼容。CocoaPods 依赖 Ruby，`ffi` 又是安装链中的依赖；这次失败的原因是使用了旧 Ruby，并不是 Flutter 项目写错了。

我的处理方式是通过 Homebrew 安装新版 Ruby，保留系统 Ruby。尚未安装 Homebrew 的读者，可先按 [Homebrew 官网](https://brew.sh/)的说明安装；Ruby 包信息见 [Homebrew Ruby 页面](https://formulae.brew.sh/formula/ruby)。

```bash
brew install ruby
```

安装后按照 Homebrew 输出配置 PATH。我的 Apple Silicon Mac 当时安装的是 Ruby 4.0.7，gem 可执行目录是 `4.0.0/bin`，所以在 `~/.zshrc` 中加入：

```bash
export PATH="/opt/homebrew/opt/ruby/bin:/opt/homebrew/lib/ruby/gems/4.0.0/bin:$PATH"
```

这里的 `4.0.0` 是该版本的 gem 目录，**不是所有人的固定值**；以后升级 Ruby 时也应重新确认。Intel Mac 的 Homebrew 前缀也可能不同，应按自己安装输出填写。

保存后执行：

```bash
source ~/.zshrc
which ruby
ruby -v
which gem
gem environment gemdir
gem install cocoapods
which pod
pod --version
flutter doctor -v
```

确认 `ruby`、`gem` 来自 Homebrew 后，再安装 CocoaPods，不加 `sudo`。我的最终检查已能识别 CocoaPods 1.17.0，Xcode 一项也通过了。这个版本是本次记录，不是要求读者固定安装的版本。

旧的系统 Ruby 可以保留，PATH 已决定优先使用新版，不需要删除 `/usr/bin/ruby`。也不建议只照报错逐个锁定旧版 `ffi`，后续依赖仍可能不兼容。Ruby 安装建议见 [CocoaPods 官方指南](https://guides.cocoapods.org/using/getting-started)。

如果只想安装 CocoaPods，也可以按 [Homebrew CocoaPods 页面](https://formulae.brew.sh/formula/cocoapods)执行 `brew install cocoapods`，由 Homebrew 管理依赖；它是另一条安装路线，不必与 `gem install cocoapods` 重复使用。

iPhone 真机还需要信任电脑、开启开发者模式并配置 Xcode 签名。本文先使用模拟器，运行步骤见下一节。

### 2. Web 和桌面：暂作了解

本次重点是 Android、iOS，其他平台可以以后再配。Web 可按[官方说明](https://docs.flutter.dev/platform-integration/web/setup)安装 Chrome 后执行 `flutter run -d chrome`。如果已有项目提示未配置 Web，保存或提交改动后，在项目根目录执行 `flutter create --platforms=web .` 补齐平台文件。

桌面端需要对应主机的工具链，例如 macOS 配置完成后可运行 `flutter run -d macos`，详见[官方 macOS 配置文档](https://docs.flutter.dev/platform-integration/macos/setup)。

## 六、用 VS Code 打开现有 flutter_demo

在 VS Code 中使用 **File → Open Folder**，打开 `MyProject/FlutterProject/flutter_demo`，确保这一层能看到 `pubspec.yaml`。

下面假设 `MyProject` 位于用户主目录；如果实际放在其他地方，请换成真实路径：

```bash
cd ~/MyProject/FlutterProject/flutter_demo
flutter pub get
flutter devices
```

`flutter pub get` 根据 `pubspec.yaml` 获取依赖，并结合锁文件解析版本。应用项目通常应提交 `pubspec.lock`，让团队使用一致的依赖版本。

VS Code 和 Android Studio 日常开发都应打开整个 Flutter 项目；仅处理原生工程时，再打开 `android/` 或 `ios/Runner.xcworkspace`。Android Studio 需安装 Flutter/Dart 插件，并使用 Flutter 运行配置。

运行可以任选一种方式：

- 在 VS Code 状态栏选择已配置好的 Android、iOS、Web 或桌面目标，然后按 `F5` 启动调试。
- 在终端执行下面的命令，其中设备 ID 来自 `flutter devices`。

```bash
flutter run -d <设备ID>
```

`<设备ID>` 是占位符，实际执行时需要替换且不保留尖括号。只有一个合适的目标设备时，也可以直接执行 `flutter run`。

第一次运行会下载依赖并构建对应平台工程，耗时通常比后续启动长。看到 Gradle 输出时，可以沿用 Android 开发的排查习惯，先定位具体错误和失败环节。

### 在 Android 模拟器上运行

先在 Android Studio 的 Device Manager 启动模拟器，再在项目根目录执行：

```bash
flutter devices
```

我的设备列表中出现了下面这个 Android 目标，说明 Flutter 已经识别到模拟器：

```text
sdk gphone16k arm64 (mobile) • emulator-5554 • android-arm64 • Android 17 (API 37) (emulator)
```

指定它运行：

```bash
flutter run -d emulator-5554
```

`emulator-5554` 是这次的设备 ID，其他人的模拟器 ID 可能不同，以自己的设备列表为准。`No wireless devices were found` 仅表示没有发现无线设备，不影响已经连接的模拟器。

首次运行先构建 APK，再安装并启动。`Running Gradle task 'assembleDebug'...` 表示仍在构建。这次解决网络和 NDK 问题后，我在 Android Studio 中运行成功：

![flutter0_android_run](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter0_android_run.png)

如果长时间没有进展，先查看终端最后的输出。可以按 `Ctrl + C` 停止当前命令，再通过详细日志重试：

```bash
flutter run -d emulator-5554 -v
```

#### NDK 安装不完整：改用已有版本后运行成功

第一次构建长时间没有进展，配置终端代理后开始下载依赖，随后又停在 NDK 安装阶段。取消 Android Studio 同步后，出现了明确报错：

```text
[CXX1101] NDK at .../ndk/28.2.13676358 did not have a source.properties file
```

原因是 **NDK 28.2 安装不完整**。项目原来的配置是：

```kotlin
ndkVersion = flutter.ndkVersion
```

本机 Flutter 默认指定 `28.2.13676358`，因此即使已经安装 NDK 30，也不会自动使用它。SDK Manager 中的 Installed 是汇总状态，需要勾选 **Show Package Details** 查看具体版本。

有两种处理方式：

- **重装指定版本**：停止构建，移走不完整的 `ndk/28.2.13676358` 目录，再通过 SDK Manager 安装该版本。不要自己补写 `source.properties`。[官方 NDK 安装说明](https://developer.android.com/studio/projects/install-ndk)
- **使用已有版本**：当前 Demo 没有额外的第三方原生插件，我将 `android/app/build.gradle.kts` 中原来的配置替换为：

```kotlin
ndkVersion = "30.0.16248370"
```

保存后重新同步，**最终在 Android Studio 中成功运行到模拟器**。这验证了 NDK 30 适用于当前 Demo，其他项目仍需检查插件要求并实际构建验证。[Flutter Android 构建配置](https://docs.flutter.dev/deployment/android#reviewing-the-gradle-build-configuration)

另外，终端代理不会自动应用到 Android Studio。IDE 下载不通时，应检查其 HTTP Proxy 设置；避免在两个编辑器中同时启动构建。

### 在 iOS 模拟器上运行

这一步我遇到了一个很容易忽略的问题：用 Xcode 打开整个 `flutter_demo` 文件夹后，没有看到预期的 Runner 工程和 iPhone 运行目标。

**VS Code 打开 Flutter 项目根目录；Xcode 则应打开 iOS 工程的 `ios/Runner.xcworkspace`。** 在项目根目录可以执行：

```bash
open ios/Runner.xcworkspace
```

打开后，在 Xcode 顶部选择 Runner 和已安装的 iPhone 模拟器。若没有可选设备，先在 Xcode 设置中安装 iOS 模拟器运行时，再通过 **Window → Devices and Simulators → Simulators** 检查或创建模拟设备。

我使用的 Xcode 26 也可以从 macOS 终端直接启动模拟器：

```bash
open -a Simulator
```

回到 VS Code 的项目终端，确认 iPhone 已被识别：

```bash
flutter devices
```

从输出中复制 iOS 模拟器对应的设备 ID，再运行：

```bash
flutter run -d <iOS模拟器设备ID>
```

设备 ID 要换成实际值，不保留尖括号。也可以直接在 VS Code 右下角选择这台 iPhone，然后按 `F5`。日常写 Dart 和调试可以继续使用 VS Code，不要求一直在 Xcode 里操作。

目前 Demo 已在 iOS 模拟器上运行成功。运行截图放在下面，与前面的 Android 结果对应；也可以点击计数按钮检查交互。

![flutter0_ios_run](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter0_ios_run.png)

## 七、先认清项目目录

```text
flutter_demo/
├── lib/
│   └── main.dart        # Dart 应用入口和初始页面
├── android/             # Android 宿主工程
├── ios/                 # iOS 宿主工程（若创建时包含该平台）
├── web/                 # Web 启动相关文件（若包含该平台）
├── macos/               # macOS 宿主工程（若包含该平台）
├── windows/             # Windows 宿主工程（若包含该平台）
├── linux/               # Linux 宿主工程（若包含该平台）
├── test/                # 测试代码
├── pubspec.yaml         # 依赖、资源和项目信息
└── pubspec.lock         # 已解析的依赖版本
```

平台目录是否存在，取决于创建项目时启用了哪些平台。它们连接各自的工具链，共享代码主要在 `lib/` 中。若缺少目标目录，先保存或提交现有修改，再在根目录使用 `flutter create --platforms=web .` 等命令补充平台；将 `web` 换成需要的平台，并检查生成的改动。

先从 `lib/main.dart` 开始：`main()` 是 Dart 入口，`runApp()` 挂载根 Widget。日常页面主要写在 Dart 层，不必为每个页面创建 Android Activity 或 iOS UIViewController。

`pubspec.yaml` 用于声明共享依赖、图片和字体；平台专属配置分别在 `android/` 和 `ios/` 工程中处理。

| 配置 | Android | iOS |
| --- | --- | --- |
| 权限 | 在 `android/app/src/main/AndroidManifest.xml` 中声明所需权限 | 在 `ios/Runner/Info.plist` 中填写相机、麦克风等权限用途说明 |
| 应用标识 | `android/app/build.gradle.kts` 中的 `applicationId` | Xcode → Runner target → Signing & Capabilities 中的 Bundle Identifier |
| 签名与平台能力 | Android 签名配置、Manifest 及相关平台配置 | Xcode → Signing & Capabilities 中的开发团队、签名和所需能力 |
| 最低系统版本 | Android 构建配置中的 `minSdk` | Xcode 中的 iOS Deployment Target；使用 CocoaPods 时需与 Podfile 的平台版本要求保持一致 |

声明权限或填写用途说明后，需要运行时授权的功能，仍应通过插件或原生 API 发起请求。两端权限机制不同，要分别配置和验证。

日常共享页面主要写在 `lib/`；需要调整 iOS 配置时，用 Xcode 打开 `ios/Runner.xcworkspace`，选择 Runner target，而不是只打开整个 Flutter 文件夹。

## 八、原生开发经验怎么迁移？

### Android 开发者

下面这些是帮助入门的类比，不是严格的一一替代。官方也提供了[面向 Android 开发者的 Flutter 指南](https://docs.flutter.dev/flutter-for/android-devs)。

| Android 中熟悉的概念 | Flutter 中先认识什么 | 需要注意的差别 |
| --- | --- | --- |
| Kotlin / Java | Dart | 语法和异步模型需要重新学习 |
| XML 布局、View | Widget 组合 | Widget 是不可变的配置描述，不是可随意修改的 View 实例 |
| Compose 的 `@Composable` | Widget 的 `build()` | 都根据状态描述 UI，但状态订阅和更新机制不同 |
| `LinearLayout` / Compose `Row`、`Column` | `Row`、`Column` | 名字容易理解，布局约束规则仍需单独学习 |
| `RecyclerView` / `LazyColumn` | `ListView.builder` | 都适合按需创建列表项，复用机制不同 |
| 页面跳转、返回栈 | `Navigator`、`Route` | Flutter 路由不等于 Android Activity |
| Gradle 中声明库依赖 | `pubspec.yaml`、`flutter pub get` | Android 平台构建仍由 Gradle 参与完成 |
| Kotlin 协程、`suspend` | `Future`、`async`、`await` | 都支持异步表达，但不能直接套用协程的调度语义 |
| 调用 Android 系统 API | 插件、平台通道 | 必要时仍可以编写 Kotlin/Java 实现 |

还有两个容易误会的地方：

**`StatefulWidget` 的状态放在 `State` 对象中。** Widget 自身仍是不可变配置。`setState()` 通知框架状态发生了变化，触发后续构建；不要把它当成命令式修改某个控件的方法。

**Widget 生命周期不等于 Activity 生命周期。** `initState()` 用于某个 State 的一次性初始化，`dispose()` 用于清理其持有的资源；应用进入前后台则是另一类生命周期事件，不能直接把它们对应成 `onCreate()`、`onDestroy()`。

### iOS 开发者

熟悉 UIKit，可以从组件、页面和导航的角度理解；熟悉 SwiftUI，则可以从“状态决定界面”的思路切入。官方分别提供了 [UIKit 迁移指南](https://docs.flutter.dev/flutter-for/uikit-devs)和 [SwiftUI 迁移指南](https://docs.flutter.dev/flutter-for/swiftui-devs)。

| iOS 中熟悉的概念 | Flutter 中先认识什么 | 需要注意的差别 |
| --- | --- | --- |
| Swift / Objective-C | Dart | 空安全、类型与异步机制需要重新学习 |
| UIKit `UIView`、SwiftUI `View` | Widget | Widget 是不可变配置，不是 UIKit 的可变视图实例 |
| SwiftUI `body` | `build()` | 都描述界面，但更新机制不同 |
| `HStack`、`VStack`、`ZStack` | `Row`、`Column`、`Stack` | 布局约束、尺寸和默认对齐规则不同 |
| `UITableView`、`UICollectionView` | `ListView.builder`、`GridView.builder` | 都可按需创建内容，复用机制不同 |
| `UINavigationController`、`NavigationStack` | `Navigator`、`Route` | Flutter 页面不等于一个 `UIViewController` |
| SwiftUI `@State` | `State` 与 `setState()` | 可用于理解局部状态，不能直接照搬属性包装器机制 |
| Swift `async/await`、`Task` | Dart `Future`、`async/await` | 调度、取消和并发模型不同 |
| Asset Catalog、SPM / CocoaPods | `pubspec.yaml` 中的资源和依赖 | iOS 原生依赖仍可能由 SPM / CocoaPods 管理 |

`initState()`、`dispose()` 也不能直接对应 `viewDidLoad`、`viewDidDisappear`。Flutter 的 State 创建与销毁、路由可见性、应用前后台是不同层面的事件。

已有的签名、权限和系统 API 经验仍然有用；需要平台能力时可以使用插件，或通过平台通道调用 Swift/Objective-C。Cupertino 组件提供 iOS 风格的界面，但仍需在 iOS 上验证交互和系统行为。

## 九、体验一次热重载

先点击 Demo 的 `+` 几次，再修改 `lib/main.dart` 中的页面标题并保存。VS Code 或 Android Studio 使用 Flutter 调试会话运行时，点击 **⚡ Hot Reload**；通过 `flutter run` 启动时，在对应终端按小写 `r`。

标题更新、计数保留，就完成了热重载验证。模拟器保持运行，日常改 UI 不需要每次停止重装。

![热重载效果](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter0_hot.png)

三种操作需要分清：

| 操作 | 适用情况 | 状态变化 |
| --- | --- | --- |
| Hot Reload，终端按 `r` | 调整大部分 Dart UI 和逻辑 | 通常保留现有状态，不重新执行 `main()`、`initState()` |
| Hot Restart，终端按 `R` | 需要重新执行 Dart 初始化逻辑 | 重启 Dart 应用，丢弃当前状态 |
| 停止后重新运行 | 修改原生代码或需要重新构建的平台配置 | 重新启动应用并按需构建 |

热重载主要用于 debug 模式，并非所有修改都能生效。遇到初始化值没有变化、原生代码没有更新等情况，先判断需要哪种重启方式。详见[官方热重载说明](https://docs.flutter.dev/tools/hot-reload)。

## 十、网络超时和其他排查方法

### 网络问题：安装 SDK 成功，不等于依赖下载都正常

我最初的 doctor 同时访问 GitHub 和 Google Maven 超时，后来只剩 Google Maven：

```text
A network error occurred while checking "https://maven.google.com/": Operation timed out
Doctor found issues in 1 category.
```

工具链检查通过后，网络仍可能影响 Android 依赖下载。

可以先在同一个终端测试：

```bash
curl -I --connect-timeout 15 https://maven.google.com/
curl -I --connect-timeout 15 https://github.com/
```

如果使用代理软件，找到它提供的“复制终端代理命令”，粘贴到准备运行 Flutter 的终端，再执行 `flutter doctor -v`。端口以软件实际设置为准，不要照抄别人的端口。

浏览器能访问，不代表终端也能访问。终端代理通常仅对当前会话及其后续进程生效；Android Studio、Gradle 还可能需要各自配置代理。先看日志中下载失败的地址，再处理对应工具的网络。

### 其他问题速查

| 现象 | 优先检查 |
| --- | --- |
| `flutter: command not found` | PATH 是否指向 SDK 的 `bin`，终端和 VS Code 是否重启 |
| 终端可以执行，编辑器不识别 Flutter | 是否打开包含 `pubspec.yaml` 的目录，扩展是否启用，SDK 路径是否一致 |
| Android toolchain 报错 | `flutter doctor -v` 中的 SDK、Java 路径和缺失组件 |
| 没有 Android 设备 | 模拟器是否启动，真机 USB 调试、授权和数据线是否正常 |
| 下载依赖失败 | 区分是 pub、Flutter SDK 资源还是 Gradle 仓库下载失败，再检查网络配置 |
| 更换文字却看不到变化 | 是否运行了当前项目、是否触发热重载、是否需要热重启 |

不要一遇到问题就执行 `flutter clean`。先读错误，它会删除构建产物，并不能修复错误的 SDK 路径、许可证或网络配置。更多问题可查[官方安装排错文档](https://docs.flutter.dev/install/troubleshoot)。

## 十一、这一篇的完成标准

同一个 `flutter_demo` 已在 Android 和 iOS 模拟器上运行成功，当前移动端学习所需的环境搭建完成。Android 端使用本机已有的 NDK 30；iOS 端的 Xcode 和 CocoaPods 也已通过实际运行验证。Web 和桌面暂不作为这一阶段的学习目标。

环境是否搭好，可以用四件事验收：

1. `flutter --version` 能显示版本。
2. `flutter doctor -v` 没有阻塞所选目标平台开发的问题。
3. `flutter devices` 能识别目标设备，`flutter_demo` 能在该设备上运行。
4. 修改页面文字后，能通过热重载看到结果。

接下来学习 Dart 基础、空安全和异步，再练习 Widget、布局、状态更新与导航。同一份页面分别在 Android 和 iOS 上运行，观察两端差异。

这次搭环境，也感受到了 AI 辅助学习的方便。从 Ruby 版本不兼容，到 NDK 安装不完整，再到找不到 iOS 模拟器，我一路把报错和截图发给 AI，跟着分析、修改、验证，最后把两端的 Demo 都跑了起来。以前可能要搜半天的问题，现在可以边问边试，学习新东西没那么容易卡住了。

遇到不懂的地方就问，让 AI 解释原因、给出步骤，再亲手跑一遍，把解决过程记下来。这样用 AI 陪着学，我觉得很靠谱。环境折腾完了，接下来就好好写代码，加油学吧。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
