Claude Desktop 实施计划
                                                                                                                          
  For Claude: REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

  Goal: 构建一个 Electron + React 桌面应用，将 Claude Code CLI 包装成类 claude.ai 网页版风格的 GUI，并打包为 Windows NSIS   安装包。
                                                                                                                          
  Architecture: 主进程通过 child_process 启动 claude CLI 子进程，用 stream-json 格式获取流式输出，通过 IPC 推送到 React
  渲染进程实时显示。三个 Agent 并行开发：主进程通信层、UI 组件层、状态集成层。

  Tech Stack: Electron 33 + electron-vite + React 19 + TypeScript + shadcn/ui + Tailwind CSS 4 + Zustand + shiki +        
  electron-builder

  ---
  阶段一：项目脚手架（顺序执行，Agent 1 负责）

  Task 1: 初始化 electron-vite 项目

  Files:
  - Create: claude-desktop/ 整个项目目录

  Step 1: 在桌面端目录下初始化项目

  cd "C:/Users/29007/Desktop/桌面端"
  npm create @quick-start/electron@latest claude-desktop -- --template react-ts
  cd claude-desktop
  npm install

  Step 2: 验证项目能启动

  npm run dev
  Expected: Electron 窗口打开，显示默认 React 页面

  Step 3: 提交初始脚手架

  git init
  git add .
  git commit -m "chore: init electron-vite react-ts project"

  ---
  Task 2: 安装所有依赖

  Files:
  - Modify: claude-desktop/package.json

  Step 1: 安装运行时依赖

  npm install zustand react-markdown remark-gfm shiki uuid
  npm install @radix-ui/react-dialog @radix-ui/react-scroll-area @radix-ui/react-tooltip

  Step 2: 安装开发依赖

  npm install -D tailwindcss @tailwindcss/vite electron-builder
  npm install -D @types/uuid

  Step 3: 初始化 Tailwind

  npx tailwindcss init

  Step 4: 提交

  git add .
  git commit -m "chore: install all dependencies"

  ---
  Task 3: 配置 Tailwind + shadcn/ui 基础样式

  Files:
  - Modify: src/renderer/src/assets/main.css
  - Create: src/renderer/src/lib/utils.ts
  - Modify: electron.vite.config.ts

  Step 1: 配置 Tailwind CSS v4（在 vite 插件方式）

  在 electron.vite.config.ts 的 renderer 配置中加入：
  import tailwindcss from '@tailwindcss/vite'
  // renderer plugins 中加入：
  plugins: [react(), tailwindcss()]

  Step 2: 更新 main.css

  @import "tailwindcss";

  :root {
    --sidebar-bg: #1a1a1a;
    --sidebar-selected: #2d2d2d;
    --sidebar-text: #e5e5e5;
    --chat-bg: #ffffff;
    --user-bubble: #f4f4f4;
    --code-bg: #f6f8fa;
    --terminal-bg: #1e1e1e;
    --accent: #d97706;
  }

  Step 3: 创建 utils.ts

  import { type ClassValue, clsx } from 'clsx'
  import { twMerge } from 'tailwind-merge'

  export function cn(...inputs: ClassValue[]) {
    return twMerge(clsx(inputs))
  }

  npm install clsx tailwind-merge

  Step 4: 提交

  git add .
  git commit -m "chore: configure tailwind and shadcn utils"

  ---
  阶段二：类型定义与 IPC 协议（Agent 1，其他 Agent 依赖此阶段完成）

  Task 4: 定义所有共享类型

  Files:
  - Create: src/types/claude.ts
  - Create: src/types/session.ts

  Step 1: 创建 src/types/claude.ts

  export type StreamChunk =
    | { type: 'text'; content: string }
    | { type: 'tool_use'; name: string; input: unknown }
    | { type: 'tool_result'; content: string; isError: boolean }
    | { type: 'bash'; command: string; output: string }
    | { type: 'file_read'; path: string; content: string }
    | { type: 'file_write'; path: string }
    | { type: 'file_edit'; path: string; diff: string }
    | { type: 'progress'; message: string; percent?: number }
    | { type: 'confirm'; message: string; id: string }
    | { type: 'todo'; items: TodoItem[] }
    | { type: 'system'; subtype: 'init'; session_id: string }
    | { type: 'error'; message: string }

  export interface TodoItem {
    id: string
    content: string
    done: boolean
  }

  Step 2: 创建 src/types/session.ts

  import { StreamChunk } from './claude'

  export interface Session {
    id: string
    name: string
    resumeId: string | null
    createdAt: number
    messages: Message[]
  }

  export interface Message {
    id: string
    role: 'user' | 'assistant'
    blocks: StreamChunk[]
    createdAt: number
    status: 'streaming' | 'done' | 'error' | 'interrupted'
  }

  Step 3: 提交

  git add .
  git commit -m "feat: define shared types for StreamChunk and Session"

  ---
  阶段三：主进程开发（Agent 1）

  Task 5: 实现 Claude CLI 路径检测

  Files:
  - Create: src/main/utils/detectClaude.ts

  Step 1: 实现检测逻辑

  import { execSync } from 'child_process'
  import { existsSync } from 'fs'
  import { join } from 'path'

  export function detectClaudePath(): string | null {
    // 1. 尝试 where 命令
    try {
      const result = execSync('where claude', { encoding: 'utf8' })
      const path = result.trim().split('\n')[0].trim()
      if (path && existsSync(path)) return path
    } catch {}

    // 2. 检查常见路径
    const candidates = [
      join(process.env.APPDATA || '', 'npm', 'claude.cmd'),
      join(process.env.LOCALAPPDATA || '', 'Programs', 'claude', 'claude.exe'),
    ]
    for (const p of candidates) {
      if (existsSync(p)) return p
    }

    return null
  }

  Step 2: 提交

  git add .
  git commit -m "feat: implement claude CLI path detection"

  ---
  Task 6: 实现 Claude 进程管理器

  Files:
  - Create: src/main/claude.ts

  Step 1: 实现 ClaudeProcessManager

  import { spawn, ChildProcess } from 'child_process'
  import { BrowserWindow } from 'electron'
  import { StreamChunk } from '../types/claude'

  interface ClaudeSession {
    sessionId: string
    resumeId: string | null
    process: ChildProcess | null
    status: 'idle' | 'running' | 'error'
  }

  export class ClaudeProcessManager {
    private sessions = new Map<string, ClaudeSession>()
    private claudePath: string
    private win: BrowserWindow

    constructor(claudePath: string, win: BrowserWindow) {
      this.claudePath = claudePath
      this.win = win
    }

    createSession(sessionId: string): void {
      this.sessions.set(sessionId, {
        sessionId,
        resumeId: null,
        process: null,
        status: 'idle'
      })
    }

    deleteSession(sessionId: string): void {
      const session = this.sessions.get(sessionId)
      if (session?.process) session.process.kill()
      this.sessions.delete(sessionId)
    }

    send(sessionId: string, message: string): void {
      const session = this.sessions.get(sessionId)
      if (!session) return

      const args = ['--output-format', 'stream-json', '--print', message]
      if (session.resumeId) {
        args.push('--resume', session.resumeId)
      }

      const proc = spawn(this.claudePath, args, { shell: true })
      session.process = proc
      session.status = 'running'

      let buffer = ''
      proc.stdout.on('data', (data: Buffer) => {
        buffer += data.toString()
        const lines = buffer.split('\n')
        buffer = lines.pop() || ''
        for (const line of lines) {
          if (!line.trim()) continue
          try {
            const chunk: StreamChunk = JSON.parse(line)
            if (chunk.type === 'system' && chunk.subtype === 'init') {
              session.resumeId = chunk.session_id
            }
            this.win.webContents.send('claude:chunk', { sessionId, chunk })
          } catch {}
        }
      })

      proc.stderr.on('data', (data: Buffer) => {
        const error = data.toString()
        this.win.webContents.send('claude:error', { sessionId, error })
      })

      proc.on('close', () => {
        session.status = 'idle'
        session.process = null
        this.win.webContents.send('claude:done', {
          sessionId,
          resumeId: session.resumeId
        })
      })
    }

    stop(sessionId: string): void {
      const session = this.sessions.get(sessionId)
      if (session?.process) {
        session.process.kill()
        session.status = 'idle'
      }
    }
  }

  Step 2: 提交

  git add .
  git commit -m "feat: implement ClaudeProcessManager with stream-json parsing"

  ---
  Task 7: 实现主进程入口和 IPC 处理器

  Files:
  - Modify: src/main/index.ts
  - Create: src/main/ipc.ts

  Step 1: 更新 src/main/index.ts

  import { app, BrowserWindow, dialog } from 'electron'
  import { join } from 'path'
  import { detectClaudePath } from './utils/detectClaude'
  import { registerIpcHandlers } from './ipc'
  import { ClaudeProcessManager } from './claude'

  let manager: ClaudeProcessManager

  function createWindow(): BrowserWindow {
    const win = new BrowserWindow({
      width: 1200,
      height: 800,
      frame: false,
      webPreferences: {
        preload: join(__dirname, '../preload/index.js'),
        contextIsolation: true,
        nodeIntegration: false
      }
    })

    if (process.env.NODE_ENV === 'development') {
      win.loadURL(process.env.ELECTRON_RENDERER_URL!)
    } else {
      win.loadFile(join(__dirname, '../renderer/index.html'))
    }

    return win
  }

  app.whenReady().then(() => {
    const claudePath = detectClaudePath()
    if (!claudePath) {
      dialog.showErrorBox(
        'Claude CLI 未找到',
        '请先安装 Claude CLI：https://claude.ai/download\n安装后重启应用。'
      )
      app.quit()
      return
    }

    const win = createWindow()
    manager = new ClaudeProcessManager(claudePath, win)
    registerIpcHandlers(manager)
  })

  app.on('window-all-closed', () => app.quit())

  Step 2: 创建 src/main/ipc.ts

  import { ipcMain } from 'electron'
  import { v4 as uuidv4 } from 'uuid'
  import { ClaudeProcessManager } from './claude'

  export function registerIpcHandlers(manager: ClaudeProcessManager): void {
    ipcMain.handle('session:create', () => {
      const id = uuidv4()
      manager.createSession(id)
      return { id, name: '新对话', resumeId: null, createdAt: Date.now(), messages: [] }
    })

    ipcMain.handle('session:delete', (_, { sessionId }) => {
      manager.deleteSession(sessionId)
    })

    ipcMain.handle('claude:send', (_, { sessionId, message }) => {
      manager.send(sessionId, message)
    })

    ipcMain.handle('claude:stop', (_, { sessionId }) => {
      manager.stop(sessionId)
    })
  }

  Step 3: 提交

  git add .
  git commit -m "feat: implement main process entry and IPC handlers"

  ---
  Task 8: 实现 Preload 脚本

  Files:
  - Modify: src/preload/index.ts

  Step 1: 实现 contextBridge

  import { contextBridge, ipcRenderer } from 'electron'

  contextBridge.exposeInMainWorld('claudeAPI', {
    // invoke（有返回值）
    createSession: () => ipcRenderer.invoke('session:create'),
    deleteSession: (sessionId: string) => ipcRenderer.invoke('session:delete', { sessionId }),
    sendMessage: (sessionId: string, message: string) =>
      ipcRenderer.invoke('claude:send', { sessionId, message }),
    stopGeneration: (sessionId: string) => ipcRenderer.invoke('claude:stop', { sessionId }),

    // on（事件监听）
    onChunk: (cb: (data: { sessionId: string; chunk: unknown }) => void) => {
      ipcRenderer.on('claude:chunk', (_, data) => cb(data))
    },
    onDone: (cb: (data: { sessionId: string; resumeId: string }) => void) => {
      ipcRenderer.on('claude:done', (_, data) => cb(data))
    },
    onError: (cb: (data: { sessionId: string; error: string }) => void) => {
      ipcRenderer.on('claude:error', (_, data) => cb(data))
    },

    // 移除监听器
    removeAllListeners: (channel: string) => ipcRenderer.removeAllListeners(channel)
  })

  Step 2: 添加类型声明 src/renderer/src/env.d.ts

  interface Window {
    claudeAPI: {
      createSession: () => Promise<import('../../types/session').Session>
      deleteSession: (sessionId: string) => Promise<void>
      sendMessage: (sessionId: string, message: string) => Promise<void>
      stopGeneration: (sessionId: string) => Promise<void>
      onChunk: (cb: (data: { sessionId: string; chunk: unknown }) => void) => void
      onDone: (cb: (data: { sessionId: string; resumeId: string }) => void) => void
      onError: (cb: (data: { sessionId: string; error: string }) => void) => void
      removeAllListeners: (channel: string) => void
    }
  }

  Step 3: 提交

  git add .
  git commit -m "feat: implement preload contextBridge API"

  ---
  阶段四：状态管理（Agent 3，依赖 Task 4）

  Task 9: 实现 Zustand Store 和会话持久化

  Files:
  - Create: src/renderer/src/store/index.ts

  Step 1: 实现 store

  import { create } from 'zustand'
  import { Session, Message } from '../../../types/session'
  import { StreamChunk } from '../../../types/claude'
  import { v4 as uuidv4 } from 'uuid'

  interface AppStore {
    sessions: Session[]
    activeSessionId: string | null
    setActiveSession: (id: string) => void
    addSession: (session: Session) => void
    removeSession: (id: string) => void
    updateSessionResumeId: (sessionId: string, resumeId: string) => void
    addUserMessage: (sessionId: string, content: string) => string
    addAssistantMessage: (sessionId: string) => string
    appendChunk: (sessionId: string, messageId: string, chunk: StreamChunk) => void
    setMessageStatus: (sessionId: string, messageId: string, status: Message['status']) => void
    loadFromStorage: () => void
    saveToStorage: () => void
  }

  export const useStore = create<AppStore>((set, get) => ({
    sessions: [],
    activeSessionId: null,

    setActiveSession: (id) => set({ activeSessionId: id }),

    addSession: (session) => {
      set((s) => ({ sessions: [...s.sessions, session] }))
      get().saveToStorage()
    },

    removeSession: (id) => {
      set((s) => ({
        sessions: s.sessions.filter((s) => s.id !== id),
        activeSessionId: s.activeSessionId === id ? null : s.activeSessionId
      }))
      get().saveToStorage()
    },

    updateSessionResumeId: (sessionId, resumeId) => {
      set((s) => ({
        sessions: s.sessions.map((sess) =>
          sess.id === sessionId ? { ...sess, resumeId } : sess
        )
      }))
      get().saveToStorage()
    },

    addUserMessage: (sessionId, content) => {
      const id = uuidv4()
      const message: Message = {
        id,
        role: 'user',
        blocks: [{ type: 'text', content }],
        createdAt: Date.now(),
        status: 'done'
      }
      set((s) => ({
        sessions: s.sessions.map((sess) =>
          sess.id === sessionId
            ? {
                ...sess,
                messages: [...sess.messages, message],
                name: sess.messages.length === 0 ? content.slice(0, 20) : sess.name
              }
            : sess
        )
      }))
      get().saveToStorage()
      return id
    },

    addAssistantMessage: (sessionId) => {
      const id = uuidv4()
      const message: Message = {
        id,
        role: 'assistant',
        blocks: [],
        createdAt: Date.now(),
        status: 'streaming'
      }
      set((s) => ({
        sessions: s.sessions.map((sess) =>
          sess.id === sessionId
            ? { ...sess, messages: [...sess.messages, message] }
            : sess
        )
      }))
      return id
    },

    appendChunk: (sessionId, messageId, chunk) => {
      set((s) => ({
        sessions: s.sessions.map((sess) =>
          sess.id === sessionId
            ? {
                ...sess,
                messages: sess.messages.map((msg) =>
                  msg.id === messageId
                    ? { ...msg, blocks: [...msg.blocks, chunk] }
                    : msg
                )
              }
            : sess
        )
      }))
    },

    setMessageStatus: (sessionId, messageId, status) => {
      set((s) => ({
        sessions: s.sessions.map((sess) =>
          sess.id === sessionId
            ? {
                ...sess,
                messages: sess.messages.map((msg) =>
                  msg.id === messageId ? { ...msg, status } : msg
                )
              }
            : sess
        )
      }))
      get().saveToStorage()
    },

    loadFromStorage: () => {
      try {
        const raw = localStorage.getItem('claude-desktop-sessions')
        if (raw) {
          const sessions: Session[] = JSON.parse(raw)
          set({ sessions, activeSessionId: sessions[0]?.id ?? null })
        }
      } catch {}
    },

    saveToStorage: () => {
      try {
        localStorage.setItem(
          'claude-desktop-sessions',
          JSON.stringify(get().sessions)
        )
      } catch {}
    }
  }))

  Step 2: 提交

  git add .
  git commit -m "feat: implement zustand store with session persistence"

  ---
  Task 10: 实现 useClaude Hook

  Files:
  - Create: src/renderer/src/hooks/useClaude.ts

  Step 1: 实现 hook

  import { useEffect, useRef } from 'react'
  import { useStore } from '../store'
  import { StreamChunk } from '../../../types/claude'

  export function useClaude() {
    const store = useStore()
    const activeMessageId = useRef<string | null>(null)

    useEffect(() => {
      window.claudeAPI.onChunk(({ sessionId, chunk }) => {
        if (!activeMessageId.current) {
          activeMessageId.current = store.addAssistantMessage(sessionId)
        }
        store.appendChunk(sessionId, activeMessageId.current, chunk as StreamChunk)
      })

      window.claudeAPI.onDone(({ sessionId, resumeId }) => {
        if (activeMessageId.current) {
          store.setMessageStatus(sessionId, activeMessageId.current, 'done')
          activeMessageId.current = null
        }
        if (resumeId) store.updateSessionResumeId(sessionId, resumeId)
      })

      window.claudeAPI.onError(({ sessionId, error }) => {
        if (activeMessageId.current) {
          store.appendChunk(sessionId, activeMessageId.current, {
            type: 'error',
            message: error
          })
          store.setMessageStatus(sessionId, activeMessageId.current, 'error')
          activeMessageId.current = null
        }
      })

      return () => {
        window.claudeAPI.removeAllListeners('claude:chunk')
        window.claudeAPI.removeAllListeners('claude:done')
        window.claudeAPI.removeAllListeners('claude:error')
      }
    }, [])

    const sendMessage = async (sessionId: string, message: string) => {
      store.addUserMessage(sessionId, message)
      await window.claudeAPI.sendMessage(sessionId, message)
    }

    const stopGeneration = async (sessionId: string) => {
      if (activeMessageId.current) {
        store.setMessageStatus(sessionId, activeMessageId.current, 'interrupted')
        activeMessageId.current = null
      }
      await window.claudeAPI.stopGeneration(sessionId)
    }

    return { sendMessage, stopGeneration }
  }

  Step 2: 提交

  git add .
  git commit -m "feat: implement useClaude hook for IPC communication"

  ---
  阶段五：UI 组件开发（Agent 2，依赖 Task 4）

  Task 11: 实现消息块组件

  Files:
  - Create: src/renderer/src/components/MessageBlocks/TextMessage.tsx
  - Create: src/renderer/src/components/MessageBlocks/BashOutputBlock.tsx
  - Create: src/renderer/src/components/MessageBlocks/FileOperationBlock.tsx
  - Create: src/renderer/src/components/MessageBlocks/ToolCallBlock.tsx
  - Create: src/renderer/src/components/MessageBlocks/ProgressBlock.tsx
  - Create: src/renderer/src/components/MessageBlocks/ConfirmPrompt.tsx
  - Create: src/renderer/src/components/MessageBlocks/TodoListCard.tsx

  Step 1: TextMessage.tsx

  import ReactMarkdown from 'react-markdown'
  import remarkGfm from 'remark-gfm'
  import { useState } from 'react'

  export function TextMessage({ content }: { content: string }) {
    return (
      <div className="prose prose-sm max-w-none">
        <ReactMarkdown
          remarkPlugins={[remarkGfm]}
          components={{
            code({ className, children }) {
              const isBlock = className?.includes('language-')
              if (!isBlock) return <code className="bg-[#f6f8fa] px-1 rounded text-sm">{children}</code>
              return (
                <div className="relative group">
                  <pre className="bg-[#f6f8fa] p-4 rounded-lg overflow-x-auto text-sm">
                    <code>{children}</code>
                  </pre>
                  <CopyButton text={String(children)} />
                </div>
              )
            }
          }}
        >
          {content}
        </ReactMarkdown>
      </div>
    )
  }

  function CopyButton({ text }: { text: string }) {
    const [copied, setCopied] = useState(false)
    return (
      <button
        className="absolute top-2 right-2 opacity-0 group-hover:opacity-100 text-xs bg-white border rounded px-2 py-1"    
        onClick={() => { navigator.clipboard.writeText(text); setCopied(true); setTimeout(() => setCopied(false), 2000) }}
      >
        {copied ? '已复制' : '复制'}
      </button>
    )
  }

  Step 2: BashOutputBlock.tsx

  import { useState } from 'react'

  export function BashOutputBlock({ command, output }: { command: string; output: string }) {
    const [expanded, setExpanded] = useState(true)
    return (
      <div className="rounded-lg overflow-hidden border border-gray-200 my-2">
        <button
          className="w-full flex items-center gap-2 px-3 py-2 bg-[#1e1e1e] text-gray-300 text-sm font-mono"
          onClick={() => setExpanded(!expanded)}
        >
          <span className="text-green-400">$</span>
          <span className="flex-1 text-left truncate">{command}</span>
          <span>{expanded ? '▲' : '▼'}</span>
        </button>
        {expanded && (
          <pre className="bg-[#1e1e1e] text-gray-200 px-3 py-2 text-xs overflow-x-auto max-h-64 overflow-y-auto">
            {output}
          </pre>
        )}
      </div>
    )
  }

  Step 3: FileOperationBlock.tsx

  import { useState } from 'react'

  type Props =
    | { type: 'file_read'; path: string; content: string }
    | { type: 'file_write'; path: string }
    | { type: 'file_edit'; path: string; diff: string }

  export function FileOperationBlock(props: Props) {
    const [expanded, setExpanded] = useState(false)
    const icon = props.type === 'file_read' ? '📖' : props.type === 'file_write' ? '✏️' : '🔧'
    const label = props.type === 'file_read' ? '读取' : props.type === 'file_write' ? '写入' : '编辑'

    return (
      <div className="border border-gray-200 rounded-lg my-2">
        <button
          className="w-full flex items-center gap-2 px-3 py-2 text-sm text-gray-700 hover:bg-gray-50"
          onClick={() => setExpanded(!expanded)}
        >
          <span>{icon}</span>
          <span className="text-gray-500">{label}</span>
          <span className="font-mono text-xs flex-1 text-left truncate">{props.path}</span>
          {(props.type === 'file_read' || props.type === 'file_edit') && (
            <span className="text-gray-400">{expanded ? '▲' : '▼'}</span>
          )}
        </button>
        {expanded && props.type === 'file_read' && (
          <pre className="px-3 py-2 text-xs bg-[#f6f8fa] overflow-x-auto max-h-48 overflow-y-auto border-t">
            {props.content}
          </pre>
        )}
        {expanded && props.type === 'file_edit' && (
          <pre className="px-3 py-2 text-xs bg-[#f6f8fa] overflow-x-auto max-h-48 overflow-y-auto border-t">
            {props.diff}
          </pre>
        )}
      </div>
    )
  }

  Step 4: ToolCallBlock.tsx

  import { useState } from 'react'

  export function ToolCallBlock({ name, input, result, isError }: {
    name: string; input: unknown; result?: string; isError?: boolean
  }) {
    const [expanded, setExpanded] = useState(false)
    return (
      <div className="border border-gray-200 rounded-lg my-2">
        <button
          className="w-full flex items-center gap-2 px-3 py-2 text-sm hover:bg-gray-50"
          onClick={() => setExpanded(!expanded)}
        >
          <span>🔧</span>
          <span className="font-mono text-xs font-medium">{name}</span>
          {isError && <span className="text-red-500 text-xs">错误</span>}
          <span className="ml-auto text-gray-400">{expanded ? '▲' : '▼'}</span>
        </button>
        {expanded && (
          <div className="border-t px-3 py-2 text-xs font-mono bg-[#f6f8fa]">
            <div className="text-gray-500 mb-1">输入：</div>
            <pre className="overflow-x-auto">{JSON.stringify(input, null, 2)}</pre>
            {result && (
              <>
                <div className="text-gray-500 mt-2 mb-1">结果：</div>
                <pre className={`overflow-x-auto ${isError ? 'text-red-600' : ''}`}>{result}</pre>
              </>
            )}