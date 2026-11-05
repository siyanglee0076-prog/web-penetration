# VulnHub - Wakanda 靶机渗透测试

## 一、靶场信息

| 项目     | 内容                                                         |
| -------- | ------------------------------------------------------------ |
| 靶场名称 | Wakanda                                                      |
| 靶场平台 | VulnHub                                                      |
| 靶场 IP  | `172.20.10.3`                                                |
| 攻击机   | Kali Linux                                                   |
| 目标系统 | Linux                                                        |
| 渗透目标 | 获取 Root 权限                                               |
| 涉及技术 | 信息收集、PHP 文件包含、Base64 解码、SSH 登录、反弹 Shell、Python、pip 提权 |

## 二、信息收集

### 1. 主机发现

通过主机发现确定目标靶机 IP：

```
172.20.10.3
```

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

### 2. 端口扫描

使用 Nmap 对目标进行端口扫描：

```
nmap -sV -sC -p- 172.20.10.3
```

发现开放端口：

| 端口      | 状态 | 服务      |
| --------- | ---- | --------- |
| 80/tcp    | open | HTTP      |
| 111/tcp   | open | rpcbind   |
| 3333/tcp  | open | dec-notes |
| 55693/tcp | open | unknown   |

重点关注 80 端口，优先进行 Web 信息收集。

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

## 三、Web 信息收集与漏洞发现

### 1. 敏感路径扫描

使用目录扫描工具进行路径探测：

```
gobuster dir -u http://172.20.10.3 -w /usr/share/wordlists/dirb/common.txt
```

发现以下路径：

| 路径           | 状态码 | 大小 |
| -------------- | ------ | ---- |
| /admin         | 200    | 0    |
| /backup        | 200    | 0    |
| /index.php     | 200    | 1527 |
| /secret        | 200    | 0    |
| /server-status | 403    | 299  |
| /shell         | 200    | 0    |

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

### 2. 发现文件包含线索

查看网页源代码，发现被注释的语言切换链接：

```
<!-- <a class="nav-link active" href="?lang=fr">Fr/a> -->
```

推测网站可能通过 `lang` 参数加载不同语言文件，存在文件包含的可能性。

访问：

```
http://172.20.10.3/?lang=fr
```

结合页面行为，重点检查 PHP 文件包含机制。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

### 3. 利用 PHP 伪协议读取源码

尝试使用 PHP Filter 伪协议读取 `index.php`：

```
http://172.20.10.3/?lang=php://filter/convert.base64-encode/resource=index
```

页面返回一段 Base64 编码内容。

将其复制到本地进行解码：

```
echo 'Base64编码内容' | base64 -d
```

得到 PHP 源码：

```
<?php
$password ="Niamey4Ever227!!!" ;//I have to remember it

if (isset($_GET['lang']))
{
    include($_GET['lang'].".php");
}
?>
```

发现关键代码：

```
include($_GET['lang'].".php");
```

网站直接将用户传入的 `lang` 参数拼接为 PHP 文件名并执行包含操作，存在本地文件包含（LFI）风险。

同时，源码中泄露了一个密码：

```
Niamey4Ever227!!!
```

这为后续登录提供了重要线索。

## 四、获取初始访问权限

### 1. SSH 登录

根据网页源码中的作者信息：

```
mamadou
```

结合泄露的密码进行账号密码猜测。

最终成功通过 SSH 登录：

```
ssh mamadou@172.20.10.3
```

密码：

```
Niamey4Ever227!!!
```

成功获得 `mamadou` 用户的 Shell。

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

### 2. 获取交互式 Shell

为了获得更完整的终端交互体验，执行：

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

## 五、权限提升一：获取 Flag1

初始登录后，首先进行系统信息收集：

```
whoami
id
uname -a
sudo -l
```

查找 Flag 文件：

```
find / -name "flag*" 2>/dev/null
```

成功找到第一个 Flag：

```
d86b9ad71ca887f4dd1dac86ba1c4dfc
```

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

## 六、权限提升二：利用 devops 用户

### 1. 查找 devops 相关文件

继续进行系统枚举，重点检查其他用户创建的文件及可写文件：

```
ls -la /home
```

结合文件权限和文件内容，发现与 `devops` 用户有关的 Python 代码。

### 2. 分析 Python 代码与相关服务

进一步检查 Python 脚本的内容、文件权限以及相关服务的运行情况。

发现存在可编辑的 Python 代码，并且该代码与系统中的某个服务存在关联。

分析其运行机制后，判断可以通过修改 Python 代码改变服务执行的行为。

### 3. 利用 Python 代码获取反弹 Shell

在确认脚本确实会被目标服务调用、且当前用户拥有相应修改权限后，修改 Python 代码，使其在授权实验环境中建立反向连接。

在 Kali 攻击机上开启监听，等待目标服务触发代码。

成功获得 `devops` 用户的 Shell。

至此，完成从 `mamadou` 到 `devops` 的用户权限提升。

<img src=".\images\08.png" alt="01" style="zoom:33%;" />

## 七、权限提升三：利用 pip 获取 Root

### 1. 检查 devops 用户权限

登录 devops 后，检查当前用户具有的特殊权限：

```
sudo -l
```

发现：

```
devops 用户具有 pip 的 root 权限
```

这意味着可以重点分析 pip 在 sudo 环境下的执行机制。

### 2. 分析 pip 提权条件

pip 本身是 Python 的包管理工具，但如果允许用户通过 sudo 以 Root 身份执行 pip，且没有对安装内容及执行行为进行有效限制，就可能造成权限提升。

本质上是利用高权限运行的 Python 包安装过程执行具有系统权限的代码。

### 3. 利用 pip 获取 Root 权限

在授权靶场中，构造包含恶意 Python 安装逻辑的本地安装包，并利用具有 Root 权限的 pip 执行安装。

最终成功获取 Root Shell。

验证：

```
whoami
```

返回：

```
root
```

至此，完成 Wakanda 靶机的完整渗透测试。

<img src=".\images\09.png" alt="01" style="zoom:33%;" />

## 八、渗透过程总结

本次靶机主要涉及三个阶段：

| 阶段      | 利用点                   | 结果         |
| --------- | ------------------------ | ------------ |
| 初始访问  | PHP 文件包含源码读取     | 获取明文密码 |
| 初始登录  | SSH 弱口令/密码复用      | 登录 mamadou |
| 用户提权  | 可编辑 Python 脚本与服务 | 获取 devops  |
| Root 提权 | sudo pip 权限配置不当    | 获取 Root    |

### 本次涉及的知识点

1. Nmap 端口扫描与服务识别
2. Gobuster 目录扫描
3. PHP 文件包含漏洞
4. PHP Filter 伪协议读取源码
5. Base64 编码与解码
6. SSH 登录与交互式 Shell
7. Linux 用户及文件权限枚举
8. Python 脚本分析与服务利用
9. 反弹 Shell
10. sudo 权限配置不当
11. pip 提权原理

### 实战收获

- Web 页面中被注释的代码也可能隐藏重要线索。
- PHP Filter 伪协议可以帮助读取 PHP 源码，进一步发现硬编码的敏感信息。
- 初始权限获取后，不能只关注当前用户，还需要检查其他用户目录、可写文件以及运行中的服务。
- Linux 提权不一定依赖内核漏洞，错误的文件权限和 sudo 配置同样可能成为提权入口。
- 对于具有高权限的 Python 包管理工具，需要重点检查其执行权限和安装过程中的代码执行风险。