

环境：

> **自建本地 bank 靶场（PHP + MySQL + Apache）**
> **工具：Burp Suite + HackBar**

目录：

```text
Web安全学习
 └── CSRF
      ├── CSRF原理.md
      ├── CSRF构造方法.md
      ├── CSRF绕过.md
      └── CSRF防御.md
```

------

# CSRF 原理

## 1. 漏洞介绍

CSRF（Cross Site Request Forgery）

中文：

> 跨站请求伪造

核心：

攻击者诱导用户访问恶意页面，利用用户已经登录的身份，对目标网站发送非法请求。

简单理解：

用户：

```
登录银行网站
↓
浏览器保存 Cookie
↓
访问攻击者页面
↓
自动携带 Cookie 请求银行接口
↓
完成转账操作
```

------

## 2. 漏洞产生条件

CSRF 成立通常需要：

### ① 用户已经登录目标网站

例如：

```
admin.com
```

浏览器存在：

```
Cookie: session=xxxx
```

------

### ② 网站使用 Cookie 作为身份认证

请求：

```http
POST /update.php HTTP/1.1

Cookie: PHPSESSID=xxxx

password=123456
```

服务器通过 Cookie 判断用户身份。

------

### ③ 服务端没有 CSRF 防护

例如：

修改密码：

```http
POST /change.php

password=123456
```

没有验证请求来源。

------

## 3. CSRF 与 XSS 区别

| 漏洞 | 特点                  |
| ---- | --------------------- |
| XSS  | 攻击者执行 JavaScript |
| CSRF | 借用用户身份发送请求  |

XSS：

```
攻击者 → 网站 → 执行JS
```

CSRF：

```
攻击者 → 用户浏览器 → 网站请求
```

核心区别：

> XSS 是利用用户权限执行代码，CSRF 是利用用户身份发送请求。

------

# CSRF 构造方法

## 1. GET 型 CSRF

如果网站：

```php
update.php?id=1
```

攻击者构造：

```html
<img src="http://target.com/update.php?id=1">
```

用户访问页面：

浏览器自动发送请求。

<img src=".\images\01.png" alt="01" style="zoom:30%;" />

<img src=".\images\02.png" alt="02" style="zoom:30%;" />

<img src=".\images\03.png" alt="03" style="zoom:30%;" />

------

## 2. POST 型 CSRF

例如修改密码：

正常请求：

```http
POST /change.php


password=123456
```

构造页面：

```html
<form action="http://target.com/change.php" method="POST">

<input name="password" value="123456">

</form>


<script>
document.forms[0].submit();
</script>
```

用户访问：

自动提交表单。

<img src=".\images\04.png" alt="04" style="zoom:30%;" />

<img src=".\images\05.png" alt="05" style="zoom:30%;" />

------

## 3. Burp 分析流程

步骤：

1. 登录目标网站
2. 抓取敏感操作请求

例如：

```http
POST /user/update.php
```

1. 查看：

- 请求方法
- 参数
- Cookie
- Token

1. 删除无关参数测试

判断：

是否只需要 Cookie 就可以完成操作。

<img src=".\images\06.png" alt="06" style="zoom:30%;" />

<img src=".\images\07.png" alt="07" style="zoom:30%;" />

<img src=".\images\08.png" alt="08" style="zoom:30%;" />

------

## 4. 常见利用位置

CSRF 常见功能：

- 修改密码
- 修改邮箱
- 添加管理员
- 删除用户
- 转账操作
- 发布内容

------

# CSRF 绕过

## 1. 无 Token

正常防御：

请求：

```http
POST /change.php

password=123456

token=xxxx
```

如果：

```
token不存在
```

直接提交成功。

------

## 2. Token 校验不严格

例如：

服务端只判断：

```php
token参数存在
```

但是不验证：

```
token是否属于当前用户
```

可以绕过。

------

## 3. Token 泄露

如果 Token 出现在：

- 页面源码
- URL
- Referer
- JS 文件

攻击者可能获取 Token。

------

## 4. Referer 校验绕过

部分网站：

检查：

```http
Referer
```

判断请求来源。

可能存在：

- 校验不完整
- 允许空 Referer

------

## 5. SameSite Cookie 绕过

Cookie：

```http
SameSite=None
```

允许跨站发送。

如果：

```http
SameSite=Strict
```

跨站请求不会携带 Cookie。

------

# CSRF 防御

## 1. CSRF Token（核心）

服务器生成随机 Token。

例如：

表单：

```html
<input type="hidden" 
name="token"
value="abc123">
```

提交时：

```
用户Cookie
+
Token
```

服务器验证：

```
Token是否匹配
```

------

## 2. 验证 Referer / Origin

检查请求来源：

```http
Referer:
https://target.com
```

或者：

```http
Origin:
https://target.com
```

判断是否来自可信网站。

------

## 3. SameSite Cookie

设置：

```http
Set-Cookie:
session=xxx;
SameSite=Strict
```

作用：

限制 Cookie 跨站发送。

------

## 4. 二次验证

敏感操作：

例如：

- 修改密码
- 转账

增加：

- 验证码
- 短信验证
- 密码确认

------

## 5. 后端验证请求方式

避免：

GET 请求执行敏感操作。

例如：

错误：

```
GET /delete?id=1
```

正确：

```
POST /delete
```

------

# 总结

## CSRF 核心理解

CSRF：

> 利用用户已经登录的身份，让用户浏览器自动发送恶意请求。

攻击流程：

```
用户登录
 ↓
保存Cookie
 ↓
访问攻击页面
 ↓
自动携带Cookie请求
 ↓
服务器执行操作
```

------

## 与 XSS 联系

XSS：

```
获取执行权限
```

CSRF：

```
利用身份权限
```

实际攻击中：

```
XSS + CSRF
```

结合效果更强。
```
<script>
xmlhttp=new XMLHttpRequest();
xmlhttp.open("post","http://172.20.10.2/cms/admin/user.action.php",false);
xmlhttp.setRequestHeader("Content-type","application/x-www-form-urlencoded");
xmlhttp.send("act=add&username=datou&password=123456&password2=123456&button=%E6%B7%BB%E5%8A%A0%E7%94%A8%E6%88%B7&userid=0");
</script>
```

<img src=".\images\09.png" alt="09" style="zoom:30%;" />

<img src=".\images\10.png" alt="10" style="zoom:30%;" />

<img src=".\images\11.png" alt="11" style="zoom:30%;" />





------

## 防御核心

主要方法：

```
CSRF Token
+
SameSite Cookie
+
Origin/Referer检测
+
敏感操作二次验证
```

------

后续你学 **SSRF、文件上传、越权、JWT、OAuth** 时，也可以继续按照这个目录接在 `Web安全学习` 下面。