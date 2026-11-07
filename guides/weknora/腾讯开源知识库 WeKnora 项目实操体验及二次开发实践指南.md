# 腾讯开源知识库 WeKnora 项目实操体验及二次开发实践指南

WeKnora（维娜拉）是腾讯开源的知识库问答系统，企业级文档理解、RAG 检索、Agent 推理与 Wiki 知识组织一体化平台。  
[官方 GitHub 仓库](https://github.com/Tencent/WeKnora) · [官方产品文档](https://weknora.weixin.qq.com/docs/01-getting-started/01-introduction)

南哥AGI研习社·开源门户：[https://nangeai.top](https://nangeai.top) 

---

## 一、基础准备

| 条件                      | 是否必须 | 说明                         |
| ----------------------- | ---- | -------------------------- |
| AI 编程工具                 | ✅ 必须 | Cursor、Claude Code、Codex 等 |
| GitHub                  | ✅ 必须 | 托管代码、跟踪 Issue、走 PR 流程      |
| GitHub CLI（gh）          | ✅ 必须 | Agent 创建 PR、关联 Issue等      |
| Git                     | ✅ 必须 | 本地代码版本管理                   |
| Docker + Docker Compose | ✅ 必须 | 部署项目所需各服务                  |

### 1.1 注册 GitHub 并准备 SSH

1. 打开 [https://github.com](https://github.com) 注册账号。
2. 进入 **Settings** → **SSH and GPG keys**。
3. 如果还没有 SSH 密钥，在终端执行：

```bash
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
```

复制公钥：

macOS：

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

Linux（需要 `xclip`）：

```bash
xclip -sel clip < ~/.ssh/id_ed25519.pub
```

Windows Git Bash：

```bash
cat ~/.ssh/id_ed25519.pub | clip
```

回到 GitHub，点击 **New SSH key**，粘贴公钥并保存。

测试 SSH：

```bash
ssh -T git@github.com
```

成功后会看到类似 `Hi 你的用户名!`

### 1.2 安装 Git

#### macOS

```bash
git --version
xcode-select --install
git --version
```

看到类似 `git version 2.x.x` 即成功。

#### Windows

1. 打开 [https://gitforwindows.org](https://gitforwindows.org) 下载 Git for Windows。
2. 一路使用默认配置安装。
3. 打开 Git Bash 或命令提示符，输入：

```bash
git --version
```

看到类似 `git version 2.x.windows.x` 即成功。

#### Linux

Ubuntu/Debian：

```bash
sudo apt-get update
sudo apt-get install git
git --version
```

CentOS/Fedora：

```bash
sudo yum install git
git --version
```

### 1.3 安装 GitHub CLI (gh)

#### macOS

```bash
gh --version
# 如果未安装，执行下列命令进行安装
brew install gh

# 安装后进行身份认证（建议使用 SSH 方式）
gh auth login

# （如遇权限问题）授予当前用户配置目录的权限
sudo chown -R "$(whoami):staff" ~/.config/gh

# 检查认证状态
gh auth status
```

#### Windows

1. 访问 [https://github.com/cli/cli/releases](https://github.com/cli/cli/releases) 下载 Windows 安装包（`gh_*_windows_amd64.msi`）。
2. 双击运行安装，完成后在“命令提示符”或 PowerShell 执行：

```bash
gh --version
```

3. 初始化并认证 GitHub 账户（建议选择 SSH 方式）：

```bash
gh auth login
```

4. 检查认证状态：

```bash
gh auth status
```

#### Linux

（以 Ubuntu/Debian 为例）

```bash
type -p curl >/dev/null || sudo apt install curl -y
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh -y

gh auth login
gh auth status
```

看到 `gh version x.x.x` 信息和认证通过即可。

### 1.4 安装 Docker

WeKnora 官方推荐用 Docker Compose 启动全栈。安装 [Docker Desktop](https://www.docker.com/) 或 Linux 上的 Docker Engine + Compose 插件后验证：

```bash
docker --version
docker compose version
```

## 二、上手实操

### 2.1 新建项目目录

```bash
mkdir my-weknora
cd my-weknora
# Cursor为例说明，打开目录
cursor .
```

### 2.2 克隆官方仓库到本地

```bash
cd my-weknora
git clone https://github.com/Tencent/WeKnora.git .
```

**说明**：`git clone` 把官方整份代码下载到当前项目根目录。

克隆完成后，在项目根目录应能看到 `README.md`、`LICENSE`、`docker-compose.yml`、`.env.example`、`frontend/`、`docreader/`、`cmd/`、`internal/` 等（以官方当前结构为准）。

**检查当前远程**（此时通常只有一个叫 `origin` 的远程，指向官方）：

```bash
git remote -v
```

类似：

```text
origin  https://github.com/Tencent/WeKnora.git (fetch)
origin  https://github.com/Tencent/WeKnora.git (push)
```

接下来把它改成：**upstream = 上游官方仓库，origin = 你的私有托管仓库**。

### 2.3 把 origin 改名为 upstream

```bash
git remote rename origin upstream
git remote -v
```

**成功标志**：

```text
upstream  https://github.com/Tencent/WeKnora.git (fetch)
upstream  https://github.com/Tencent/WeKnora.git (push)
```

### 2.4 在 GitHub 创建空私有仓库

1. 浏览器打开 [https://github.com/new](https://github.com/new)
2. 填写：
   - **Repository name**：`my-weknora`（或你喜欢的名字）
   - **Description**：可选，如「WeKnora 私有化定制版」
   - 选择 **Private**（闭源商用必选）
   - **不要**勾选 “Add a README file” / “Add .gitignore”
3. 点 **Create repository**，复制 SSH 地址：`git@github.com:你的用户名/my-weknora.git`

### 2.5 添加私有仓库为 origin

```bash
git remote add origin git@github.com:YOUR_USER/my-weknora.git
git remote -v
```

**成功标志**：

```text
origin    git@github.com:YOUR_USER/my-weknora.git (fetch)
origin    git@github.com:YOUR_USER/my-weknora.git (push)
upstream  https://github.com/Tencent/WeKnora.git (fetch)
upstream  https://github.com/Tencent/WeKnora.git (push)
```

### 2.6 创建定制分支 custom/main

```bash
git branch
git checkout -b custom/main
```

**成功标志**：当前分支为 `* custom/main`。

从当前所在的分支（往往是 `main` 或从官方同步后的分支）再拉出一条「你自己的定制线」，名字叫 `custom/main`。  
以后的二次开发、实验性改动优先在这条分支上做，避免直接改动「和上游对齐」的 `main`。

### 2.7 第一次推送到私有仓库

```bash
git push -u origin custom/main
```

**成功标志**：GitHub 私有库 `my-weknora` 分支 `custom/main` 可见完整代码。

### 2.8 创建 CUSTOM.md（强烈建议）

WeKnora 定制化功能务必从第一天记录。在仓库根目录新建 `CUSTOM.md`，可先照抄：

```markdown
# 相对 Tencent/WeKnora 的定制说明

## 品牌
- 产品名：（待填写）
- Logo / Favicon 路径：（待填写，常见在 frontend 静态资源目录）
- 默认语言 / 文案策略：（待填写）

## 部署与环境
- 目标部署方式：Docker Compose / Helm / Lite 单二进制 / 其他
- 关键 .env 覆盖项：（如注册模式、SYSTEM_AES_KEY、模型厂商 API）
- 是否启用可选 Profile：neo4j / minio / langfuse / searxng 等

## 我改过的文件
| 文件/目录 | 改了什么 | 日期 |
|-----------|----------|------|
| | | |

## 我故意删除或关闭的能力
| 路径 / 功能 | 原因 |
|-------------|------|
| | |

## 合并官方记录
| 日期 | 官方版本 | 有没有冲突 | 备注 |
|------|----------|------------|------|
| | | | |
```

```bash
git add CUSTOM.md
git commit -m "docs: 添加定制记录文件 CUSTOM.md"
git push
```

### 2.9 用官方 Compose 跑通基线

在 **尚未大改代码** 时跑通一次，后续的定制化改动与合并上游才有对照组：

```bash
cp .env.example .env
# 按需编辑 .env（模型、密钥等，见官方配置文档）
docker compose up -d
```

浏览器打开 **http://localhost**，按引导完成首次空间/账号配置。

## 三、AI辅助定制化开发

到这里，大家可以按照自己的 AI Coding 方法进行，下面我推荐一种可控 AI Coding SOP 方法，供大家参考。该 SOP 下 Skills 的获取方式，请点击下方链接查看了解。  
获取方式1（国内）:

++[https://mall.bilibili.com/neul-next/detailuniversal/detail.html?page=detailuniversal_detail&itemsId=41424824&loadingShow=1&noTitleBar=1#noReffer=true&msource=merchant_share](https://mall.bilibili.com/neul-next/detailuniversal/detail.html?page=detailuniversal_detail&itemsId=41424824&loadingShow=1&noTitleBar=1#noReffer=true&msource=merchant_share)++

获取方式2（国外）:

++[https://www.patreon.com/nangeagi/posts/ni-shi-bu-shi-ye-166882633?utm_medium=clipboard_copy&utm_source=copyLink&utm_campaign=postshare_creator&utm_content=join_link](https://www.patreon.com/nangeagi/posts/ni-shi-bu-shi-ye-166882633?utm_medium=clipboard_copy&utm_source=copyLink&utm_campaign=postshare_creator&utm_content=join_link)++

该可控 AI Coding SOP 的具体内容，大家可以参考以下视频合集，其中介绍得非常详细：  
B站链接（国内）：[https://www.bilibili.com/video/BV1zQNM6MEqf/](https://www.bilibili.com/video/BV1zQNM6MEqf/)  
YouTube链接（国外）：[https://www.youtube.com/playlist?list=PLRsjhp02BBRE](https://www.youtube.com/playlist?list=PLRsjhp02BBRE)



### 3.1 项目初始配置 — `/setup-skills`

它会依次问你 Issue 跟踪器、Triage 标签、领域文档布局、Repo Wiki。

**自然语言（在 Cursor 里说）**：

```bash
/setup-skills
```

| 步骤 | 问什么 | WeKnora 跟做怎么选 |
| --- | --- | --- |
| A | Issue 跟踪器 | **GitHub**（功能面大、迭代快） |
| B | Triage 标签 | 首次接受默认；有 UI Issue 时用 `design-input` |
| C | 领域文档 | **Single-context**（根目录 `CONTEXT.md` + `docs/adr/`，Go/Vue/Python 共用术语） |
| D | Repo Wiki 要不要？ | **启用**（语言默认 `zh-CN`）；WeKnora 模块多，后续用 `/repo-wiki` 沉淀人机共读导读；空仓布局可先只有说明文件 |

完成后生成 `AGENTS.md`（或 `CLAUDE.md`）与 `docs/agents/*`（含 `wiki.md`）。**每个仓库只跑一次**，改配置直接编辑 `docs/agents/*.md`。

**自然语言**：

```
提交并 push
```

### 3.2 查看代码结构 — `/zoom-out`

**自然语言（在 Cursor 里说）**：

```
/zoom-out

这是 Tencent WeKnora：Go 后端 cmd/server + internal/，Vue3 前端 frontend/，Python docreader/，
Docker Compose 部署。请画出：租户/RBAC、知识库与检索管道、Agent/MCP/Skills、
异步任务与可选 Neo4j/向量库 之间的关系；标注主要目录职责，不要贴大段代码。
```

### 3.3 开工前对齐 — `/grill-with-docs`

**自然语言（在 Cursor 里说）**：

```
/grill-with-docs 

我想基于 WeKnora 做私有化二次开发，改成自己的品牌产品（闭源部署给客户）。
品牌名称是：NANGEBASE。

v1 目标：
- 替换 Web 控制台产品名、Logo、登录页与关键 i18n 文案
- 不破坏 RAG 检索与 Agent 核心链路；定制写入 CUSTOM.md
- 生产部署走 Docker Compose

请一步步拷问我
```

**你要做的**：认真回答；术语写入 `CONTEXT.md`；重大决策写 ADR；UI 方向写入 `docs/design/`（若启用）。

```
提交并 push
```

### 3.4 沉淀需求 — `/to-prd`

```
/to-prd

把 v1 的讨论整理成 PRD。
```

确认无误后：

```
确认PRD全文，创建issue

完成后，提交并推送
```

### 3.5 拆任务 — `/to-issues`

v1 按 **垂直切片** 拆：

**自然语言**：

```
/to-issues 

按垂直切片拆 v1 PRD。
```

```
确认拆分合理，创建 issue
```

避免「先改完所有 Vue 页面再改 Go」的水平切片；后端行为变更与前端展示应通过 Issue 依赖顺序衔接。

### 3.6 写代码 — 分支 + `/tdd` + PR

每个 Issue：**拉分支 →** `/tdd` **→ push → CI → PR → 合并**。

**步骤 1：为 Issue 建分支**

**自然语言（在 Cursor 里说）**：

```
从最新 main 创建新分支，准备实现 todo-cli 的 GitHub Issue #[num]。
```

**步骤 2：在当前对话或新对话里跑** `/tdd`

**自然语言（在 Cursor 里说）**：

```
/tdd

实现该 Issue。
```

`/tdd` 会先按 issue「**PRD 必读**」3 项打开 PRD 对应段落，再进入红-绿-重构循环；每个 RED 循环按验收标准的 **PRD 依据** 精读 PRD。

**步骤 2**：

```
/tdd

从最新 custom/main 创建新分支，实现 Issue #[num]
```

**步骤 3：接入 CI**

在同一条分支上新增 GitHub Actions workflow，让 push 后云端自动跑。

**自然语言（在 Cursor 里说）**：

```
【首次设置即可】设置任意分支 push 和 PR 时都（不）要跑 CI。
```

**步骤 4：push 并等 CI 变绿**

**自然语言（在 Cursor 里说）**：

```
提交并 push。
```

push 后会自动运行 CI 检测，打开 GitHub → **Actions**，确认 workflow **CI** 最近一次运行是 ✅。若失败，把 Actions 日志贴给 `/diagnose`。完成之后，还未将代码合并到 main，需要提交 PR。

**自然语言（在 Cursor 里说）**：

```
创建 PR。
```

PR 后会自动运行 CI 检测，打开 GitHub → **Actions**，确认 workflow **CI** 最近一次运行是 ✅。若失败，把 Actions 日志贴给 `/diagnose`。也可以通过自然语言查看 CI 状态。

**自然语言（在 Cursor 里说）**：

```
请确认 CI 是否通过；失败则摘要原因并建议修复。
```

PR 完成之后，将代码合并到 main。

**自然语言（在 Cursor 里说）**：

```
合并该 PR 到 main，不要删除该分支，并确认该 Issue 已关闭。
```

PR 页 **Checks** 全绿后，再 **Merge pull request** 进 `main`。之后每张功能单重复：拉新分支 → `/tdd` → push → **等 CI** → PR → 合并。第一张合并后，可以跑一次 `/repo-wiki`，把当时的代码结构留给自己看。

### 3.7 出问题时 — `/diagnose`

WeKnora 常见卡点：Compose 服务不健康、docreader gRPC、Embedding 维度与向量索引、Agent 沙箱镜像、合并上游后 `.env.example` 新增变量未同步。

```
/diagnose

[描述现象：如 docker compose 某服务 restart、检索无结果、Agent 工具报错等]
```

### 3.8 v1 跑通后的维护 — `/improve-codebase-architecture`

```
/improve-codebase-architecture

WeKnora 体量大，请从「租户/知识库/检索/Agent」边界找可以加深模块、收窄接口的地方，
避免在 frontend 直接耦合 internal 领域类型。
```
