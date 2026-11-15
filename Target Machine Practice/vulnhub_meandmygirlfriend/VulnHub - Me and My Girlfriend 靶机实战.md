# VulnHub 靶机实战

## 一、靶机信息

- **靶机 IP：** `192.168.31.135`
- **开放端口：**

| 端口 | 服务 |
| ---- | ---- |
| 22   | SSH  |
| 80   | HTTP |

## 二、信息收集

### 1. 端口扫描

发现开放端口：

- 22：SSH
- 80：HTTP

### 2. Web 信息收集

访问 Web 页面后，发现页面提示：

```
<!-- Maybe you can search how to use x-forwarded-for -->
```

提示我们关注 HTTP 请求头中的 `X-Forwarded-For`。

**知识点：X-Forwarded-For（XFF）**

XFF 是一个 HTTP 请求头，通常用于传递客户端原始 IP 地址。

例如：

```
X-Forwarded-For: 127.0.0.1
```

如果网站使用 XFF 判断用户来源或限制访问，并且没有正确验证该请求头，就可能出现 IP 限制绕过问题。

使用 Burp Suite 拦截请求，在请求头中添加：

```
X-Forwarded-For: 127.0.0.1
```

观察页面响应是否发生变化。

随后页面提示：

```
Great! What you need now is reconn, attack and got the shell
```

提示继续进行信息收集、攻击并获取 Shell。

<img src=".\images\01.png" alt="01" style="zoom: 33%;" />

## 三、获取初始权限

### 1. 获取用户凭据

发现网页有注册页面，注册一个用户后，发现修改id参数可以实现水平越权。

<img src=".\images\02.png" alt="01" style="zoom: 33%;" />

水平越权后得到多个用户信息，爆破得到关键用户账密：

```
alice
4lic3
```

### 2. SSH 登录

使用 SSH 登录靶机：

```
ssh alice@192.168.31.135
```

登录成功后获得普通用户 Shell：

```
alice@gfriEND:/usr/bin$
```

当前用户为 `alice`，所在目录为 `/usr/bin`。

## 四、Linux 本地提权

### 1. 查看 sudo 权限

使用 sudo 相关命令检查当前用户可以执行的特权命令：

```
sudo -l
```

发现可以利用 PHP 执行系统命令。

### 2. PHP 提权

执行：

```
sudo php -r 'system("/bin/sh -i");'
```

命令解释：

- `sudo`：以更高权限执行命令。
- `php -r`：直接执行 PHP 代码，不需要创建 PHP 文件。
- `system()`：执行系统命令。
- `/bin/sh -i`：启动交互式 Shell。

如果当前用户被允许以 root 权限执行 PHP，那么 PHP 就可以调用系统 Shell，从而获得 root 权限。

### 3. 验证权限

```
whoami
id
```

如果输出显示 `root` 或 `uid=0(root)`，说明提权成功。

<img src=".\images\03.png" alt="01" style="zoom: 33%;" />

## 五、最终结果

| 阶段         | 结果                      |
| ------------ | ------------------------- |
| Web 信息收集 | 发现 X-Forwarded-For 提示 |
| IP 限制绕过  | 使用 XFF 尝试绕过         |
| 获取凭据     | `alice : 4lic3`           |
| 初始权限     | Alice 普通用户            |
| 本地提权     | sudo + PHP 执行系统命令   |
| 最终权限     | Root（需通过 `id` 验证）  |

## 六、实战总结

### 1. X-Forwarded-For

当 Web 页面源码或提示中出现 XFF 时，可以检查网站是否使用请求头判断客户端 IP。

但要注意：XFF 本身并不具备身份认证能力，不能直接相信用户提交的 IP。

### 2. Linux 提权

本次重点是 `sudo` 权限配置不当。

如果普通用户可以通过 sudo 以 root 权限执行 PHP，就可能利用 PHP 的系统命令执行能力启动高权限 Shell。

### 3. 关键命令

```
# SSH 登录
ssh alice@192.168.31.135

# 查看 sudo 权限
sudo -l

# PHP 执行系统命令
sudo php -r 'system("/bin/sh -i");'

# 验证权限
whoami
id
```