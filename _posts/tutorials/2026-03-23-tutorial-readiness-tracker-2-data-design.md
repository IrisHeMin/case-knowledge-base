---
layout: post
title: "Tutorial: 从零搭建 GitHub Pages 网站 — 第2篇：数据设计与 JSON 文件创建"
date: 2026-03-23
categories: [Tutorial, Web-Development]
tags: [github-pages, html, javascript, tutorial, beginner, json, data-design, readiness-tracker]
type: "tutorial"
---

# 从零搭建 GitHub Pages 网站 — 第2篇：数据设计与 JSON 文件创建

> 📚 系列导航：
> - [第1篇：工具准备与项目初始化](../tutorial-readiness-tracker-1-setup/)
> - **第2篇：数据设计与 JSON 文件创建**（本篇）
> - 第3篇：HTML 骨架与 Tailwind CSS 布局
> - 第4篇：JavaScript 动态渲染与 Chart.js 图表
> - 第5篇：localStorage 数据持久化与 GitHub Pages 部署

---

## 📌 为什么先写数据？

**核心原则：先确定数据结构，再写展示代码。**

这就像做 Excel 报表——你先确定"表头有哪些列"（姓名、进度、日期...），才能设计报表格式。

如果先写 HTML，写到一半发现数据字段不对、结构不合理，就得全部返工。
先把数据定义好，后面写 JavaScript 时才知道：
- 要读什么字段
- 要显示什么内容
- 数据之间是什么关系

---

## 📖 什么是 JSON？

**JSON（JavaScript Object Notation）**= JavaScript 对象表示法。

就是一种通用的数据格式，用纯文本来表示结构化数据。

**类比理解：**
- Excel 表格 → 人类看的数据格式
- JSON 文件 → 程序看的数据格式

**JSON 的基本语法规则：**

```json
{
  "name": "Alice",
  "age": 28,
  "isActive": true,
  "skills": ["Networking", "DNS", "VPN"],
  "address": {
    "city": "Shanghai",
    "country": "China"
  }
}
```

| 语法 | 含义 | 例子 |
|------|------|------|
| `{ }` | 对象（一组键值对） | `{ "name": "Alice" }` |
| `[ ]` | 数组（有序列表） | `["DNS", "VPN"]` |
| `"key": "value"` | 键值对（key 必须用双引号） | `"name": "Alice"` |
| `"string"` | 字符串（文本） | `"Hello"` |
| `123` | 数字（不加引号） | `28` |
| `true / false` | 布尔值 | `true` |
| `null` | 空值 | `null` |

**JSON vs JavaScript 对象的区别：**
- JSON 的 key 必须加双引号：`"name"` ✅，`name` ❌
- JSON 不能有注释（`//` 和 `/* */` 都不行）
- JSON 不能有结尾逗号：`"a": 1, "b": 2` ✅，`"a": 1, "b": 2,` ❌

**为什么用 JSON 而不用 Excel？**
因为 JavaScript 可以直接读取和解析 JSON，但不能直接读取 Excel。
JSON 就是"程序员的表格"。

---

## 📋 数据架构设计

我们的数据分三层：

```
团队配置 (team.json)
  └── 成员列表
        └── 每个成员的学习计划
              └── 每个计划分成多个阶段 (Phase)
                    └── 每个阶段有多个里程碑 (Milestone)
```

形象地说，就像这样：

```
Cloud Networking Support Team（团队）
├── Alice Wang（成员）
│   └── Azure Networking Readiness（计划）
│       ├── Phase 1: Foundation（阶段）
│       │   ├── ✅ 完成基础学习路径（里程碑）
│       │   ├── ✅ 搭建实验环境（里程碑）
│       │   └── ✅ 完成 NSG Lab（里程碑）
│       ├── Phase 2: Intermediate
│       │   ├── ✅ 处理 3 个 Case
│       │   ├── ✅ VPN Lab
│       │   ├── 🔄 DNS Lab（进行中）
│       │   └── ✅ Shadow Senior
│       └── Phase 3: Advanced
│           ├── 🔄 AZ-700 认证（进行中）
│           ├── ⬜ 技术分享文档（未开始）
│           ├── 🔄 5 个高复杂 Case（进行中）
│           └── ⬜ 团队分享（未开始）
├── Bob Zhang
│   └── ...
└── Carol Li
    └── ...
```

---

## 🔧 创建文件 1：团队配置 — `data/team.json`

**操作**：在 VS Code 左侧，右键 `data` 文件夹 → **New File** → 输入 `team.json`

```json
{
  "team_name": "Cloud Networking Support Team",
  "manager": "David Chen",
  "members": [
    { "id": "alice",   "name": "Alice Wang",    "role": "Support Engineer II",     "focus_areas": ["Networking", "DNS"] },
    { "id": "bob",     "name": "Bob Zhang",     "role": "Support Engineer",        "focus_areas": ["Networking", "VPN"] },
    { "id": "carol",   "name": "Carol Li",      "role": "Support Engineer II",     "focus_areas": ["Storage", "Networking"] },
    { "id": "dave",    "name": "Dave Liu",      "role": "Support Engineer",        "focus_areas": ["Firewall", "Networking"] },
    { "id": "emma",    "name": "Emma Xu",       "role": "Senior Support Engineer", "focus_areas": ["Load Balancer", "Networking"] },
    { "id": "frank",   "name": "Frank Zhao",    "role": "Support Engineer",        "focus_areas": ["Networking", "ExpressRoute"] }
  ]
}
```

**每个字段的设计意图：**

| 字段 | 为什么要有它 |
|------|-------------|
| `"id": "alice"` | 唯一标识符。后面 JavaScript 用 `demoData["alice"]` 来查找这个人的数据。用简短的英文 id 而不是用名字，是因为名字可能重复或修改，id 不变 |
| `"name"` | 显示在页面上的可读名字 |
| `"role"` | 显示在名字旁边的角色标签 |
| `"focus_areas"` | 专注领域，供管理者了解团队技能分布（可选字段） |

**为什么 `members` 是数组 `[]` 而不是对象 `{}`？**
- 数组 = 有序列表，适合"一组同类型的东西"（6 个成员）
- 对象 = 无序的键值对，适合"一个东西的多个属性"（一个成员的 name/role/id）
- 数组可以轻松遍历（`members.forEach(...)`），对象需要用 `Object.keys()` 才能遍历

---

## 🔧 创建文件 2：计划模板 — `data/templates/azure-networking.json`

**操作**：右键 `data/templates` → New File → `azure-networking.json`

```json
{
  "template_id": "azure-networking",
  "title": "Azure Networking Readiness",
  "description": "Azure 网络技术技能提升计划，涵盖基础到高级",
  "estimated_weeks": 8,
  "phases": [
    {
      "phase": 1,
      "title": "基础概念 (Foundation)",
      "duration_weeks": 2,
      "milestones": [
        { "id": "net-101", "title": "完成 Azure Networking 基础学习路径", "type": "learning" },
        { "id": "net-102", "title": "搭建 Hub-Spoke 网络实验环境",       "type": "hands-on" },
        { "id": "net-103", "title": "完成 NSG/UDR 基础 Lab",            "type": "hands-on" }
      ]
    },
    {
      "phase": 2,
      "title": "进阶实践 (Intermediate)",
      "duration_weeks": 3,
      "milestones": [
        { "id": "net-201", "title": "独立处理 3 个 VNet Peering/Connectivity Case", "type": "case-practice" },
        { "id": "net-202", "title": "完成 VPN Gateway 排查 Lab",                     "type": "hands-on" },
        { "id": "net-203", "title": "完成 DNS 解析排查 Lab",                          "type": "hands-on" },
        { "id": "net-204", "title": "Shadow 一位 Senior 处理复杂 Case",               "type": "mentoring" }
      ]
    },
    {
      "phase": 3,
      "title": "深度掌握 (Advanced)",
      "duration_weeks": 3,
      "milestones": [
        { "id": "net-301", "title": "通过 AZ-700 认证",                "type": "certification" },
        { "id": "net-302", "title": "输出一篇技术分享文档 (TSG/KB)",    "type": "knowledge-sharing" },
        { "id": "net-303", "title": "独立处理 5 个高复杂度 Networking Case", "type": "case-practice" },
        { "id": "net-304", "title": "完成一次团队内部技术分享",          "type": "presentation" }
      ]
    }
  ]
}
```

**模板的作用：**
模板 = 预设的学习路径。成员创建计划时可以选择模板作为起点，然后根据自己的情况自定义。

**为什么 milestone 有 `"type"` 字段？**
方便后续做筛选和统计，比如"这个人完成了几个 hands-on 实验"。
虽然当前的 index.html 没有用到这个字段，但提前设计好方便以后扩展。
这就是**数据设计的前瞻性**——多存一点元数据，不会增加成本，但以后想用的时候就有了。

---

## 🔧 创建文件 3-8：每个成员的进度 — `data/plans/xxx.json`

为 6 个成员各创建一个文件。以 Alice 为例：

**操作**：右键 `data/plans` → New File → `alice.json`

```json
{
  "member_id": "alice",
  "plans": [
    {
      "plan_id": "alice-net-2026",
      "template_id": "azure-networking",
      "title": "Azure Networking Readiness",
      "start_date": "2026-01-15",
      "target_date": "2026-04-15",
      "status": "in_progress",
      "progress": [
        { "milestone_id": "net-101", "status": "completed",   "completed_date": "2026-01-28" },
        { "milestone_id": "net-102", "status": "completed",   "completed_date": "2026-02-05" },
        { "milestone_id": "net-103", "status": "completed",   "completed_date": "2026-02-12" },
        { "milestone_id": "net-201", "status": "completed",   "completed_date": "2026-03-01" },
        { "milestone_id": "net-202", "status": "completed",   "completed_date": "2026-03-08" },
        { "milestone_id": "net-203", "status": "in_progress"  },
        { "milestone_id": "net-204", "status": "completed",   "completed_date": "2026-02-20" },
        { "milestone_id": "net-301", "status": "in_progress"  },
        { "milestone_id": "net-302", "status": "not_started"  },
        { "milestone_id": "net-303", "status": "in_progress"  },
        { "milestone_id": "net-304", "status": "not_started"  }
      ]
    }
  ]
}
```

**其他成员创建类似文件**：`bob.json`、`carol.json`、`dave.json`、`emma.json`、`frank.json`。
每人的 progress 里 milestone 完成状态不同，体现不同的进度。

---

## ⚠️ 重要说明：这些 JSON 文件 ≠ 运行时数据

**一个关键概念要理解清楚：**

这些 `data/` 目录下的 JSON 文件，**不是网页运行时直接读取的数据源**。

```
                    ┌──────────────────────────┐
                    │   data/*.json 文件        │
                    │   （参考数据 / 设计文档）   │
                    │                          │
                    │   ❌ 网页不直接读取这些文件  │
                    └──────────────────────────┘

                    ┌──────────────────────────┐
                    │   浏览器 localStorage     │
                    │   （运行时实际数据源）      │
                    │                          │
                    │   ✅ index.html 读这里     │
                    │   ✅ my-plan.html 写这里   │
                    └──────────────────────────┘

                    ┌──────────────────────────┐
                    │   index.html 内嵌的        │
                    │   demoData 对象            │
                    │                          │
                    │   ✅ localStorage 没数据时  │
                    │      用这个兜底显示         │
                    └──────────────────────────┘
```

**为什么浏览器不直接读取 JSON 文件？**
浏览器出于安全限制，不允许 JavaScript 直接读取本地文件系统（你不会希望随便一个网页能读你电脑上的文件吧？）。

**那 data/ 目录有什么用？**
1. **设计参考**：帮你在写代码前确认数据结构
2. **文档作用**：其他开发者看到这些文件就知道数据长什么样
3. **未来扩展**：如果将来用 GitHub API 做后端，可以直接读写这些文件

**实际运行时的数据来源有两个：**
1. `index.html` 代码里硬编码的 `demoData` 对象（兜底数据）
2. 浏览器 `localStorage` 里的真实数据（由 `my-plan.html` 个人页面写入）

这个概念非常重要，理解了它，后面写 JavaScript 时就不会困惑"数据从哪里来"。

---

## ✅ 第2篇完成！

到这里，你完成了：
- ✅ 理解了 JSON 格式和语法
- ✅ 设计了三层数据架构（团队 → 成员 → 计划 → 阶段 → 里程碑）
- ✅ 创建了 `data/team.json`（团队配置）
- ✅ 创建了 `data/templates/azure-networking.json`（计划模板）
- ✅ 创建了 `data/plans/*.json`（成员进度）
- ✅ 理解了 JSON 文件 vs localStorage vs demoData 的关系

现在你的项目结构：
```
readiness-tracker/
├── .github/workflows/
├── data/
│   ├── team.json                   ← ✅ 团队配置
│   ├── templates/
│   │   └── azure-networking.json   ← ✅ 计划模板
│   └── plans/
│       ├── alice.json              ← ✅ 成员进度
│       ├── bob.json
│       ├── carol.json
│       ├── dave.json
│       ├── emma.json
│       └── frank.json
└── README.md
```

**下一篇**：开始写 `index.html` —— HTML 骨架和 Tailwind CSS 页面布局。

---

*📖 系列索引：[工具准备](../tutorial-readiness-tracker-1-setup/) → 数据设计 → [HTML 骨架](../tutorial-readiness-tracker-3-html-layout/) → [JS 动态渲染](../tutorial-readiness-tracker-4-javascript-rendering/) → [部署上线](../tutorial-readiness-tracker-5-deploy/)*
