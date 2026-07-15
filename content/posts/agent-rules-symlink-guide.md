---
title: "把 Agent 项目规则收回治理层：用软链接维护统一规则源"
date: 2026-07-15T21:00:00+08:00
draft: false
categories: ["AI"]
tags: ["AI", "Agent", "Claude Code", "Cursor", "Gemini CLI", "软链接", "项目治理"]
---

最近在整理项目里的 Agent 规则，发现一个问题：工具越来越多，规则也越来越容易散。因此今日份整理一下，怎么把 Agent 项目规则从某个工具目录里抽出来，放到项目治理层统一维护吧。

先说清楚一点：`.agent/` 不是 Agent 行业标准目录，也不是所有工具都会自动读取的目录。

它只是团队约定的 Single Source of Truth（唯一真实来源）。你也可以叫 `.ai/`、`.agents/`、`docs/agent/`。

共用入口也不一定非要叫 `.agent/AGENTS.md`。如果某个工具明确要求在仓库根目录放 `AGENTS.md`，那就按工具要求放一个薄入口，再指向 `.agent/AGENTS.md` 或你们真正的规则源。关键不在名字，而在这件事：

> 项目规则只维护一份，各工具入口再指向这份规则源。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/other/2026/agent_guide0.png)

这个“指向”可以有两种方式：

| 方式 | 适合场景 |
|---|---|
| 入口文件写明统一规则源位置 | 更通用，也更推荐，适合大多数工具 |
| 软链接到统一规则源的某些目录 | 工具确认会读取该路径，并且团队环境支持软链接时再用 |

所以不要一上来就把软链接当成标准答案。先确认工具真正读取哪里，再决定要不要用软链接。

这篇文章不是在介绍某个 Agent 工具的标准配置方式，而是在多个 Agent 工具共存时，怎么把项目规则从工具目录里抽离出来，建立一个统一治理入口。

软链接只是其中一种实现手段，而且不一定是首选。很多时候，先用各工具自己的入口文件指向统一规则源，会比直接软链接更稳。

真正想解决的是规则漂移、多工具同步、项目规范分散这些问题。

## 一、一句话方案

把项目里的 Agent 项目规则拆成两层：

```text
.agent/       # 团队约定的统一规则源，不是工具标准目录
.claude/      # Claude Code 入口层，具体文件按 Claude Code 约定
.cursor/      # Cursor 入口层，具体规则位置按 Cursor 文档
.gemini/      # Gemini CLI 入口层，具体入口按 Gemini CLI 文档
.codex/       # Codex 入口层，具体入口按 Codex 文档
```

团队真正维护的是：

```text
.agent/
```

各工具目录只负责“指路”。

这里的重点不是把 `.agent/` 硬塞给所有工具，也不是假设 `.cursor/rules`、`.gemini/rules` 这种路径一定有效。每个工具到底读取哪个文件、哪个目录，要看它自己的文档和团队实际验证结果。

可以把它想成公司里的公共资料柜：资料只放一份，每个办公室门口贴一张指路纸。以后改资料，去资料柜改，不要每个办公室各复印一份。

这里有个边界要先记住：软链接只能解决“路径复用”，不能解决“工具发现机制”。

真正让工具读到规则的，还是各工具自己支持的入口机制。例如 Claude Code 可以通过项目里的 `CLAUDE.md` 写明规则位置；其他工具也应该按它们各自官方文档和团队验证结果来配置入口。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/other/2026/agent_guide1.png)

## 二、为什么需要这样做

### 2.1 避免规则漂移

现在很多团队可能不只用一个 Agent 工具。

有人用 Claude Code，有人用 Cursor，有人用 Gemini CLI，有人用 Codex，也可能还有 CodeBuddy、Trae 或团队内部自研 Agent。每个工具都有自己习惯读取的配置目录和入口文件，项目里慢慢就会出现这些目录：

```text
.claude/
.cursor/
.gemini/
.codex/
.codebuddy/
```

目录下面又可能都有类似内容：

```text
rules/
commands/
skills/
agents/
CLAUDE.md
某些工具约定的项目入口文件
团队自定义的入口文件
```

如果每个工具都维护一份规则，很容易出现：Claude 的规则更新了，Cursor 忘了同步；Gemini 补了新说明，Codex 还在读旧版本。

最后项目里不是没有规范，而是有太多份“看起来差不多但其实不一样”的规范。

### 2.2 降低维护成本

这些内容本质上经常是同一类东西：

- 项目怎么启动；
- 代码风格是什么；
- 哪些目录不要乱改；
- 遇到特定任务时应该读哪些说明；
- 提交前要跑哪些检查；
- 哪些规则是团队红线。

如果每个工具都维护一份，新增、修改、删除规则时都要同步多处。次数一多，人一定会漏。

### 2.3 把规则提升到项目治理层

Agent 规则和普通文档不太一样。

普通文档过期，最多是人看了困惑；Agent 规则过期，它可能生成不符合团队规范的代码、采用旧架构方案，或者执行错误的操作流程。

所以项目规则应该像源码一样，有明确的唯一真实来源。

今天团队主要用 Claude Code，规则全写在 `.claude/` 里还说得过去。明天团队又加了 Cursor、Gemini CLI、Codex，如果每个工具目录都复制一份规则，项目治理就会被工具目录拆散。

统一之后，团队只需要记住一件事：

```text
真实规则只改 .agent/
```

`.claude/`、`.cursor/`、`.gemini/` 这些目录只是工具入口，不再承载第二份规则。

## 三、推荐目录结构

可以在项目根目录建立一个团队自定义目录：

```text
.agent/
├── AGENTS.md
├── rules/
├── commands/
├── skills/
└── agents/
```

这里每个目录都只是团队约定，不代表所有工具都原生支持：

| 路径 | 性质 | 用途 |
|---|---|---|
| `.agent/AGENTS.md` | 团队自定义入口 | 所有 Agent 共用的入口说明，名字也可以换成 `README.md` |
| `.agent/rules/` | 团队自定义目录 | 必须遵守的项目规则 |
| `.agent/commands/` | 团队自定义目录 | 项目命令、流程、常用操作，不等同于某个工具的 Command 标准目录 |
| `.agent/skills/` | 团队自定义目录 | 团队维护的任务说明文档，不一定等同于某个 Agent 产品里的 Skill 机制 |
| `.agent/agents/` | 团队自定义目录 | 子 Agent 或角色说明，不等同于某个工具的 Agent 标准目录 |

下面这个结构只是一种可能方案，用来展示统一规则源的设计方式，不代表所有 Agent 工具都必须采用相同目录结构。实际位置需要根据具体 Agent 工具支持的规则机制进行调整。

特别是 `.cursor/rules` 这类路径，不要直接照抄。Cursor 规则文件的放置位置、文件命名、是否需要放在特定目录，都应该以 Cursor 当前文档和团队验证结果为准。

```text
.claude/
├── CLAUDE.md
└── <可选链接或入口说明> -> ../.agent/...

.cursor/
├── <Cursor 当前版本约定的规则文件或目录>
└── <可选链接或入口说明> -> ../.agent/...

.gemini/
├── <Gemini CLI 当前版本约定的入口文件>
└── <可选链接或入口说明> -> ../.agent/...
```

如果确认某个工具会读取某个目录，再把那个目录软链接到 `.agent/`。否则更稳的做法是：在工具自己的入口文件里明确写“请阅读 `.agent/AGENTS.md`”。

## 四、先用入口文件指路

比起一上来就创建一堆软链接，我更建议先做一件事：在各工具自己的入口文件里写清楚统一规则源在哪里。

例如 Claude Code 的项目入口可以写：

```markdown
# CLAUDE.md

本文件是 Claude Code 的项目入口。

本项目的 Agent 共用规则统一维护在 `.agent/` 下，请先阅读：

- `.agent/AGENTS.md`

处理具体任务时，再按 `.agent/AGENTS.md` 的说明读取 `.agent/rules/`、`.agent/commands/` 和 `.agent/skills/`。
```

其他工具也类似，但入口文件名不要靠猜。比如 Cursor、Gemini CLI、Codex 等工具，应该按它们当前文档确认项目规则应该放在哪、入口文件叫什么、是否支持引用其他文件。

如果某个工具约定读取 `.xxx/PROJECT.md`，也可以只写类似内容：

```markdown
# PROJECT.md

本文件是某个 Agent 工具的项目入口。

本项目的 Agent 共用规则统一维护在 `.agent/` 下，请先阅读：

- `.agent/AGENTS.md`
```

这样做有几个好处：

1. 工具专属入口仍然存在；
2. 不同工具可以按自己的真实约定使用入口文件名；
3. 真正的项目规则仍然只维护 `.agent/AGENTS.md`；
4. 不会因为复制入口说明而造成规则漂移。

注意这里的重点是“入口指向统一规则源”，不是“所有 Agent 自动读取 `.agent/`”。

## 五、软链接是什么

软链接可以理解成文件系统里的“快捷方式”。

真实目录在这里：

```text
.agent/rules
```

但可以创建一个入口：

```text
.claude/rules -> ../.agent/rules
```

这样当某个工具或入口说明去读取：

```text
.claude/rules/code-style.md
```

实际访问的是：

```text
.agent/rules/code-style.md
```

也就是：

```text
多个入口，一份内容。
```

软链接本身不是复制。它只是告诉系统：“你从这里进去，其实要去另一个地方。”

但它不会替你解决工具行为问题。工具会不会读取 `.claude/rules/`、`.cursor/rules/`、`.gemini/rules/`，取决于工具自己的规则和团队入口配置。

## 六、什么时候再考虑软链接

入口文件能解决大部分问题。只有在确认某个工具会读取某个固定路径时，我才会考虑软链接。

比如团队已经验证某个工具会读取：

```text
<工具自己的规则目录>/rules
```

那可以把这个已验证路径指向 `.agent/rules`。假设以 `.claude/rules` 为例，命令可以写成：

```bash
mkdir -p .agent/rules .agent/commands .agent/skills .agent/agents
mkdir -p .claude

ln -s ../.agent/rules .claude/rules
```

为什么是 `../.agent/rules`，不是 `.agent/rules`？

因为软链接的目标路径通常按“软链接所在目录”来理解。`.claude/rules` 位于 `.claude/` 目录下，从 `.claude/` 回到项目根目录需要 `..`，再进入 `.agent/rules`，所以目标写成：

```text
../.agent/rules
```

如果目标路径已经存在，先看清楚它是什么：

```bash
ls -la .claude/rules
readlink .claude/rules
```

如果确认它只是旧软链接，可以用替换旧链接的写法：

```bash
ln -snf ../.agent/rules .claude/rules
```

但如果 `.claude/rules` 是真实目录，不是软链接，就不要直接覆盖。先把里面已有规则迁移到 `.agent/rules/`，再处理目录。

所以软链接这一节更像“可选增强”，不是第一步。先统一入口，再决定哪些已验证路径需要链接。

## 七、`.agent/AGENTS.md` 写什么

`.agent/AGENTS.md` 是团队约定的共用入口。这个名字不是强制的，`.agent/README.md` 也可以。

它不需要很长，只要回答几个问题：

```markdown
# AGENTS.md

本项目的 Agent 共用规则统一维护在 `.agent/` 下。

## 目录说明

- `.agent/rules/`：必须遵守的项目规则。
- `.agent/commands/`：团队自定义的项目常用命令和标准流程。
- `.agent/skills/`：团队自定义的特定任务操作手册。
- `.agent/agents/`：团队自定义的子 Agent 或角色说明。

## 工作原则

- 不要覆盖用户已有未提交改动。
- 修改业务代码前，先确认涉及模块和职责边界。
- 遇到日志、富文本、网络、数据库等专项任务时，先阅读对应说明。
- 提交前执行项目要求的验证命令。

## 维护约定

- 真实规则只维护 `.agent/`。
- 不要在 `.claude/`、`.cursor/`、`.gemini/` 下维护第二份规则。
- 如果工具目录需要读取规则，通过入口文件或软链接指向 `.agent/`。
```

这份文件适合作为所有 Agent 进入项目后的第一站。

## 八、怎么确认软链接正确

查看目录：

```bash
ls -la .claude .cursor .gemini
```

你会看到类似：

```text
rules -> ../.agent/rules
commands -> ../.agent/commands
skills -> ../.agent/skills
```

也可以直接看某个链接指向：

```bash
readlink .claude/rules
```

如果输出：

```text
../.agent/rules
```

说明链接指向没问题。

还可以真的读一个文件试一下：

```bash
touch .agent/rules/code-style.md
ls .claude/rules/code-style.md
```

如果 `.claude/rules/code-style.md` 能看到同一个文件，就说明入口和真实目录连上了。

但这只能证明软链接没问题，不能证明某个 Agent 工具会自动读取它。工具侧还要单独验证：打开项目后，让工具执行一个依赖规则的任务，看它是否真的按入口说明读到了 `.agent/`。

## 九、删除软链接会删除真实文件吗

通常不会。

比如：

```bash
rm .claude/rules
```

删除的是 `.claude/rules` 这个链接本身，不会删除 `.agent/rules`。

我自己的习惯是：删除前先看一眼它是不是链接。

```bash
ls -la .claude/rules
```

如果看到类似：

```text
.claude/rules -> ../.agent/rules
```

再删除链接本身。

重点就一句：确认路径类型后再删，不要凭感觉删规则目录。

## 十、Git 和 Windows 要注意什么

Git 会把软链接作为一种特殊文件类型记录，保存链接目标路径，而不是保存目标目录内容。

比如 Git 会记录：

```text
.claude/rules -> ../.agent/rules
```

如果想确认 Git 看到的是软链接，可以执行：

```bash
git ls-files -s .claude/rules
```

软链接在 Git 里通常会显示成 `120000` 这种 mode：

```text
120000 <hash> 0 .claude/rules
```

`120000` 表示 Git 记录的是软链接类型，不是普通文件，也不是普通目录。

不过 clone 之后的行为和操作系统、文件系统、Git 配置都有关系。macOS、Linux 通常比较自然，Windows 上要额外确认。

几个容易影响结果的点：

| 影响项 | 可能结果 |
|---|---|
| 文件系统是否支持符号链接 | 不支持时，链接可能无法按预期创建或检出 |
| Windows 是否开启 Developer Mode | 未开启时，创建符号链接可能需要管理员权限 |
| Git 的 `core.symlinks` 配置 | 为 `false` 时，Git 可能把链接检出成普通文本文件 |
| 团队成员使用 Git Bash、WSL 还是原生终端 | 不同环境对链接的体验不一样 |

可以查看当前仓库配置：

```bash
git config core.symlinks
```

团队里有 Windows 用户时，最好在 Windows 机器上实际拉取一次项目，然后确认三件事：

1. clone 后仍然是软链接；
2. 链接仍指向 `.agent/`；
3. Agent 工具实际能通过入口说明读到统一规则源。

不要只在 macOS 上验证完就默认全团队都没问题。

## 十一、什么时候适合用软链接

软链接方案适合这些场景：

- 已经确认某个工具会读取某个固定目录；
- 多个入口需要访问同一批 rules、commands、skills；
- 各工具对目录格式要求差异不大；
- 团队希望先低成本统一规则源；
- 不想维护额外的同步脚本；
- 不希望每次改规则都复制多份文件。

它最大的优点是简单：没有生成脚本，没有复杂构建，没有额外依赖，文件系统自己就能完成路径指向。

但前提还是那句：工具真的会读这个路径。不然链接建得再漂亮，也只是目录结构好看。

## 十二、什么时候软链接不够用

软链接不是万能的。

如果不同工具需要的格式完全不同，比如：

```text
.agent/skills/       # 通用 Markdown 源
.claude/commands/    # 某个工具需要自己的命令格式
.codex/agents/       # 某个工具需要自己的角色 frontmatter
.cursor/rules/       # 某个工具需要另一套 rule 格式
```

这时软链接只能共享文件，不能转换格式。

如果需要把一份源文件转换成多种工具专用格式，就应该引入同步脚本，例如：

```bash
python3 tools/sync_agent_configs.py
```

也就是说：

| 需求 | 更适合的做法 |
|---|---|
| 同一份内容，且工具确认会读取某个固定路径 | 软链接 |
| 一份源文件，生成多种格式 | 同步脚本 |
| 每个工具都有不同字段、frontmatter、命令格式 | 同步脚本或模板生成 |

不要一开始就上很重的生成系统。先用入口文件把统一规则源定下来；如果某个工具确实需要读取固定路径，再补软链接。等真的出现格式转换需求时，再升级为脚本生成。

## 十三、推荐落地步骤

可以按这个顺序改造项目：

1. 先定团队约定：统一规则源放在哪里。可以叫 `.agent/`，也可以换成你们更喜欢的名字。

2. 建立目录：

```bash
mkdir -p .agent/rules .agent/commands .agent/skills .agent/agents
```

3. 把已有规则、命令、技能移动到 `.agent/`。

4. 创建共用入口，例如：

```text
.agent/AGENTS.md
```

如果某个工具明确要求根目录 `AGENTS.md` 或其他固定文件名，就在那个位置放薄入口，让它指向 `.agent/AGENTS.md`，不要把规则正文复制过去。

5. 保留工具专属入口文件，但只写薄说明，让工具去读统一规则源。

6. 如果某个工具已验证会读取固定目录，再考虑软链接。下面只以 `.claude/` 为例：

```bash
ln -s ../.agent/rules .claude/rules
ln -s ../.agent/commands .claude/commands
ln -s ../.agent/skills .claude/skills
```

7. 如果目标路径已经存在，先判断它是链接还是目录。链接可以按需替换，真实目录要先迁移内容。

8. 在团队文档里明确：

```text
只维护 .agent/。
不要直接在 .claude/、.cursor/、.gemini/ 下复制规则。
工具目录里的 rules、commands、skills 应该通过入口文件或软链接指向 .agent/。
```

9. 找一台新环境拉取项目，跑一次验证命令：

```bash
ls -la .claude .cursor .gemini
readlink .claude/rules
git ls-files -s .claude/rules
git config core.symlinks
```

10. 分别用团队正在使用的 Agent 工具打开项目，确认它们真的通过入口说明读到了 `.agent/`。

这一步很重要。别只在已经配置好的电脑上看结果。

## 十四、一个完整示例

以下结构用于展示统一规则源的设计方式，不代表所有 Agent 工具必须采用相同目录结构。

最终可以形成这样的结构：

```text
project-root/
├── .agent/
│   ├── AGENTS.md
│   ├── rules/
│   │   ├── code-style.md
│   │   └── git-workflow.md
│   ├── commands/
│   │   ├── verify.md
│   │   └── release.md
│   ├── skills/
│   │   ├── android-clean-architecture.md
│   │   └── logging.md
│   └── agents/
│       ├── reviewer.md
│       └── explorer.md
│
├── AGENTS.md                  # 可选：如果某个工具要求根目录入口，只做指路
│
├── .claude/
│   ├── CLAUDE.md              # Claude Code 入口，写明去读 .agent/AGENTS.md
│   └── <已验证路径> -> ../.agent/...
│
├── .cursor/
│   └── <Cursor 文档约定的规则文件或目录>
│
└── .gemini/
    └── <Gemini CLI 文档约定的入口文件>
```

如果 Cursor 当前版本要求把规则放在某个固定位置，就按 Cursor 文档来；如果 Gemini CLI 要求另一个入口，也按它的文档来。不要为了统一外观，强行让所有工具目录长得一样。

从此以后，团队只需要维护：

```text
.agent/
```

其他目录只是给不同 Agent 工具看的入口。

## 十五、几个关键点

1. `.agent/` 是团队约定的 Single Source of Truth，不是行业标准目录。
2. 不要假设所有 Agent 都会自动读取 `.agent/`，要通过各工具入口指向它。
3. `.agent/commands/`、`.agent/skills/`、`.agent/agents/` 都是团队自定义目录，不等同于工具标准。
4. 入口文件指路是更通用的第一步，软链接只在工具路径已验证、团队环境支持时再用。
5. 软链接适合“同一份内容，多处读取”，不负责格式转换。
6. 删除链接前先用 `ls -la` 看清楚，不要凭感觉删规则目录。
7. Windows、文件系统和 `core.symlinks` 会影响 clone 后的软链接行为，要实际验证。
8. 等工具格式真的分叉后，再考虑同步脚本或模板生成。

这类规则整理一开始看起来不大，但项目里 Agent 工具越多，越需要一个统一入口。先把真实规则从工具目录里抽出来，放到项目治理层，后面维护起来会轻松很多。

今日份整理先到这。

____

🌈关注我吖~❤️

公众号：**妮K妮K妮**
