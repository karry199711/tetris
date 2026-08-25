# GitHub 推送失败问题总结（学习笔记）

> 记录一次 `git push` 时遇到 `Connection was reset` 错误的完整排查过程。
> 场景：Windows + Git Bash，首次把本地项目推送到 GitHub。

---

## 一、问题表现

### 1. 原始报错

在 Git Bash 执行推送命令时：

```bash
git push -u origin main
```

报错：

```
fatal: unable to access 'https://github.com/karry199711/tetris.git/': Recv failure: Connection was reset
```

翻译过来就是：**无法访问 GitHub，连接被重置了**。

### 2. 关键特征

- 命令本身没写错（`remote add`、`push` 语法都对）
- 报错出现在「访问远程地址」这一步，说明是**网络层**的问题，不是 git 配置问题
- 表现是超时/重置，而不是「账号密码错误」（那会是 `Authentication failed`）

---

## 二、诊断过程

先用 `curl`（命令行 HTTP 请求工具）测试各个 GitHub 域名的连通性：

```bash
curl -sS -o /dev/null -w "HTTP %{http_code}" --connect-timeout 8 https://github.com
```

| 域名 | 结果 | 说明 |
| --- | --- | --- |
| `github.com` | ❌ 超时 (HTTP 000) | 主站，也就是 push 用的域名 |
| `api.github.com` | ✅ 200 | GitHub 的 API 接口，正常 |
| `codeload.github.com` | ✅ 301 | 下载源码用的，正常 |
| `raw.githubusercontent.com` | ❌ 超时 | 原始文件/CDN，超时 |

再用 `nslookup`（DNS 查询工具）看域名解析到哪个 IP：

```bash
nslookup github.com
```

关键输出：

```
github.com       →  20.205.243.166
api.github.com   →  20.205.243.168
```

最后手动测试 GitHub 真实的 IP 段（`140.82.x.x`）：

```bash
curl --resolve github.com:443:140.82.113.3 https://github.com
```

| IP | 结果 |
| --- | --- |
| `140.82.113.3` | ✅ 200 |
| `140.82.114.3` | ✅ 200 |
| `140.82.116.3` | ✅ 200 |
| `140.82.112.4` | ✅ 200 |

**结论**：不是全断了，而是 `github.com` 被「解析到了错误的 IP」，只要指回正确 IP 就能通。

---

## 三、问题原因：DNS 污染

### 1. 什么是 DNS

DNS（Domain Name System，域名系统）相当于「互联网的电话簿」：

```
你输入  github.com   →  DNS 查询  →  返回 IP 地址   →  浏览器/git 连接这个 IP
```

正常时，`github.com` 应该解析到 `140.82.x.x` 段的 IP。

### 2. DNS 污染是什么

这次的问题叫 **DNS 污染（DNS pollution / DNS hijacking）**：

- 你的电脑查询 `github.com` 时，返回了一个**被故意干扰/封锁的假 IP** `20.205.243.166`
- 这个 IP 连不通，于是 git 报「连接被重置/超时」
- 而 `api.github.com` 恰好没被污染（或污染到的 IP 还能通），所以它能正常访问

> 这是国内访问 GitHub 常见的网络问题，和你的电脑、你的 git 操作无关。

### 3. 为什么有的域名能通、有的不能

- 污染往往是「针对个别热门域名」的，`github.com` 是重点对象
- 同一时刻 `api.github.com`、`codeload.github.com` 可能没被污染，所以正常

---

## 四、处理方法：修改 hosts 文件

### 1. 原理

`hosts` 文件是本机的「本地 DNS 表」，**优先级高于 DNS 服务器**：

```
hosts 文件里有记录 → 直接用它 → 不经过 DNS 查询 → 绕开污染
```

所以只要在 hosts 里写「github.com → 正确 IP」，就绕过了被污染的结果。

### 2. 具体步骤

**① 备份原文件**（好习惯）：

```bash
cp /c/Windows/System32/drivers/etc/hosts /c/Windows/System32/drivers/etc/hosts.bak
```

**② 追加两行**（用记事本以管理员身份打开，或命令行追加）：

```
# GitHub fix (DNS pollution)
140.82.113.3 github.com
```

**③ 刷新 DNS 缓存**（让改动立即生效）：

```bash
ipconfig /flushdns
```

**④ 验证**：

```bash
curl -sS -o /dev/null -w "HTTP %{http_code}" https://github.com
# 返回 HTTP 200 即成功
```

### 3. hosts 文件位置（各系统）

| 系统 | 路径 |
| --- | --- |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |
| macOS / Linux | `/etc/hosts` |

格式：`IP地址   域名`（中间用空格或 Tab 分隔）。

---

## 五、相关知识补充

### 1. GitHub 相关 IP 段（供以后失效时更换）

| 用途 | IP 段 / 常用 IP |
| --- | --- |
| github.com 主站 | `140.82.112.0/20`，常用 `140.82.113.3`、`140.82.114.3`、`140.82.116.3` |
| raw.githubusercontent.com | `185.199.108.133` ~ `185.199.111.133` |
| api.github.com | 一般无需手动指定（本次未被污染） |

> 这些 IP 不是永久不变的。如果哪天又连不上，重新用 `curl --resolve` 测一批新 IP 再更新 hosts。

### 2. 常用诊断命令

| 命令 | 作用 |
| --- | --- |
| `ping 域名` | 看域名能否连通（有时 ICMP 被禁，不太准） |
| `nslookup 域名` | 查域名解析到哪个 IP（诊断污染的关键） |
| `curl -I https://域名` | 测试 HTTP 能否访问 |
| `curl --resolve 域名:443:IP https://域名` | 强制用指定 IP 测试，判断「是不是 IP 的问题」 |
| `ipconfig /flushdns` | 刷新 Windows 的 DNS 缓存 |
| `git remote -v` | 查看本地配置的远程地址 |

### 3. git push 的认证方式（本次用到的知识点）

git 通过 HTTPS 推送时，GitHub 需要验证身份，常见三种：

1. **用户名 + Token**：`Username` 填用户名，`Password` 填 Personal Access Token（不再是账号密码）
2. **credential.helper store**：`git config --global credential.helper store` 让 git 记住凭据，存到 `~/.git-credentials`
3. **GitHub CLI**：`gh auth login` 走浏览器登录，最省心

> Token 相当于密码，不要提交进仓库或公开分享。

### 4. 排查「连不上 GitHub」的思路框架

```
1. 是命令错了还是网络错了？      → 看报错：Connection reset / timeout = 网络
2. 全断还是部分域名断？         → curl 测 github.com / api / codeload / raw
3. 是 DNS 问题还是 IP 问题？    → nslookup 看解析结果，curl --resolve 测正确 IP
4. 对症下药：
   - DNS 污染 → 改 hosts 指到正确 IP
   - 整体被墙 → 用代理/VPN，或 git 配置代理
```

---

## 六、本次小结

- **表现**：`git push` 报 `Connection was reset`
- **原因**：`github.com` 被 DNS 污染，解析到被封 IP `20.205.243.166`
- **处理**：hosts 里加 `140.82.113.3 github.com` → 刷新 DNS → 恢复
- **本质**：用本机 hosts 的「本地解析」绕开被污染的「远程解析」
