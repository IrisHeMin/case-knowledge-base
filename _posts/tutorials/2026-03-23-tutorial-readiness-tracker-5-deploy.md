---
layout: post
title: "Tutorial: 从零搭建 GitHub Pages 网站 — 第5篇：localStorage 数据持久化与 GitHub Pages 部署"
date: 2026-03-23
categories: [Tutorial, Web-Development]
tags: [github-pages, html, javascript, tutorial, beginner, localstorage, deployment, github-actions, readiness-tracker]
type: "tutorial"
---

# 从零搭建 GitHub Pages 网站 — 第5篇：localStorage 数据持久化与 GitHub Pages 部署

> 📚 系列导航：
> - [第1篇：工具准备与项目初始化](../tutorial-readiness-tracker-1-setup/)
> - [第2篇：数据设计与 JSON 文件创建](../tutorial-readiness-tracker-2-data-design/)
> - [第3篇：HTML 骨架与 Tailwind CSS 布局](../tutorial-readiness-tracker-3-html-layout/)
> - [第4篇：JavaScript 动态渲染与 Chart.js 图表](../tutorial-readiness-tracker-4-javascript-rendering/)
> - **第5篇：localStorage 数据持久化与 GitHub Pages 部署**（本篇）

---

## 📌 本篇目标

1. 深入理解 localStorage——页面之间怎么共享数据
2. 理解 index.html 和 my-plan.html 的数据流
3. 创建 GitHub Actions 部署配置
4. 推送代码到 GitHub
5. 开启 GitHub Pages，让网站上线

---

## 📖 深入理解 localStorage

### localStorage 是什么？

浏览器内置的一个**键值对存储**，就像一个只有两列的 Excel：

| Key（键） | Value（值） |
|-----------|------------|
| `readiness-team-config` | `{"teamName":"My Team","members":[...]}` |
| `readiness-shared-alice` | `{"plans":[...],"overallPct":68}` |
| `readiness-shared-bob` | `{"plans":[...],"overallPct":23}` |

### localStorage 的特性

| 特性 | 说明 |
|------|------|
| **持久化** | 关闭浏览器再打开，数据还在 |
| **同源策略** | 只有同一个域名下的页面能读写同一份 localStorage。比如 `irishemin.github.io` 下的所有页面共享一份 |
| **容量** | 约 5-10 MB（足够存几千个成员的数据） |
| **只能存字符串** | 存对象时要 `JSON.stringify()`，读出来要 `JSON.parse()` |
| **同步操作** | 读写是瞬间完成的（不像网络请求要等） |

### 为什么 index.html 和 my-plan.html 能共享数据？

因为它们部署在**同一个域名**下：

```
https://irishemin.github.io/readiness-tracker/index.html
https://irishemin.github.io/readiness-tracker/my-plan.html
                    ↑ 同一个域名 ↑
```

同一个域名下的所有页面，访问的是**同一份 localStorage**。
所以 my-plan.html 写入的数据，index.html 能读到。

### 数据流全景图

```
┌──────────────────────────────────────────────────────────────────┐
│                        浏览器 localStorage                       │
│                                                                  │
│  ┌─────────────────────────────┐  ┌────────────────────────────┐ │
│  │ readiness-team-config       │  │ readiness-shared-alice     │ │
│  │ {"teamName":"...",          │  │ {"plans":[...],"pct":68}   │ │
│  │  "members":[...]}           │  │                            │ │
│  └──────────┬──────────────────┘  └──────┬─────────────────────┘ │
│             │                            │                       │
│  ┌──────────┴──────────┐    ┌────────────┴───────────┐          │
│  │ readiness-shared-bob│    │readiness-shared-carol   │ ...      │
│  │ {"plans":...}       │    │{"plans":...}            │          │
│  └─────────────────────┘    └────────────────────────┘          │
└──────────────────────────────────────────────────────────────────┘
         ↑ 读取                          ↑ 写入
         │                               │
┌────────┴─────────┐            ┌────────┴──────────┐
│   index.html      │            │   my-plan.html     │
│  (Team Dashboard)  │            │ (Personal Workspace)│
│                    │            │                     │
│ • 读取团队配置     │            │ • 读取/写入团队配置  │
│ • 读取每个成员数据  │            │ • 写入当前用户的     │
│ • 只读，不修改成员  │            │   计划进度数据       │
│   进度数据         │            │                     │
│ • 每30秒自动刷新   │            │                     │
└────────────────────┘            └─────────────────────┘
```

**核心约定**：
- `readiness-team-config` → 团队配置（两个页面都会读写）
- `readiness-shared-{成员id}` → 每个成员的进度（my-plan.html 写入，index.html 读取）

这个约定就是两个页面之间的"协议"。只要双方都遵守这个 key 命名规则，数据就能正确传递。

### localStorage 的局限性

| 局限 | 影响 | 未来解决方案 |
|------|------|-------------|
| 数据只在当前浏览器 | 换电脑或换浏览器看不到数据 | 用 GitHub API 存到仓库的 JSON 文件 |
| 清除浏览器数据会丢失 | 用户清缓存后数据消失 | 同上 |
| 不能跨用户共享 | 每个人只能看自己的浏览器数据 | 需要后端服务或 GitHub API |

**这就是为什么 index.html 里有 `demoData`**——它是"保险"。
即使 localStorage 是空的，页面也不会白屏，而是显示预设的示例数据。

---

## 📖 怎么查看和调试 localStorage？

在浏览器里按 **F12** 打开 DevTools → 点 **Application** 标签 → 左侧找 **Local Storage** → 展开你的域名。

你能看到所有的 key-value 对，还能手动编辑、删除。

这在开发调试时非常有用：
- 想测试"有真实数据时的效果"→ 手动添加一个 key
- 想测试"没有数据时的效果"→ 删除所有 key
- 数据出 bug 了 → 查看具体的 value 是否符合预期

---

## 🔧 创建部署配置 — `.github/workflows/deploy.yml`

**为什么需要这个文件？**

GitHub Pages 需要知道"怎么把你的仓库变成网站"。
这个 YAML 文件告诉 GitHub："每当我 push 代码到 master 分支时，请自动部署。"

**什么是 GitHub Actions？**
GitHub 提供的自动化服务。你定义一组步骤（workflow），GitHub 在云端的虚拟机上自动执行。
常见用途：自动测试、自动构建、自动部署。
我们这里用它来自动把文件部署到 GitHub Pages。

**什么是 YAML？**
一种配置文件格式，用缩进表示层级。类似 JSON，但更适合人类阅读。

**操作**：在 VS Code 里，右键 `.github/workflows` → New File → `deploy.yml`

```yaml
name: Deploy to GitHub Pages
# ↑ 工作流的名字，会显示在 GitHub Actions 页面上

on:
  push:
    branches: ["master"]
# ↑ 触发条件：当有代码 push 到 master 分支时，自动执行
# 如果你的默认分支是 main，这里改成 "main"

  workflow_dispatch:
# ↑ 也允许在 GitHub 网页上手动点按钮触发

permissions:
  contents: read     # 允许读取仓库代码
  pages: write       # 允许写入 GitHub Pages
  id-token: write    # 身份验证令牌

concurrency:
  group: "pages"
  cancel-in-progress: false
# ↑ 并发控制：如果多次 push 同时触发，不取消正在进行的部署

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    # ↑ 部署完成后，这个 URL 就是你的网站地址

    runs-on: ubuntu-latest
    # ↑ 在 Ubuntu Linux 虚拟机上运行（GitHub 免费提供的）

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      # ↑ 第1步：从 GitHub 下载你的代码到虚拟机上
      # 类似你在本地 git clone 的过程

      - name: Setup Pages
        uses: actions/configure-pages@v5
      # ↑ 第2步：配置 GitHub Pages 设置

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: '.'
      # ↑ 第3步：把当前目录的所有文件打包上传
      # '.' 表示上传整个项目目录

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
      # ↑ 第4步：执行部署
```

**YAML 格式注意事项：**
- 用**空格**缩进（不能用 Tab！）
- 同级元素对齐
- `#` 后面是注释
- `-` 表示列表项
- `:` 后面必须有空格

---

## 🚀 推送代码到 GitHub

### 第一次推送

打开终端（PowerShell），确保你在项目目录：

```powershell
cd C:\Users\minh\readiness-tracker
```

然后依次执行：

```powershell
# 1. 查看有哪些文件变动
git status
```

你会看到一堆红色的文件名（untracked files = 还没被 Git 管理的文件）。

```powershell
# 2. 把所有文件加入暂存区
git add .
```

**`git add .` 是什么意思？**
- `git add` = 把文件标记为"准备提交"
- `.` = 当前目录下的所有文件
- 被 add 的文件会变成绿色（staged = 已暂存）

```powershell
# 3. 提交（创建一个版本快照）
git commit -m "Initial readiness tracker with team dashboard"
```

**`git commit` 是什么意思？**
- 把暂存区的文件保存为一个版本快照（像游戏存档）
- `-m "消息"` = 附带一条说明，描述这次改了什么
- 每次 commit 都有一个唯一的哈希值（如 `c716274`），可以随时回退到这个版本

```powershell
# 4. 推送到 GitHub
git push
```

**`git push` 是什么意思？**
把本地的所有 commit 上传到 GitHub 远程仓库。
推送完成后，刷新 GitHub 网页，你应该能看到所有文件。

### Git 工作流总结

```
本地文件修改 → git add → git commit → git push → GitHub 仓库
                ↑          ↑           ↑
               暂存        存档        上传

日后想修改网站：
1. 在 VS Code 里改文件
2. 保存文件
3. git add . && git commit -m "描述" && git push
4. GitHub Actions 自动重新部署
5. 1-2 分钟后网站更新
```

---

## 🌐 开启 GitHub Pages

### 操作步骤

1. 打开 `https://github.com/你的用户名/readiness-tracker`
2. 点页面上方的 **Settings**（⚙️ 图标）
3. 左侧菜单找到 **Pages**
4. **Build and deployment** 部分：
   - Source 下拉框选 **GitHub Actions**（不是 "Deploy from a branch"！）
5. 不需要其他设置，保存

### 验证部署

1. 回到仓库主页，点上方的 **Actions** 标签
2. 你应该看到一个名为 "Deploy to GitHub Pages" 的工作流正在运行（🟡 黄色圆圈）
3. 等 1-2 分钟变成 ✅ 绿色 = 部署成功
4. 你的网站地址就是：`https://你的用户名.github.io/readiness-tracker/`

### 如果部署失败？

点击失败的工作流 → 查看错误日志。常见问题：
- YAML 缩进错误（用了 Tab 而不是空格）
- 分支名不对（`deploy.yml` 里写的是 `master`，但你的默认分支是 `main`）
- 没有开启 Pages 设置中的 GitHub Actions 选项

---

## 🔄 日后更新网站的完整流程

```
[VS Code 里修改代码]
      │
      ▼
[浏览器双击 index.html 本地测试]
      │
      ├── 不满意 → 继续修改
      │
      └── 满意 ↓
            │
[终端执行三行命令]
  git add .
  git commit -m "修改说明"
  git push
      │
      ▼
[GitHub Actions 自动执行部署]
      │
      ▼ （约 1-2 分钟）
[网站自动更新 ✅]
```

**每次改代码后，只需要这三行命令就能更新网站。** 没有构建、没有编译、没有 npm install。

---

## 📁 最终项目结构

```
readiness-tracker/
├── .github/
│   └── workflows/
│       └── deploy.yml              ← GitHub Pages 自动部署配置
├── data/
│   ├── team.json                   ← 团队花名册（参考数据）
│   ├── templates/
│   │   └── azure-networking.json   ← 学习计划模板
│   └── plans/
│       ├── alice.json              ← 各成员的进度示例
│       ├── bob.json
│       ├── carol.json
│       ├── dave.json
│       ├── emma.json
│       └── frank.json
├── index.html                      ← 🏠 Team Dashboard（本教程实现的）
├── my-plan.html                    ← 🚀 个人工作台（下一步可实现）
└── README.md                       ← 项目说明
```

---

## 🎯 整个项目的技术栈回顾

| 层级 | 技术 | 为什么选它 |
|------|------|-----------|
| **结构** | HTML5 | 浏览器原生支持，不需要任何框架 |
| **样式** | Tailwind CSS (CDN) | 一行引入就能用，不需要构建工具 |
| **图表** | Chart.js (CDN) | 轻量级，一个 `<canvas>` + `new Chart()` 就能画图 |
| **逻辑** | 原生 JavaScript | 不用 React/Vue，代码量小的项目用原生更简单 |
| **数据** | localStorage | 不需要数据库和后端，浏览器自带 |
| **托管** | GitHub Pages | 免费、自动部署、全球 CDN 加速 |
| **CI/CD** | GitHub Actions | 内置在 GitHub 里，push 即部署 |

**总成本：$0**。零服务器费用，零域名费用（用 github.io 子域名）。

---

## 🗺️ 下一步可以做什么？

### 添加 my-plan.html（个人工作台）
- 让每个成员能创建自己的学习计划
- 勾选子任务，自动计算进度
- 数据自动写入 localStorage，Team Dashboard 自动显示

### 添加更多模板
- Azure Storage、Azure Compute/VM 等
- 在 `data/templates/` 里加新 JSON 文件

### 进阶功能
- GitHub API 集成（跨设备数据同步）
- 团队成员认证（GitHub OAuth）
- 进度趋势图（历史数据记录）
- 导出 PDF/PowerPoint 报告

---

## ✅ 系列教程完成！

恭喜！🎉 你现在完全理解了：

| 知识点 | 掌握程度 |
|--------|---------|
| HTML 文档结构和标签语法 | ✅ |
| Tailwind CSS utility class 样式 | ✅ |
| JSON 数据格式设计 | ✅ |
| JavaScript 数组操作（map/filter/reduce） | ✅ |
| DOM 操作（getElementById + innerHTML） | ✅ |
| Chart.js 图表创建 | ✅ |
| localStorage 持久化存储 | ✅ |
| GitHub Pages + Actions 自动部署 | ✅ |
| Git 基本工作流（add/commit/push） | ✅ |

**最重要的收获**：你理解了一个完整的前端项目从数据设计 → 页面布局 → 逻辑编写 → 部署上线的全流程。

这些知识可以复用到任何类似的项目：个人博客、项目管理面板、数据可视化页面……
核心模式都是一样的：**HTML 搭骨架 → CSS 加样式 → JavaScript 填数据 → Git 推上线**。

---

*📖 完整系列：[工具准备](../tutorial-readiness-tracker-1-setup/) → [数据设计](../tutorial-readiness-tracker-2-data-design/) → [HTML 骨架](../tutorial-readiness-tracker-3-html-layout/) → [JS 动态渲染](../tutorial-readiness-tracker-4-javascript-rendering/) → 部署上线*
