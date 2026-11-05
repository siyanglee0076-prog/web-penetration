# VulnHub - 64Base 靶机实战

## 1. 靶机信息

| 项目     | 内容                              |
| -------- | --------------------------------- |
| 靶机名称 | 64Base                            |
| 靶场平台 | VulnHub                           |
| 目标 IP  | `172.20.10.2`                     |
| 攻击机   | Kali Linux                        |
| 实验目标 | 获取全部 Flag，最终获取 Root 权限 |

## 2. 主机发现与端口扫描

### 2.1 目标主机

```
172.20.10.2
```

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

### 2.2 Nmap 扫描结果

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
4899/tcp  open  radmin
62964/tcp open  unknown
```

初步分析：

- 22：SSH 远程登录服务
- 80：HTTP Web 服务
- 4899：Radmin 服务标识
- 62964：未知服务，需要进一步枚举
- <img src=".\images\02.png" alt="01" style="zoom:33%;" />

## 3. Web 信息收集与线索分析

访问目标网站，发现以下信息。

### 3.1 线索一：Base64 编码

发现字符串：

```
dmlldyBzb3VyY2UgO0QK
```

解码：

```
echo "dmlldyBzb3VyY2UgO0QK" | base64 -d
```

结果：

```
view source ;D
```

提示：查看网页源代码。

### 3.2 线索二：身份验证提示

网页出现：

```
Only respond if you are a real Imperial-Class BountyHunter
```

提示：Imperial-Class 后添加 BountyHunter。

### 3.3 线索三：多层编码

网页源代码发现3条 Hex 字符串（合并）：

```
5a6d78685a7a4637546d705361566c59546d785062464a7654587056656c464953587055616b4a56576b644752574e7151586853534842575555684b6246524551586454656b5a77596d316a4d454e6e5054313943673d3d0a
```

解码得到：

```
flag1{NjRiYXNlOlRoMzUzQHIzTjBUZGFEcjAxRHpVQHJlTDAwSzFpbmc0Cg==}
```

继续解码 Flag 内容得到：

```
64base:Th353@r3N0TdaDr01DzU@reL00K1ing4
```

获得一组 用户：密码

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

## 4. Flag 2：进一步信息收集

目录遍历，找到正确登录页面。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

登录后网页源代码发现第二段 Hex 编码：

```
5a6d78685a7a4a37595568534d474e4954545a4d65546b7a5a444e6a645756584f54466b53465a70576c4d31616d49794d485a6b4d6b597757544a6e4c3252714d544a54626d51315a45566157464655614446525557383966516f3d0a
```

先进行 Hex 解码，再进行 Base64 解码，得到：

```
flag2{aHR0cHM6Ly93d3cueW91dHViZS5jb20vd2F0Y2g/dj12Snd5dEZXQTh1QQo=}
```

继续 Base64 解码：

```
https://www.youtube.com/watch?v=vJwytFWA8uA
```

访问视频链接，获取后续线索：使用Bp。

## 5. Flag 3：Web 路径发现

抓包重发拿到：

```
flag3{NTNjcjN0NWgzNzcvSW1wZXJpYWwtQ2xhc3MvQm91bnR5SHVudGVyL2xvZ2luLnBocD9mPWV4ZWMmYz1pZAo=}
```

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

Base64 解码得到：

```
53cr3t5h377/Imperial-Class/BountyHunter/login.php?f=exec&c=id
```

发现隐藏登录页面及相关参数：

- 隐藏目录：`53cr3t5h377`
- 页面：`Imperial-Class/BountyHunter/login.php`
- 参数：`f=exec`
- 参数：`c=id`

## 6. Flag 4：获取第二组凭据

网页Post下提示用system替换exec发现：

```
flag4{NjRiYXNlOjY0YmFzZTVoMzc3Cg==}
```

<img src=".\images\08.png" alt="01" style="zoom:33%;" />

Base64 解码得到：

```
64base:64base5h377
```

拿到一组 用户：密码

## 7. Flag 5：文件线索

linux远程登陆 64base 用户后，在目录下发现：

```
flag5{TG9vayBJbnNpZGUhIDpECg==}
```

Base64 解码得到：

```
Look Inside! :D
```

提示：继续检查相关文件及其内部信息。

## 8. 加密文件分析

查看flag5完整信息，发现一段 Hex 编码字符串：

执行：

```
echo "HEX字符串" | xxd -p -r | base64 --decode
```

得到一段密钥，并保存密钥文件。

<img src=".\images\09.png" alt="01" style="zoom:33%;" />

查看图片，记录发现的密码线索：

```
usetheforce
```

使用该线索进一步分析加密数据。

## 9. Flag 6：最终文件分析

利用密钥+密码，ssh登录root用户，拿到：

<img src=".\images\10.png" alt="01" style="zoom:33%;" />

```
flag6{NGU1NDZiMzI1YTQ0NTEzMjRlMzI0NTMxNTk1NDU1MzA0ZTU0NmI3YTRkNDQ1MTM1NGU0NDRkN2E0ZDU0NWE2OTRlNDQ2YjMwNGQ3YTRkMzU0ZDdhNDkzMTRmNTQ1NTM0NGU0NDZiMzM0ZTZhNTk3OTRlNDQ2MzdhNGY1NDVhNjg0ZTU0NmIzMTRlN2E2MzMzNGU3YTU5MzA1OTdhNWE2YjRlN2E2NzdhNGQ1NDU5Nzg0ZDdhNDkzMTRlNmE0ZDM0NGU2YTQ5MzA0ZTdhNTUzMjRlMzI0NTMyNGQ3YTYzMzU0ZDdhNTUzMzRmNTQ1NjY4NGU1NDYzMzA0ZTZhNjM3YTRlNDQ0ZDMyNGU3YTRlNmI0ZDMyNTE3NzU5NTE2ZjNkMGEK}
```

逐层解码：

1. Base64 解码。
2. Hex 解码。
3. Base64 解码。
4. Hex 解码。
5. 最终得到 Base64 字符串。

最终解码结果：

```
base64 -d /var/local/.luke|less.real
```

执行命令，拿到设计图纸，完成靶机实战。

<img src=".\images\11.png" alt="01" style="zoom:33%;" />

## 10. 实验总结

### 10.1 获取的线索与凭据

| 阶段     | 发现内容                          |
| -------- | --------------------------------- |
| Flag 1   | 64base@r3N0TdaDr01DzU@reL00K1ing4 |
| Flag 2   | YouTube 视频链接                  |
| Flag 3   | 隐藏 Web 登录路径及命令执行参数   |
| Flag 4   | 64base:64base5h377                |
| Flag 5   | Look Inside!                      |
| 加密分析 | usetheforce                       |
| Flag 6   | 隐藏文件及命令执行线索            |

### 10.2 涉及的技术

- Nmap 主机与端口扫描
- Web 源代码分析
- Base64 编码与解码
- Hex 十六进制转换
- 多层编码分析
- 隐藏目录发现
- HTTP 参数分析
- 加密数据分析
- Linux 文件与命令分析

### 10.3 实验收获

本次实验主要通过网页线索、编码数据及隐藏文件逐步推进。过程中涉及多层 Base64 和 Hex 解码，需要结合上下文判断编码方式，而不能仅依赖工具进行机械解码。

通过本次实验，进一步熟悉了 Web 信息收集、编码分析、隐藏路径发现及 Linux 文件分析等基础渗透测试技术。