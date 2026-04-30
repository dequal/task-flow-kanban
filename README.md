

# TaskFlow Kanban 代码结构分解与说明

## 一、项目架构总览

这是一个**纯前端单页面应用（SPA）**，技术栈为 HTML + CSS + JavaScript，数据持久化使用 `localStorage`，无需后端服务器。

---

## 二、HTML 结构分解

### 2.1 页面头部区域
```html
<div class="app-header">
  <h1>📋 拖拽任务管理工具（完整版）</h1>
  <button id="themeToggle">切换暗黑模式</button>
</div>
```
| 元素 | 功能说明 |
|------|----------|
| `app-header` | 应用标题容器 |
| `themeToggle` | 主题切换按钮，点击切换浅色/暗黑模式 |

### 2.2 任务输入区域
```html
<div class="task-input">
  <input id="taskInput" placeholder="输入任务内容...">
  <select id="prioritySelect">  <!-- 优先级选择 -->
  <select id="remindMin">       <!-- 提前提醒时间 -->
  <input type="datetime-local" id="deadlineTime">  <!-- 截止时间 -->
  <button id="addBtn">添加任务</button>
</div>
```
| 元素 | 功能说明 |
|------|----------|
| `taskInput` | 输入任务内容文本 |
| `prioritySelect` | 选择任务优先级（高/中/低） |
| `remindMin` | 设置提前提醒时长（分钟） |
| `deadlineTime` | 设置任务截止时间 |
| `addBtn` | 提交新任务 |

### 2.3 看板区域（三列布局）
```html
<div class="board">
  <div class="column" id="todo">待处理</div>
  <div class="column" id="doing">进行中</div>
  <div class="column" id="done">已完成</div>
</div>
```
| 列ID | 状态含义 | 任务流转 |
|------|----------|----------|
| `todo` | 待处理 | 初始状态 |
| `doing` | 进行中 | 处理中 |
| `done` | 已完成 | 归档状态 |

### 2.4 编辑弹窗
```html
<div class="modal" id="editModal">...</div>
```
用于修改已有任务信息，支持内容、优先级、提醒、截止时间编辑。

---

## 三、CSS 样式系统

### 3.1 主题变量（CSS 自定义属性）
```css
:root {
  --bg-color: #f5f7fa;          /* 背景色 */
  --task-bg: #ffffff;           /* 任务卡片背景 */
  --primary-color: #409eff;     /* 主色调 */
  --high-priority: #f56c6c;     /* 高优先级颜色 */
  --medium-priority: #e6a23c;   /* 中优先级颜色 */
  --low-priority: #67c23a;      /* 低优先级颜色 */
  --drop-hover: #dcdde1;        /* 拖拽悬停背景色 */
}
```
**设计目的**：统一管理配色，便于主题扩展和维护。

### 3.2 暗黑模式支持
```css
.dark-mode {
  --bg-color: #1a1a2e;
  --task-bg: #2a2a42;
  --drop-hover: #3a3a5a;
}
```
**实现逻辑**：切换 `body` 标签的 `dark-mode` 类，主题偏好自动保存到 `localStorage`。

### 3.3 拖拽交互样式
```css
.column.drag-over { background: var(--drop-hover); }  /* 拖拽悬停提示 */
.task.dragging { opacity: 0.5; }                      /* 拖拽中透明度 */
```
**视觉反馈**：拖拽时任务卡片半透明，目标列背景高亮。

---

## 四、JavaScript 核心功能

### 4.1 数据管理

**数据加载**
```javascript
let tasks = JSON.parse(localStorage.getItem("tasks")) || [];
```

**任务数据结构**
```javascript
{
  id: Date.now(),           // 唯一标识（时间戳）
  content: "任务内容",       // 任务描述
  status: "todo",           // 状态：todo/doing/done
  priority: "high",         // 优先级：high/medium/low
  deadline: "2024-01-01T00:00",  // 截止时间
  remindMin: 15,            // 提前提醒分钟数（0=不提醒）
  reminded: false,          // 是否已发送截止提醒
  preReminded: false        // 是否已发送提前预提醒
}
```

### 4.2 主要功能模块

| 函数名 | 功能说明 | 关键操作 |
|--------|----------|----------|
| `renderTasks()` | 渲染任务卡片 | 根据tasks数组生成DOM，处理过期标红 |
| `addTask()` | 添加新任务 | 创建任务对象→保存→重新渲染 |
| `deleteTask(id)` | 删除任务 | 从数组移除→保存→重新渲染 |
| `editTask(id)` | 打开编辑弹窗 | 填充当前任务信息 |
| `saveEditTask()` | 保存编辑 | 更新任务→保存→重新渲染 |
| `sortByPriority(columnId)` | 优先级排序 | 按高→中→低重新排列 |
| `checkRemind()` | 提醒检查 | 每30秒轮询，发送Notification |
| `toggleTheme()` | 主题切换 | 切换dark-mode类，保存偏好 |

### 4.3 拖拽逻辑核心
```javascript
columns.forEach(column => {
  column.addEventListener("dragover", e => e.preventDefault());
  column.addEventListener("dragleave", () => column.classList.remove("drag-over"));
  column.addEventListener("drop", e => {
    e.preventDefault();
    const dragging = document.querySelector(".dragging");
    if(!dragging) return;
    
    const taskList = column.querySelector(".task-list");
    taskList.appendChild(dragging);
    
    const taskId = dragging.dataset.id;
    const newStatus = taskList.id;
    tasks = tasks.map(t => t.id == taskId ? {...t, status: newStatus} : t);
    
    saveTasks();
    renderTasks();
    column.classList.remove("drag-over");
  });
});
```
**关键点**：
1. 移动DOM元素与更新数据模型同步
2. 确保页面展示与数据一致
3. 拖拽完成后自动保存

### 4.4 提醒系统
```javascript
setInterval(checkRemind, 30000);  // 每30秒检查一次

function checkRemind() {
  const now = Date.now();
  const MS_PER_MIN = 60 * 1000;
  
  tasks.forEach(task => {
    if(!task.deadline) return;
    const deadlineTimeStamp = new Date(task.deadline).getTime();
    const preRemindTime = deadlineTimeStamp - task.remindMin * MS_PER_MIN;
    
    // 提前预提醒
    if(task.remindMin > 0 && !task.preReminded && now >= preRemindTime && now < deadlineTimeStamp) {
      new Notification("任务即将到期", {
        body: `🔔 还有${task.remindMin}分钟：${task.content}`
      });
      task.preReminded = true;
    }
    
    // 截止提醒
    if(!task.reminded && now >= deadlineTimeStamp) {
      new Notification("任务已截止", {
        body: `⏰ 任务已到截止时间：${task.content}`
      });
      task.reminded = true;
    }
  });
  
  saveTasks();
  renderTasks();
}
```
**提醒逻辑**：
- 提前预提醒 + 截止提醒双重保障
- 每个提醒仅发送一次，不重复骚扰
- 使用浏览器 Notification API

---

## 五、数据流程图

```
用户操作 → 更新 tasks 数组 → 保存到 localStorage → 重新渲染 DOM
                                    ↓
                            检查提醒 → 发送 Notification
```

---

## 六、潜在问题与改进建议

| 问题类型 | 风险描述 | 改进建议 |
|----------|----------|----------|
| **XSS 安全** | 任务内容通过 `innerHTML` 渲染，未做转义 | 使用 `textContent` 或转义特殊字符 |
| **性能** | 每次操作重新渲染所有任务，数据量大时卡顿 | 使用虚拟列表或局部更新 |
| **可访问性** | 缺少 ARIA 标签，屏幕阅读器支持不足 | 添加 `role`、`aria-label` 等属性 |
| **移动端** | 拖拽在移动端体验不佳 | 增加触摸事件支持或改用点击切换 |
| **数据备份** | 本地存储数据易丢失 | 增加导出/导入功能 |

---

## 七、代码质量检查清单

- ✅ 数据结构清晰，字段定义完整
- ✅ 主题切换逻辑完善，支持持久化
- ✅ 拖拽交互流畅，视觉反馈明确
- ⚠️ 缺少 XSS 防护，需添加输入转义
- ⚠️ 缺少错误处理，如 localStorage 不可用时
- ⚠️ 缺少 ARIA 标签，无障碍支持不足

---

## 八、总结

TaskFlow Kanban 是一个**功能完整、轻量易用**的前端任务管理工具，核心实现了：

1. **任务 CRUD**（增删改查）
2. **状态管理**（待处理/进行中/已完成）
3. **优先级分类**（高/中/低三级）
4. **时间提醒**（提前预提醒 + 截止提醒）
5. **拖拽交互**（直观的状态切换）
6. **主题切换**（浅色/暗黑双主题）
7. **数据持久化**（localStorage）

代码结构清晰，适合作为前端开发参考项目，后续可在**安全性、性能、可访问性**方面进一步优化。