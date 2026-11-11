# PUBLISHING · 把这个项目发到 GitHub

这份文件是给你自己看的操作手册，不是给贡献者看的。

---

## 0. 现状检查

本机（Windows）当前**没有安装 git**。`git --version` 报 "not recognized"。

两条路，任选：

- **路线 A：装 git，用命令行**（推荐，以后维护省事）
- **路线 B：不装 git，用 GitHub 网页上传**（5 分钟能上线，但后续每次改动都麻烦）

---

## 路线 A：安装 git 并推送

### A1. 安装（在 PowerShell 里执行）

```powershell
winget install --id Git.Git -e --source winget
```

装完**新开一个** PowerShell 窗口（当前窗口不会刷新 PATH），然后验证：

```powershell
git --version
```

### A2. 配置身份（只需一次）

```powershell
git config --global user.name "你的名字"
git config --global user.email "你的GitHub邮箱"
git config --global init.defaultBranch main
```

### A3. 初始化仓库

```powershell
cd "C:\Users\Administrator\Doubao\chats\2026-11-11\new-chat"
git init
git add .
git commit -m "Initial commit: concept, threat model, architecture, hardware plan (zh/EN)"
```

### A4. 在 GitHub 上建空仓库

1. 打开 https://github.com/new
2. Repository name 建议：`physical-isolation-wall`
3. Description 见下面第 3 节
4. **不要**勾选 "Add a README file"（会和本地文件冲突）
5. License 可以先不选，之后手动放 LICENSE
6. Create repository

### A5. 推送

```powershell
git remote add origin https://github.com/<你的用户名>/physical-isolation-wall.git
git branch -M main
git push -u origin main
```

首次推送会要求认证。两个选择：

- **浏览器登录**：新版本 git 会弹窗走 GitHub 的 OAuth，点几下即可。
- **Personal Access Token**：GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens，权限勾 `Contents: Read and write`，把 token 当密码填。
  > Windows 会把凭据存进"凭据管理器"，一次输入之后就不用再输。

---

## 路线 B：网页上传（不装 git）

1. https://github.com/new → 建空仓库（同样不勾 README）
2. 进入仓库 → **Add file** → **Upload files**
3. 把 `new-chat` 目录下**所有文件**拖进去（含 `docs` 文件夹，GitHub 网页支持拖拽文件夹）
4. Commit message 填 `Initial commit`
5. 提交

缺点：以后再改文件要重新上传，且历史记录会很乱。等你愿意装 git 了，可以再补：

```powershell
git init
git remote add origin https://github.com/<用户名>/physical-isolation-wall.git
git fetch origin
git reset --soft origin/main
git add .
git commit -m "Import"
git push
```

---

## 1. 仓库元信息（照抄即可）

**Description（英文，GitHub 显示在仓库标题下，限制 350 字符）**

```
A hardware write gate between your computer and the data you cannot afford to lose.
Default-closed, one-way request protocol, physical presence required for every write.
物理隔离墙：用硬件闸门把"写入"变成必须人在场亲手确认的物理行为。
```

**Website**：留空或指向你自己的说明页。

**Topics（标签，最多 20 个，建议这 12 个）**

```
open-hardware
hardware-security
data-integrity
physical-security
write-blocker
nfc
rp2040
rust
embedded
threat-model
air-gap
ransomware-protection
```

---

## 2. 发布前必须做的三件小事

1. **LICENSE 已放全文**（MIT），GitHub 会自动识别并在仓库页显示 MIT 徽标，无需再处理。
2. **确认 `docs/img` 不是空目录**，否则 git 不会提交它（已放了 `.gitkeep`）。
3. **在 README 顶部确认语言切换链接可用**：`README.zh-CN.md` 与 `README.md` 互相指向。

---

## 3. 建好之后建议立刻开的三类 Issue

开源项目最容易死于"没人知道从哪下手"。用 issue 把工作拆成可认领的块：

| 标题模板 | 用途 |
|---|---|
| `TASK: Verify TS3USB221-class mux defaults to disconnected at power-on` | 第一个硬件任务，门槛低，人人都能领 |
| `RFC: One-way request protocol and the no-return-channel problem` | 把 docs/02 的未解问题搬到台面上讨论 |
| `TASK: Translate docs/01–04 into Chinese` | 让中文社区能参与 |

再把 `docs/05-roadmap.md` 里的"Open research questions"逐条开成 `RFC:` issue——这些是最容易引来高质量评论的地方。

---

## 4. 一条关于呈现的提醒

GitHub 首屏只显示 `README.md`（英文版），中文版靠顶部的链接。如果你更希望中文读者第一眼看到中文，有两个办法：

- 把 `README.zh-CN.md` 的内容放到 `README.md` 顶部，用 `<details>` 折叠英文（会牺牲一点社区观感）；
- 保持现状，但在仓库 Description 里放一句中文（上面已经放了）。

社区惯例是英文优先，因为贡献者构成如此；你的目标如果是"让更多人参与"，现状更合适。

---

## 5. 后续维护的最小节奏

- 每合并一个 RFC，更新 `docs/05-roadmap.md` 的 RFC 索引。
- 每做出一块硬件，在 `docs/test-results/` 放一份实测记录，**包括失败的那些**。
- 打标签（release）前必须跑过模糊测试，这是 [CONTRIBUTING](CONTRIBUTING.md) 里写死的规则。

---

## 6. 如果以后要建组织/主页

暂时不需要。这类项目早期最值钱的是**一份能被挑刺的设计文档**，不是一个漂亮的落地页。等 P1 有实物照片了再考虑。
