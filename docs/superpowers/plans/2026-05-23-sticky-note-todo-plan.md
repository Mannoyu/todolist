# Sticky-Note Todo — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a four-quadrant dashboard todo app with tag-based routing, drag-to-reorder, due dates, and LocalStorage persistence.

**Architecture:** Single-page React app with CSS Grid dashboard layout. Zustand store persisted to LocalStorage. React Router v6 with nested routes — left sidebar tags navigate to `/tag/:tagId`, right panel renders filtered task list via `<Outlet />`. Native HTML5 Drag & Drop for reordering.

**Tech Stack:** React 18, TypeScript, Vite, Zustand, Tailwind CSS, date-fns, React Router v6

---

## File Structure

```
vibe/
├── index.html
├── package.json
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── vite.config.ts
├── tailwind.config.js
├── postcss.config.js
├── src/
│   ├── main.tsx                    # ReactDOM + BrowserRouter
│   ├── App.tsx                     # Route definitions
│   ├── index.css                   # Tailwind directives
│   ├── types.ts                    # Task, Tag interfaces
│   ├── store.ts                    # Zustand store + persist
│   ├── utils/
│   │   └── date.ts                 # date-fns helpers
│   └── components/
│       ├── DashboardLayout.tsx      # 2×2 CSS Grid shell
│       ├── StatsPanel.tsx           # Top-left: stats cards + progress
│       ├── TagSidebar.tsx            # Bottom-left: tag list + add
│       ├── TagItem.tsx              # Single tag row (click → navigate)
│       ├── AddTaskButton.tsx        # Top-right: "+ 新建任务"
│       ├── TaskList.tsx             # Routed outlet content
│       ├── TaskItem.tsx             # Single task row (dnd, checkbox, actions)
│       ├── TaskFormModal.tsx        # Create/edit modal
│       ├── TagFormModal.tsx         # Manage tags modal
│       └── ConfirmDialog.tsx        # Generic confirm dialog
```

Each file has one responsibility. Components that belong together (TagSidebar + TagItem) live side by side.

---

### Task 1: Project Scaffolding

**Files:**
- Create: `vibe/` (entire project via Vite)

- [ ] **Step 1: Scaffold Vite project**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npm create vite@latest . -- --template react-ts
```

- [ ] **Step 2: Install dependencies**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npm install zustand react-router-dom date-fns && npm install -D tailwindcss @tailwindcss/vite
```

- [ ] **Step 3: Configure Tailwind with Vite plugin**

Read `vite.config.ts`, then replace with:

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
})
```

- [ ] **Step 4: Replace CSS with Tailwind directives**

Write `src/index.css`:

```css
@import "tailwindcss";
```

- [ ] **Step 5: Verify scaffolding**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npm run dev
```

Expected: Vite dev server starts without errors. Open browser to confirm React + Tailwind work.

- [ ] **Step 6: Clean default Vite files**

Delete `src/App.css`. Remove all default content from `src/App.tsx` (keep empty component shell). Remove unused assets.

- [ ] **Step 7: Commit**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && git init && git add -A && git commit -m "feat: scaffold Vite + React + TS + Tailwind project"
```

---

### Task 2: Types

**Files:**
- Create: `src/types.ts`

- [ ] **Step 1: Write types**

```ts
export interface Task {
  id: string;
  title: string;
  completed: boolean;
  tagId: string | null;
  dueDate: string | null;
  createdAt: string;
  order: number;
}

export interface Tag {
  id: string;
  name: string;
  color: string;
}

export const TAG_COLORS = [
  '#ef4444', // red
  '#f97316', // orange
  '#eab308', // yellow
  '#22c55e', // green
  '#3b82f6', // blue
  '#8b5cf6', // purple
  '#ec4899', // pink
  '#6b7280', // gray
] as const;
```

- [ ] **Step 2: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

Expected: No type errors.

- [ ] **Step 3: Commit**

```bash
git add src/types.ts && git commit -m "feat: add Task and Tag types"
```

---

### Task 3: Zustand Store

**Files:**
- Create: `src/store.ts`

- [ ] **Step 1: Write store with persist middleware**

```ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { Tag, Task } from './types';

interface TaskStore {
  tasks: Task[];
  tags: Tag[];

  addTask: (title: string, tagId: string | null, dueDate: string | null) => void;
  updateTask: (id: string, updates: Partial<Pick<Task, 'title' | 'tagId' | 'dueDate' | 'completed'>>) => void;
  deleteTask: (id: string) => void;
  toggleComplete: (id: string) => void;
  reorderTasks: (fromIndex: number, toIndex: number) => void;

  addTag: (name: string, color: string) => void;
  updateTag: (id: string, updates: Partial<Pick<Tag, 'name' | 'color'>>) => void;
  deleteTag: (id: string) => void;
}

export const useStore = create<TaskStore>()(
  persist(
    (set) => ({
      tasks: [],
      tags: [],

      addTask: (title, tagId, dueDate) =>
        set((state) => {
          const maxOrder = state.tasks.reduce((max, t) => Math.max(max, t.order), 0);
          const task: Task = {
            id: crypto.randomUUID(),
            title,
            completed: false,
            tagId,
            dueDate,
            createdAt: new Date().toISOString(),
            order: maxOrder + 1,
          };
          return { tasks: [...state.tasks, task] };
        }),

      updateTask: (id, updates) =>
        set((state) => ({
          tasks: state.tasks.map((t) => (t.id === id ? { ...t, ...updates } : t)),
        })),

      deleteTask: (id) =>
        set((state) => ({
          tasks: state.tasks.filter((t) => t.id !== id),
        })),

      toggleComplete: (id) =>
        set((state) => ({
          tasks: state.tasks.map((t) =>
            t.id === id ? { ...t, completed: !t.completed } : t
          ),
        })),

      reorderTasks: (fromIndex, toIndex) =>
        set((state) => {
          const tasks = [...state.tasks].sort((a, b) => a.order - b.order);
          const [moved] = tasks.splice(fromIndex, 1);
          tasks.splice(toIndex, 0, moved);
          return {
            tasks: tasks.map((t, i) => ({ ...t, order: i })),
          };
        }),

      addTag: (name, color) =>
        set((state) => {
          const tag: Tag = { id: crypto.randomUUID(), name, color };
          return { tags: [...state.tags, tag] };
        }),

      updateTag: (id, updates) =>
        set((state) => ({
          tags: state.tags.map((t) => (t.id === id ? { ...t, ...updates } : t)),
        })),

      deleteTag: (id) =>
        set((state) => ({
          tags: state.tags.filter((t) => t.id !== id),
          tasks: state.tasks.map((t) => (t.tagId === id ? { ...t, tagId: null } : t)),
        })),
    }),
    { name: 'sticky-todo-store' }
  )
);
```

- [ ] **Step 2: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

Expected: No type errors.

- [ ] **Step 3: Commit**

```bash
git add src/store.ts && git commit -m "feat: add Zustand store with LocalStorage persistence"
```

---

### Task 4: Date Utilities

**Files:**
- Create: `src/utils/date.ts`

- [ ] **Step 1: Write date helpers**

```ts
import { format, formatDistanceToNow, isPast, isToday, parseISO } from 'date-fns';
import { zhCN } from 'date-fns/locale';

export function formatDueDate(dateStr: string | null): string | null {
  if (!dateStr) return null;
  return format(parseISO(dateStr), 'MM/dd');
}

export function formatRelative(dateStr: string | null): string | null {
  if (!dateStr) return null;
  const date = parseISO(dateStr);
  const relative = formatDistanceToNow(date, { addSuffix: true, locale: zhCN });
  if (isPast(date) && !isToday(date)) {
    return `已过期 · ${relative}`;
  }
  return relative;
}

export function isOverdue(dateStr: string | null): boolean {
  if (!dateStr) return false;
  const date = parseISO(dateStr);
  return isPast(date) && !isToday(date);
}
```

- [ ] **Step 2: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

Expected: No type errors.

- [ ] **Step 3: Commit**

```bash
git add src/utils/date.ts && git commit -m "feat: add date formatting utilities"
```

---

### Task 5: DashboardLayout

**Files:**
- Create: `src/components/DashboardLayout.tsx`

- [ ] **Step 1: Write layout component**

```tsx
import { Outlet } from 'react-router-dom';
import { StatsPanel } from './StatsPanel';
import { TagSidebar } from './TagSidebar';
import { AddTaskButton } from './AddTaskButton';

export function DashboardLayout() {
  return (
    <div className="min-h-screen grid grid-cols-1 md:grid-cols-[280px_1fr] grid-rows-[auto_auto_auto_1fr] md:grid-rows-[auto_1fr] gap-4 p-4 bg-gray-50">
      <div className="md:col-start-1 md:row-start-1">
        <StatsPanel />
      </div>
      <div className="md:col-start-2 md:row-start-1 flex justify-end items-start">
        <AddTaskButton />
      </div>
      <div className="md:col-start-1 md:row-start-2 overflow-auto">
        <TagSidebar />
      </div>
      <div className="md:col-start-2 md:row-start-2 overflow-auto bg-white rounded-xl shadow-sm p-4">
        <Outlet />
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Create placeholder components so compilation passes**

Create stub files for `StatsPanel.tsx`, `TagSidebar.tsx`, `AddTaskButton.tsx`:

```tsx
// StatsPanel.tsx (stub)
export function StatsPanel() {
  return <div className="bg-white rounded-xl shadow-sm p-4">Stats</div>;
}
```

```tsx
// TagSidebar.tsx (stub)
export function TagSidebar() {
  return <div className="bg-white rounded-xl shadow-sm p-4">Tags</div>;
}
```

```tsx
// AddTaskButton.tsx (stub)
export function AddTaskButton() {
  return <button className="px-4 py-2 bg-blue-500 text-white rounded-lg">+ 新建任务</button>;
}
```

- [ ] **Step 3: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

Expected: No type errors.

- [ ] **Step 4: Set up App.tsx with router and test**

Write `src/App.tsx`:

```tsx
import { Routes, Route } from 'react-router-dom';
import { DashboardLayout } from './components/DashboardLayout';
import { TaskList } from './components/TaskList';

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<DashboardLayout />}>
        <Route index element={<TaskList />} />
        <Route path="tag/:tagId" element={<TaskList />} />
      </Route>
    </Routes>
  );
}
```

Write `src/main.tsx`:

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

Create stub `src/components/TaskList.tsx`:

```tsx
export function TaskList() {
  return <div>Task List</div>;
}
```

- [ ] **Step 5: Run dev server and verify layout renders**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npm run dev
```

Expected: Browser shows four-quadrant layout on desktop, stacked on mobile.

- [ ] **Step 6: Commit**

```bash
git add src/components/ src/App.tsx src/main.tsx && git commit -m "feat: add DashboardLayout with router setup"
```

---

### Task 6: StatsPanel

**Files:**
- Create/Overwrite: `src/components/StatsPanel.tsx`

- [ ] **Step 1: Write StatsPanel**

```tsx
import { useStore } from '../store';

export function StatsPanel() {
  const tasks = useStore((s) => s.tasks);
  const total = tasks.length;
  const completed = tasks.filter((t) => t.completed).length;
  const incomplete = total - completed;
  const rate = total === 0 ? 0 : Math.round((completed / total) * 100);

  return (
    <div className="bg-white rounded-xl shadow-sm p-5">
      <h2 className="text-sm font-semibold text-gray-500 uppercase tracking-wide mb-3">统计</h2>
      <div className="grid grid-cols-2 gap-3 mb-4">
        <StatCard label="已完成" value={completed} color="text-green-600" />
        <StatCard label="未完成" value={incomplete} color="text-red-500" />
        <StatCard label="剩余待完成" value={incomplete} color="text-orange-500" />
        <StatCard label="完成率" value={`${rate}%`} color="text-blue-600" />
      </div>
      <div className="w-full bg-gray-200 rounded-full h-2">
        <div
          className="bg-blue-500 h-2 rounded-full transition-all duration-300"
          style={{ width: `${rate}%` }}
        />
      </div>
    </div>
  );
}

function StatCard({ label, value, color }: { label: string; value: string | number; color: string }) {
  return (
    <div className="text-center">
      <div className={`text-xl font-bold ${color}`}>{value}</div>
      <div className="text-xs text-gray-400 mt-0.5">{label}</div>
    </div>
  );
}
```

- [ ] **Step 2: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

- [ ] **Step 3: Commit**

```bash
git add src/components/StatsPanel.tsx && git commit -m "feat: add StatsPanel with completion stats"
```

---

### Task 7: TagSidebar + TagItem

**Files:**
- Overwrite: `src/components/TagSidebar.tsx`
- Create: `src/components/TagItem.tsx`

- [ ] **Step 1: Write TagItem**

```tsx
import { NavLink } from 'react-router-dom';
import { Tag } from '../types';
import { useStore } from '../store';

export function TagItem({ tag }: { tag: Tag }) {
  const taskCount = useStore((s) => s.tasks.filter((t) => t.tagId === tag.id).length);

  return (
    <NavLink
      to={`/tag/${tag.id}`}
      className={({ isActive }) =>
        `flex items-center gap-2 px-3 py-2 rounded-lg text-sm transition-colors ${
          isActive ? 'bg-gray-100 font-medium' : 'hover:bg-gray-50'
        }`
      }
    >
      <span
        className="w-3 h-3 rounded-full flex-shrink-0"
        style={{ backgroundColor: tag.color }}
      />
      <span className="flex-1 truncate">{tag.name}</span>
      <span className="text-xs text-gray-400">{taskCount}</span>
    </NavLink>
  );
}
```

- [ ] **Step 2: Write TagSidebar**

```tsx
import { useState } from 'react';
import { NavLink } from 'react-router-dom';
import { useStore } from '../store';
import { TagItem } from './TagItem';
import { TagFormModal } from './TagFormModal';

export function TagSidebar() {
  const tags = useStore((s) => s.tags);
  const [showModal, setShowModal] = useState(false);

  return (
    <div className="bg-white rounded-xl shadow-sm p-5">
      <h2 className="text-sm font-semibold text-gray-500 uppercase tracking-wide mb-3">标签</h2>
      <nav className="flex flex-col gap-1">
        <NavLink
          to="/"
          end
          className={({ isActive }) =>
            `flex items-center gap-2 px-3 py-2 rounded-lg text-sm transition-colors ${
              isActive ? 'bg-gray-100 font-medium' : 'hover:bg-gray-50'
            }`
          }
        >
          <span className="w-3 h-3 rounded-full bg-gray-400 flex-shrink-0" />
          <span>全部</span>
        </NavLink>
        {tags.length === 0 ? (
          <p className="text-xs text-gray-400 px-3 py-2">暂无标签，点击下方创建</p>
        ) : (
          tags.map((tag) => <TagItem key={tag.id} tag={tag} />)
        )}
      </nav>
      <button
        onClick={() => setShowModal(true)}
        className="mt-3 w-full text-sm text-blue-500 hover:text-blue-600 py-2 border border-dashed border-gray-300 rounded-lg hover:border-blue-300 transition-colors"
      >
        + 新建标签
      </button>
      {showModal && <TagFormModal onClose={() => setShowModal(false)} />}
    </div>
  );
}
```

Create stub `src/components/TagFormModal.tsx`:

```tsx
export function TagFormModal({ onClose }: { onClose: () => void }) {
  return (
    <div className="fixed inset-0 bg-black/30 flex items-center justify-center z-50" onClick={onClose}>
      <div className="bg-white rounded-xl p-6 shadow-xl" onClick={(e) => e.stopPropagation()}>
        Tag Manager (stub)
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

- [ ] **Step 4: Commit**

```bash
git add src/components/TagSidebar.tsx src/components/TagItem.tsx src/components/TagFormModal.tsx && git commit -m "feat: add TagSidebar and TagItem with routing"
```

---

### Task 8: AddTaskButton

**Files:**
- Overwrite: `src/components/AddTaskButton.tsx`

- [ ] **Step 1: Write AddTaskButton**

```tsx
import { useState } from 'react';
import { TaskFormModal } from './TaskFormModal';

export function AddTaskButton() {
  const [showModal, setShowModal] = useState(false);

  return (
    <>
      <button
        onClick={() => setShowModal(true)}
        className="px-5 py-2.5 bg-blue-500 hover:bg-blue-600 text-white font-medium rounded-lg shadow-sm transition-colors"
      >
        + 新建任务
      </button>
      {showModal && <TaskFormModal onClose={() => setShowModal(false)} />}
    </>
  );
}
```

Create stub `src/components/TaskFormModal.tsx`:

```tsx
export function TaskFormModal({ onClose, task }: { onClose: () => void; task?: undefined }) {
  return (
    <div className="fixed inset-0 bg-black/30 flex items-center justify-center z-50" onClick={onClose}>
      <div className="bg-white rounded-xl p-6 shadow-xl" onClick={(e) => e.stopPropagation()}>
        Task Form (stub)
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

- [ ] **Step 3: Commit**

```bash
git add src/components/AddTaskButton.tsx src/components/TaskFormModal.tsx && git commit -m "feat: add AddTaskButton with modal trigger"
```

---

### Task 9: TaskList + TaskItem

**Files:**
- Overwrite: `src/components/TaskList.tsx`
- Create: `src/components/TaskItem.tsx`

- [ ] **Step 1: Write TaskItem with drag & drop**

```tsx
import { useState } from 'react';
import { useStore } from '../store';
import { Task } from '../types';
import { formatDueDate, isOverdue } from '../utils/date';
import { TaskFormModal } from './TaskFormModal';
import { ConfirmDialog } from './ConfirmDialog';

export function TaskItem({ task, index }: { task: Task; index: number }) {
  const toggleComplete = useStore((s) => s.toggleComplete);
  const deleteTask = useStore((s) => s.deleteTask);
  const tags = useStore((s) => s.tags);
  const reorderTasks = useStore((s) => s.reorderTasks);
  const tag = tags.find((t) => t.id === task.tagId);

  const [showEdit, setShowEdit] = useState(false);
  const [showDelete, setShowDelete] = useState(false);
  const [dragOver, setDragOver] = useState<'above' | 'below' | null>(null);
  const overdue = !task.completed && isOverdue(task.dueDate);
  const dueDateLabel = formatDueDate(task.dueDate);

  function handleDragStart(e: React.DragEvent) {
    e.dataTransfer.setData('text/plain', String(index));
    e.dataTransfer.effectAllowed = 'move';
  }

  function handleDragOver(e: React.DragEvent) {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';
    const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
    const mid = (rect.top + rect.bottom) / 2;
    setDragOver(e.clientY < mid ? 'above' : 'below');
  }

  function handleDragLeave() {
    setDragOver(null);
  }

  function handleDrop(e: React.DragEvent) {
    e.preventDefault();
    setDragOver(null);
    const fromIndex = Number(e.dataTransfer.getData('text/plain'));
    if (fromIndex === index) return;
    reorderTasks(fromIndex, index);
  }

  return (
    <>
      <div
        draggable
        onDragStart={handleDragStart}
        onDragOver={handleDragOver}
        onDragLeave={handleDragLeave}
        onDrop={handleDrop}
        className={`group flex items-center gap-3 px-3 py-2.5 rounded-lg cursor-pointer transition-colors hover:bg-gray-50 ${
          dragOver === 'above' ? 'border-t-2 border-blue-400' : ''
        } ${dragOver === 'below' ? 'border-b-2 border-blue-400' : ''}`}
      >
        <input
          type="checkbox"
          checked={task.completed}
          onChange={() => toggleComplete(task.id)}
          className="w-4 h-4 rounded border-gray-300 text-blue-500 focus:ring-blue-400 flex-shrink-0 cursor-pointer"
        />
        <span
          className={`flex-1 truncate text-sm ${
            task.completed ? 'line-through text-gray-400' : 'text-gray-800'
          }`}
          onClick={() => setShowEdit(true)}
        >
          {task.title}
        </span>
        {tag && (
          <span
            className="text-xs px-2 py-0.5 rounded-full text-white flex-shrink-0 max-w-[80px] truncate"
            style={{ backgroundColor: tag.color }}
          >
            {tag.name}
          </span>
        )}
        {dueDateLabel && (
          <span
            className={`text-xs flex-shrink-0 ${
              overdue ? 'text-red-500 font-medium' : 'text-gray-400'
            }`}
          >
            {dueDateLabel}
          </span>
        )}
        <button
          onClick={() => setShowEdit(true)}
          className="text-gray-300 hover:text-blue-500 opacity-0 group-hover:opacity-100 transition-all flex-shrink-0 text-sm"
        >
          编辑
        </button>
        <button
          onClick={() => setShowDelete(true)}
          className="text-gray-300 hover:text-red-500 opacity-0 group-hover:opacity-100 transition-all flex-shrink-0 text-sm"
        >
          删除
        </button>
      </div>
      {showEdit && <TaskFormModal task={task} onClose={() => setShowEdit(false)} />}
      {showDelete && (
        <ConfirmDialog
          title="删除任务"
          message={`确定要删除「${task.title}」吗？`}
          onConfirm={() => { deleteTask(task.id); setShowDelete(false); }}
          onCancel={() => setShowDelete(false)}
        />
      )}
    </>
  );
}
```

- [ ] **Step 2: Write TaskList**

```tsx
import { useParams } from 'react-router-dom';
import { useStore } from '../store';
import { TaskItem } from './TaskItem';

export function TaskList() {
  const { tagId } = useParams<{ tagId: string }>();
  const tasks = useStore((s) => s.tasks);

  const filtered = tagId
    ? tasks.filter((t) => t.tagId === tagId)
    : tasks;

  const sorted = [...filtered].sort((a, b) => a.order - b.order);

  if (sorted.length === 0) {
    return (
      <div className="flex flex-col items-center justify-center py-16 text-gray-400">
        <p className="text-lg">暂无任务</p>
        <p className="text-sm mt-1">点击右上角「+ 新建任务」创建第一个</p>
      </div>
    );
  }

  return (
    <div className="flex flex-col">
      {sorted.map((task, i) => (
        <TaskItem key={task.id} task={task} index={i} />
      ))}
    </div>
  );
}
```

Create stub `src/components/ConfirmDialog.tsx`:

```tsx
interface ConfirmDialogProps {
  title: string;
  message: string;
  onConfirm: () => void;
  onCancel: () => void;
}

export function ConfirmDialog({ title, message, onConfirm, onCancel }: ConfirmDialogProps) {
  return (
    <div className="fixed inset-0 bg-black/30 flex items-center justify-center z-50" onClick={onCancel}>
      <div className="bg-white rounded-xl p-6 shadow-xl max-w-sm w-full mx-4" onClick={(e) => e.stopPropagation()}>
        <h3 className="font-semibold text-gray-800 mb-2">{title}</h3>
        <p className="text-sm text-gray-500 mb-4">{message}</p>
        <div className="flex justify-end gap-3">
          <button onClick={onCancel} className="px-4 py-2 text-sm text-gray-600 hover:bg-gray-100 rounded-lg">取消</button>
          <button onClick={onConfirm} className="px-4 py-2 text-sm text-white bg-red-500 hover:bg-red-600 rounded-lg">确定</button>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

- [ ] **Step 4: Commit**

```bash
git add src/components/TaskList.tsx src/components/TaskItem.tsx src/components/ConfirmDialog.tsx && git commit -m "feat: add TaskList and TaskItem with drag-and-drop"
```

---

### Task 10: TaskFormModal

**Files:**
- Overwrite: `src/components/TaskFormModal.tsx`

- [ ] **Step 1: Write TaskFormModal**

```tsx
import { useState, useEffect } from 'react';
import { useStore } from '../store';
import { Task } from '../types';

interface TaskFormModalProps {
  onClose: () => void;
  task?: Task;
}

export function TaskFormModal({ onClose, task }: TaskFormModalProps) {
  const addTask = useStore((s) => s.addTask);
  const updateTask = useStore((s) => s.updateTask);
  const tags = useStore((s) => s.tags);

  const [title, setTitle] = useState(task?.title ?? '');
  const [tagId, setTagId] = useState<string | null>(task?.tagId ?? null);
  const [dueDate, setDueDate] = useState(task?.dueDate?.split('T')[0] ?? '');

  useEffect(() => {
    function handleKeyDown(e: KeyboardEvent) {
      if (e.key === 'Escape') onClose();
    }
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [onClose]);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const trimmed = title.trim();
    if (!trimmed) return;

    const isoDate = dueDate ? new Date(dueDate).toISOString() : null;

    if (task) {
      updateTask(task.id, { title: trimmed, tagId, dueDate: isoDate });
    } else {
      addTask(trimmed, tagId, isoDate);
    }
    onClose();
  }

  return (
    <div className="fixed inset-0 bg-black/30 flex items-center justify-center z-50 p-4" onClick={onClose}>
      <div
        className="bg-white rounded-xl shadow-xl w-full max-w-md"
        onClick={(e) => e.stopPropagation()}
      >
        <form onSubmit={handleSubmit} className="p-6">
          <h2 className="text-lg font-semibold text-gray-800 mb-4">
            {task ? '编辑任务' : '新建任务'}
          </h2>

          <div className="mb-4">
            <label className="block text-sm font-medium text-gray-600 mb-1">任务标题</label>
            <input
              type="text"
              value={title}
              onChange={(e) => setTitle(e.target.value)}
              placeholder="输入任务内容..."
              autoFocus
              className="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent"
            />
          </div>

          <div className="mb-4">
            <label className="block text-sm font-medium text-gray-600 mb-1">标签分类</label>
            <select
              value={tagId ?? ''}
              onChange={(e) => setTagId(e.target.value || null)}
              className="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent"
            >
              <option value="">无分类</option>
              {tags.map((tag) => (
                <option key={tag.id} value={tag.id}>
                  {tag.name}
                </option>
              ))}
            </select>
          </div>

          <div className="mb-5">
            <label className="block text-sm font-medium text-gray-600 mb-1">截止日期</label>
            <input
              type="date"
              value={dueDate}
              onChange={(e) => setDueDate(e.target.value)}
              className="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent"
            />
          </div>

          <div className="flex justify-end gap-3">
            <button
              type="button"
              onClick={onClose}
              className="px-4 py-2 text-sm text-gray-600 hover:bg-gray-100 rounded-lg transition-colors"
            >
              取消
            </button>
            <button
              type="submit"
              className="px-4 py-2 text-sm text-white bg-blue-500 hover:bg-blue-600 rounded-lg font-medium transition-colors"
            >
              {task ? '保存' : '创建'}
            </button>
          </div>
        </form>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

- [ ] **Step 3: Commit**

```bash
git add src/components/TaskFormModal.tsx && git commit -m "feat: add TaskFormModal for creating and editing tasks"
```

---

### Task 11: TagFormModal

**Files:**
- Overwrite: `src/components/TagFormModal.tsx`

- [ ] **Step 1: Write TagFormModal**

```tsx
import { useState, useEffect } from 'react';
import { useStore } from '../store';
import { TAG_COLORS, Tag } from '../types';

export function TagFormModal({ onClose }: { onClose: () => void }) {
  const tags = useStore((s) => s.tags);
  const addTag = useStore((s) => s.addTag);
  const updateTag = useStore((s) => s.updateTag);
  const deleteTag = useStore((s) => s.deleteTag);

  const [name, setName] = useState('');
  const [color, setColor] = useState(TAG_COLORS[0]);
  const [editingId, setEditingId] = useState<string | null>(null);

  useEffect(() => {
    function handleKeyDown(e: KeyboardEvent) {
      if (e.key === 'Escape') onClose();
    }
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [onClose]);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const trimmed = name.trim();
    if (!trimmed) return;

    if (editingId) {
      updateTag(editingId, { name: trimmed, color });
      setEditingId(null);
    } else {
      addTag(trimmed, color);
    }
    setName('');
    setColor(TAG_COLORS[0]);
  }

  function startEdit(tag: Tag) {
    setEditingId(tag.id);
    setName(tag.name);
    setColor(tag.color);
  }

  function cancelEdit() {
    setEditingId(null);
    setName('');
    setColor(TAG_COLORS[0]);
  }

  return (
    <div className="fixed inset-0 bg-black/30 flex items-center justify-center z-50 p-4" onClick={onClose}>
      <div
        className="bg-white rounded-xl shadow-xl w-full max-w-md"
        onClick={(e) => e.stopPropagation()}
      >
        <div className="p-6">
          <h2 className="text-lg font-semibold text-gray-800 mb-4">管理标签</h2>

          <form onSubmit={handleSubmit} className="flex gap-2 mb-4">
            <input
              type="text"
              value={name}
              onChange={(e) => setName(e.target.value)}
              placeholder="标签名称"
              className="flex-1 px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-400"
            />
            <div className="flex gap-2">
              {TAG_COLORS.map((c) => (
                <button
                  key={c}
                  type="button"
                  onClick={() => setColor(c)}
                  className={`w-7 h-7 rounded-full border-2 transition-colors ${
                    color === c ? 'border-gray-800 scale-110' : 'border-transparent'
                  }`}
                  style={{ backgroundColor: c }}
                />
              ))}
            </div>
            {editingId ? (
              <>
                <button
                  type="submit"
                  className="px-3 py-2 text-sm text-white bg-blue-500 hover:bg-blue-600 rounded-lg"
                >
                  更新
                </button>
                <button
                  type="button"
                  onClick={cancelEdit}
                  className="px-3 py-2 text-sm text-gray-600 hover:bg-gray-100 rounded-lg"
                >
                  取消
                </button>
              </>
            ) : (
              <button
                type="submit"
                className="px-3 py-2 text-sm text-white bg-blue-500 hover:bg-blue-600 rounded-lg"
              >
                添加
              </button>
            )}
          </form>

          <div className="space-y-1 max-h-64 overflow-auto">
            {tags.map((tag) => (
              <div
                key={tag.id}
                className="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-gray-50 group"
              >
                <span
                  className="w-3 h-3 rounded-full flex-shrink-0"
                  style={{ backgroundColor: tag.color }}
                />
                <span className="flex-1 text-sm truncate">{tag.name}</span>
                <button
                  onClick={() => startEdit(tag)}
                  className="text-xs text-gray-300 hover:text-blue-500 opacity-0 group-hover:opacity-100 transition-all"
                >
                  编辑
                </button>
                <button
                  onClick={() => deleteTag(tag.id)}
                  className="text-xs text-gray-300 hover:text-red-500 opacity-0 group-hover:opacity-100 transition-all"
                >
                  删除
                </button>
              </div>
            ))}
            {tags.length === 0 && (
              <p className="text-sm text-gray-400 text-center py-4">暂无标签</p>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verify compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

- [ ] **Step 3: Commit**

```bash
git add src/components/TagFormModal.tsx && git commit -m "feat: add TagFormModal for managing tags"
```

---

### Task 12: Final Integration & Polish

- [ ] **Step 1: Verify full compilation**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npx tsc --noEmit
```

- [ ] **Step 2: Run dev server and test manually**

```bash
cd "c:/Users/123/Desktop/vibe codingweb/vibe" && npm run dev
```

Manually test:
1. Create a few tags (工作/生活/学习) with different colors
2. Create tasks with and without tags, with and without due dates
3. Toggle completion — verify strikethrough and stat updates
4. Click different tags — verify filtering works via the right panel
5. Edit a task — verify modal pre-fills data
6. Delete a task — verify confirm dialog
7. Drag to reorder — verify order persists after refresh
8. Create a task with a past date — verify red overdue text
9. Delete a tag — verify tasks fall back to null tagId
10. Refresh the page — verify all data persists

- [ ] **Step 3: Fix any issues found**

None expected, but address any that come up.

- [ ] **Step 4: Final commit**

```bash
git add -A && git commit -m "feat: complete sticky-note todo app"
```

---

## Self-Review

### Spec Coverage

| Spec Requirement | Covered By |
|-----------------|------------|
| Four-quadrant dashboard layout | Task 5 (DashboardLayout) |
| Stats panel (completed/incomplete/remaining/rate) | Task 6 (StatsPanel) |
| Tag sidebar with routing | Task 7 (TagSidebar + TagItem) |
| Add task button (top-right) | Task 8 (AddTaskButton) |
| Task list as router outlet | Task 5 + Task 9 (TaskList) |
| List-style task display | Task 9 (TaskItem) |
| Create/edit/delete tasks | Task 10 (TaskFormModal) + Task 9 |
| Drag-to-reorder | Task 9 (HTML5 DnD) |
| Due dates with overdue indicators | Task 4 (date utils) + Task 9 |
| Tag management (CRUD) | Task 11 (TagFormModal) |
| User-defined tags with colors | Task 11 |
| LocalStorage persistence | Task 3 (Zustand persist) |
| Empty states | Task 7 + Task 9 |
| Delete tag → null tagId on tasks | Task 3 (deleteTag) |
| No placeholder found | All tasks have complete code |
| Type consistency | Types defined in Task 2, used consistently throughout |
