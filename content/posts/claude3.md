---
title: '第 4 篇：别把所有规则塞进 CLAUDE.md——CLAUDE.md、Rules、Skill、Hook 怎么分工'
date: 2026-08-13T21:00:00+08:00
draft: true
tags: ["AI", "Claude Code", "Codex", "Agent", "项目治理"]
categories: ["AI"]
---

前面三篇，我们已经解决了三个问题：

**第 1 篇：**

> CLAUDE.md 是什么？

它不是项目百科全书，而是 AI 在项目中工作时需要知道的长期规则。

**第 2 篇：**

> 什么值得写进去？

重点记录 AI 难以自行确认，但又会影响工作结果的隐性知识、项目约定和工作要求。

**第 3 篇：**

> 已经写了一大堆怎么办？

定期健检，删除过期规则，把不属于 CLAUDE.md 的内容迁移到更合适的位置。

到了这里，其实还剩下一个非常重要的问题：

> **如果不是所有规则都应该放进 CLAUDE.md，那它们应该放在哪里？**

很多人使用 Coding Agent 时，会把所有需求都写成：

```text
CLAUDE.md
```

然后慢慢变成：

```text
CLAUDE.md
├── 项目介绍
├── 架构规范
├── 编码规范
├── 测试流程
├── 发布流程
├── Git 规范
├── 安全规则
├── 前端规则
├── 后端规则
├── 数据库规则
├── 各种历史问题
└── 一大堆 Don't
```

最后一个文件承担了所有职责。

但真正合理的设计应该更像：

```text
                    Agent 工作体系
                          │
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
   长期项目规则       特定任务流程       强制执行机制
        │                 │                 │
   CLAUDE.md / Rules     Skill            Hook / CI
```

也就是说：

> **不是把所有规则都告诉 AI，而是让每条规则进入最适合它的机制。**

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/ai/2026/claude3.png)

---

## 一、先建立一个最重要的判断模型

以后遇到一条规则，不要先问：

> “我要不要写进 CLAUDE.md？”

先问三个问题：

### 1. 这条规则影响多大范围？

```text
所有项目
    ↓
整个项目
    ↓
某个目录
    ↓
某类任务
```

---

### 2. 什么时候需要它？

```text
每一次任务
    ↓
某类任务
    ↓
偶尔使用
```

---

### 3. 必须严格执行吗？

```text
希望 AI 遵守
    ↓
希望 AI 检查
    ↓
绝对不能违反
```

把这三个维度放在一起，就很容易理解各种机制之间的区别。

---

## 二、CLAUDE.md：长期、稳定、项目级的规则

先从最熟悉的开始。

CLAUDE.md 最适合放：

> **整个项目长期有效，而且 Agent 经常需要知道的规则。**

例如：

```md
# Project Rules

- UI 层禁止直接访问 DataSource。
- 所有金额统一使用分作为存储单位。
- 修改 API 后必须同步更新 Mock。
- 修改数据库 Schema 必须创建 Migration。
```

这些规则有几个共同特点：

```text
稳定
+
项目级
+
经常相关
+
无法完全从代码推断
```

Anthropic 当前文档把项目级 `CLAUDE.md` 定位为团队共享指令，并建议其中记录项目架构、编码标准和常见工作流程；同时支持通过 `@path` 导入其他文件。

所以：

> **CLAUDE.md 更像项目的“长期工作协议”。**

---

### 但“项目级”不代表什么都放根目录

这是一个非常容易产生的误区。

假设项目结构：

```text
project/
├── android/
├── backend/
├── frontend/
└── scripts/
```

现在有一条规则：

```text
frontend 页面修改后必须执行截图测试。
```

这条规则没有问题。

但如果把它写进根目录 CLAUDE.md，那么 Agent 修改 `scripts/deploy.sh` 时，也会看到这条和前端完全无关的规则，这显然没有必要。

所以：

> **规则的作用范围越小，就越应该靠近它实际生效的位置。**

---

## 三、Rules：把局部规则放到局部

对于 Claude Code 来说，可以通过 `.claude/rules/` 等机制进一步组织规则。

它解决的是一个问题：

> **规则本身有价值，但没有必要让它成为整个项目的全局规则。**

例如：

```text
.claude/
└── rules/
    ├── android.md
    ├── backend.md
    └── testing.md
```

可以分别描述：

```text
android.md
    ↓
Android 相关规则

backend.md
    ↓
后端规则

testing.md
    ↓
测试相关规则
```

这样就形成：

```text
CLAUDE.md
    ↓
项目级规则

Rules
    ↓
更细粒度的规则
```

### 举个 Android 项目的例子

根目录：

```md
# CLAUDE.md

- 所有业务逻辑必须经过明确的业务层。
- 修改代码后根据影响范围执行对应检查。
- 不要直接修改生成文件。
```

Android 相关规则：

```md
# Android Rules

- UI 状态统一通过 UiState 暴露。
- 不允许 View 直接访问 Repository。
- 新页面遵循现有 Feature 模块结构。
```

数据库相关规则：

```md
# Database Rules

- Schema 修改必须包含 Migration。
- Migration 必须增加对应测试。
```

这就比全部塞到根目录更合理。

---

### 但 Rules 和 CLAUDE.md 不要理解成“两个完全不同的东西”

这里需要特别注意。

它们本质上都属于：

> **给 Agent 的规则。**

区别主要在：

```text
组织方式
+
作用范围
+
加载时机
```

所以不要陷入“CLAUDE.md 是规则，Rules 是另外一种规则”的误区，更准确的理解是：

```text
项目指令体系
│
├── 全局 / 项目级指令
│      └── CLAUDE.md
│
└── 更细粒度指令
       └── Rules
```

具体加载方式、匹配规则和优先级应该以当前 Agent 的官方文档为准，不要因为 Claude Code 和 Codex 都存在“规则文件”，就默认两者机制完全一致。

---

## 四、Skill：当规则变成“一个完整流程”

这是整个体系里非常重要的一次升级。

假设有一个发布流程：

```text
发布 Android 新版本：

1. 修改 versionCode
2. 修改 versionName
3. 检查 Release Notes
4. 执行单元测试
5. 执行 lint
6. 构建 Release APK
7. 上传测试环境
8. 验证安装
9. 创建 Tag
10. 创建 Release
```

当然可以把它全部写进 CLAUDE.md，但这样做非常糟糕。

因为不是每一次写代码都在发布版本。如果每次 Agent 工作都带着这十条流程，就会增加无关上下文。

所以更适合放进：

```text
Skill
```

把它理解成：

> **遇到某类任务时，Agent 可以调用的一套专业工作方法。**

---

### Skill 和 CLAUDE.md 最大的区别

可以用一句话理解：

> **CLAUDE.md 是“平时应该知道什么”，Skill 是“做某类事情时应该怎么做”。**

例如 CLAUDE.md 里有一条长期规则：

```md
- 修改数据库 Schema 必须创建 Migration。
```

而 `database-migration` 这个 Skill 里面可以定义：

```text
1. 检查当前 Schema
2. 分析版本差异
3. 创建 Migration
4. 更新测试
5. 执行 Migration Test
6. 检查回滚风险
```

只有当真在做数据库迁移时，才需要调用这个 Skill。

所以：

```text
CLAUDE.md
    ↓
知道规则

Skill
    ↓
执行流程
```

---

### Skill 特别适合什么？

一般来说，可以优先考虑这几类：发布流程（`release`）、数据迁移（`database-migration`）、UI 检查（`ui-review`）、PR Review（`code-review`）、特定测试流程（`integration-test`）。

这些任务都有一个共同特点：

> **不是每次开发都需要，但一旦触发，就有一套相对固定的步骤。**

这正是 Skill 的价值。OpenAI 当前 Codex 的官方资料也把 Skills 描述为可以保存、复用工作流的机制，并直接将“Save workflows as skills”作为 Codex 的使用场景。

---

## 五、Hook / CI：如果这件事情真的不能犯错

现在进入另外一个非常重要的概念：

> Hook。

很多人会把“不要提交密钥”写进 CLAUDE.md。

这当然有用。

但它只能告诉 AI：

> “不要这么做。”

它不能保证“这件事情永远不会发生”。

假设：

```text
API_KEY=xxxx
```

不小心进入 Git。这时候最可靠的方案不是 CLAUDE.md，而是：

```text
Hook / CI / 安全扫描
```

例如：

```text
git commit
    ↓
Secret Scan
    ↓
发现 API Key
    ↓
阻止提交
```

这和：

```text
AI 看到规则
    ↓
AI 记住规则
    ↓
AI 自己决定遵守
```

完全不是一个可靠性等级。

---

### 把“提示”与“强制”彻底分开

这是这篇最值得记住的一个概念：

> **Prompt 是指导，自动化是约束。**

#### 情况一

```text
修改 Kotlin 后建议运行对应测试。
```

适合 CLAUDE.md，因为这是工作习惯。

#### 情况二

```text
修改支付模块必须执行 PaymentTest。
```

可以 `CLAUDE.md + CI` 双层配合。

#### 情况三

```text
任何包含 API Key 的提交都必须阻止。
```

应该在 `Hook + CI + Secret Scanner`，因为这是安全边界。

---

### 为什么 Hook / CI 比 CLAUDE.md 可靠？

假设 CLAUDE.md 里写：

```md
# Rules

绝对不能修改 production 配置。
```

Agent 可能理解了规则，但也可能因为当前任务特殊、上下文冲突、规则被忽略或模型判断错误，最后还是修改了。

而 CI：

```text
Pull Request
      ↓
检查
      ↓
失败
      ↓
无法合并
```

它不需要模型“记住”。

所以：

> **越接近“绝对不能发生”的事情，越应该从 Prompt 下沉到自动化系统。**

---

## 六、CLAUDE.md 是索引，不是知识库

还有一类东西：Agent 根本不应该记住。

例如：

```text
API 完整文档
数据库设计文档
产品需求
品牌规范
部署说明
```

这些信息可能非常长，没必要复制进 CLAUDE.md。

更合理的是：

```text
CLAUDE.md
    ↓
告诉 Agent 去哪里找

docs/api.md
docs/database.md
docs/release.md
    ↓
真正的详细内容
```

所以：

> **CLAUDE.md 是索引，不是知识库。**

---

## 七、一个真实项目可以怎么组织？

假设是一个 Android 项目，可以设计成：

```text
project/
│
├── CLAUDE.md
│
├── .claude/
│   ├── rules/
│   │   ├── android.md
│   │   ├── architecture.md
│   │   └── testing.md
│   │
│   └── skills/
│       ├── release/
│       ├── database-migration/
│       └── ui-review/
│
├── docs/
│   ├── architecture.md
│   ├── api.md
│   ├── database.md
│   └── release.md
│
├── app/
├── feature-home/
├── feature-login/
└── ...
```

整体关系：

```text
CLAUDE.md
    │
    ├── 项目级长期规则
    │
    ├── 指向重要文档
    │
    └── 指明关键工作方式
           │
           ↓
       Rules          Skills          docs        Hook / CI
           │               │               │              │
           └── 更细粒度规则  └── 特定任务流程  └── 详细知识  └── 强制检查
```

这样整个系统就不会全部挤在一个文件里，各司其职。

---

## 八、判断表 + 最容易犯的三个错误

以后新增一条规则，可以直接看这个表：

| 内容           | 推荐位置              | 原因           |
| ------------ | ----------------- | ------------ |
| 项目级长期规则      | CLAUDE.md         | Agent 经常需要知道 |
| 项目隐性业务规则     | CLAUDE.md         | 不容易从代码推断     |
| 某目录专属规则      | Rules / 目录级指令     | 缩小作用范围       |
| 某类任务的完整流程    | Skill             | 按需加载         |
| 详细技术文档       | docs              | 不需要每次加载      |
| API 完整说明     | docs              | 内容太长且变化频繁    |
| “建议运行测试”     | CLAUDE.md         | 工作习惯         |
| “必须通过测试才能合并” | CI                | 强制保证         |
| “禁止提交密钥”     | Hook / CI         | 安全边界         |
| 一次性任务要求      | 当前 Prompt / Issue | 不应该永久保存      |

### 错误一：把所有东西都塞进 CLAUDE.md

结果：

```text
CLAUDE.md
500 行
```

然后任何任务都要加载全部规则。

这不是工程化，而是把所有东西堆在一起。

### 错误二：把 Skill 当成“更长的 CLAUDE.md”

Skill 不是超长规则文件，它更适合：

```text
一个明确的任务
+
一套执行流程
+
必要的工具或资料
```

所以“项目必须使用 xxx 架构”应该是规则，而“如何执行一次完整的数据库迁移”更像 Skill。

### 错误三：把安全问题交给 AI 自觉

例如“不要提交密码”这句话可以存在，但不能只有这句话。

真正的安全保障应该是：

```text
AI 提醒
+
Hook
+
CI
```

形成多层防线。

---

## 九、最终可以形成一个“规则漏斗”

把今天学到的内容浓缩成一张图：

```text
                    所有项目知识
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
         Agent 能自己发现？       不能自己发现？
              │                     │
              ↓                     ↓
          不必写规则          是否长期有效？
                                  │
                         ┌────────┴────────┐
                         ↓                 ↓
                       是                 否
                         │                 │
                         ↓                 ↓
                    是否全局适用？       特定任务？
                         │                 │
                   ┌─────┴─────┐           ↓
                   ↓           ↓         Skill
                  是           否
                   │           │
                   ↓           ↓
              CLAUDE.md      Rules
                   │
                   ↓
              是否必须强制？
                   │
                   ↓
              Hook / CI
```

这个漏斗其实比记住各种文件名更重要。因为未来即使 `CLAUDE.md`、`Rules`、`Skill`、`Hook` 这些名字可能会变化，但背后的设计思想不会轻易变化：

> **按作用范围、触发时机和可靠性分配信息。**

---

## 这一篇真正需要记住的 5 句话

### ① CLAUDE.md 管长期规则

> AI 在这个项目里长期应该知道什么？

### ② Rules 管更细的范围

> 哪些规则只针对某个目录、模块或领域？

### ③ Skill 管完整流程

> 当我要做某类任务时，应该按照什么步骤完成？

### ④ Hook / CI 管强制约束

> 什么事情无论 AI 怎么想，都绝对不能发生？

### ⑤ 不要让一个文件承担所有职责

真正成熟的 Agent 项目不是：

```text
一个超级 CLAUDE.md
```

而是：

```text
规则
+
文档
+
Skill
+
自动化检查
```

各司其职。

---


到这里，我们已经把：

```text
CLAUDE.md
↓
Rules
↓
Skill
↓
Hook / CI
```

这套基本架构建立起来了。

但还有一个特别实际的问题：

> **如果我同时使用 Claude Code、Codex、Cursor、Gemini CLI 等多个 Agent，难道我要维护四五套规则吗？**

这也是我们在实际使用多个 Coding Agent 时很容易遇到的问题。

下一篇再来解决这个问题：

* `CLAUDE.md`
* `AGENTS.md`
* Cursor Rules
* Gemini CLI
* `.agent/`
* Git Soft Link / Symbolic Link
* 如何建立“一份规则，多种 Agent 共享”的目录结构

最终把整个系列从：

> **“如何写规则”**

进一步推进到：

> **“如何管理一套真正可维护的 Agent 工程配置”。**

今日份学习先到这。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**