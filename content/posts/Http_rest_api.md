---
title: "HTTP 基础与 REST API 入门"
date: 2026-07-07T10:00:00+08:00
draft: false
categories: ["Web"]
tags: ["HTTP", "REST", "API"]
---

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/other/2026/http_rest_api0.jpeg)

前后端交互一直是入门时的坎。URL、请求方法、状态码、CORS——每个概念单独看都不难，串在一起用的时候容易乱。

最近把这块系统捋了一遍，整理一下。

## 一、API

API（Application Programming Interface）是一套软件之间交互的约定，规定了请求方式、参数格式以及返回结果。

## 二、REST 的核心思路

API 有很多种风格，REST 是目前最流行的一种。

REST 全称是 Representational State Transfer，也就是说：

> **用 URL 表示「要操作的东西」，用 HTTP 方法表示「想怎么操作它」。**

URL 定位资源，方法定位动作。比如操作「用户信息」：

- 资源地址：`https://api.example.com/users`
- 想干什么？增、删、改、查？

这就引出四个 HTTP 方法。不过先停一下。

**REST 是一种设计风格，不是协议**。HTTP API 可以遵循 REST，也可以用别的设计方式，比如 RPC：

```
REST 风格：  GET /users/1        → 用路径定位资源
RPC 风格：   /getUser?id=1       → 用函数调用定位资源
```

两种都能干活，只是 REST 更强调「以资源为中心」，目前社区用得更多。接下来看 REST 的四个方法。

### 2.1 四个动词

| 方法 | 干什么 | 类比 |
|------|--------|------|
| **GET** | 查询/获取数据 | 看菜单 |
| **POST** | 创建新数据 | 下单 |
| **PUT** | 完整替换资源 | 整页重写 |
| **PATCH** | 部分更新资源 | 改一个字 |
| **DELETE** | 删除数据 | 取消订单 |
| **OPTIONS** | 预检请求（CORS 用） | 先敲门问问能不能进 |

以「文章」资源为例：

```
GET    /articles        → 获取文章列表
GET    /articles/5      → 获取 id=5 的文章
POST   /articles        → 新建文章
PUT    /articles/5      → 完整替换 id=5 的文章
PATCH  /articles/5      → 部分更新 id=5 的文章
DELETE /articles/5      → 删除 id=5 的文章
```

PUT 和 PATCH 的区别看一个例子就清楚了。假设用户数据是：

```json
{ "name": "Tom", "age": 18 }
```

要把 age 改成 20：

```
PUT   /users/1   请求体：{ "name": "Tom", "age": 20 }   → 完整替换，没写的字段会被清掉
PATCH /users/1   请求体：{ "age": 20 }                  → 只改 age，name 保持不动
```

**URL 说的是操作对象，HTTP 方法说的是操作动作**，组合在一起就清楚了。

### 2.2 URL 的结构

拆开 `https://api.example.com/users/42`：

```
https://          → 协议（HTTPS 安全传输）
api.example.com   → 域名（服务器在哪）
/users/42         → 路径（id=42 的用户）
```

路径里只放资源名，不放动作：

```
不好： /getUser?id=42      → 动词塞进了 URL
      /deleteUser?id=42

REST： GET /users/42       → 动作交给 HTTP 方法
       DELETE /users/42
```

后面 REST 约定部分还会再提到这个。

查询参数跟在 `?` 后面：

```
/articles?page=2&size=10
```

意思是「第 2 页，每页 10 条」。

### 2.3 HTTP vs HTTPS

`http://` 和 `https://` 差一个字母，含义差很多：

- **HTTP**：数据在传输过程中没有加密保护，网络链路上的攻击者可能窃取或篡改数据。
- **HTTPS**：在 HTTP 上加了一层 TLS 加密，被截到了也看不懂。

现在正规 API 都强制 HTTPS。实际开发中，Token、密码等敏感信息走 HTTP 就是裸奔。

## 三、请求和响应

API 的交互模式是一来一回：客户端发 **请求（Request）**，服务器回 **响应（Response）**。

### 3.1 请求的格式

一个请求分三块：**请求行**、**Headers**、**Body**。Body 不是所有请求都有——GET 一般没有，POST/PUT/PATCH 通常有。

先看 GET（数据通过 URL 传）：

```
GET /users/42 HTTP/1.1            ← 请求行：方法 + 路径 + 协议版本
Host: api.example.com             ← 从这里往下都是 Headers
Authorization: Bearer eyJhbGc...
Accept: application/json
```

再看 POST（数据通过 Body 传）：

```
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json    ← 告诉服务器 Body 是 JSON 格式
Authorization: Bearer eyJhbGc...

{ "name": "Tom", "age": 18 }      ← 请求 Body，真正的数据在这
```

三种传参方式的区别：

| 位置 | 怎么传 | 什么时候用 |
|------|--------|------------|
| **Header** | 键值对，跟在请求行后面 | 元信息：身份令牌、格式声明、缓存控制 |
| **Query** | URL 问号后面 `/users?page=2` | 过滤、分页、搜索关键词 |
| **Body** | 请求体，Header 空行之后 | 创建/更新数据时传 JSON |

### 3.2 响应的格式

```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache

{
  "id": 42,
  "name": "小明",
  "email": "xiaoming@example.com"
}
```

返回的数据一般是 JSON——用花括号包着，键值对形式，人和机器都好读。

## 四、Headers：请求和响应的附加信息

Headers 是 HTTP 报文里的元信息，告诉对方消息格式、缓存策略、身份信息等。

### 4.1 常见请求头

| Header | 作用 | 示例 |
|--------|------|------|
| `Content-Type` | 发送数据的格式 | `Content-Type: application/json` |
| `Accept` | 期望返回的格式 | `Accept: application/json` |
| `Authorization` | 身份凭证 | `Authorization: Bearer <token>` |
| `Host` | 目标服务器域名 | `Host: api.example.com` |
| `User-Agent` | 客户端标识 | `User-Agent: Mozilla/5.0 ...` |

### 4.2 常见响应头

| Header | 作用 | 示例 |
|--------|------|------|
| `Content-Type` | 返回数据的格式 | `Content-Type: application/json` |
| `Cache-Control` | 缓存控制 | `Cache-Control: max-age=3600`（缓存 1 小时） |
| `Set-Cookie` | 服务器下发 Cookie | `Set-Cookie: session=abc123` |
| `Location` | 配合 3xx 重定向 | `Location: https://new.example.com` |
| `Access-Control-Allow-Origin` | 跨域白名单 | `Access-Control-Allow-Origin: *` |

> 请求 Body 是 JSON 时，需要带 `Content-Type: application/json`，告诉服务器怎么解析。GET 请求没有 Body，不需要这个 Header。

## 五、状态码

响应里的状态码告诉客户端「这次请求结果怎么样」。按首位数字分五大类。

### 5.1 1xx — 信息类

请求收到，处理中。实际开发中很少直接碰到。

| 状态码 | 含义 |
|--------|------|
| **100** Continue | 收到了请求头，客户端可以继续发请求体 |
| **101** Switching Protocols | 切换协议，比如 HTTP → WebSocket |

### 5.2 2xx — 成功

| 状态码 | 含义 | 出现场景 |
|--------|------|----------|
| **200** OK | 成功 | GET 查询成功 |
| **201** Created | 创建成功 | POST 新建资源 |
| **204** No Content | 成功但无返回内容 | DELETE 删除、PUT 更新（不需要返数据时） |

### 5.3 3xx — 重定向

资源不在这，去别处找。浏览器自动跟着跳。

| 状态码 | 含义 | 出现场景 |
|--------|------|----------|
| **301** Moved Permanently | 永久迁移 | 换域名，旧地址永久跳转 |
| **302** Found | 临时迁移 | 临时跳转，搜索引擎不更新索引 |
| **304** Not Modified | 内容没变 | 命中浏览器缓存，不用重新下载 |

### 5.4 4xx — 客户端错误

| 状态码 | 含义 | 出现场景 |
|--------|------|----------|
| **400** Bad Request | 请求格式有问题 | 参数缺失、JSON 格式错、校验不通过 |
| **401** Unauthorized | 身份认证失败 | 没 Token、Token 过期或无效 |
| **403** Forbidden | 没权限 | 已登录但无权访问 |
| **404** Not Found | 找不到 | URL 写错或资源已删除 |
| **405** Method Not Allowed | 方法不对 | 只支持 GET 却发了 POST |
| **409** Conflict | 冲突 | 要创建的资源已存在（如重复用户名） |
| **422** Unprocessable Entity | 格式对但内容不合法 | 校验不通过 |
| **429** Too Many Requests | 请求太频繁 | 触发限流 |

> 400 vs 422 理论上区分得很清楚，但实际项目中很多后端统一返回 400，不会严格区分。看团队约定就好，不用太纠结。

> **401 vs 403 的区别：**
> - 401 = 不认识你是谁 → 没登录
> - 403 = 知道你是谁但不能进 → 登录了但没权限

### 5.5 5xx — 服务器错误

| 状态码 | 含义 | 出现场景 |
|--------|------|----------|
| **500** Internal Server Error | 内部出错 | 代码 bug、未处理异常 |
| **502** Bad Gateway | 网关收到坏响应 | 上游服务返回无效数据 |
| **503** Service Unavailable | 暂时不可用 | 过载或维护中 |
| **504** Gateway Timeout | 网关超时 | 上游响应太慢，常见于微服务调用链 |

### 5.6 一句话记

- `2xx`：成了
- `3xx`：去别处
- `4xx`：请求的锅
- `5xx`：服务器的锅

## 六、认证：Token 方式

大部分 API 不能随便调，服务器需要知道是谁在请求。

最常用的方式是 Token：

1. 先用用户名密码登录，服务器返回一个 Token
2. 之后每次请求在 Header 里带上 Token
3. 服务器验证 Token，处理请求

常见的 Token 格式是 JWT（JSON Web Token），长这样：

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

> JWT 不是加密，是 Base64 编码 + 签名校验。中间那段 Payload 任何人都能解码读取——所以**敏感数据不要直接放进 JWT**。它的作用是防篡改（签名校验），不是防偷看。

## 七、一个完整请求的例子

假设笔记 App 获取笔记列表：

```
1. App 发请求：
   GET https://api.notesapp.com/notes
   Authorization: Bearer <token>

2. 服务器收到 → 验证 token → 查数据库

3. 服务器返回：
   200 OK
   [
     { "id": 1, "title": "买菜清单", "content": "土豆、西红柿" },
     { "id": 2, "title": "工作计划", "content": "下午开会" }
   ]

4. App 拿到数据 → 显示在界面上
```

App 上看到的数据，大部分都是这样「要来的」。

## 八、幂等性：同一个请求发多次会怎样

**幂等（Idempotent）**：不管发几次，结果一样，不会因为重复就出问题。

| 方法 | 幂等？ | 原因 |
|------|--------|------|
| **GET** | ✅ 是 | 查询不改数据，查 100 次结果一样 |
| **PUT** | ✅ 是 | 完整替换，重复替换结果一样 |
| **DELETE** | ✅ 是 | 多次执行不会重复产生删除效果，即使第二次可能返回 404 |
| **POST** | ❌ 否 | 每次调用都新建一条，发 3 次建 3 条 |

**为什么需要关心这个：** 网络不稳定时，请求超时，客户端不确定服务器有没有收到，会重试。

- GET/PUT/DELETE 重试没问题
- POST 重试可能创建重复数据——重复下单、重复扣款

所以支付、下单类的 POST 接口通常需要额外做幂等性设计（比如带唯一请求 ID），防止重复操作。

## 九、CORS 跨域：请求被浏览器拦截了

前后端分离开发几乎必然碰到跨域。

### 9.1 什么是跨域

浏览器有一个安全策略叫「同源策略」：页面只能请求和自己同源的服务器。同源 = 协议 + 域名 + 端口三个都一样。

```
前端：http://localhost:3000
后端：http://localhost:8080   ← 端口不同 → 跨域

前端：https://www.myapp.com
后端：https://api.myapp.com   ← 子域名不同 → 跨域
```

### 9.2 浏览器的处理方式

不是所有跨域请求都会预检。**简单请求**（比如普通 GET，不带自定义 Header）浏览器直接发，不预检。**非简单请求**（比如带 `Authorization` 的 JSON POST），浏览器会先发一个 **OPTIONS 预检请求**，问服务器「允许跨域吗？」

服务器在响应头里表态：

```
Access-Control-Allow-Origin: https://www.myapp.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
```

没这几行，浏览器就拦截请求，控制台报 CORS 错。

### 9.3 怎么处理

- **后端配置**：加 `Access-Control-Allow-Origin` 响应头，最正确的方式
- **开发环境**：用前端框架的代理（Vite / webpack devServer proxy）绕过
- **`*` 通配符**：`Access-Control-Allow-Origin: *` 允许所有来源，只适合公开 API，有认证的接口不能这样配

> CORS 是**浏览器的限制**，不是 HTTP 协议本身的限制。Postman 或后端直接发请求不受跨域影响。

## 十、HTTP/1.1 vs HTTP/2

### 10.1 HTTP/1.1 的瓶颈

HTTP/1.1 支持连接复用（keep-alive），但**单个 TCP 连接**内有队头阻塞——一个请求卡住，后面的都得等。浏览器通常会开多个 TCP 连接（一般 6 个）来缓解，但治标不治本。

```
请求 A ──→ 等待响应 A ──→ 请求 B ──→ 等待响应 B ──→ 请求 C ...
```

一个网页要加载几十个资源（JS、CSS、图片），排队一个个来，慢。

### 10.2 HTTP/2 做了什么

**多路复用**：一个连接同时发多个请求，不用排队。

```
        ┌─→ 请求 A ──→ 响应 A
一个连接 ├─→ 请求 B ──→ 响应 B
        └─→ 请求 C ──→ 响应 C
```

还有：

- **头部压缩**：Headers 重复内容多，HTTP/2 压缩后减少传输量

### 10.3 对开发者的影响

大多数情况下 HTTP/2 是透明的——服务器和 CDN 开启后，浏览器自动用，不需要改代码。不过 HTTP/2 的多路复用跑在 TCP 之上，TCP 层面的队头阻塞仍然存在（丢一个包，整个连接等重传）。HTTP/3 换成了基于 UDP 的 QUIC 协议，在传输层解决了这个问题，目前正在逐步推广。

## 十一、REST 的几个约定

REST 不是硬性规定，是大家认可的习惯：

- **URL 用名词不用动词**：`/users` 而不是 `/getUsers`
- **常见约定用复数**：`/articles` 而不是 `/article`（社区有单复数之争，但复数更普遍）
- **层级关系用斜杠**：`/users/42/orders` 表示「42 号用户的订单」
- **返回 JSON**：现在几乎都是 JSON，偶尔还有 XML（老古董）

## 十二、工具推荐

学 HTTP / REST 最好的方式就是实际调一调。最简单的：打开浏览器，地址栏输入：

```
https://api.github.com/users/github
```

回车就能看到 GitHub 官方账号的信息以 JSON 返回。不需要装任何东西。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/other/2026/http_rest_api1.png)

想要更系统调试的话，**Postman**（免费）很适合——可以自由设置方法、Headers、Body，看完整的请求和响应。

几个公开 API 可以玩：

- `https://api.github.com` — GitHub API
- `https://jsonplaceholder.typicode.com/posts` — 假的 REST API，随便测
- `https://httpbin.org/get` — 调试 HTTP 请求的神器

---

**几个关键点：**

1. API 是软件之间交互的约定，REST 是目前最流行的一种风格
2. URL 定位资源，HTTP 方法定位动作，加起来就是 REST 的核心
3. 状态码按首位分五类：2 成 3 跳 4 你错 5 我错
4. GET/PUT/DELETE 幂等可安全重试，POST 不幂等要注意防重复
5. CORS 是浏览器限制，不是 HTTP 限制——Postman 不受影响

今日份学习先到这。

🌈关注我吖~❤️

公众号：**妮K妮K妮**
