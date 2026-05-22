# 便签式待办事项 — 设计文档

## Overview

一个四象限仪表盘布局的待办事项应用。左上统计面板，左下标签分类，右上添加任务入口，右下任务列表（第二路由出口，按标签过滤）。

## Tech Stack

- React 18 + TypeScript + Vite
- Zustand (状态管理 + LocalStorage 持久化)
- Tailwind CSS (样式)
- date-fns (日期格式化)
- React Router v6 (标签路由)

## Layout

```
┌──────────────────────────────────────────┐
│ Top-Left: Stats Panel   │ Top-Right: Add  │
│ - completed count       │   Task Button   │
│ - incomplete count      │   [+ 新建任务]   │
│ - remaining count       │                 │
│ - completion rate bar   │                 │
├─────────────────────────┼─────────────────┤
│ Bottom-Left: Tags       │ Bottom-Right:   │
│  🟢 工作 (3)            │   Task List     │
│  🔵 生活 (2)            │   (router       │
│  🟠 学习 (1)            │    outlet)      │
│  ➕ 新建标签             │                 │
└──────────────────────────────────────────┘
```

- 大屏: CSS Grid 四象限 (2×2)
- 小屏 (≤768px): 四区域上下堆叠，统计 → 添加 → 标签 → 任务列表

## Routes

| Route | Content |
|-------|---------|
| `/` | 任务列表显示全部任务 (按创建时间/排序) |
| `/tag/:tagId` | 任务列表过滤显示该标签的任务 |

右下角为 `<Outlet />`，左侧标签点击时导航切换。

## Data Models

```ts
interface Task {
  id: string;
  title: string;
  completed: boolean;
  tagId: string | null;
  dueDate: string | null;  // ISO date string
  createdAt: string;        // ISO date string
  order: number;            // sort order for drag
}

interface Tag {
  id: string;
  name: string;
  color: string;            // hex color, 8 presets
}
```

## Component Tree

```
App
├── DashboardLayout (CSS Grid)
│   ├── StatsPanel
│   │   ├── StatCard (completed)
│   │   ├── StatCard (incomplete)
│   │   ├── StatCard (remaining)
│   │   └── CompletionRateBar
│   ├── TagSidebar
│   │   ├── TagList
│   │   │   └── TagItem (click → navigate)
│   │   └── AddTagButton → TagFormModal
│   ├── AddTaskButton → TaskFormModal
│   └── TaskListArea (<Outlet />)
│       └── TaskList
│           └── TaskItem (checkbox, title, tag badge, due date, delete)
├── TaskFormModal (create/edit)
├── TagFormModal (create/edit/delete tags)
└── ConfirmDialog (delete confirmation)
```

## State (Zustand Store)

```ts
interface TaskStore {
  tasks: Task[];
  tags: Tag[];
  // Task actions
  addTask: (task: Omit<Task, 'id' | 'createdAt' | 'order'>) => void;
  updateTask: (id: string, updates: Partial<Task>) => void;
  deleteTask: (id: string) => void;
  toggleComplete: (id: string) => void;
  reorderTasks: (fromIndex: number, toIndex: number, tagId?: string) => void;
  // Tag actions
  addTag: (tag: Omit<Tag, 'id'>) => void;
  updateTag: (id: string, updates: Partial<Tag>) => void;
  deleteTag: (id: string) => void;
  // Derived
  getTasksByTag: (tagId?: string) => Task[];
  getStats: () => { completed: number; incomplete: number; total: number; rate: number };
}
```

使用 `zustand/middleware` 的 `persist` 中间件，自动同步到 LocalStorage，key 为 `sticky-todo-store`。

## Key Interactions

| Interaction | Implementation |
|-------------|---------------|
| 添加任务 | 右上角按钮 → TaskFormModal (title, tag select, due date picker) |
| 编辑任务 | 点击任务行 → TaskFormModal 预填数据 |
| 完成/取消完成 | 点击 checkbox → toggleComplete |
| 删除任务 | 行尾删除按钮 → ConfirmDialog → deleteTask |
| 拖拽排序 | 列表内 drag & drop (用 HTML5 DnD 或轻量库) |
| 截止日期显示 | date-fns: formatDistanceToNow, 过期标红色文字 |
| 标签管理 | 标签区底部按钮 → TagFormModal 列表 (增/删/改) |

## Task List Display

列表式布局，每行从左到右:
- [checkbox] 标题文字 ── 标签色块 ── 截止日期 ── [编辑] [删除]

已完成任务: 标题加删除线 + 灰色; 过期未完成: 日期红色

## Edge Cases

- 删除标签时，该标签下的任务 tagId 置为 null (无分类)
- 空状态: 任务列表为空时显示占位提示 "暂无任务，点击右上角创建第一个"
- 标签为空时标签区显示 "暂无标签，点击下方创建"
- 日期选择不限制范围 (允许过去日期以触发过期提醒)
- 拖拽排序使用原生 HTML5 Drag & Drop API，不引入额外依赖
