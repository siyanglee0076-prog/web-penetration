# VulnHub - DC4 靶机渗透测试

## 一、靶场信息

| 项目     | 内容                                                         |
| -------- | ------------------------------------------------------------ |
| 靶场名称 | DC4                                                          |
| 靶场平台 | VulnHub                                                      |
| 靶场 IP  | `192.168.31.128`                                             |
| 攻击机   | Kali Linux                                                   |
| 目标系统 | Linux                                                        |
| 渗透目标 | 获取 Root 权限                                               |
| 涉及技术 | Nmap 端口扫描、用户名收集、Hydra SSH 爆破、敏感信息泄露、用户凭据获取、teehee 提权 |

## 二、信息收集

### 1. 主机发现

通过主机发现确定目标靶机 IP：

```
192.168.31.128
```

### 2. 端口扫描

使用 Nmap 对目标进行端口扫描：

```
nmap -sV -sC -p- 192.168.31.128
```

发现开放端口：

| 端口   | 状态 | 服务 |
| ------ | ---- | ---- |
| 22/tcp | open | SSH  |
| 80/tcp | open | HTTP |

目标开放 SSH 和 HTTP 服务，优先对 Web 服务进行信息收集。

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

## 三、用户名收集与 SSH 爆破

利用简单账户密码字典，在bp里对登录页面进行爆破。

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

登录后bp抓包发现可以直接执行系统命令，进而尝试反弹shell。

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

### 1. 用户名收集

在信息收集过程中发现三个用户名：

```
charles
jim
sam
```

### 2. SSH 弱口令爆破

使用 Hydra 对 jim 用户进行 SSH 密码爆破：

```
hydra -l jim -P /home/kali/Desktop/mm.txt 192.168.31.128 ssh -Vv
```

爆破成功，获取 SSH 登录凭据：

```
[22][ssh] host: 192.168.31.128
login: jim
password: jibril04
```

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

### 3. SSH 远程登录

使用获取的账号密码登录目标主机：

```
ssh jim@192.168.31.128
```

登录成功后，查看当前用户信息：

```
id
```

## 四、敏感信息收集与用户凭据获取

### 1. 查找敏感信息

登录 Jim 用户后，继续进行系统信息收集。

发现一封 Charles 留给 Jim 的邮件，内容如下：

```
Hi Jim,

I'm heading off on holidays at the end of today,
so the boss asked me to give you my password
just in case anything goes wrong.

Password is: ^xHhA&hvim0y

See ya,
Charles
```

### 2. 获取 Charles 用户密码

从邮件中提取到 Charles 的登录密码：

```
用户名：charles
密码：^xHhA&hvim0y
```

该密码为后续用户切换提供了凭据。

### 3. 用户切换

通过获取的 Charles 凭据进行用户切换。

具体执行命令及切换后的权限信息：待补充。

## 五、Linux 权限提升

### 1. 权限信息收集

继续对当前用户的权限进行枚举，重点检查 sudo 权限配置：

```
sudo -l
```

### 2. teehee 提权

在权限枚举过程中发现 `teehee` 相关提权利用条件。

利用 teehee 成功获取 Root 权限。

<此处补充 teehee 的具体 sudo 配置、利用命令及执行过程截图>

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

### 3. 验证 Root 权限

提权成功后执行：

```
whoami
id
```

确认当前用户已经获得 Root 权限。

至此，完成 DC4 靶机的渗透测试。

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

## 六、渗透过程总结

### 1. 核心攻击路径

| 阶段       | 利用点           | 结果                   |
| ---------- | ---------------- | ---------------------- |
| 信息收集   | Nmap 端口扫描    | 发现 SSH、HTTP 服务    |
| 用户名收集 | 用户名信息收集   | 获取 charles、jim、sam |
| SSH 爆破   | Hydra 弱口令爆破 | 获取 Jim 登录密码      |
| 远程登录   | SSH              | 成功登录 Jim 用户      |
| 信息泄露   | 邮件敏感信息     | 获取 Charles 用户密码  |
| 用户切换   | Charles 用户凭据 | 进一步获取用户权限     |
| Linux 提权 | teehee           | 获取 Root 权限         |

### 2. 本次涉及的知识点

1. Nmap 端口扫描与服务识别。
2. 用户名信息收集。
3. Hydra SSH 弱口令爆破。
4. SSH 远程登录。
5. Linux 用户信息及权限枚举。
6. 邮件敏感信息泄露。
7. 用户凭据复用与用户切换。
8. Linux sudo 权限配置。
9. teehee 提权。

### 3. 实战收获

- SSH 服务可能存在弱口令风险，可以通过用户名收集结合密码字典进行安全测试。
- 获取初始用户权限后，不应只关注当前用户的权限，还需要继续收集系统中的敏感信息。
- 用户之间可能通过邮件、文件等方式传递登录凭据，敏感信息泄露可能成为进一步渗透的入口。
- Linux 提权过程中需要重点检查 `sudo -l`，关注当前用户被授权执行的特殊命令。
- teehee 可以作为 Linux 提权的利用入口，需要结合具体 sudo 配置分析其利用条件。
- 渗透测试不能止步于获取普通用户 Shell，还需要继续进行权限枚举和提权，最终验证是否获得 Root 权限。