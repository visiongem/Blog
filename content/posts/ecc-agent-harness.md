---

## title: "ECC：一个 21 万+ Star 的 AI 编程助手「操作系统」"

date: 2026-06-22T21:00:00+08:00
draft: false
categories: ["AI"]
tags: ["AI", "Claude Code", "Cursor", "Agent", "生产力"]

最近看到一个很不错的库，叫 [affaan-m/ECC](https://github.com/affaan-m/ECC)。

**ECC** 全称 Everything Claude Code，现在把自己定位成 AI 编程助手的 agent harness「操作系统」——不只是 Claude Code 的配置包，而是一套可跨平台复用的工作流系统。

截至目前，GitHub 上已有 **21 万+ Star、3 万+ Fork**（数字还在涨，以仓库页为准）。

作者是 Affaan Khan，2025 年 9 月在 Anthropic × Forum Ventures 黑客马拉松上靠 [zenith.chat](https://zenith.chat) 拿的第一名。ECC 是他把 10 个多月高强度日常使用和真实产品开发里沉淀下来的配置，整理成的一套完整系统。

它不是那种「收藏即学会」的 awesome list，也不是某个框架的封装。

可在 Claude Code、Codex、Cursor、OpenCode、Gemini 及其他 AI 智能体框架中通用。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/other/2026/ecc0.png)
---

## 它到底解决了啥

用 AI 写代码有几个痛点：

- AI 写完代码就跑，测试从来不写。让它补，它写了，跑不过。再改，还是跑不过。
- 让它改个功能，它一把梭改了五六个文件，中间也不确认步骤，最后 git diff 长得都不想看。
- 每次开新会话，项目约定全部清零。只能手动把「请用 kebab-case 命名文件」「请写测试」这些话再贴一遍。
- 让它跑个长任务，跑到一半会话断了，进度全丢，也不知道它到底改了什么。

ECC 就是盯着这些问题来的。所有内容均经过 10 个多月的高强度日常使用和真正的产品开发迭代，相当于给 AI 定制了一层标准工作流。

---

## 它是什么

ECC 相当于给常用的 AI 编程工具（Claude Code、Cursor、Copilot 这些）加了一个「专业模式」。

怎么加的呢？靠四个东西：

### Skills —— 把每次都要说一遍的话存起来

比如想让 AI 用 TDD 的方式写代码。没装 ECC 之前，每次都得解释：「先写测试用例，确认测试跑不过，再写实现，再跑测试，通过了才算完。」说三遍就不想说了。

装了 ECC 之后，触发 `tdd-workflow` 这个 Skill，AI 自动进入 TDD 模式，不用废话。

类似的还有 `/ecc:security-review`（安全审查）、`/ecc:plan`（先规划再动手）、`/ecc:refactor-clean`（清理死代码）。总共 271 个 Skill，覆盖测试、安全、重构、文档、部署这些高频场景。记住几个常用的就行，其他忘了可以 `/plugin list ecc@ecc` 查。

> 旧版短命令如 `/tdd` 已迁入 `legacy-command-shims/`，默认不启用。新用法以 Skill 和 `/ecc:`* 命令为准。

### Hooks —— 不用喊，自己跑

这个东西比 Skills 更省心。Skills 还要手动敲命令，Hooks 直接监听事件自动触发。

比如让 AI 跑个 `npm run build`，PreToolUse 钩子发现不在 tmux 会话里，自动弹一句「先开个 tmux 再跑，别问我怎么知道的」。不是拦着，只是提醒——长命令在裸终端里跑，断了真的想死。

Hooks 可以监听六种事件：工具执行前/后、发消息时、AI 回完话时、上下文压缩前、权限请求时。每个事件都能挂自定义脚本。说实话，大部分人不一定用得上，但 Stop 钩子（会话结束时自动存摘要）和 SessionStart 钩子（新会话开始时自动加载上次约定），这两个用了就知道有多香。

### Subagents —— 把活分给专业的人干

细想一下：主 AI 什么都要会，前端、后端、数据库、安全、构建。结果就是什么都懂一点，什么都不精。

ECC 的思路是——主 AI 当包工头，遇到专项任务就派子代理去干。Code Reviewer 只看代码质量，Security Reviewer 只查安全漏洞，Build Error Resolver 专门盯着编译报错。每个子代理只干自己擅长的事，免费用（开源 MIT 协议）。

光是从「一个 AI 干所有事」改成「包工头 + 专业工」，质量就提了一大截。

### Rules —— 让 AI 记住项目该怎么做

按语言分层，`rules/typescript`、`rules/golang`、`rules/kotlin`。每个规则文件写清楚了该语言的惯用写法、常见坑、命名约定。AI 加载这些规则之后写出来的代码不会东一榔头西一棒子。

---

## 几个设计理念值得提一下

精简指南里反复强调了三件事。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/other/2026/ecc1.png)

**把配置当调音，别当盖楼。** 原话是 "Don't overcomplicate — treat configuration like fine-tuning, not architecture。"不需要一次性启用所有组件。从几个常用 Skill 和两条规则开始，用着顺手再往上加。

**上下文窗口很贵。** 每多开一个 MCP 或插件，上下文窗口就被吃掉一块。作者的建议：配置 20-30 个 MCP 没问题，但同时启用的控制在 10 个以内，活跃工具不超过 80 个。Skills 同理——装 271 个没问题，但要清楚每个 Skill 加载时的上下文开销。

**Skills 是主体，Commands 只是过渡。** ECC 保留了 `commands/` 目录，但那是给旧版斜杠命令的兼容层。真正的逻辑放在 Skills 里。Skills 还能附带 codemap（代码导航索引），让 AI 快速定位项目结构而不消耗上下文去探索。作者的用法是——Skills 负责定义工作流，Commands 只做入口映射，后者迟早会被替掉。

---

## 跨平台这件事

ECC 最早是给 Claude Code 做的，但 v2.0 之后，同一套规则和技能可以跨七个平台用：Claude Code、Codex、Cursor、OpenCode、Gemini、Zed、GitHub Copilot。在 Claude Code 里配好的 `tdd-workflow`，到 Cursor 里还是同一套行为。不用重新折腾。

具体怎么装每个平台不一样，Claude Code 最省事——两行命令搞定。其他平台基本就是复制规则文件到对应的配置目录，README 里都写清楚了。

---

## 怎么装

Claude Code 用户最省事：

```bash
# 1. 添加市场
/plugin marketplace add https://github.com/affaan-m/ECC

# 2. 安装插件
/plugin install ecc@ecc

# 3. 手动把规则文件拷过去（插件不能自动分发 rules）
#    建议放到 ecc 命名空间，避免和其他规则冲突
git clone https://github.com/affaan-m/ECC.git && cd ECC
mkdir -p ~/.claude/rules/ecc
cp -R rules/common ~/.claude/rules/ecc/
cp -R rules/typescript ~/.claude/rules/ecc/
```

装完就有 67 个专业子代理、271 个技能、92 个命令。

有一点要特别注意：如果走了上面这套插件安装，就不要再跑 `./install.sh --profile full` 了。两套方式叠在一起会让技能文件重复，运行时会各种奇怪报错。插件、技能和命令，规则文件手动复制就完事了。

---

## 几个实际场景，就知道好不好用了

**新项目初始化：**

```bash
/ecc:plan "用 Next.js + Prisma 搭一个博客，支持 Markdown 编辑和 RSS"
```

以前 AI 可能直接开始创建文件了。现在 Plan 技能先让它列出计划——数据库怎么设计，API 路由有哪些，前端页面分几个模块，测试覆盖哪些点。看一眼，确认没问题，它才开始干活。干活的每一步 `tdd-workflow` 技能介入：先写测试，确认跑不过，再写实现，跑过，接着下一步。

**安全审查：**

代码写得差不多了，跑一句 `/ecc:security-review`。安全审查子代理按 checklist 逐项扫：有没有硬编码的密钥、SQL 拼接有没有问题、用户输入校验了没有、依赖库有没有已知漏洞。输出一份报告，标出严重程度和怎么修。比手动搜要全得多。

**跨会话记忆：**

昨天跟 AI 商量好了文件统一用 `kebab-case`，组件用 `PascalCase`，测试文件放在 `__tests__` 目录。今天开新会话，Stop 钩子在昨天结束的时候已经把约定写进了记忆文件，SessionStart 钩子今天一上来就加载好了。AI 直接按昨天的规矩干活，一个字都不用重复。

---

## 说清楚它不做什么

装了两周，有些东西得坦率说：

- 它不是替代 Claude Code 或 Cursor 的。还是在用原来的工具，ECC 只是让它更顺手。
- 规则文件得手动拷。插件能自动更新技能和命令，但 rules 不行。这不是 ECC 的 bug，是 Claude Code 插件系统的限制。
- `multi-`* 系列命令（并行多代理）需要额外装运行时 `npx ccg-workflow`，基础安装里不带。
- Pro 版是付费的，$19/seat/月，提供 GitHub App 私有仓库支持。开源部分 MIT 协议，永远免费。个人开发者用开源版完全够了。

---

## 最后说两句

21 万+ Star，3 万+ Fork，由 Affaan 主导维护、230+ 社区贡献者参与，每周发版，兼容七个 AI 编程平台。一个项目干了这么多事，持续了 10 个月，背后不是热爱是什么。

更让我觉得有意思的是——AI 编程助手这个领域还远没到「有标准」的阶段。每个工具都有自己的配置方式、自己的生态。ECC 正在用一种社区驱动的方式，把 Skills 目录结构、Hooks 事件模型、Subagents 角色设计推成某种跨平台配置范式，Cursor、Codex 等平台也在往类似方向走。

日常用 AI 写代码的话，花 10 分钟装上试试。反正不要钱。

---

## 相关链接

- [GitHub: affaan-m/ECC](https://github.com/affaan-m/ECC)
- [官方网站: ecc.tools](https://ecc.tools)
- [精简指南（英文）](https://github.com/affaan-m/ECC/blob/main/the-shortform-guide.md)
- [安全指南（英文）](https://github.com/affaan-m/ECC/blob/main/the-security-guide.md)

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
