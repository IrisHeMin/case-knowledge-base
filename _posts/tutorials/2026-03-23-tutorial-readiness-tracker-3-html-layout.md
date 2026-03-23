---
layout: post
title: "Tutorial: 从零搭建 GitHub Pages 网站 — 第3篇：HTML 骨架与 Tailwind CSS 布局"
date: 2026-03-23
categories: [Tutorial, Web-Development]
tags: [github-pages, html, css, tailwind, tutorial, beginner, layout, readiness-tracker]
type: "tutorial"
---

# 从零搭建 GitHub Pages 网站 — 第3篇：HTML 骨架与 Tailwind CSS 布局

> 📚 系列导航：
> - [第1篇：工具准备与项目初始化](../tutorial-readiness-tracker-1-setup/)
> - [第2篇：数据设计与 JSON 文件创建](../tutorial-readiness-tracker-2-data-design/)
> - **第3篇：HTML 骨架与 Tailwind CSS 布局**（本篇）
> - 第4篇：JavaScript 动态渲染与 Chart.js 图表
> - 第5篇：localStorage 数据持久化与 GitHub Pages 部署

---

## 📌 本篇目标

写出 `index.html` 的 **HTML 结构部分**（不含 JavaScript 逻辑），让页面有完整的布局框架。
就像盖房子——先搭骨架（钢筋混凝土），后面再装修（填数据、画图表）。

---

## 📖 HTML 基础概念（3 分钟入门）

HTML = **H**yper**T**ext **M**arkup **L**anguage（超文本标记语言）。

它不是编程语言，而是一种**描述文档结构**的标记语言。
用"标签"（tag）来告诉浏览器"这段内容是标题"、"这段内容是段落"、"这里放一个按钮"。

### 标签的基本格式

```html
<标签名 属性="值">内容</标签名>
```

例如：
```html
<h1 class="text-xl font-bold">这是标题</h1>
<p>这是一段文字</p>
<div id="container">这是一个容器</div>
<button onclick="alert('hi')">点我</button>
```

| 标签 | 用途 | 类比 |
|------|------|------|
| `<h1>` ~ `<h6>` | 标题（h1 最大，h6 最小） | Word 里的"标题 1"~"标题 6" |
| `<p>` | 段落 | Word 里的普通段落 |
| `<div>` | 通用容器（不带任何样式） | 一个空的盒子，用来分组 |
| `<span>` | 行内容器（不换行） | 文字里的一小段标记 |
| `<a href="...">` | 超链接 | 点击跳转到另一个页面 |
| `<button>` | 按钮 | 可点击的交互元素 |
| `<input>` | 输入框 | 让用户输入文字 |
| `<table>` | 表格 | Excel 的表格 |
| `<canvas>` | 画布 | 用来画图（Chart.js 需要它） |

### id 和 class 的区别

```html
<div id="stats-cards" class="glass rounded-2xl shadow-lg">
```

| 属性 | 作用 | 规则 |
|------|------|------|
| `id` | 唯一标识，给 JavaScript 用来找这个元素 | 整个页面里不能重复 |
| `class` | 样式类名，给 CSS 用来加样式 | 可以多个，可以重复使用 |

**类比**：`id` 就像身份证号（唯一），`class` 就像标签/分类（可以多个）。

---

## 📖 什么是 Tailwind CSS？

**传统 CSS 写法：**
```html
<style>
    .card {
        background-color: white;
        border-radius: 16px;
        padding: 24px;
        box-shadow: 0 4px 15px rgba(0,0,0,0.1);
    }
</style>
<div class="card">内容</div>
```

**Tailwind CSS 写法：**
```html
<div class="bg-white rounded-2xl p-6 shadow-lg">内容</div>
```

**Tailwind 的核心思路**：把每个 CSS 属性都变成一个小 class，直接写在 HTML 标签上。

### Tailwind 常用 class 速查

| Tailwind class | 等价的 CSS | 含义 |
|---------------|-----------|------|
| `bg-white` | `background: white` | 白色背景 |
| `bg-gray-50` | `background: #f9fafb` | 极浅灰背景 |
| `bg-blue-500` | `background: #3b82f6` | 蓝色背景 |
| `text-gray-800` | `color: #1f2937` | 深灰文字 |
| `text-xl` | `font-size: 1.25rem` | 稍大文字 |
| `text-3xl` | `font-size: 1.875rem` | 大文字 |
| `font-bold` | `font-weight: 700` | 加粗 |
| `p-6` | `padding: 1.5rem` | 四周内边距 24px |
| `px-6` | `padding-left/right: 1.5rem` | 左右内边距 |
| `py-4` | `padding-top/bottom: 1rem` | 上下内边距 |
| `mb-8` | `margin-bottom: 2rem` | 下方外边距 |
| `rounded-2xl` | `border-radius: 1rem` | 大圆角 |
| `shadow-lg` | `box-shadow: ...` | 大阴影 |
| `flex` | `display: flex` | 弹性布局 |
| `grid` | `display: grid` | 网格布局 |
| `grid-cols-4` | `grid-template-columns: repeat(4, 1fr)` | 分成 4 列 |
| `gap-6` | `gap: 1.5rem` | 网格间距 |
| `hidden` | `display: none` | 隐藏元素 |
| `w-full` | `width: 100%` | 宽度 100% |
| `min-h-screen` | `min-height: 100vh` | 最小高度 = 屏幕高度 |

**为什么用 Tailwind 而不自己写 CSS？**
1. 不用在 HTML 和 CSS 文件之间来回切换
2. class 名就是样式的描述，看到 class 就知道长什么样
3. 不需要想 class 名（传统 CSS 最头疼的就是起名字）
4. 通过 CDN 引入一行代码就能用，零配置

---

## 🔧 开始写 index.html

在项目根目录创建 `index.html`，我们分区域逐步写入。

### Part 1：文件头部（Head）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Readiness Plan Tracker - Team Dashboard</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.7/dist/chart.umd.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');
        body { font-family: 'Inter', sans-serif; }
        .glass { background: rgba(255,255,255,0.85); backdrop-filter: blur(12px); }
        .heatmap-cell { transition: all 0.3s ease; cursor: pointer; }
        .heatmap-cell:hover { transform: scale(1.06); box-shadow: 0 4px 15px rgba(0,0,0,0.15); z-index: 10; position: relative; }
        .progress-bar-inner { transition: width 1s ease-in-out; }
        .card-hover:hover { transform: translateY(-2px); box-shadow: 0 8px 25px rgba(0,0,0,0.1); }
        .card-hover { transition: all 0.3s ease; }
        .fade-in { animation: fadeIn 0.5s ease-in; }
        @keyframes fadeIn { from { opacity:0; transform:translateY(10px); } to { opacity:1; transform:translateY(0); } }
        .sync-dot { display:inline-block; width:8px; height:8px; border-radius:50%; }
        .sync-dot.live { background:#10b981; box-shadow: 0 0 6px #10b981; }
        .sync-dot.none { background:#d1d5db; }
    </style>
</head>
```

**逐行解释：**

| 代码 | 为什么必须写 |
|------|-------------|
| `<!DOCTYPE html>` | 告诉浏览器这是 HTML5 文件。不写可能触发怪异模式（quirks mode），样式会出问题 |
| `<html lang="zh-CN">` | 根标签。`lang="zh-CN"` 告诉搜索引擎和屏幕阅读器内容是中文 |
| `<meta charset="UTF-8">` | 指定字符编码。不写的话，中文可能显示为乱码 |
| `<meta name="viewport" ...>` | 让页面在手机上正常显示。不写的话手机打开会看到缩小的桌面版 |
| `<title>` | 浏览器标签页上显示的标题 |
| `<script src="cdn.tailwindcss.com">` | 引入 Tailwind CSS。浏览器加载时会从 CDN 下载这个 JS 文件 |
| `<script src="chart.js...">` | 引入 Chart.js 图表库。后面画柱状图需要它 |
| `<style>` 里的内容 | Tailwind 虽然覆盖了大部分样式，但一些特殊效果（毛玻璃、动画）需要自定义 CSS |

**`<style>` 里每个自定义 class 的作用：**

| class | 效果 | 用在哪里 |
|-------|------|---------|
| `.glass` | 半透明白色 + 毛玻璃模糊效果 | 卡片背景，看起来更有层次感 |
| `.heatmap-cell:hover` | 鼠标悬停时放大 1.06 倍 + 阴影 | 热力图的每个格子 |
| `.progress-bar-inner` | 宽度变化时有 1 秒动画过渡 | 进度条从 0 到实际百分比的动画 |
| `.card-hover:hover` | 鼠标悬停时向上浮起 2px + 阴影 | 各种卡片，增加交互感 |
| `.fade-in` | 渐入动画（从下方淡入） | 成员卡片加载时的入场动画 |
| `.sync-dot` | 8px 小圆点 | 数据同步状态指示灯（绿色=实时，灰色=demo） |

### Part 2：页面顶部导航栏（Header）

```html
<body class="bg-gradient-to-br from-slate-50 to-blue-50 min-h-screen">

<header class="bg-white/80 backdrop-blur-md shadow-sm sticky top-0 z-50">
    <div class="max-w-7xl mx-auto px-6 py-4 flex items-center justify-between">
        <div class="flex items-center gap-3">
            <!-- Logo 图标 -->
            <div class="w-10 h-10 bg-gradient-to-br from-blue-500 to-indigo-600 rounded-xl flex items-center justify-center">
                <svg class="w-6 h-6 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z"/>
                </svg>
            </div>
            <!-- 标题 -->
            <div>
                <h1 class="text-xl font-bold text-gray-800">Readiness Plan Tracker</h1>
                <p class="text-xs text-gray-500" id="team-name-label">Cloud Networking Support Team</p>
            </div>
        </div>
        <!-- 右侧按钮 -->
        <div class="flex items-center gap-3">
            <span class="text-xs text-gray-400" id="data-source-label"></span>
            <button onclick="openTeamSettings()" class="px-4 py-2 text-sm bg-white border border-gray-200 rounded-xl hover:bg-gray-50 transition">⚙️ Team Settings</button>
            <a href="my-plan.html" class="px-4 py-2 text-sm bg-gradient-to-r from-blue-500 to-indigo-600 text-white rounded-xl hover:shadow-lg transition">🚀 My Plan</a>
        </div>
    </div>
</header>
```

**布局解析：**

```
┌──────────────────────────────────────────────────────────┐
│  [图标] Readiness Plan Tracker          [Settings] [My Plan] │
│         Cloud Networking Support Team                     │
└──────────────────────────────────────────────────────────┘
```

**关键 class 解释：**

| class | 作用 |
|-------|------|
| `bg-white/80` | 白色背景 + 80% 透明度（让滚动时内容能透过来） |
| `backdrop-blur-md` | 毛玻璃效果（模糊背后的内容） |
| `sticky top-0 z-50` | 粘性定位：滚动页面时导航栏始终固定在顶部。`z-50` 确保它在其他元素之上 |
| `max-w-7xl mx-auto` | 内容最大宽度 80rem（约 1280px），`mx-auto` 水平居中 |
| `flex items-center justify-between` | 弹性布局：子元素水平排列，垂直居中，两端对齐（左边logo+标题，右边按钮） |
| `bg-gradient-to-r from-blue-500 to-indigo-600` | 渐变背景：从左到右，蓝色渐变到靛蓝 |

### Part 3：主内容区域（空容器）

```html
<main class="max-w-7xl mx-auto px-6 py-8">
    <!-- 4 张统计卡片 -->
    <div class="grid grid-cols-4 gap-6 mb-8" id="stats-cards"></div>

    <!-- 进度条 + 柱状图（5列布局，左3右2） -->
    <div class="grid grid-cols-5 gap-6 mb-8">
        <div class="col-span-3 glass rounded-2xl shadow-lg p-6 card-hover">
            <h2 class="text-lg font-semibold text-gray-800 mb-4">📈 Individual Progress</h2>
            <div id="progress-bars"></div>
        </div>
        <div class="col-span-2 glass rounded-2xl shadow-lg p-6 card-hover">
            <h2 class="text-lg font-semibold text-gray-800 mb-4">📊 Team Comparison</h2>
            <canvas id="comparisonChart" height="280"></canvas>
        </div>
    </div>

    <!-- 热力图 -->
    <div class="glass rounded-2xl shadow-lg p-6 mb-8 card-hover">
        <h2 class="text-lg font-semibold text-gray-800 mb-4">🗺️ Plan Readiness Heatmap</h2>
        <div id="heatmap-container" class="overflow-x-auto"></div>
    </div>

    <!-- 成员详情卡片 -->
    <h2 class="text-lg font-semibold text-gray-800 mb-4">👥 Member Plan Details</h2>
    <div class="grid grid-cols-2 gap-6 mb-8" id="member-cards"></div>
</main>
```

**布局结构图：**

```
┌────────┬────────┬────────┬────────┐
│ 统计1  │ 统计2  │ 统计3  │ 统计4  │  ← grid-cols-4
└────────┴────────┴────────┴────────┘
┌────────────────────┬──────────────┐
│                    │              │
│   进度条列表        │  柱状图      │  ← grid-cols-5 (3+2)
│   (col-span-3)     │ (col-span-2) │
│                    │              │
└────────────────────┴──────────────┘
┌──────────────────────────────────┐
│           热力图表格              │  ← 全宽
└──────────────────────────────────┘
┌───────────────┬───────────────────┐
│  成员卡片 1    │  成员卡片 2        │  ← grid-cols-2
├───────────────┼───────────────────┤
│  成员卡片 3    │  成员卡片 4        │
└───────────────┴───────────────────┘
```

**为什么这些 div 的内容是空的？**
因为这些容器的内容会由 JavaScript 动态生成（下一篇）。
HTML 只需要把"空盒子"摆好位置，给每个盒子一个 `id`，JavaScript 通过 `id` 找到它们，把内容塞进去。

**关键设计模式 —— id 命名约定：**

| id | JavaScript 会往里填什么 |
|----|----------------------|
| `stats-cards` | 4 张统计卡片的 HTML |
| `progress-bars` | 6 个成员的进度条 HTML |
| `comparisonChart` | Chart.js 图表（画在 canvas 上） |
| `heatmap-container` | 动态生成的 HTML table |
| `member-cards` | 6 张成员详情卡片 |

### Part 4：页脚

```html
<footer class="text-center py-6 text-xs text-gray-400">
    Readiness Plan Tracker v2.1 · Data syncs via localStorage (same browser) ·
    <a href="my-plan.html" class="text-blue-400 hover:underline">Open My Plan →</a>
</footer>
```

### Part 5：Team Settings 弹窗（Modal）

```html
<!-- 弹窗覆盖层 -->
<div id="modal-team" class="fixed inset-0 bg-black/40 backdrop-blur-sm hidden z-[100] flex items-center justify-center">
    <!-- 弹窗内容 -->
    <div class="bg-white/95 backdrop-blur-xl rounded-2xl shadow-2xl p-8 max-w-lg w-full mx-4 max-h-[85vh] overflow-y-auto">
        <div class="flex items-center justify-between mb-6">
            <h2 class="text-xl font-bold text-gray-800">⚙️ Team Settings</h2>
            <button onclick="closeTeamSettings()" class="text-gray-400 hover:text-gray-600 text-xl">✕</button>
        </div>
        <!-- 团队名称输入框 -->
        <div class="mb-5">
            <label class="text-sm font-medium text-gray-600 block mb-1">团队名称</label>
            <input id="ts-team-name" type="text" class="w-full px-4 py-2.5 border border-gray-200 rounded-xl bg-white focus:ring-2 focus:ring-blue-200 outline-none">
        </div>
        <!-- 成员列表（JS 动态渲染） -->
        <div class="mb-4">
            <label class="text-sm font-medium text-gray-600 block mb-2">团队成员</label>
            <div id="ts-members-list" class="space-y-2"></div>
        </div>
        <!-- 添加成员 -->
        <div class="bg-gray-50 rounded-xl p-4 mb-6">
            <p class="text-sm font-medium text-gray-600 mb-2">➕ 添加成员</p>
            <div class="grid grid-cols-5 gap-2">
                <input id="ts-new-name" type="text" placeholder="姓名" class="col-span-2 px-3 py-2 border border-gray-200 rounded-lg text-sm">
                <input id="ts-new-role" type="text" placeholder="角色" class="col-span-2 px-3 py-2 border border-gray-200 rounded-lg text-sm">
                <button onclick="tsAddMember()" class="px-3 py-2 bg-blue-500 text-white rounded-lg text-sm font-medium">Add</button>
            </div>
        </div>
        <!-- 底部按钮 -->
        <div class="flex gap-3">
            <button onclick="tsSave()" class="flex-1 py-2.5 bg-blue-500 text-white rounded-xl font-medium">💾 Save</button>
            <button onclick="tsReset()" class="px-4 py-2.5 bg-red-50 text-red-500 rounded-xl text-sm">Reset to Demo</button>
            <button onclick="closeTeamSettings()" class="px-6 py-2.5 bg-gray-100 text-gray-600 rounded-xl">Cancel</button>
        </div>
    </div>
</div>
```

**弹窗（Modal）的实现原理：**

```
正常状态：弹窗有 class="hidden"，不显示

点击 "Team Settings" 按钮
  → JavaScript 执行 openTeamSettings()
  → 移除 hidden class
  → 弹窗显示

┌──────────────────────────────────┐
│      半透明黑色遮罩              │  ← fixed inset-0（全屏覆盖）
│      bg-black/40                │     z-[100]（最顶层）
│                                  │
│    ┌────────────────────┐       │
│    │  白色弹窗内容       │       │  ← 居中显示
│    │  ...               │       │
│    └────────────────────┘       │
│                                  │
└──────────────────────────────────┘

点击 "Cancel" 或 "✕"
  → JavaScript 执行 closeTeamSettings()
  → 加回 hidden class
  → 弹窗消失
```

---

## 🧪 验证：双击 index.html 看效果

现在你的 `index.html` 只有 HTML 结构（没有 JavaScript），打开后应该看到：
- 顶部导航栏（有标题和按钮）
- 空白的统计卡片区域
- "Individual Progress" 和 "Team Comparison" 两个卡片（内容为空）
- "Plan Readiness Heatmap" 卡片（内容为空）
- 空白的成员卡片区域

**这是正常的！** 因为我们还没写 JavaScript 来填充内容。
但你已经能看到页面的整体布局和样式了。

---

## ✅ 第3篇完成！

- ✅ 理解了 HTML 标签的基本语法
- ✅ 理解了 Tailwind CSS 的 class 命名逻辑
- ✅ 写完了 index.html 的所有 HTML 结构
- ✅ 理解了 Grid 网格布局的工作方式
- ✅ 理解了弹窗（Modal）的显示/隐藏原理
- ✅ 理解了"空容器 + id"的设计模式（HTML 负责布局，JS 负责填内容）

**下一篇**：写 JavaScript —— 让数据"活"起来，动态渲染进度条、图表、热力图。

---

*📖 系列索引：[工具准备](../tutorial-readiness-tracker-1-setup/) → [数据设计](../tutorial-readiness-tracker-2-data-design/) → HTML 骨架 → [JS 动态渲染](../tutorial-readiness-tracker-4-javascript-rendering/) → [部署上线](../tutorial-readiness-tracker-5-deploy/)*
