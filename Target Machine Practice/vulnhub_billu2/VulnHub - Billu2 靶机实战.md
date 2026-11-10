# VulnHub - Billu2 靶机实战

## 一、靶机信息

| 项目     | 内容                                       |
| -------- | ------------------------------------------ |
| 靶机 IP  | `192.168.31.131`                           |
| 开放端口 | `22`、`80`、`111`、`8080`、`45251`         |
| Web 服务 | HTTP                                       |
| 主要技术 | Drupalgeddon2、Metasploit、SUID、PATH 劫持 |
| 实战目标 | 获取 Shell 并提权至 root                   |

## 二、信息收集

### 1. 端口扫描

扫描发现以下开放端口：

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
8080/tcp  open  http-proxy
45251/tcp open  unknown
```

初步判断：

- 22：SSH 远程登录服务。

- 80：HTTP Web 服务，优先进行信息收集。

- 111：RPC 服务。

- 8080：HTTP-Proxy 服务。

- 45251：未知服务，需要进一步识别。

  <img src=".\images\01.png" alt="01" style="zoom: 33%;" />

### 2. 目录扫描

发现以下路径：

```
http://192.168.31.131/robots.txt
http://192.168.31.131/README.txt
```

<img src=".\images\02.png" alt="01" style="zoom: 33%;" />

## 三、Web 信息收集与漏洞发现

### 1. Drupal 版本识别

访问：

```
http://192.168.31.131/core/install.php
```

发现目标使用：

```
Drupal 8.3.5
```

<img src=".\images\03.png" alt="01" style="zoom: 33%;" />

<img src=".\images\04.png" alt="04" style="zoom: 33%;" />

### 2. Drupalgeddon2 漏洞

发现目标版本符合 Drupalgeddon2 漏洞影响范围：

- Drupal 8.3.9 以下
- Drupal 8.4.6 以下
- Drupal 8.5.1 以下

漏洞名称：

```
Drupalgeddon2
```

漏洞类型：

```
Remote Code Execution (RCE)
```

利用工具：Metasploit。

## 四、漏洞利用与获取 Shell

使用 Metasploit 搜索并使用漏洞利用模块：

```
exploit/unix/webapp/drupal_drupalgeddon2
```

成功利用 Drupalgeddon2 漏洞，获取目标机器 Shell。

<img src=".\images\05.png" alt="01" style="zoom: 33%;" />

## 五、Linux 本地信息收集

### 1. 发现 SUID 文件

获取 Shell 后进行 Linux 本地信息收集，发现：

```
/opt/s
```

该文件具有 SUID 权限，进一步分析其执行行为。

<img src=".\images\06.png" alt="01" style="zoom: 33%;" />

### 2. 分析程序执行命令

发现程序执行以下命令：

```
scp -r /root/* b0x@127.0.0.1:/var/backup
```

分析：

<img src=".\images\07.png" alt="01" style="zoom: 33%;" />

程序调用了 `scp` 命令，并且使用的是命令名称，而不是绝对路径。

因此，可以进一步检查 PATH 环境变量对程序命令搜索路径的影响。

## 六、Linux 权限提升：SUID + PATH 劫持

### 1. 修改 PATH 环境变量

将 `/tmp` 放在 PATH 搜索路径的最前面：

```
export PATH=/tmp:$PATH
```

### 2. 利用 PATH 劫持

在 `/tmp` 下准备同名 `scp` 命令，使程序调用时优先搜索到该路径下的命令。

### 3. 执行 SUID 文件

```
./s
```

执行 SUID 程序，触发命令调用。

### 4. 验证提权结果

```
whoami
id
```

成功获得 root 权限。

<img src=".\images\08.png" alt="01" style="zoom: 33%;" />

## 七、最终结果

<badge color="success">提权成功</badge>

| 阶段         | 利用方式         | 结果              |
| ------------ | ---------------- | ----------------- |
| Web 信息收集 | Drupal 版本识别  | 发现 Drupal 8.3.5 |
| 漏洞利用     | Drupalgeddon2    | 获取 Shell        |
| 本地信息收集 | 查找 SUID 文件   | 发现 `/opt/s`     |
| 权限提升     | SUID + PATH 劫持 | 获取 root 权限    |

## 八、实战总结

本次靶机主要涉及两个阶段：

### 1. Web 漏洞利用

- 通过 Web 页面识别 Drupal 版本。
- 根据版本判断 Drupalgeddon2 漏洞。
- 使用 Metasploit 漏洞利用模块。
- 成功获取目标机器 Shell。

### 2. Linux 本地权限提升

- 发现具有 SUID 权限的 `/opt/s`。
- 分析程序调用的 `scp` 命令。
- 理解 PATH 环境变量的搜索顺序。
- 通过 PATH 劫持触发 SUID 程序中的命令调用。
- 最终获得 root 权限。

### 本次涉及的工具与命令

| 工具 / 命令    | 用途                   |
| -------------- | ---------------------- |
| Nmap           | 端口扫描               |
| 目录扫描工具   | Web 路径发现           |
| Metasploit     | Drupalgeddon2 漏洞利用 |
| `sysinfo`      | 查看目标系统信息       |
| `find`         | 查找 SUID 文件         |
| `ls -l`        | 查看文件权限           |
| `export PATH`  | 修改环境变量           |
| `scp`          | 文件传输命令           |
| `whoami`、`id` | 验证当前权限           |

### 个人复盘

本次实战让我进一步理解了 SUID 与 PATH 环境变量之间的关系。

之前只知道使用 `find` 查找 SUID 文件，这次通过分析 `/opt/s` 的执行行为，发现程序调用外部命令时，如果没有使用绝对路径，就需要考虑 PATH 劫持的可能性。

**核心收获：拿到 Shell 后，不要只关注系统版本，还要检查 SUID 文件，并结合程序执行逻辑寻找本地提权入口。**