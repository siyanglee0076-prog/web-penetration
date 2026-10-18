# **环境统一：**

> **自建本地 CMS、XSSChallenge 靶场（PHP + MySQL + Apache）**
> **工具：Burp Suite + HackBar**

------

# XSS 盲打

## 1. 漏洞原理

XSS 盲打（Blind XSS）是一种特殊的存储型 XSS。

与普通存储型 XSS 不同：

- 用户输入点和触发点不在同一个页面
- 攻击者提交 Payload 后，无法立即看到执行效果
- Payload 会被管理员或其他高权限用户访问后台时触发

例如：

用户注册页面：

```
username=test
message=<script>alert(1)</script>
```

数据保存到数据库。

管理员进入后台：

```
后台留言查看页面
```

如果后台没有过滤：

```
<script>alert(1)</script>
```

会在管理员浏览器执行。

<img src="images\01.png" alt="01" style="zoom:30%;" />

<img src="..\XSS\images\02.png" alt="02" style="zoom:30%;" />

------

## 2. 常见位置

盲打通常存在：

- 留言板
- 用户反馈
- 注册信息
- 工单系统
- 评论系统
- 后台日志
- 用户资料

------

## 3. 测试思路

1. 寻找可能被管理员查看的位置
2. 输入测试 Payload

例如：

```html
<script>alert(1)</script>
```

1. 等待管理员访问后台
2. 判断是否触发 XSS

------

## 4. 实际学习理解

普通 XSS：

```
提交 → 当前页面触发
```

盲打：

```
提交 → 数据库存储 → 管理员查看 → 触发
```

核心区别：

> 触发者和提交者不是同一个人。

------







# XSS 构造方法

## 1. Script 标签注入

最基础方式：

```html
<script>alert(1)</script>
```

原理：

浏览器解析 HTML 时执行 script 标签。

<img src="images\03.png" alt="03" style="zoom:30%;" />

------

## 2. HTML 标签属性事件

当 script 被过滤时，可以利用事件触发。

例如：

```html
<img src=x onerror=alert(1)>
```

解释：

```
img
```

图片标签。

```
src=x
```

加载不存在的资源。

```
onerror
```

加载失败触发事件。

| 事件          | 含义                     | 常见元素/场景        |
| ------------- | ------------------------ | -------------------- |
| `onclick`     | 点击时触发               | 按钮、链接、普通元素 |
| `onfocus`     | 获得焦点时触发           | `input` 等表单元素   |
| `onblur`      | 失去焦点时触发           | `input` 等表单元素   |
| `onmouseover` | 鼠标移入时触发           | 大多数 HTML 元素     |
| `onmouseout`  | 鼠标移出时触发           | 大多数 HTML 元素     |
| `onmousemove` | 鼠标移动时触发           | 页面元素             |
| `onkeydown`   | 按下键盘按键时触发       | 输入框、页面         |
| `onkeyup`     | 松开键盘按键时触发       | 输入框、页面         |
| `onchange`    | 内容发生变化并确认后触发 | `input`、`select`    |
| `onsubmit`    | 提交表单时触发           | `<form>`             |
| `onload`      | 页面/资源加载完成时触发  | `body`、图片等       |
| `onerror`     | 资源加载失败时触发       | `img` 等             |

------

<img src="images\04.png" alt="04" style="zoom:30%;" />





## 3. 输入到标签属性中

例如：

原代码：

```html
<input value="用户输入">
```

输入：

```
" onmouseover="alert(1)
```

闭合原来的属性：

```html
<input value="" onmouseover="alert(1)">
```

------

<img src="C:\Users\27696\Desktop\web-penetration\XSS\images\05.png" alt="05" style="zoom:30%;" />



## 4. 常见 Payload

弹窗测试：

```html
<script>alert(1)</script>
```

图片事件：

```html
<img src=x onerror=alert(1)>
```

SVG：

```html
<svg onload=alert(1)>
```

焦点事件：

```html
<input autofocus onfocus=alert(1)>
```

------

# XSS 变形

## 1. 大小写绕过

部分过滤规则不完善：

```html
<ScRiPt>alert(1)</ScRiPt>
```

原因：

HTML 标签通常不区分大小写。

<img src=".\images\07.png" alt="07" style="zoom:30%;" />

------

## 2. 编码绕过

HTML 实体编码：

```html
&#60;script&#62;
```

浏览器解析后：

```
<script>
```

------

## 3. 空格绕过

过滤：

```
script
```

可以尝试：

```html
<script    >
```

或者：

```html
<img src=x onerror = alert(1)>
```

------

## 4. 关键字替换绕过

过滤：

```
alert
```

尝试：

```javascript
window['alert'](1)
```

或者：

```javascript
confirm(1)
```

------

## 5. 事件绕过

script 被过滤：

```html
<script>alert(1)</script>
```

使用：

```html
<img src=x onerror=alert(1)>  / <a href="javascript:alert(1)">1</a>
```

核心：

> 不一定非要 script 标签，HTML 事件也可以执行 JS。

------

<img src=".\images\06.png" alt="06" style="zoom:30%;" />

## 6. 双写关键词

script 被过滤：

```html
<script>alert(1)</script>
```

使用：

```html
<sscriptcript>alert(1)</scscriptript>
```

核心：

> 将关键词双写，绕过替换

<img src=".\images\08.png" alt="08" style="zoom:30%;" />

# XSS 防御

## 1. 输入过滤

对用户输入进行限制：

例如：

过滤：

```
<script>
onerror
javascript:
```

但是：

单纯黑名单过滤不安全。

原因：

攻击者可以不断变形绕过。

------

## 2. 输出编码（核心）

根据输出位置编码。

HTML 内容：

```
<  转义为  &lt;
>  转义为  &gt;
```

属性：

```
"  转义
'  转义
```

例如：

用户输入：

```html
<script>alert(1)</script>
```

输出：

```html
&lt;script&gt;alert(1)&lt;/script&gt;
```

浏览器会认为是普通文本。

------

## 3. 使用安全函数

PHP：

```php
htmlspecialchars()
```

作用：

把特殊字符转换成 HTML 实体。

例如：

输入：

```
<script>
```

输出：

```
&lt;script&gt;
```

------

## 4. CSP 防护

Content Security Policy

限制浏览器加载执行脚本。

例如：

禁止：

```
inline javascript
```

减少 XSS 利用。

------

## 5. HttpOnly Cookie

设置：

```
HttpOnly
```

作用：

禁止 JavaScript 读取 Cookie。

防止：

```javascript
document.cookie
```

直接获取登录凭证。

------

# 总结

## XSS 类型理解

| 类型       | 特点                             |
| ---------- | -------------------------------- |
| 反射型 XSS | 输入后立即返回页面执行           |
| 存储型 XSS | 输入保存服务器，访问时执行       |
| 盲打 XSS   | 提交后等待管理员等高权限用户触发 |

## 学习重点

XSS 本质：

> 用户输入 → 浏览器解析 → JavaScript 执行

攻击核心：

```
绕过过滤
突破上下文
触发执行
```

防御核心：

```
不要相信用户输入
输出时进行正确编码
```

------

这个四个 Markdown 可以直接拆成：

```
Web安全学习
 └── XSS
      ├── XSS盲打.md
      ├── XSS构造方法.md
      ├── XSS变形绕过.md
      └── XSS防御.md
```