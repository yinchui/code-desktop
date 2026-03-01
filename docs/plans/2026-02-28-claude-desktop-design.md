# Claude Desktop 详细设计文档

**日期：** 2026-02-28
**项目：** Claude Code 桌面端 GUI
**交付物：** Windows NSIS 安装包（.exe），开箱即用
**版本：** v1.0

---

## 1. 项目目标与范围

### 1.1 目标

将 Claude Code CLI 包装成一个类 claude.ai 网页版风格的 Windows 桌面应用。用户下载安装包后直接使用，无需任何额外配置。

### 1.2 核心需求

- 完全复用系统已安装的 `claude` CLI 及 `cc switch` 管理的账号配置
- 支持多会话（左侧会话列表，可新建/切换/删除）
- 流式输出，实时显示 AI 回复
- 复用 claude CLI 的 `--resume` 功能持久化会话历史
- 完整还原 Claude Code 所有输出类型（工具调用、进度条、确认提示等）
- 最终交付 Windows NSIS 安装包

### 1.3 不在范围内

- macOS / Linux 支持
- 内置账号管理或 API Key 输入
- 云端同步

---

## 2. 技术栈

| 层级 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 桌面框架 | Electron | ^33.x | 主流稳定版 |
| 脚手架 | electron-vite | ^3.x | 开发体验最佳 |
| 前端框架 | React | ^19.x | 配合 TypeScript |
| UI 组件库 | shadcn/ui | latest | 基于 Radix UI |
| 样式 | Tailwind CSS | ^4.x | 原子化 CSS |
| 状态管理 | Zustand | ^5.x | 轻量，适合 Electron |
| Markdown 渲染 | react-markdown + remark-gfm | latest | 支持 GFM |
| 代码高亮 | shiki | latest | 与 VS Code 同款 |
| 打包 | electron-builder | ^25.x | NSIS 安装包 |
| 语言 | TypeScript | ^5.x | 全栈 TS |

---

## 3. 整体架构

```
┌──────────────────────────────────────────────────────┐
│                  Electron 主进程 (Main)               │
│                                                      │
│  ┌─────────────────┐    ┌──────────────────────────┐ │
│  │  WindowManager  │    │    ClaudeProcessManager  │ │
│  │  - 创建/管理窗口 │    │  - 检测 claude CLI 路径   │ │
│  │  - 应用生命周期  │    │  - 每会话一个子进程        │ │
│  │  - 系统托盘      │    │  - 流式 JSON 解析         │ │
│  └────────┬────────┘    └────────────┬─────────────┘ │
│           │                          │               │
└───────────┼──────────────────────────┼───────────────┘
            │         IPC Bridge       │
            │   (contextBridge API)    │
┌───────────▼──────────────────────────▼───────────────┐
│                   Preload Script                     │
│  - 安全暴露 window.claudeAPI 给渲染进程               │
│  - 封装所有 ipcRenderer 调用                          │
└───────────────────────────┬──────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────┐
│                渲染进程 (Renderer)                    │
│                                                      │
│  ┌──────────┐  ┌────────────────────────────────┐   │
│  │ Sidebar  │  │         ChatWindow             │   │
│  │          │  │  ┌──────────────────────────┐  │   │
│  │ 会话列表  │  │  │     MessageList          │  │   │
│  │ 新建会话  │  │  │  - TextMessage           │  │   │
│  │ 删除会话  │  │  │  - ToolCallBlock         │  │   │
│  │ 会话名称  │  │  │  - BashOutputBlock       │  │   │
│  │          │  │  │  - FileOperationBlock    │  │   │
│  │          │  │  │  - ProgressBlock         │  │   │
│  │          │  │  │  - ConfirmPrompt         │  │   │
│  │          │  │  │  - TodoListCard          │  │   │
│  └──────────┘  │  └──────────────────────────┘  │   │
│                │  ┌──────────────────────────┐  │   │
│                │  │       InputBar           │  │   │
│                │  │  - 多行文本输入            │  │   │
│                │  │  - 发送按钮               │  │   │
│                │  │  - 停止生成按钮            │  │   │
│                │  └──────────────────────────┘  │   │
│                └────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
            │ stdin / stdout / stderr
┌───────────▼──────────────────────────────────────────┐
│              Claude CLI 子进程                        │
│  claude --output-format stream-json [--resume <id>]  │
│  - 复用 cc switch 当前激活的账号配置                   │
│  - 每个会话独立进程，独立历史                           │
└──────────────────────────────────────────────────────┘
```

---

## 4. 项目目录结构

```
claude-desktop/
├── src/
│   ├── main/
│   │   ├── index.ts              # 主进程入口，窗口创建，应用生命周期
│   │   ├── claude.ts             # ClaudeProcessManager，子进程管理
│   │   ├── ipc.ts                # 所有 ipcMain.handle 注册
│   │   └── utils/
│   │       └── detectClaude.ts   # 自动检测 claude CLI 路径
│   ├── preload/
│   │   └── index.ts              # contextBridge，暴露 window.claudeAPI
│   └── renderer/
│       ├── index.html
│       ├── main.tsx              # React 入口
│       ├── App.tsx               # 根布局（Sidebar + ChatWindow）
│       ├── components/
│       │   ├── Sidebar/
│       │   │   ├── Sidebar.tsx
│       │   │   ├── SessionItem.tsx
│       │   │   └── NewSessionButton.tsx
│       │   ├── ChatWindow/
│       │   │   ├── ChatWindow.tsx
│       │   │   ├── MessageList.tsx
│       │   │   └── InputBar.tsx
│       │   └── MessageBlocks/
│       │       ├── TextMessage.tsx        # 普通文字 + Markdown
│       │       ├── ToolCallBlock.tsx      # 工具调用容器
│       │       ├── BashOutputBlock.tsx    # bash 命令输出
│       │       ├── FileOperationBlock.tsx # 文件读写操作
│       │       ├── ProgressBlock.tsx      # 进度条
│       │       ├── ConfirmPrompt.tsx      # 确认提示（yes/no）
│       │       └── TodoListCard.tsx       # Todo list 卡片
│       ├── hooks/
│       │   ├── useClaude.ts       # 核心 hook，管理消息发送和流式接收
│       │   └── useSessions.ts     # 会话列表管理
│       ├── store/
│       │   └── index.ts           # Zustand store（会话、消息、UI状态）
│       ├── types/
│       │   └── claude.ts          # Claude stream-json 输出类型定义
│       └── lib/
│           └── utils.ts           # shadcn/ui 工具函数（cn）
├── resources/
│   └── icon.ico                   # 应用图标
├── electron.vite.config.ts
├── electron-builder.yml           # 打包配置
├── package.json
├── tailwind.config.ts
└── tsconfig.json
```

---

## 5. 核心模块详细设计

### 5.1 Claude CLI 路径检测（detectClaude.ts）

启动时按以下顺序检测 `claude` 可执行文件路径：

1. 执行 `where claude`，取第一个结果
2. 检查常见路径：
   - `%APPDATA%\npm\claude.cmd`
   - `%LOCALAPPDATA%\Programs\claude\claude.exe`
3. 检查 PATH 环境变量中的每个目录
4. 若未找到，弹出错误对话框提示用户安装 Claude CLI，并提供安装链接

### 5.2 Claude 进程管理（claude.ts）

```typescript
// 每个会话对应一个 ClaudeSession 实例
interface ClaudeSession {
  sessionId: string        // 本地生成的 UUID
  resumeId: string | null  // claude CLI 的 --resume ID（首次为 null）
  process: ChildProcess | null
  status: 'idle' | 'running' | 'error'
}
```

**进程启动命令：**
```bash
# 新会话
claude --output-format stream-json --print "<user_message>"

# 恢复历史会话
claude --output-format stream-json --resume <resumeId> --print "<user_message>"
```

**流式 JSON 解析：**
- 监听 `process.stdout` 的 `data` 事件
- 按换行符分割，每行是一个完整 JSON 对象
- 解析后通过 `ipcMain` 推送到渲染进程
- 监听 `process.stderr` 捕获错误
- 监听 `process.close` 事件，更新会话状态

**进程生命周期：**
- 每次用户发送消息时启动新进程（`--print` 模式，单次执行）
- 进程结束后保存返回的 `resumeId`，下次发送时用 `--resume` 恢复
- 用户点击"停止生成"时，调用 `process.kill()`

### 5.3 IPC 协议

**渲染进程 → 主进程（invoke，有返回值）：**

| 频道 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `claude:send` | `{ sessionId, message }` | `void` | 发送消息 |
| `claude:stop` | `{ sessionId }` | `void` | 停止当前生成 |
| `session:create` | `void` | `Session` | 新建会话 |
| `session:delete` | `{ sessionId }` | `void` | 删除会话 |
| `session:list` | `void` | `Session[]` | 获取所有会话 |

**主进程 → 渲染进程（on，推送事件）：**

| 频道 | 数据 | 说明 |
|------|------|------|
| `claude:chunk` | `{ sessionId, chunk: StreamChunk }` | 流式输出片段 |
| `claude:done` | `{ sessionId, resumeId }` | 回复完成，携带新的 resumeId |
| `claude:error` | `{ sessionId, error: string }` | 错误信息 |

### 5.4 Stream JSON 数据类型（types/claude.ts）

Claude CLI 的 `--output-format stream-json` 输出以下类型，需全部处理：

```typescript
type StreamChunk =
  | { type: 'text'; content: string }                    // 普通文字
  | { type: 'tool_use'; name: string; input: unknown }   // 工具调用开始
  | { type: 'tool_result'; content: string; isError: boolean } // 工具结果
  | { type: 'bash'; command: string; output: string }    // bash 执行
  | { type: 'file_read'; path: string; content: string } // 文件读取
  | { type: 'file_write'; path: string }                 // 文件写入
  | { type: 'file_edit'; path: string; diff: string }    // 文件编辑
  | { type: 'progress'; message: string; percent?: number } // 进度
  | { type: 'confirm'; message: string; id: string }     // 确认提示
  | { type: 'todo'; items: TodoItem[] }                  // Todo 列表
  | { type: 'system'; subtype: 'init'; session_id: string } // 会话初始化
```

### 5.5 消息块渲染组件

每种 `StreamChunk` 类型对应一个专属渲染组件：

| 组件 | 对应类型 | 视觉样式 |
|------|---------|---------|
| `TextMessage` | `text` | Markdown 渲染，代码块用 shiki 高亮，有复制按钮 |
| `BashOutputBlock` | `bash` | 深色终端风格，显示命令和输出，可折叠 |
| `FileOperationBlock` | `file_read/write/edit` | 文件图标 + 路径，edit 显示 diff |
| `ToolCallBlock` | `tool_use/tool_result` | 折叠卡片，显示工具名和参数 |
| `ProgressBlock` | `progress` | 进度条 + 文字描述 |
| `ConfirmPrompt` | `confirm` | 内联 Yes/No 按钮，用户点击后发送响应 |
| `TodoListCard` | `todo` | 卡片式 Todo 列表，勾选框显示完成状态 |

### 5.6 会话管理（store/index.ts）

```typescript
interface Session {
  id: string           // 本地 UUID
  name: string         // 显示名称（默认取第一条消息前20字）
  resumeId: string | null  // claude CLI 的 session ID
  createdAt: number
  messages: Message[]
}

interface Message {
  id: string
  role: 'user' | 'assistant'
  blocks: StreamChunk[]  // 一条消息由多个块组成
  createdAt: number
  status: 'streaming' | 'done' | 'error'
}
```

**会话持久化：**
- 使用 Electron 的 `app.getPath('userData')` 获取用户数据目录
- 将 sessions 序列化为 JSON 存储到 `userData/sessions.json`
- 应用启动时加载，每次会话变更时写入

---

## 6. UI 设计规范

### 6.1 整体布局

```
┌─────────────────────────────────────────────────────┐
│  标题栏（自定义，无边框窗口，可拖拽）                   │
├──────────────┬──────────────────────────────────────┤
│              │                                      │
│   Sidebar    │           ChatWindow                 │
│   260px      │           flex-1                     │
│              │                                      │
│  深色背景     │  左侧深色 sidebar，右侧浅色聊天区       │
│  #1a1a1a     │  聊天区背景 #ffffff                   │
│              │                                      │
│  会话列表     │  消息列表（可滚动）                    │
│              │                                      │
│              ├──────────────────────────────────────┤
│              │  InputBar（固定底部）                  │
│              │  背景 #f9f9f9，圆角输入框              │
└──────────────┴──────────────────────────────────────┘
```

### 6.2 颜色规范

| 元素 | 颜色值 |
|------|--------|
| Sidebar 背景 | `#1a1a1a` |
| Sidebar 选中项 | `#2d2d2d` |
| Sidebar 文字 | `#e5e5e5` |
| 聊天区背景 | `#ffffff` |
| 用户消息气泡 | `#f4f4f4`，右对齐 |
| AI 消息 | 无气泡，左对齐，直接渲染 |
| 代码块背景 | `#f6f8fa` |
| 终端输出背景 | `#1e1e1e`（深色） |
| 主色调（按钮等） | `#d97706`（amber，与 Claude 品牌色接近） |

### 6.3 交互细节

- **输入框**：Enter 发送，Shift+Enter 换行，发送中禁用输入，显示停止按钮
- **流式输出**：逐字显示，末尾有闪烁光标 `▋`
- **代码块**：右上角有复制按钮，hover 时显示
- **工具调用块**：默认折叠，点击展开详情
- **确认提示**：内联显示 Yes/No 按钮，点击后按钮变为已选状态并禁用
- **会话名称**：自动取第一条用户消息的前 20 个字符，双击可重命名
- **滚动行为**：新消息自动滚动到底部，用户手动向上滚动时停止自动滚动

---

## 7. 打包与分发

### 7.1 electron-builder 配置（electron-builder.yml）

```yaml
appId: com.claudedesktop.app
productName: Claude Desktop
directories:
  output: dist-installer
win:
  target:
    - target: nsis
      arch: [x64]
  icon: resources/icon.ico
nsis:
  oneClick: false           # 显示安装向导
  allowToChangeInstallationDirectory: true
  createDesktopShortcut: true
  createStartMenuShortcut: true
  shortcutName: Claude Desktop
```

### 7.2 构建命令

```bash
# 开发模式
npm run dev

# 构建安装包
npm run build:win
# 输出：dist-installer/Claude Desktop Setup x.x.x.exe
```

---

## 8. 错误处理

| 场景 | 处理方式 |
|------|---------|
| claude CLI 未找到 | 启动时弹出对话框，提示安装，提供官方链接 |
| claude CLI 未登录 | 捕获 stderr 中的认证错误，显示提示让用户在终端运行 `claude` 登录 |
| 进程意外退出 | 在消息列表中显示红色错误块，提供重试按钮 |
| 网络超时 | 同上，显示错误信息 |
| 用户主动停止 | 标记消息为 `interrupted`，显示灰色提示 |

---

## 9. Agent Teams 开发策略

使用 Claude Code Agent Teams 并行开发，三个 Agent 同时工作：

### Agent 1 — 主进程与通信层
**负责文件：** `src/main/`, `src/preload/`
**任务：**
- 实现 `detectClaude.ts`，自动检测 CLI 路径
- 实现 `claude.ts`，完整的进程管理和 stream-json 解析
- 实现 `ipc.ts`，注册所有 IPC 处理器
- 实现 `preload/index.ts`，安全暴露 API

### Agent 2 — UI 组件层
**负责文件：** `src/renderer/components/`
**任务：**
- 实现所有 MessageBlock 组件（7种）
- 实现 Sidebar、ChatWindow、InputBar
- 实现 Markdown 渲染和代码高亮
- 按照 UI 设计规范实现样式

### Agent 3 — 状态与集成层
**负责文件：** `src/renderer/store/`, `src/renderer/hooks/`, `src/renderer/App.tsx`
**任务：**
- 实现 Zustand store 和会话持久化
- 实现 `useClaude.ts` 和 `useSessions.ts` hooks
- 将主进程 IPC 事件接入 store
- 端到端联调，确保数据流通畅

**并行开发前提：** Agent 1 先完成 IPC 协议和类型定义（`src/types/claude.ts`），Agent 2 和 Agent 3 基于类型定义并行开发，最后 Agent 3 负责集成联调。

---

## 10. 成功标准

- [ ] 安装包可在 Windows 11 上一键安装，桌面有快捷方式
- [ ] 安装后直接打开，无需任何配置
- [ ] 能正常发送消息并流式接收 AI 回复
- [ ] 所有 7 种消息块类型正确渲染
- [ ] 多会话正常工作（新建、切换、删除）
- [ ] 关闭重开后历史会话通过 `--resume` 恢复
- [ ] `cc switch` 切换账号后新会话自动同步
- [ ] 确认提示（Yes/No）可以正常交互
- [ ] 停止生成按钮正常工作
- [ ] 应用可以正常卸载
