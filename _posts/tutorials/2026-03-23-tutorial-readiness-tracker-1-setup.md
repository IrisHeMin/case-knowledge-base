---
layout: post
title: "Tutorial: 从零搭建 GitHub Pages 网站 — 第1篇：工具准备与项目初始化"
date: 2026-03-23
categories: [Tutorial, Web-Development]
tags: [github-pages, html, javascript, tutorial, beginner, git, vscode, readiness-tracker]
type: "tutorial"
---

# 从零搭建 GitHub Pages 网站 — 第1篇：工具准备与项目初始化

> 📚 本系列共 5 篇，手把手教你从零搭建一个团队技能追踪面板网站 (Readiness Tracker)。
> 完全面向小白，每一步都有操作说明和原理解释。
>
> 🔗 系列导航：
> - **第1篇：工具准备与项目初始化**（本篇）
> - 第2篇：数据设计与 JSON 文件创建
> - 第3篇：HTML 骨架与 Tailwind CSS 布局
> - 第4篇：JavaScript 动态渲染与 Chart.js 图表
> - 第5篇：localStorage 数据持久化与 GitHub Pages 部署

---

## 📌 我们要搭建什么？

一个**静态网页**，部署在 GitHub Pages 上，免费在线访问。

**最终效果**：[https://irishemin.github.io/readiness-tracker/](https://irishemin.github.io/readiness-tracker/)

它是一个团队技能成长追踪面板，能看到：
- 每个成员的学习进度百分比
- 团队整体统计数据（平均进度、完成人数、风险人数）
- 热力图（Heatmap）展示每个人在每个计划上的完成度
- 可交互的柱状图对比

### "静态网页"是什么意思？

| 类型 | 举例 | 特点 |
|------|------|------|
| **静态网页** | GitHub Pages、个人博客 | 就是一堆 HTML/CSS/JS 文件，浏览器直接打开就能看 |
| **动态网页** | 淘宝、知乎、Gmail | 需要服务器运行程序（Python/Java/Node.js），需要数据库 |

我们做的是**静态网页**——不需要服务器，不需要数据库。数据存在浏览器的 localStorage 里。

---

## 🛠️ 需要的工具（只有 3 个）

### 工具 1：VS Code（文本编辑器）

**是什么**：写代码的工具，就像"高级记事本"。

**为什么用它而不用 Windows 记事本？**
- 语法高亮：HTML 标签、JavaScript 关键字会显示不同颜色，方便阅读
- 自动补全：输入 `<div` 按 Tab 自动补全为 `<div></div>`
- 错误提示：拼写错误、语法错误会有红色波浪线提示
- 文件树：左侧可以看到整个项目的文件夹结构

**下载安装**：
1. 打开浏览器访问 [https://code.visualstudio.com/](https://code.visualstudio.com/)
2. 点击 **Download for Windows**
3. 运行安装程序，一路点 Next → Install
4. 安装时建议勾选 **"Add to PATH"**（这样你能在终端里输入 `code .` 直接打开）

### 工具 2：浏览器（Chrome 或 Edge）

**用途**：双击 `.html` 文件就能在浏览器里看到网页效果。

你的电脑已经有 Edge 了，不需要额外安装。Chrome 也可以。

### 工具 3：Git + GitHub 账号

**Git 是什么？**
一个代码版本管理工具。你可以把它理解为"代码的时光机"：
- 每次保存一个版本（commit），之后可以随时回退到任何历史版本
- 能把本地代码上传（push）到 GitHub

**GitHub 是什么？**
一个代码托管平台。你把代码放在 GitHub 上，它帮你：
- 存储代码（云端备份）
- 提供 GitHub Pages 服务（免费把 HTML 文件变成网站）
- 团队协作（多人同时编辑代码）

**安装 Git**：
1. 打开 [https://git-scm.com/download/win](https://git-scm.com/download/win)
2. 下载安装，所有选项保持默认即可

**注册 GitHub 账号**：
1. 打开 [https://github.com](https://github.com)
2. 点 **Sign up** → 输入邮箱、密码、用户名 → 完成注册

### ❌ 不需要安装的东西

| 工具 | 为什么不需要 |
|------|-------------|
| Node.js | 我们不用 React/Vue 等框架，不需要 npm |
| Python | 没有后端程序 |
| 数据库 | 用浏览器 localStorage 代替 |
| Webpack/Vite | 没有构建步骤，HTML 文件直接就能运行 |

---

## 🚀 第一步操作：在 GitHub 上创建仓库

**仓库（Repository）是什么？**
就是一个"云端文件夹"。你的所有代码文件都放在里面。GitHub Pages 会自动把仓库里的 HTML 文件变成网站。

**具体操作：**

1. 登录 [https://github.com](https://github.com)
2. 点页面右上角的 **"+"** 号 → 选择 **"New repository"**
3. 填写以下信息：
   - **Repository name**: `readiness-tracker`（仓库名，也会成为网址的一部分）
   - **Description**（可选）: `Team readiness tracking dashboard`
   - **Visibility**: 选 **Public**（公开）
   - ✅ 勾选 **"Add a README file"**（自动创建一个说明文件）
4. 点 **"Create repository"**

**为什么选 Public？**
GitHub Pages 的免费版只支持公开仓库。如果你的组织有 GitHub Enterprise，可以用私有仓库。

完成后你会看到仓库页面，地址类似：
```
https://github.com/你的用户名/readiness-tracker
```

---

## 🚀 第二步操作：Clone 仓库到本地电脑

**Clone 是什么？**
就是把 GitHub 上的仓库"下载"到你的电脑上，这样你才能在本地编辑文件。
Clone 和普通下载的区别：Clone 会保留 Git 版本历史，以后可以方便地 push 回 GitHub。

**具体操作：**

打开 **PowerShell 终端**（你现在用的这个就行），输入：

```powershell
cd C:\Users\minh
git clone https://github.com/你的用户名/readiness-tracker.git
cd readiness-tracker
```

**每行命令的含义：**

| 命令 | 含义 |
|------|------|
| `cd C:\Users\minh` | cd = Change Directory（切换目录），进入你的用户文件夹 |
| `git clone https://...` | 从 GitHub 下载仓库，会在当前目录创建一个 `readiness-tracker` 文件夹 |
| `cd readiness-tracker` | 进入刚下载的项目文件夹 |

执行完后，你的 `C:\Users\minh\readiness-tracker` 文件夹里应该只有一个 `README.md` 文件。

---

## 🚀 第三步操作：创建文件夹结构

**为什么需要文件夹？**
和你电脑上"文档"、"图片"文件夹一样，把不同类型的文件归类，方便管理和查找。

**具体操作：**

继续在终端输入：

```powershell
mkdir data
mkdir data\plans
mkdir data\templates
mkdir .github
mkdir .github\workflows
```

**每个文件夹的用途：**

| 文件夹 | 存什么 | 谁会读取它 |
|--------|--------|-----------|
| `data\` | 数据文件的根目录 | 开发者参考用 |
| `data\plans\` | 每个成员的进度数据（JSON 文件） | 开发者参考用 |
| `data\templates\` | 学习计划模板 | 开发者参考用 |
| `.github\workflows\` | 自动部署配置文件 | GitHub Actions 自动读取 |

**为什么文件夹名以 `.` 开头（`.github`）？**
以 `.` 开头的文件夹在 Windows 资源管理器里默认是隐藏的。这是一种约定：
- `.github` → GitHub 专用配置（自动化流程、Issue 模板等）
- `.git` → Git 内部数据（clone 时自动创建）

现在你的项目结构：
```
readiness-tracker/
├── .github/
│   └── workflows/          ← GitHub Actions 自动部署配置
├── data/
│   ├── plans/              ← 成员进度数据
│   └── templates/          ← 学习计划模板
└── README.md               ← 项目说明
```

---

## 🚀 第四步操作：用 VS Code 打开项目

```powershell
code .
```

**这行命令做了什么？**
- `code` → 启动 VS Code 程序
- `.` → 表示"当前目录"，即打开 `readiness-tracker` 文件夹

VS Code 打开后：
- 左侧面板显示文件树（你能看到所有文件夹和文件）
- 之后的所有文件操作，都在 VS Code 里完成

**在 VS Code 里创建新文件：**
- 方法 1：左侧文件树 → 右键某个文件夹 → **New File** → 输入文件名
- 方法 2：快捷键 `Ctrl + N` 创建新文件 → `Ctrl + S` 保存时选择路径和文件名

---

## ✅ 第1篇完成！

到这里，你的开发环境已经准备好了：
- ✅ VS Code 安装完毕
- ✅ Git 安装完毕
- ✅ GitHub 仓库创建完毕
- ✅ 仓库 Clone 到本地
- ✅ 文件夹结构创建完毕
- ✅ VS Code 已打开项目

**下一篇**：我们开始设计数据结构，创建 JSON 数据文件——这是写代码之前最重要的一步。

---

*📖 系列索引：工具准备 → [数据设计](../tutorial-readiness-tracker-2-data-design/) → [HTML 骨架](../tutorial-readiness-tracker-3-html-layout/) → [JS 动态渲染](../tutorial-readiness-tracker-4-javascript-rendering/) → [部署上线](../tutorial-readiness-tracker-5-deploy/)*
