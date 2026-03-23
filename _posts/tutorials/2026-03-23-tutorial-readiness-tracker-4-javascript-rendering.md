---
layout: post
title: "Tutorial: 从零搭建 GitHub Pages 网站 — 第4篇：JavaScript 动态渲染与 Chart.js 图表"
date: 2026-03-23
categories: [Tutorial, Web-Development]
tags: [github-pages, html, javascript, chartjs, tutorial, beginner, dom, localstorage, readiness-tracker]
type: "tutorial"
---

# 从零搭建 GitHub Pages 网站 — 第4篇：JavaScript 动态渲染与 Chart.js 图表

> 📚 系列导航：
> - [第1篇：工具准备与项目初始化](../tutorial-readiness-tracker-1-setup/)
> - [第2篇：数据设计与 JSON 文件创建](../tutorial-readiness-tracker-2-data-design/)
> - [第3篇：HTML 骨架与 Tailwind CSS 布局](../tutorial-readiness-tracker-3-html-layout/)
> - **第4篇：JavaScript 动态渲染与 Chart.js 图表**（本篇）
> - 第5篇：localStorage 数据持久化与 GitHub Pages 部署

---

## 📌 本篇目标

写出 `index.html` 的 **`<script>` 部分**——让空的 HTML 容器被数据填满，图表画出来，页面"活"起来。

这是整个项目**代码量最大、逻辑最核心**的部分。我们把它拆成 7 个模块逐步完成。

---

## 📖 JavaScript 基础概念（5 分钟速通）

### 变量声明

```javascript
const name = "Alice";    // const = 常量，声明后不能重新赋值
let count = 0;           // let = 变量，可以重新赋值
count = 5;               // ✅ 可以
// name = "Bob";          // ❌ 报错，const 不能改
```

**什么时候用 const，什么时候用 let？**
- 默认用 `const`（大多数值定义后不需要改）
- 只有你确定后面要重新赋值时，才用 `let`

### 数组的三大操作：map / filter / reduce

```javascript
const nums = [10, 20, 30, 40, 50];

// map = 变换：把每个元素变成新东西，返回新数组
nums.map(n => n * 2);              // → [20, 40, 60, 80, 100]

// filter = 过滤：只保留满足条件的元素
nums.filter(n => n > 25);          // → [30, 40, 50]

// reduce = 归纳：把整个数组"浓缩"成一个值
nums.reduce((sum, n) => sum + n, 0); // → 150（求和）
//                              ↑ 初始值=0
```

**为什么这三个操作这么重要？**
因为我们的页面本质就是：**数据数组 → 变换/过滤/统计 → 生成 HTML**。

### 模板字符串（Template Literals）

```javascript
const name = "Alice";
const pct = 68;

// 传统字符串拼接（繁琐、易出错）
const html1 = "<p>" + name + ": " + pct + "%</p>";

// 模板字符串（推荐！用反引号 ` 包裹）
const html2 = `<p>${name}: ${pct}%</p>`;
```

**反引号 `` ` `` 在键盘上的位置**：左上角，数字 `1` 旁边，`Esc` 下面。

模板字符串的好处：
- `${...}` 里可以放任何 JavaScript 表达式
- 可以换行（传统字符串不能直接换行）
- 代码更清晰，少了一堆 `+` 号

### DOM 操作：JavaScript 如何修改页面

```javascript
// 找到 id="stats-cards" 的元素
const container = document.getElementById('stats-cards');

// 用新的 HTML 内容替换它内部的内容
container.innerHTML = '<p>Hello World</p>';
```

**这就是 JavaScript 和 HTML 的交互方式：**
1. HTML 先画好空容器（带 `id`）
2. JavaScript 用 `document.getElementById(id)` 找到它
3. 用 `.innerHTML = ...` 把内容塞进去

---

## 🔧 模块 1：团队配置与 Demo 数据

在 `</footer>` 之后、`</body>` 之前，加入 `<script>` 标签：

```html
<script>
// ========== 模块 1：团队配置 ==========

// localStorage 的 key（约定好的名字，两个页面都用同一个 key 来读写）
const TEAM_CONFIG_KEY = 'readiness-team-config';

// 默认团队成员（当 localStorage 没有数据时使用）
const defaultTeamMembers = [
    { id:"alice", name:"Alice Wang",  role:"Support Engineer II" },
    { id:"bob",   name:"Bob Zhang",   role:"Support Engineer" },
    { id:"carol", name:"Carol Li",    role:"Support Engineer II" },
    { id:"dave",  name:"Dave Liu",    role:"Support Engineer" },
    { id:"emma",  name:"Emma Xu",     role:"Senior Support Engineer" },
    { id:"frank", name:"Frank Zhao",  role:"Support Engineer" }
];

// 从 localStorage 读取团队配置
function loadTeamConfig() {
    try {
        const raw = localStorage.getItem(TEAM_CONFIG_KEY);
        // localStorage.getItem(key) → 返回字符串，key 不存在则返回 null
        if (raw) return JSON.parse(raw);
        // JSON.parse() → 把 JSON 字符串转回 JavaScript 对象
    } catch(e) {
        // 如果 JSON 格式损坏，JSON.parse() 会抛异常
        // catch 住它，避免整个页面崩溃
    }
    // 没有存储数据或数据损坏，返回默认值
    return { teamName: "Cloud Networking Support Team", members: defaultTeamMembers };
}

// 把团队配置保存到 localStorage
function saveTeamConfig(config) {
    localStorage.setItem(TEAM_CONFIG_KEY, JSON.stringify(config));
    // JSON.stringify() → 把 JavaScript 对象转成 JSON 字符串
    // localStorage 只能存字符串，所以必须先 stringify
}

// 初始化：加载配置
let teamConfig = loadTeamConfig();
let teamMembers = teamConfig.members;
```

**为什么用 try-catch？**

```javascript
try {
    // 尝试执行可能出错的代码
    JSON.parse("这不是合法的JSON");  // 💥 会抛出异常
} catch(e) {
    // 如果上面出错了，跳到这里处理
    // 不写 try-catch 的话，整个页面就白屏了
}
```

localStorage 的数据可能被用户在浏览器 DevTools 里手动修改过。
如果有人不小心把 JSON 改坏了，`JSON.parse()` 会报错。
`try-catch` 就是安全网——"试试看，出错了不要紧，用默认值就好"。

---

## 🔧 模块 2：Demo 兜底数据

```javascript
// ========== 模块 2：Demo 数据 ==========
// 当 localStorage 没有真实数据时，用这些假数据来展示页面效果
// 让用户第一次打开网站时就能看到完整的 Dashboard，而不是一片空白

const demoData = {
    alice: {
        plans: [{
            title: "Azure Networking Readiness",
            target_date: "2026-04-15",
            phases: [
                { title: "Foundation",   icon: "📚", total: 3, done: 3,   pct: 100 },
                { title: "Intermediate", icon: "🔧", total: 4, done: 3.5, pct: 88 },
                { title: "Advanced",     icon: "🏆", total: 4, done: 1,   pct: 25 }
            ],
            total: 11, done: 7.5, pct: 68
        }],
        overallPct: 68
    },
    bob:   { plans:[{title:"Azure Networking Readiness",target_date:"2026-05-01",
             phases:[{title:"Foundation",icon:"📚",total:3,done:2.5,pct:83},
                     {title:"Intermediate",icon:"🔧",total:4,done:0,pct:0},
                     {title:"Advanced",icon:"🏆",total:4,done:0,pct:0}],
             total:11,done:2.5,pct:23}], overallPct:23 },
    carol: { plans:[{title:"Azure Networking Readiness",target_date:"2026-03-31",
             phases:[{title:"Foundation",icon:"📚",total:3,done:3,pct:100},
                     {title:"Intermediate",icon:"🔧",total:4,done:4,pct:100},
                     {title:"Advanced",icon:"🏆",total:4,done:3.5,pct:88}],
             total:11,done:10.5,pct:95}], overallPct:95 },
    dave:  { plans:[{title:"Azure Networking Readiness",target_date:"2026-05-15",
             phases:[{title:"Foundation",icon:"📚",total:3,done:1.5,pct:50},
                     {title:"Intermediate",icon:"🔧",total:4,done:0,pct:0},
                     {title:"Advanced",icon:"🏆",total:4,done:0,pct:0}],
             total:11,done:1.5,pct:14}], overallPct:14 },
    emma:  { plans:[{title:"Azure Networking Readiness",target_date:"2026-02-01",
             phases:[{title:"Foundation",icon:"📚",total:3,done:3,pct:100},
                     {title:"Intermediate",icon:"🔧",total:4,done:4,pct:100},
                     {title:"Advanced",icon:"🏆",total:4,done:4,pct:100}],
             total:11,done:11,pct:100}], overallPct:100 },
    frank: { plans:[{title:"Azure Networking Readiness",target_date:"2026-06-01",
             phases:[{title:"Foundation",icon:"📚",total:3,done:0.5,pct:17},
                     {title:"Intermediate",icon:"🔧",total:4,done:0,pct:0},
                     {title:"Advanced",icon:"🏆",total:4,done:0,pct:0}],
             total:11,done:0.5,pct:5}], overallPct:5 }
};
```

**数据结构说明：**

```
demoData = {
  "alice": {                          ← 用成员 id 作为 key，方便用 demoData["alice"] 查找
    plans: [                          ← 计划数组（一个人可以有多个计划）
      {
        title: "Azure Networking...", ← 计划名称
        target_date: "2026-04-15",   ← 目标完成日期
        phases: [                    ← 阶段数组
          {
            title: "Foundation",     ← 阶段名
            icon: "📚",              ← 显示用的 emoji
            total: 3,                ← 该阶段总共有 3 个里程碑
            done: 3,                 ← 已完成 3 个
            pct: 100                 ← 完成百分比 = done/total × 100
          },
          ...
        ],
        total: 11,                   ← 所有阶段里程碑总数
        done: 7.5,                   ← 已完成的（0.5 = 进行中算半个）
        pct: 68                      ← 计划总体完成百分比
      }
    ],
    overallPct: 68                   ← 这个人所有计划的综合完成百分比
  },
  "bob": { ... },
  ...
}
```

---

## 🔧 模块 3：数据加载函数

```javascript
// ========== 模块 3：数据加载 ==========

function loadAllMemberData() {
    const result = {};       // 用一个空对象来收集每个成员的数据
    let liveCount = 0;       // 计数：有多少人有真实数据（非 demo）

    teamMembers.forEach(m => {
        // 对每个成员，尝试从 localStorage 读取真实数据
        try {
            const raw = localStorage.getItem(`readiness-shared-${m.id}`);
            // key 的格式是 readiness-shared-alice、readiness-shared-bob 等
            // 这个 key 是和 my-plan.html 页面约定好的
            if (raw) {
                const parsed = JSON.parse(raw);
                if (parsed.plans && parsed.plans.length > 0) {
                    // 有真实数据！标记来源为 'live'
                    result[m.id] = { ...parsed, source: 'live' };
                    liveCount++;
                    return;  // forEach 里的 return 相当于 continue（跳过本次循环）
                }
            }
        } catch(e) {}

        // 没有真实数据，用 demo 数据兜底
        result[m.id] = { ...demoData[m.id], source: 'demo' };
    });

    // 更新页面上的数据来源标签
    document.getElementById('data-source-label').innerHTML =
        liveCount > 0
        ? `<span class="sync-dot live"></span> ${liveCount}/${teamMembers.length} synced live`
        : `<span class="sync-dot none"></span> Showing demo data`;

    return result;
}
```

**`{ ...parsed, source: 'live' }` 是什么意思？**

```javascript
const parsed = { plans: [...], overallPct: 68 };

// 展开运算符 ... 把 parsed 的所有字段复制出来，然后追加一个 source 字段
const result = { ...parsed, source: 'live' };
// result = { plans: [...], overallPct: 68, source: 'live' }
```

这叫**对象展开**（spread），作用是"复制一个对象，同时加/改某些字段"，而不修改原对象。

---

## 🔧 模块 4：工具函数

```javascript
// ========== 模块 4：工具函数 ==========

// 百分比 → 热力图颜色（Tailwind class）
// 为什么写成函数？因为热力图和成员卡片都会用到，避免重复
function getHeatColor(pct) {
    if (pct >= 90) return "bg-emerald-500 text-white";   // 深绿色
    if (pct >= 75) return "bg-emerald-400 text-white";   // 浅绿色
    if (pct >= 50) return "bg-blue-400 text-white";      // 蓝色
    if (pct >= 25) return "bg-yellow-300 text-gray-800"; // 黄色（浅色背景用深色文字）
    if (pct > 0)   return "bg-orange-200 text-gray-800"; // 橙色
    return "bg-red-100 text-gray-500";                    // 浅红色（还没开始）
}

// 百分比 → 状态标签
function statusLabel(pct) {
    if (pct >= 100) return { text: "✅ Completed",      color: "text-emerald-600" };
    if (pct >= 50)  return { text: "🔄 On Track",       color: "text-blue-600" };
    if (pct >= 20)  return { text: "🔄 In Progress",    color: "text-blue-500" };
    return                  { text: "⚠️ Needs Attention", color: "text-orange-500" };
}

// ISO 时间 → 人性化时间
// "2026-03-23T10:00:00Z" → "2h ago"
function timeAgo(iso) {
    if (!iso) return '';
    const diff = Date.now() - new Date(iso).getTime();
    // Date.now() = 当前时间的毫秒时间戳
    // new Date(iso).getTime() = 把 ISO 字符串转成毫秒时间戳
    // 两者相减 = 时间差（毫秒）
    const mins = Math.floor(diff / 60000);  // 60000毫秒 = 1分钟
    if (mins < 1) return 'just now';
    if (mins < 60) return `${mins}m ago`;
    const hrs = Math.floor(mins / 60);
    if (hrs < 24) return `${hrs}h ago`;
    return `${Math.floor(hrs / 24)}d ago`;
}

// HTML 转义（防止 XSS 攻击）
function esc(s) {
    const d = document.createElement('div');
    d.textContent = s || '';  // textContent 会自动转义特殊字符
    return d.innerHTML;       // innerHTML 返回转义后的安全 HTML
}
// 例如：esc('<script>alert("hack")</script>')
// → '&lt;script&gt;alert("hack")&lt;/script&gt;'
// 防止用户在"团队名称"输入框里注入恶意代码
```

**为什么需要 `esc()` 函数？**

如果用户在 Team Settings 的"团队名称"里输入 `<script>alert('hack')</script>`，
不做转义的话，这段代码会被浏览器执行（这叫 **XSS 攻击**——跨站脚本攻击）。
`esc()` 函数把 `<` 变成 `&lt;`，`>` 变成 `&gt;`，让浏览器把它当作普通文字显示，而不是执行。

---

## 🔧 模块 5：主渲染函数 renderDashboard()

这是**最核心的函数**，负责渲染页面上所有动态内容。

```javascript
// ========== 模块 5：主渲染函数 ==========
let compChart = null;  // 保存 Chart.js 实例的引用，用于后续销毁重建

function renderDashboard() {
    // 更新团队名称
    document.getElementById('team-name-label').textContent = teamConfig.teamName || 'My Team';

    // 加载所有成员数据
    const data = loadAllMemberData();
    // 把团队成员列表和对应的数据合并成一个数组
    const members = teamMembers.map(m => ({ ...m, d: data[m.id] }));
    // 结果：每个元素 = { id:"alice", name:"Alice Wang", role:"...", d:{ plans:[...], overallPct:68 } }

    // ===== 渲染统计卡片 =====
    const avgPct = Math.round(
        members.reduce((sum, m) => sum + (m.d.overallPct || 0), 0) / members.length
    );
    // reduce 求和 → 除以人数 → Math.round 四舍五入
    // || 0 是防御性编程：如果 overallPct 是 undefined 或 null，当作 0 处理

    const completedCount = members.filter(m => m.d.overallPct >= 100).length;
    const atRisk = members.filter(m => m.d.overallPct < 20 && m.d.overallPct < 100).length;
    const totalPlans = members.reduce((s, m) => s + (m.d.plans?.length || 0), 0);
    // m.d.plans?.length → 可选链运算符 (?.)
    // 如果 m.d.plans 是 undefined 或 null，直接返回 undefined（不报错）
    // || 0 → 如果是 undefined 就当 0

    document.getElementById('stats-cards').innerHTML = `
        <div class="glass rounded-2xl shadow-lg p-6 card-hover">
            <p class="text-sm text-gray-500 mb-1">👥 Team Members</p>
            <p class="text-3xl font-bold text-gray-800">${members.length}</p>
            <p class="text-xs text-gray-400 mt-1">${totalPlans} active plans</p>
        </div>
        <div class="glass rounded-2xl shadow-lg p-6 card-hover">
            <p class="text-sm text-gray-500 mb-1">📊 Average Progress</p>
            <p class="text-3xl font-bold text-blue-600">${avgPct}%</p>
        </div>
        <div class="glass rounded-2xl shadow-lg p-6 card-hover">
            <p class="text-sm text-gray-500 mb-1">✅ Completed Plans</p>
            <p class="text-3xl font-bold text-emerald-600">${completedCount}</p>
        </div>
        <div class="glass rounded-2xl shadow-lg p-6 card-hover">
            <p class="text-sm text-gray-500 mb-1">⚠️ Needs Attention</p>
            <p class="text-3xl font-bold text-orange-500">${atRisk}</p>
        </div>`;

    // ===== 渲染进度条（按百分比降序排列） =====
    const sorted = [...members].sort((a, b) => (b.d.overallPct || 0) - (a.d.overallPct || 0));
    // [...members] → 创建数组副本（不修改原数组）
    // sort((a,b) => b-a) → 降序排列（b 的值 - a 的值，如果结果 > 0 则 b 排前面）

    document.getElementById('progress-bars').innerHTML = sorted.map(m => {
        const pct = m.d.overallPct || 0;
        const st = statusLabel(pct);
        const barColor = pct >= 100 ? 'bg-emerald-500' : pct < 20 ? 'bg-orange-400' : 'bg-blue-500';
        // 三元运算符链：条件1 ? 值1 : 条件2 ? 值2 : 默认值
        const planNames = (m.d.plans || []).map(p => p.title).join(', ');
        const syncIcon = m.d.source === 'live'
            ? `<span class="sync-dot live" title="Live data, updated ${timeAgo(m.d.lastUpdated)}"></span>`
            : `<span class="sync-dot none" title="Demo data"></span>`;

        return `<div class="mb-4 cursor-pointer" onclick="window.location.href='my-plan.html?id=${m.id}'">
            <div class="flex items-center justify-between mb-1">
                <div class="flex items-center gap-2">
                    ${syncIcon}
                    <span class="text-sm font-semibold text-gray-700">${m.name}</span>
                    <span class="text-xs text-gray-400">${m.role}</span>
                </div>
                <div class="flex items-center gap-2">
                    <span class="text-xs ${st.color}">${st.text}</span>
                    <span class="text-sm font-bold text-gray-700">${pct}%</span>
                </div>
            </div>
            <div class="w-full bg-gray-200 rounded-full h-3 overflow-hidden">
                <div class="progress-bar-inner ${barColor} h-3 rounded-full" style="width:${pct}%"></div>
            </div>
            <p class="text-xs text-gray-400 mt-0.5 ml-1">${planNames || 'No plan yet'}</p>
        </div>`;
    }).join('');
    // .join('') → 把数组里的所有 HTML 字符串拼成一个

    // ===== 渲染 Chart.js 柱状图 =====
    if (compChart) compChart.destroy();
    // 为什么要先 destroy？因为 Chart.js 不允许在同一个 canvas 上创建两个图表
    // renderDashboard() 可能被多次调用（30秒自动刷新），所以每次先销毁旧图表

    const ctx = document.getElementById('comparisonChart').getContext('2d');
    // getContext('2d') → 获取 canvas 的 2D 绘图上下文
    // Chart.js 需要这个"画笔"来画图

    const barColors = sorted.map(m => {
        const p = m.d.overallPct || 0;
        return p >= 100 ? '#10b981' : p >= 50 ? '#3b82f6' : p >= 20 ? '#60a5fa' : '#f59e0b';
        // 颜色用十六进制值：绿 / 蓝 / 浅蓝 / 黄
    });

    compChart = new Chart(ctx, {
        type: 'bar',              // 图表类型：柱状图
        data: {
            labels: sorted.map(m => m.name.split(' ')[0]),
            // 标签 = 每个人的 first name（"Alice Wang" → "Alice"）
            datasets: [{
                data: sorted.map(m => m.d.overallPct || 0),
                // 每个柱子的值 = 完成百分比
                backgroundColor: barColors,
                borderRadius: 6,    // 柱子顶部圆角
                barThickness: 24    // 柱子粗细（像素）
            }]
        },
        options: {
            responsive: true,       // 自适应容器大小
            indexAxis: 'y',         // 'y' = 横向柱状图（柱子从左到右）
                                    // 默认是 'x' = 纵向柱状图（柱子从下到上）
            scales: {
                x: {
                    beginAtZero: true,  // X 轴从 0 开始
                    max: 100,            // X 轴最大值 100（因为是百分比）
                    ticks: { callback: v => v + '%' },  // 刻度显示为 "0%", "50%", "100%"
                    grid: { color: '#f1f5f9' }          // 网格线颜色（浅灰）
                },
                y: { grid: { display: false } }  // Y 轴不显示网格线（更清爽）
            },
            plugins: {
                legend: { display: false },  // 不显示图例（只有一组数据，不需要图例）
                tooltip: {
                    callbacks: {
                        label: ctx => ctx.parsed.x + '%'  // 鼠标悬停提示显示百分比
                    }
                }
            }
        }
    });

    // ===== 调用热力图和成员卡片渲染 =====
    renderHeatmap(members);
    renderMemberCards(members);
}
```

**进度条的视觉原理：**

```html
<div class="w-full bg-gray-200 rounded-full h-3">           ← 灰色底条（100% 宽度）
    <div class="bg-blue-500 h-3 rounded-full"
         style="width:68%">                                   ← 蓝色填充条（68% 宽度）
    </div>
</div>
```

```
灰色底条：[████████████████████████████████████████] 100%
蓝色填充：[█████████████████████████████           ] 68%
```

就是两层 div 叠在一起：外层 100% 宽灰色背景，内层按百分比宽度蓝色填充。

---

## 🔧 模块 6：热力图和成员卡片

```javascript
// ========== 模块 6：热力图渲染 ==========

function renderHeatmap(members) {
    // Step 1：收集所有计划名（去重）
    const allPlanNames = new Set();
    members.forEach(m => (m.d.plans || []).forEach(p => allPlanNames.add(p.title)));
    const planList = [...allPlanNames];
    // Set = 集合，自动去重。6 个人都有 "Azure Networking Readiness"，Set 里只有一个

    if (planList.length === 0) {
        document.getElementById('heatmap-container').innerHTML =
            '<p class="text-sm text-gray-400 text-center py-8">还没有成员创建计划</p>';
        return;  // 没有数据就提前返回，不画表格
    }

    // Step 2：构建 HTML 表格
    let html = '<table class="w-full"><thead><tr>';
    html += '<th class="text-left text-sm text-gray-500 pb-3 pr-4 min-w-[120px]">Member</th>';
    planList.forEach(name => {
        html += `<th class="text-center text-xs text-gray-500 pb-3 px-2 min-w-[100px]">${name}</th>`;
    });
    html += '<th class="text-center text-xs text-gray-500 pb-3 px-2 font-bold min-w-[80px]">Overall</th>';
    html += '</tr></thead><tbody>';

    // Step 3：每个成员一行
    members.forEach(m => {
        const overall = m.d.overallPct || 0;
        html += `<tr class="cursor-pointer hover:bg-blue-50/50 transition"
                     onclick="window.location.href='my-plan.html?id=${m.id}'">`;
        // 头像 + 名字
        html += `<td class="text-sm font-medium text-gray-700 py-2 pr-4">
            <div class="flex items-center gap-2">
                <div class="w-7 h-7 bg-gradient-to-br from-blue-400 to-indigo-500 rounded-lg
                            flex items-center justify-center text-white text-xs font-bold">
                    ${m.name.charAt(0)}
                </div>
                ${m.name}
            </div></td>`;
        // m.name.charAt(0) → 取名字的第一个字符作为头像字母

        // 每个计划一个格子
        planList.forEach(planName => {
            const memberPlan = (m.d.plans || []).find(p => p.title === planName);
            // .find() → 在数组中找到第一个满足条件的元素
            if (memberPlan) {
                html += `<td class="px-2 py-2">
                    <div class="heatmap-cell ${getHeatColor(memberPlan.pct)} rounded-lg text-center py-3 text-sm font-semibold">
                        ${memberPlan.pct}%
                    </div></td>`;
            } else {
                html += `<td class="px-2 py-2">
                    <div class="rounded-lg text-center py-3 text-sm text-gray-300 bg-gray-50">—</div></td>`;
            }
        });

        // Overall 列
        html += `<td class="px-2 py-2">
            <div class="heatmap-cell ${getHeatColor(overall)} rounded-lg text-center py-3 text-sm font-bold">
                ${overall}%
            </div></td>`;
        html += '</tr>';
    });
    html += '</tbody></table>';
    document.getElementById('heatmap-container').innerHTML = html;
}

// ========== 成员详情卡片 ==========

function renderMemberCards(members) {
    document.getElementById('member-cards').innerHTML = members.map(m => {
        const pct = m.d.overallPct || 0;
        const plans = m.d.plans || [];
        const syncInfo = m.d.source === 'live'
            ? `<span class="sync-dot live"></span> <span class="text-xs text-emerald-600">Updated ${timeAgo(m.d.lastUpdated)}</span>`
            : `<span class="sync-dot none"></span> <span class="text-xs text-gray-400">Demo data</span>`;

        // 渲染每个计划的每个阶段的微型进度条
        let phaseDetails = '';
        plans.forEach(plan => {
            phaseDetails += `<div class="mb-3">
                <p class="text-xs font-semibold text-gray-600 mb-1.5">
                    ${plan.title} <span class="text-gray-400">→ ${plan.target_date || ''}</span>
                </p>
                ${(plan.phases || []).map(ph => `
                    <div class="flex items-center gap-2 mb-1">
                        <span class="text-xs w-4">${ph.icon || '📌'}</span>
                        <span class="text-xs text-gray-600 w-28 truncate">${ph.title}</span>
                        <div class="flex-1 h-1.5 bg-gray-200 rounded-full overflow-hidden">
                            <div class="h-1.5 rounded-full transition-all
                                ${ph.pct >= 100 ? 'bg-emerald-500' : ph.pct >= 50 ? 'bg-blue-500' : 'bg-blue-300'}"
                                style="width:${ph.pct}%"></div>
                        </div>
                        <span class="text-xs font-medium w-8 text-right
                            ${ph.pct >= 100 ? 'text-emerald-600' : 'text-gray-500'}">
                            ${ph.pct}%
                        </span>
                    </div>`).join('')}
            </div>`;
        });

        if (plans.length === 0) {
            phaseDetails = '<p class="text-sm text-gray-400 py-4 text-center">尚未创建 Readiness Plan</p>';
        }

        return `<div class="glass rounded-2xl shadow-lg p-5 card-hover fade-in cursor-pointer"
                     onclick="window.location.href='my-plan.html?id=${m.id}'">
            <div class="flex items-center justify-between mb-3">
                <div class="flex items-center gap-2">
                    <div class="w-9 h-9 bg-gradient-to-br from-blue-400 to-indigo-500 rounded-xl
                                flex items-center justify-center text-white font-bold">${m.name.charAt(0)}</div>
                    <div>
                        <p class="text-sm font-bold text-gray-800">${m.name}</p>
                        <p class="text-xs text-gray-400">${m.role}</p>
                    </div>
                </div>
                <div class="text-right">
                    <p class="text-2xl font-bold ${pct >= 100 ? 'text-emerald-600' : pct >= 50 ? 'text-blue-600' : 'text-orange-500'}">${pct}%</p>
                    <div class="flex items-center gap-1 justify-end mt-0.5">${syncInfo}</div>
                </div>
            </div>
            <div class="w-full bg-gray-200 rounded-full h-2 mb-3 overflow-hidden">
                <div class="h-2 rounded-full transition-all
                    ${pct >= 100 ? 'bg-emerald-500' : pct < 20 ? 'bg-orange-400' : 'bg-blue-500'}"
                    style="width:${pct}%"></div>
            </div>
            ${phaseDetails}
        </div>`;
    }).join('');
}
```

---

## 🔧 模块 7：Team Settings 交互 + 初始化

```javascript
// ========== 模块 7：Team Settings 弹窗交互 ==========
let tsMembers = [];  // 弹窗中编辑时的临时成员列表

function openTeamSettings() {
    teamConfig = loadTeamConfig();
    tsMembers = JSON.parse(JSON.stringify(teamConfig.members));
    // 深拷贝：JSON.stringify → JSON.parse
    // 为什么要深拷贝？因为在弹窗里编辑时不想影响原始数据
    // 直接赋值 tsMembers = teamConfig.members 的话，改 tsMembers 也会改 teamConfig
    document.getElementById('ts-team-name').value = teamConfig.teamName || '';
    renderTsMembers();
    document.getElementById('modal-team').classList.remove('hidden');
    // classList.remove('hidden') → 移除 hidden class → 弹窗显示
}

function closeTeamSettings() {
    document.getElementById('modal-team').classList.add('hidden');
    // classList.add('hidden') → 加上 hidden class → 弹窗消失
}

function renderTsMembers() {
    document.getElementById('ts-members-list').innerHTML = tsMembers.length === 0
        ? '<p class="text-sm text-gray-400 py-3 text-center">还没有成员，请在下方添加</p>'
        : tsMembers.map((m, i) => `
        <div class="flex items-center gap-2 group bg-white rounded-lg px-3 py-2 border border-gray-100">
            <div class="w-7 h-7 bg-gradient-to-br from-blue-400 to-indigo-500 rounded-lg flex items-center justify-center text-white text-xs font-bold flex-shrink-0">${m.name.charAt(0)}</div>
            <input type="text" value="${esc(m.name)}" onchange="tsMembers[${i}].name=this.value"
                class="flex-1 text-sm border-none bg-transparent font-medium px-1 outline-none">
            <input type="text" value="${esc(m.role || '')}" onchange="tsMembers[${i}].role=this.value"
                class="flex-1 text-sm border-none bg-transparent text-gray-500 px-1 outline-none">
            <button onclick="tsMembers.splice(${i},1); renderTsMembers()"
                class="text-gray-300 hover:text-red-500 opacity-0 group-hover:opacity-100 transition px-1 text-lg">✕</button>
        </div>`).join('');
    // group-hover:opacity-100 → 鼠标悬停在父元素（group）上时，删除按钮才显示
    // splice(i, 1) → 从数组第 i 个位置删除 1 个元素
}

function tsAddMember() {
    const nameInput = document.getElementById('ts-new-name');
    const roleInput = document.getElementById('ts-new-role');
    const name = nameInput.value.trim();
    if (!name) { nameInput.focus(); return; }
    // 生成唯一 id：名字小写 + 随机后缀
    const id = name.toLowerCase().replace(/[^a-z0-9]/g, '') + '_' + Date.now().toString(36).slice(-4);
    tsMembers.push({ id, name, role: roleInput.value.trim() || 'Support Engineer' });
    nameInput.value = ''; roleInput.value = '';
    renderTsMembers();
    nameInput.focus();
}

function tsSave() {
    const validMembers = tsMembers.filter(m => m.name.trim());
    validMembers.forEach(m => { m.name = m.name.trim(); m.role = (m.role || '').trim(); });
    const config = {
        teamName: document.getElementById('ts-team-name').value.trim() || 'My Team',
        members: validMembers
    };
    saveTeamConfig(config);
    teamConfig = config;
    teamMembers = config.members;
    closeTeamSettings();
    renderDashboard();  // 保存后重新渲染页面
}

function tsReset() {
    if (!confirm('确定要恢复为 Demo 数据吗？这会覆盖当前团队设置。')) return;
    // confirm() → 浏览器弹出确认对话框，用户点"确定"返回 true，点"取消"返回 false
    localStorage.removeItem(TEAM_CONFIG_KEY);
    teamConfig = loadTeamConfig();
    teamMembers = teamConfig.members;
    closeTeamSettings();
    renderDashboard();
}

// ========== 初始化 ==========
document.getElementById('team-name-label').textContent = teamConfig.teamName || 'My Team';
renderDashboard();  // 首次渲染页面

// 每 30 秒自动重新渲染
// 为什么？因为用户可能在另一个标签页的 my-plan.html 里更新了进度
// index.html 需要定时去 localStorage 读取最新数据
setInterval(() => {
    teamConfig = loadTeamConfig();
    teamMembers = teamConfig.members;
    renderDashboard();
}, 30000); // 30000 毫秒 = 30 秒
</script>
</body>
</html>
```

---

## 🧪 验证：保存并测试

1. 保存 `index.html`
2. 双击用浏览器打开
3. 你应该看到完整的 Dashboard：
   - 4 张统计卡片（6 人、51%、1 完成、2 需关注）
   - 6 个彩色进度条（按百分比排序）
   - 横向柱状图
   - 彩色热力图
   - 6 张成员详情卡片

**如果页面空白？** 打开浏览器 DevTools（按 `F12`）→ Console 标签页 → 查看红色错误信息。
常见错误：
- `Unexpected token` → JSON 格式有误（检查逗号、引号）
- `xxx is not defined` → 变量名拼写错误
- `Cannot read property of null` → `getElementById` 找不到元素（检查 id 是否一致）

---

## ✅ 第4篇完成！

这是最长的一篇，但也是最核心的。你现在理解了：
- ✅ JavaScript 变量、数组操作（map/filter/reduce）
- ✅ 模板字符串和 DOM 操作
- ✅ localStorage 读写和 try-catch 错误处理
- ✅ Chart.js 图表的创建方式
- ✅ 动态 HTML 表格（热力图）的生成逻辑
- ✅ 弹窗的显示/隐藏机制
- ✅ setInterval 定时刷新

**下一篇**：localStorage 数据持久化细节 + GitHub Pages 部署上线。

---

*📖 系列索引：[工具准备](../tutorial-readiness-tracker-1-setup/) → [数据设计](../tutorial-readiness-tracker-2-data-design/) → [HTML 骨架](../tutorial-readiness-tracker-3-html-layout/) → JS 动态渲染 → [部署上线](../tutorial-readiness-tracker-5-deploy/)*
