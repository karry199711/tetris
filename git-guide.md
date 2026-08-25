# Git 上传 GitHub 教程（以本「俄罗斯方块」项目为例）

> 适用环境：Windows + Git Bash（你当前的终端）。
> 说明：这个文件夹其实已经 `git init` 并提交过一次了。下面按「从头到尾」的完整流程写，方便你熟悉每一步。重复执行 `git init` 也没关系。

---

## 0. 三个核心概念（先记住，命令就好懂了）

| 概念 | 比喻 | 说明 |
| --- | --- | --- |
| **工作区** | 你的桌面文件夹 | 就是 `index.html`、`README.md` 这些普通文件 |
| **暂存区** | 一个"购物车" | `git add` 把文件放进去，决定哪些要提交 |
| **本地仓库** | 一个"存档点" | `git commit` 把购物车里的东西正式存成一个版本 |

流程就是：**改文件 → `add`（放进购物车）→ `commit`（存档）→ `push`（传到 GitHub）**

---

## 1. 一次性准备：告诉 Git 你是谁

```bash
git config user.name "karry199711"
git config user.email "1033588792@qq.com"
```

- `config`：配置。这里设置提交记录上显示的名字和邮箱。
- 邮箱最好和你 GitHub 账号里绑定的邮箱一致，这样 GitHub 才能把提交「认领」到你名下。
- 加 `--global`（如 `git config --global user.name "..."`）则对电脑上所有项目生效；不加只对当前项目生效。

---

## 2. 初始化仓库（新项目第一次才需要）

```bash
git init
```

- 在当前文件夹里创建一个隐藏的 `.git` 目录，从此这个文件夹就变成了一个「可被版本管理」的仓库。
- 运行后终端会提示 `Initialized empty Git repository ...`。

---

## 3. 把文件加入暂存区

```bash
git add .
```

- `add`：添加。`.` 表示「当前文件夹下所有文件」。
- 这一步只是「选中」文件放进购物车，还没有真正存档。
- 想看当前状态可以随时运行：`git status`（会显示哪些文件新增/修改/已暂存）。

---

## 4. 提交（存档）

```bash
git commit -m "feat: 俄罗斯方块游戏"
```

- `commit`：提交，把暂存区的文件正式存成一个版本快照。
- `-m` 后面的引号里是提交说明，写清楚「这次做了什么」。惯例前缀：`feat:`（新功能）、`fix:`（修复）、`docs:`（文档）等。

---

## 5. 在 GitHub 上创建远程仓库（二选一）

### 方式 A：网页手动创建（推荐新手）

1. 打开 https://github.com/new
2. Repository name 填 `tetris`（或你喜欢的名字）
3. 选 **Public**（公开，免费开启 Pages）
4. **不要**勾选 "Add a README / .gitignore / license"（保持空仓库，否则会和本地冲突）
5. 点 **Create repository**，页面会给你一段命令，可以忽略，按本教程第 6、7 步做即可。

### 方式 B：用命令行 API 创建（可选，需要 Token）

```bash
curl -X POST "https://api.github.com/user/repos" \
  -H "Authorization: token 你的TOKEN" \
  -d '{"name":"tetris","description":"俄罗斯方块","public":true}'
```

- `curl`：一个命令行 HTTP 请求工具，这里调用 GitHub 的接口建仓库。
- `-H`：请求头，`Authorization: token ...` 用来带身份认证。
- `-d`：请求体（JSON），指定仓库名、描述、是否公开。

---

## 6. 关联远程仓库

```bash
git remote add origin https://github.com/karry199711/tetris.git
```

- `remote`：远程仓库。`add`：添加。`origin`：给这个远程地址起的一个别名（约定俗成叫 origin）。
- 意思是「告诉本地仓库：你的远程地址是这个 URL」。只关联一次即可。
- 查看已关联的远程地址：`git remote -v`

---

## 7. 推送到 GitHub（核心一步）

```bash
git push -u origin main
```

- `push`：推送，把本地提交传到 GitHub。
- `origin`：上一步定义的远程别名。`main`：分支名（GitHub 默认主分支）。
- `-u`：建立「本地 main ↔ 远程 main」的跟踪关系，以后只需 `git push` 就行。

> **分支名注意**：你本地当前分支可能是 `master`（旧默认名）。可以先改名为 main：
> ```bash
> git branch -M main
> ```
> `branch -M`：重命名分支。之后再 `git push -u origin main`。

### 关于身份认证（推送到私有/新仓库时需要）

推送时 GitHub 要确认「你是谁」。三种方式任选：

1. **Token 一次性推送**（最简单，不长期保存密码）：
   ```bash
   git push https://karry199711:你的TOKEN@github.com/karry199711/tetris.git main
   ```
   把 `你的TOKEN` 换成你生成的那串 `ghp_...`。URL 里带上「用户名:Token」就能通过认证。

2. **让 Git 记住凭据**（以后免输入）：
   ```bash
   git config --global credential.helper store
   ```
   之后第一次推送时输入的用户名/Token 会被保存到本机 `~/.git-credentials`。

3. **用 GitHub CLI**（官方工具，体验最好）：
   ```bash
   gh auth login   # 走浏览器登录，之后用 gh 建仓库、推送都免密
   ```

---

## 8. 常用命令速查表

| 命令 | 作用 |
| --- | --- |
| `git status` | 查看当前状态（改了啥、暂存了啥） |
| `git log --oneline` | 查看提交历史（简洁版） |
| `git diff` | 查看未暂存的改动内容 |
| `git add 文件名` | 只暂存某个文件 |
| `git push` | 把本地提交推到远程（已跟踪分支时） |
| `git pull` | 拉取远程最新代码并合并 |
| `git clone 仓库URL` | 把远程仓库下载到本地 |
| `git remote -v` | 查看远程地址 |

---

## 9. 安全提示 ⚠️

- **Token 相当于密码**，千万不要把含 Token 的命令或文件提交进仓库（会被公开）。
- 本教程里的 `你的TOKEN` 都是占位符，实际使用时替换，用完可在 GitHub 上删除/轮换该 Token。
- 检查有没有误提交：提交前运行 `git status`，确认 `git-guide.md` 里没有真实 Token（本例没有）。
