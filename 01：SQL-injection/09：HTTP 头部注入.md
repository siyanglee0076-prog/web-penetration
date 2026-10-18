# HTTP 头部注入

## 一、环境

- 靶场：自建本地 sqli-labs-master 靶场
- 工具：Burp Suite + HackBar
- 数据库：MySQL

## 二、什么是 HTTP 头部注入

HTTP 头部注入是指：

**用户可控的 HTTP 请求头数据被服务器直接带入 SQL 查询中，从而产生 SQL 注入。**

常见的 HTTP 请求头包括：

```text
User-Agent
Referer
Cookie
X-Forwarded-For
```

普通 SQL 注入一般是在：

```text
?id=1
```

这种 URL 参数中注入。

HTTP 头部注入则可能是在：

```http
User-Agent: xxx
```

或者：

```http
Cookie: xxx
```

等位置进行注入。

## 三、基本原理

例如服务器代码可能把：

```http
User-Agent: Mozilla/5.0
```

直接保存到数据库：

```sql
INSERT INTO logs(user_agent) VALUES('Mozilla/5.0')
```

如果服务器没有正确过滤，就可能变成：

```http
User-Agent: ' UNION SELECT ...
```

从而影响后面的 SQL 语句。

所以核心是：

```text
HTTP 请求头
      ↓
服务器获取请求头
      ↓
拼接 SQL
      ↓
数据库执行
      ↓
产生 SQL 注入
```

## 四、常见注入位置

### 1. User-Agent

例如：

```http
User-Agent: Mozilla/5.0
```

服务器可能记录访问者的浏览器信息。

如果这个值直接进入 SQL，就可能存在注入。

### 2. Referer

例如：

```http
Referer: http://example.com/
```

服务器可能把来源页面记录到数据库。

### 3. Cookie

例如：

```http
Cookie: user_id=1
```

如果 Cookie 中的数据被服务器直接用于 SQL 查询，也可能产生注入。

<img src=".\images\12.png" alt="image-20260923110229244" style="zoom:30%;" />

<img src=".\images\13.png" alt="image-20260923112846606" style="zoom:30%;" />

<img src=".\images\14.png" alt="image-20260923113247727" style="zoom:30%;" />

## 五、基本判断

使用 **BP（Burp Suite）+ HackBar** 抓取请求后，可以修改 HTTP 头部中的参数。

例如：

```http
User-Agent: test'
```

观察服务器返回是否出现：

- SQL 报错
- 页面响应变化
- 状态码变化
- 响应时间变化

也可以进一步测试：

```http
User-Agent: test' AND 1=1--+
```

和：

```http
User-Agent: test' AND 1=2--+
```

通过比较两次请求的响应来判断是否可能存在注入。

## 六、HTTP 头部注入和普通注入的区别

### 普通 SQL 注入

注入位置：

```text
?id=1
```

数据来源：

```text
URL 参数
```

### HTTP 头部注入

注入位置：

```text
User-Agent: xxx
```

或者：

```text
Cookie: xxx
```

数据来源：

```text
HTTP 请求头
```

所以两者本质上还是：

**用户输入 → 进入 SQL → 改变 SQL 语句。**

只是**注入点的位置不同**。

## 七、为什么需要 BP（Burp Suite）

HTTP 头部不是普通 URL 参数，直接在浏览器地址栏里修改比较麻烦。

使用 BP 可以直接看到完整 HTTP 请求：

```http
GET /xxx?id=1 HTTP/1.1
Host: xxx
User-Agent: Mozilla/5.0
Cookie: xxx
Referer: xxx
```

然后直接修改：

```http
User-Agent: test'
```

再发送请求。

所以 HTTP 头部注入非常适合使用：

**BP（Burp Suite）+ HackBar**

进行测试。

## 八、我的理解

以前学习 SQL 注入时，我主要关注：

```text
?id=1'
```

后来发现 SQL 注入的关键其实不是：

> “一定要在 URL 参数里注入。”

而是：

> **只要服务器把用户可控的数据带入 SQL，就可能形成 SQL 注入。**

所以：

```text
URL 参数
POST 参数
Cookie
User-Agent
Referer
```

都可能成为注入点。

HTTP 头部注入本质上还是 SQL 注入，只是**把注入位置从参数移动到了 HTTP 请求头**。

## 九、关键知识点

```text
HTTP 头部注入
├── User-Agent
├── Referer
├── Cookie
├── BP（Burp Suite）修改请求头
├── 判断 SQL 注入
└── 本质仍然是 SQL 注入
```

核心记忆：

**HTTP 头部注入 = 用户可控的 HTTP 请求头进入 SQL 查询，从而产生 SQL 注入。**