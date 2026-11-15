# VulnHub - Covfefe 靶机实战

## 1. 靶机信息

| 项目     | 信息                                               |
| -------- | -------------------------------------------------- |
| 靶机名称 | Covfefe                                            |
| 靶机 IP  | `192.168.31.132`                                   |
| 攻击机   | Kali Linux                                         |
| 攻击方向 | Web 信息收集 → 敏感信息泄露 → SSH → Linux 本地提权 |
| 最终结果 | 获取 root 权限                                     |

## 2. 信息收集

### 2.1 端口扫描

```
nmap -sV -sC 192.168.31.132
```

发现开放端口：

| 端口      | 状态 | 服务  |
| --------- | ---- | ----- |
| 22/tcp    | open | SSH   |
| 80/tcp    | open | HTTP  |
| 31337/tcp | open | Elite |

初步判断：

- 22：SSH 登录服务。
- 80：HTTP 服务，访问后暂未发现有效信息。
- 31337：特殊 Web 服务，进一步进行目录扫描。

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

## 3. Web 信息收集与漏洞发现

### 3.1 80 端口探测

访问：

```
http://192.168.31.132/
```

暂未发现有效信息，转向 31337 端口。



### 3.2 31337 端口目录扫描

访问：

```
http://192.168.31.132:31337/
```

使用 DIRB 进行目录扫描：

```
dirb http://192.168.31.132:31337/
```

发现以下路径：

```
/.bash_history
/.bashrc
/.profile
/.ssh
/robots.txt
```

其中：

- `.bash_history`：可能包含用户历史执行命令。
- `.ssh`：可能存在 SSH 相关文件。
- `robots.txt`：可能包含隐藏目录或敏感路径。

这些隐藏文件和目录值得进一步检查。

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

### 3.3 发现第一个 Flag

访问：

```
http://192.168.31.132:31337/taxes/
```

页面返回：

```
Good job! Here is a flag:
flag1{make_america_great_again}
```

获得第一个 Flag：

```
flag1{make_america_great_again}
```



### 3.4 SSH 敏感信息收集

访问：

```
http://192.168.31.132:31337/.ssh
```

页面返回：

```
['id_rsa', 'authorized_keys', 'id_rsa.pub']
```

发现 SSH 相关文件：

| 文件            | 作用             |
| --------------- | ---------------- |
| id_rsa          | SSH 私钥         |
| authorized_keys | SSH 公钥授权列表 |
| id_rsa.pub      | SSH 公钥         |

说明 Web 服务可能暴露了 SSH 目录中的敏感信息，进一步检查私钥文件及其可访问性。

## 4. 漏洞利用 / 获取初始 Shell

### 4.1 获取用户信息

收集到以下用户及密码信息：

```
用户名：simon
密码：starwars
```

凭据：

```
simon:starwars
```

目标开放了 SSH 22 端口，可以使用 SSH 进行登录。

```
ssh simon@192.168.31.132
```

输入密码：

```
starwars
```

成功进入 Simon 用户环境。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

## 5. 分阶段突破

### 5.1 获取第二个 Flag

在目标机器中继续进行信息收集，发现第二个 Flag：

```
flag2{use_the_source_luke}
```

### 5.2 发现可疑程序

进一步检查系统中的特殊目录和程序，发现：

```
/usr/local/sbin/message
/usr/local/bin/read_message
```

这两个文件值得重点分析。

尤其需要关注程序的执行权限、文件所有者以及内部代码逻辑。

## 6. Linux 本地提权

### 6.1 分析程序

重点分析：

```
/usr/local/sbin/message
```

以及：

```
/usr/local/bin/read_message
```

发现存在与输入长度有关的程序漏洞，进一步判断为缓冲区溢出（Buffer Overflow）。

### 6.2 缓冲区溢出利用

通过构造超长输入，触发程序缓冲区溢出。

利用过程中使用了：

```
Simonaaaaaaaaaaaaaaa/bin/sh
```

最终成功执行 `/bin/sh`，获得更高权限的 Shell。

### 6.3 提权结果

成功突破原有用户权限限制，获得 root 权限。

验证：

```
whoami
```

预期结果：

```
root
```

至此，靶机提权完成。

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

## 7. 最终结果

本次靶机获得的信息：

| 阶段     | 结果                              |
| -------- | --------------------------------- |
| 目标 IP  | 192.168.31.132                    |
| 开放端口 | 22、80、31337                     |
| Web 服务 | 31337 端口                        |
| 初始用户 | simon                             |
| 用户密码 | starwars                          |
| Flag 1   | `flag1{make_america_great_again}` |
| Flag 2   | `flag2{use_the_source_luke}`      |
| 提权文件 | `/usr/local/sbin/message`         |
| 相关程序 | `/usr/local/bin/read_message`     |
| 提权方式 | 缓冲区溢出                        |
| 最终权限 | root                              |

## 8. 实战总结

本次 Covfefe 靶机主要涉及以下知识点：

1. **端口扫描**：通过 Nmap 发现 SSH、HTTP 和特殊端口 31337。
2. **目录扫描**：通过 DIRB 发现隐藏文件，包括 `.bash_history`、`.ssh` 等。
3. **敏感信息泄露**：Web 服务暴露 SSH 相关文件。
4. **SSH 登录**：利用收集到的 Simon 用户凭据进入目标系统。
5. **Linux 本地提权**：发现 `/usr/local/sbin/message` 等可疑程序。
6. **缓冲区溢出**：通过构造超长输入触发程序漏洞，最终执行 `/bin/sh` 获得 root 权限。

### 本次重点收获

- 不能只关注常见的 80 端口，特殊端口也可能存在重要 Web 服务。
- 目录扫描不仅要关注常见后台路径，也要留意 `.bash_history`、`.ssh` 等隐藏文件。
- SSH 私钥、公钥和授权文件是信息收集的重要对象。
- Linux 提权不能只关注 SUID、sudo，还需要关注自定义程序的执行逻辑。
- 缓冲区溢出是 Linux 本地提权中值得进一步学习的方向。

## 9. 工具与命令

| 工具 / 命令 | 用途               |
| ----------- | ------------------ |
| Nmap        | 端口扫描、服务识别 |
| DIRB        | Web 目录扫描       |
| SSH         | 远程登录           |
| `cat`       | 查看文件内容       |
| `find`      | 搜索文件           |
| `file`      | 查看文件类型       |
| `ls -l`     | 查看文件权限       |
| `whoami`    | 验证当前用户       |
| `id`        | 查看用户 UID、GID  |

## 10. 个人复盘

本次靶机从 Web 信息收集入手，通过特殊端口发现敏感文件，进一步获取用户凭据并登录 SSH，最后通过分析自定义程序中的缓冲区溢出漏洞完成本地提权。

**后续需要重点补充的知识：**

- SSH 私钥的使用与权限要求。
- Linux 缓冲区溢出的基本原理。
- SUID 程序中的栈溢出漏洞分析。
- `/bin/sh` 在提权过程中的作用。