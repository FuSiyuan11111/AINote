# Git：本地目录 → 仓库 → GitHub

## 目录

- [本地普通目录初始化为 git 仓库并推送到 GitHub 的完整流程](#本地普通目录初始化为-git-仓库并推送到-github-的完整流程)

---

## 本地普通目录初始化为 git 仓库并推送到 GitHub 的完整流程
- 2026-09-28

**结论**：五步走：`git config`（一次性署名）→ 写 `.gitignore` → `git init` + `add` + `commit` → GitHub 网页建空仓库 → `git remote add origin <URL>` + `git push -u origin master`。之后日常只需 `git add . / commit / push` 三句循环。git 靠"三笔记忆"免重复配置：远程地址和分支关联在 `.git/config`，登录凭据在 Windows 凭据管理器，署名在 git config。

**背景**：把一个用 IDEA 直接打开的裸文件夹 Java 项目（AIFaceRecognition）发布到 GitHub，完整踩了一遍流程和网络坑。

**细节**：

第 0 步，一次性配置署名（只是提交记录的作者标签，不是登录凭据）：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

第 1 步，提交前先写 `.gitignore`，决定什么不上传（IDEA 的 `.idea/`、`*.iml`，编译输出 `out/`，绑定个人账号的激活文件等）。

第 2 步，初始化仓库——普通目录变 git 仓库，全部记忆（配置 + 历史）存进隐藏的 `.git` 文件夹：

```bash
git init
```

第 3 步，首次提交。**提交前必须 `git status --short` 检查暂存清单**：

```bash
git add .
git status --short
git commit -m "Initial commit: ArcSoft face recognition demo"
```

第 4 步，GitHub 网页建空仓库：New repository → 填名字 → **不勾选**任何 README / .gitignore / license 选项（本地已有，勾了会冲突）。

第 5 步，关联并推送：

```bash
git remote add origin https://github.com/用户名/AIFaceRecognition.git
git push -u origin master
```

地址错了或想换：`git remote set-url origin 新地址`（`add` 报 already exists 时用它）；查看：`git remote -v`。

首次 push 弹 Git Credential Manager 登录窗，浏览器授权后凭据存进 Windows 凭据管理器（控制面板 → 凭据管理器 → Windows 凭据 → `git:https://github.com`），换账号 = 删这条凭据。

`-u` 把"本地 master ↔ 远程 master"关联写进 `.git/config`——但它只在**成功推送后**写入；没推成功过之前，plain `git push` 不可用。

日常循环：

```bash
git add .
git commit -m "这次改了什么的简短说明"
git push
```

三笔记忆的位置：

| 记忆 | 存在哪里 | 失效条件 |
|---|---|---|
| 远程仓库地址、分支关联 | 项目里的 `.git/config` | 删除 `.git`、换电脑 |
| 登录凭据 | Windows 凭据管理器 | 手动删除、过期 |
| 作者署名 | git config（用户目录） | 重新配置 |

国内可双托管：

```bash
git remote add gitee https://gitee.com/用户名/AIFaceRecognition.git
git push -u gitee master
```

**坑**：

- **git 永久记住每一次提交**：密钥、账号绑定文件（如虹软的 `ArcFace64.dat`、填了真实 APP_ID/SDK_KEY 的 Constants.java）必须在**首次提交前**清理，提交过一次就永远留在历史里，事后删除无用。
- 教程里的占位符 URL 会被原样当真登记（`https://github.com/你的用户名/...` 是合法字符串，`remote add` 不报错），之后真地址加不进去（already exists）——改用 `set-url` 覆盖。
- **校园网可能选择性屏蔽 github.com**：现象是 `Recv failure: Connection was reset` 或超时，而 `api.github.com` 却能通；IDEA 的 "Share Project on GitHub" 走 OAuth 网页授权，被拦时报 `Incorrect credentials ... Access to this site has been restricted`——**不是密码错，是网络拦截**。诊断命令：`curl -sS -m 8 -o /dev/null -w "%{http_code}" https://github.com`。对策：换网络（手机热点）重试，或改推 Gitee。
- 推送失败时先分清是配置错还是网络错：报 `remote origin already exists` / `not a git repository` 是配置问题；`Connection was reset` / 超时是网络问题，命令本身没错，换网重试即可。
