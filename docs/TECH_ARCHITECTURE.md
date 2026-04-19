# UniClaw/DeerFlow 技术架构深度解析

**文档版本**: 1.0
**更新日期**: 2026-04-19
**基于版本**: DeerFlow 2.0

---

## 一、系统整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户浏览器 (localhost:2026)                      │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ HTTP/SSE
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Nginx 反向代理 (Port 2026)                      │
│                    路由: /api/* → gateway | /api/langgraph/* → langgraph │
└──────┬─────────────────────────────────────┬───────────────────────┘
       │                                     │
       ▼                                     ▼
┌──────────────────┐              ┌─────────────────────────────────┐
│   LangGraph Server │              │        Gateway API (FastAPI)    │
│    (Port 2024)    │              │         (Port 8001)            │
│                   │              │                                │
│  ┌─────────────┐  │              │  • /api/models (模型配置)        │
│  │ Lead Agent  │  │              │  • /api/mcp (MCP服务器)         │
│  │ + Middleware │  │              │  • /api/skills (技能管理)        │
│  │    Chain     │  │              │  • /api/memory (记忆系统)        │
│  └─────────────┘  │              │  • /api/threads (会话管理)        │
│  ┌─────────────┐  │              │  • /api/uploads (文件上传)        │
│  │   Tools     │  │              │  • /api/artifacts (产物服务)      │
│  │  Sandbox    │  │              └─────────────────────────────────┘
│  │  Subagents  │  │
│  └─────────────┘  │
└──────────────────┘
```

### 核心技术栈

| 层级 | 技术 | 作用 |
|------|------|------|
| **Agent 框架** | LangGraph SDK | 多步骤 Agent 编排、状态管理 |
| **LLM 抽象** | LangChain Core | 模型调用、Tool 绑定 |
| **API 网关** | FastAPI | REST API、Streaming |
| **沙箱执行** | LocalSandbox / AioSandbox | 隔离的代码执行环境 |
| **前端** | Next.js 16 + React 19 | Web UI |

---

## 二、Agent 执行流程详解

### 2.1 请求入口：用户发送消息

```
用户输入 "帮我研究 AI Agent 的最新发展"
         │
         ▼
Frontend (Next.js)
  └→ API: POST /api/runs/stream
         │
         ▼
Gateway API (FastAPI)
  └→ 创建/复用 Thread
  └→ 调用 LangGraph Server
```

### 2.2 LangGraph Agent 核心执行

**`make_lead_agent()`** 是核心工厂函数，创建 Agent 实例：

```python
# 关键流程
make_lead_agent(config)
  │
  ├─ 1. 模型解析 (resolve_model_name)
  │     └─ 从 config.yaml 加载模型配置
  │
  ├─ 2. Middleware Chain 构建
  │     └─ 按顺序串联 9 个中间件
  │
  ├─ 3. Tools 加载
  │     └─ 从配置加载可用工具
  │
  └─ 4. System Prompt 组装
        └─ 注入 Skills、Memory、上下文
```

### 2.3 Middleware 链（执行顺序）

Middleware 是 DeerFlow 的**核心扩展机制**，按顺序执行：

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Request 进入 Lead Agent                         │
├──────────────────────────────────────────────────────────────────────┤
│  1. ThreadDataMiddleware      → 创建线程隔离目录 (workspace/uploads/outputs) │
│  2. UploadsMiddleware        → 注入上传文件到上下文                      │
│  3. SandboxMiddleware        → 获取沙箱环境                             │
│  4. SummarizationMiddleware  → Token 超限时压缩上下文 (可选)             │
│  5. TodoMiddleware          → Plan 模式下任务跟踪 (可选)                │
│  6. TitleMiddleware         → 首轮对话后生成标题                        │
│  7. MemoryMiddleware        → 异步更新长期记忆                         │
│  8. ViewImageMiddleware     → 注入图像数据给视觉模型 (可选)              │
│  9. ClarificationMiddleware  → 拦截澄清请求 (最后一个)                   │
└──────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       Response 返回给用户                               │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.4 Agent Loop（思考-行动循环）

DeerFlow 使用 LangGraph 的 Agent Loop：

```
┌─────────────────────────────────────────────────────────────┐
│                      Agent Loop                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐              │
│   │  User   │───▶│  Model  │───▶│  Tools  │              │
│   │  Input  │    │ (LLM)   │    │ (执行)   │              │
│   └─────────┘    └────┬────┘    └────┬────┘              │
│                       │              │                    │
│                       │    ┌─────────┘                    │
│                       │    │                              │
│                       │    ▼ (Tool Result)                │
│                       │    ┌─────────┐                    │
│                       └────│  Model  │◀───┐              │
│                            │(下一轮)  │    │              │
│                            └────┬────┘    │              │
│                                 │         │              │
│                          ┌──────┴─────────┘              │
│                          │                              │
│                          ▼                              │
│                    ┌──────────┐                         │
│                    │ Finish?  │─── No ──┐                │
│                    └──────────┘         │                │
│                          │ Yes          │                │
│                          ▼              │                │
│                    ┌──────────┐         │                │
│                    │  Return  │◀────────┘                │
│                    │  Result  │                          │
│                    └──────────┘                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、Thread State（线程状态）

每个会话/任务维护一个 `ThreadState`：

```python
class ThreadState(AgentState):
    # 消息历史
    messages: list[BaseMessage]        # 对话消息
    
    # 沙箱状态
    sandbox: SandboxState              # {sandbox_id: str}
    
    # 线程数据
    thread_data: ThreadDataState      # {workspace_path, uploads_path, outputs_path}
    
    # 产物
    artifacts: list[str]              # 生成的文件列表
    
    # 任务列表 (Plan 模式)
    todos: list                       # 当前任务状态
    
    # 已上传文件
    uploaded_files: list[dict]        # 用户上传的文件
    
    # 查看过的图片
    viewed_images: dict[str, ViewedImageData]  # {image_path: {base64, mime_type}}
```

---

## 四、Tools 系统

### 4.1 工具分类

| 类别 | 工具 | 说明 |
|------|------|------|
| **Sandbox** | `bash`, `ls`, `read_file`, `write_file`, `str_replace` | 文件/命令执行 |
| **内置** | `present_files`, `ask_clarification`, `view_image`, `task` | Agent 内置工具 |
| **社区** | `web_search`, `web_fetch`, `image_search` | 第三方服务 |
| **MCP** | 任意 MCP Server | 协议扩展 |

### 4.2 工具执行流程

```
Agent 决定调用 Tool(X)
         │
         ▼
  ┌─────────────┐
  │ Tool Call   │
  │ JSON 结构   │
  └──────┬──────┘
         │
         ▼
  ┌─────────────────────────┐
  │  SandboxMiddleware      │
  │  (路由到对应沙箱)        │
  └──────┬──────────────────┘
         │
         ▼
  ┌─────────────────────────┐
  │  AioSandboxProvider    │  ← Docker 容器
  │  或 LocalSandboxProvider│  ← 本地文件系统
  └──────┬──────────────────┘
         │
         ▼
  ┌─────────────────────────┐
  │  Tool 执行              │
  │  (bash/read/write/...) │
  └──────┬──────────────────┘
         │
         ▼
  ┌─────────────────────────┐
  │  返回结果 JSON          │
  └──────┬──────────────────┘
         │
         ▼
  Agent 继续下一轮
```

### 4.3 虚拟路径映射

DeerFlow 使用虚拟路径实现线程隔离：

```
Agent 看到:     /mnt/user-data/workspace/
               /mnt/user-data/uploads/
               /mnt/user-data/outputs/
               
实际映射到:     /deer-flow/backend/.deer-flow/threads/{thread_id}/user-data/workspace/
               /deer-flow/backend/.deer-flow/threads/{thread_id}/user-data/uploads/
               /deer-flow/backend/.deer-flow/threads/{thread_id}/user-data/outputs/
```

---

## 五、Subagent（子代理）系统

### 5.1 什么时候使用 Subagent？

当任务需要**并行执行多个独立子任务**时：

```
用户: "帮我研究 AI Agent，并生成一份报告和一个网页"

Lead Agent 分析后决定:
  ├─ Subagent-1: 研究 AI Agent (并行)
  ├─ Subagent-2: 生成报告 (并行)  
  └─ Subagent-3: 生成网页 (并行)
```

### 5.2 Subagent 执行架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Lead Agent                               │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  task() tool 被调用                                  │  │
│  │  参数: { task: "研究 AI Agent", subtask: true }     │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │                                 │
│                         ▼                                 │
│  ┌──────────────────────────────────────────────────────┐│
│  │  Subagent Executor                                   ││
│  │  • 创建独立 Agent 实例                                ││
│  │  • 在独立线程/进程中运行                              ││
│  │  • 最大并发: 3 个                                    ││
│  │  • 超时: 15 分钟                                     ││
│  └──────────────────────┬───────────────────────────────┘│
│                         │                                 │
│                         ▼                                 │
│  ┌──────────────────────────────────────────────────────┐│
│  │  Subagent Result                                     ││
│  │  • status: completed/failed/timed_out                ││
│  │  • result: 返回消息                                   ││
│  │  • ai_messages: 完整对话历史                         ││
│  └──────────────────────┬───────────────────────────────┘│
│                         │                                 │
└─────────────────────────┼─────────────────────────────────┘
                          │
                          ▼
              Lead Agent 汇总 Subagent 结果
              生成最终回复给用户
```

### 5.3 配置参数

```yaml
agents:
  lead_agent:
    subagent:
      enabled: true           # 启用子代理
      max_concurrent: 3      # 最大并发数
      timeout_minutes: 15     # 超时时间
```

---

## 六、Skills 系统

### 6.1 什么是 Skill？

Skill 是一个 **Markdown 格式的能力包**，定义了：
- 工作流程
- 最佳实践
- 参考资源

### 6.2 Skill 文件结构

```markdown
---
name: research
description: 深度研究技能
version: 1.0.0
---

# 研究技能

## 适用场景
- 需要收集大量信息的任务
- 多角度分析任务

## 工作流程
1. 确定研究目标
2. 搜索相关信息
3. 整理分析
4. 输出报告

## 提示
- 使用 web_search 工具
- 多角度验证信息
```

### 6.3 Skill 加载机制

```
┌─────────────────────────────────────┐
│  配置指定 Skill 目录                  │
│  skills.container_path: /mnt/skills │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  load_skills() 递归扫描              │
│  skills/public/*/SKILL.md           │
│  skills/custom/*/SKILL.md           │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  注入到 System Prompt               │
│  (用户无感知，Agent 自动使用)         │
└─────────────────────────────────────┘
```

### 6.4 内置 Skills

| Skill | 用途 |
|-------|------|
| `research` | 深度研究 |
| `report-generation` | 报告生成 |
| `slide-creation` | PPT/幻灯片 |
| `web-page` | 网页开发 |
| `image-generation` | 图像生成 |

---

## 七、Memory（记忆系统）

### 7.1 记忆类型

| 类型 | 说明 |
|------|------|
| **User Context** | 用户的工作、偏好、关注点 |
| **History** | 历史对话摘要 |
| **Facts** | 结构化事实，带置信度 |

### 7.2 记忆更新流程

```
对话结束/达到 Token 阈值
         │
         ▼
  MemoryMiddleware 触发
         │
         ▼
  LLM 分析对话内容
  提取: 用户信息 + 关键事实
         │
         ▼
  存储到 memory.json
         │
         ▼
  下次对话时
  注入到 System Prompt
```

### 7.3 配置

```yaml
memory:
  enabled: true
  storage_path: memory.json
  debounce_seconds: 30     # 批量更新延迟
  max_facts: 100            # 最大事实数
  fact_confidence_threshold: 0.7  # 最低置信度
  injection_enabled: true     # 启用注入
  max_injection_tokens: 2000 # 最大注入 Token
```

---

## 八、Streaming（实时响应）

### 8.1 前端-后端通信

```
Frontend                          Backend
   │                                  │
   │  POST /api/runs/stream           │
   │  ──────────────────────────────▶│
   │                                  │
   │  ┌─────────────────────────────┐ │
   │  │ SSE Stream (text/event-stream)│ │
   │  │                             │ │
   │  │ ◀────────── values ──────────│ │  Agent 思考过程
   │  │ ◀────────── messages ────────│ │  工具调用
   │  │ ◀────────── tasks ──────────│ │  子任务状态
   │  │ ◀────────── [DONE] ─────────│ │  完成
   │  └─────────────────────────────┘ │
   │                                  │
   ▼                                  ▼
```

### 8.2 LangGraph Stream Modes

| Mode | 内容 |
|------|------|
| `values` | 状态快照 |
| `messages` | 新消息列表 |
| `messages-tuple` | (role, content) 元组 |
| `updates` | 节点更新 |
| `events` | 自定义事件 |
| `tasks` | 后台任务状态 |

---

## 九、Frontend-Backend 集成

### 9.1 前端核心文件

```
frontend/src/core/
├── api/
│   ├── api-client.ts      # 基础 API 客户端
│   ├── stream-mode.ts    # 流模式处理
│   └── index.ts          # 导出
├── threads/
│   └── hooks.ts          # Thread 状态管理
└── messages/
    └── handling.ts       # 消息处理
```

### 9.2 关键 API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/runs/stream` | POST | 发起带 Stream 的对话 |
| `/api/threads` | GET | 列出所有会话 |
| `/api/threads/{id}` | DELETE | 删除会话 |
| `/api/threads/{id}/uploads` | POST | 上传文件 |
| `/api/memory` | GET | 获取记忆数据 |
| `/api/models` | GET | 列出可用模型 |

---

## 十、配置系统

### 10.1 config.yaml 结构

```yaml
config_version: 7
log_level: info
token_usage:
  enabled: false

models:                    # LLM 模型配置
  - name: MiniMax-M2.7-highspeed
    display_name: MiniMax / MiniMax-M2.7-highspeed
    use: langchain_openai:ChatOpenAI
    model: MiniMax-M2.7-highspeed
    api_key: $MINIMAX_API_KEY
    base_url: https://api.minimaxi.com/v1

tool_groups:               # 工具分组
  - name: web
  - name: file:read
  - name: file:write
  - name: bash

tools:                    # 工具定义
  - name: web_search
    use: deerflow.community.ddg_search.tools:web_search_tool
    group: web

sandbox:                  # 沙箱配置
  use: deerflow.sandbox.local:LocalSandboxProvider
  allow_host_bash: true

skills:                   # Skills 配置
  container_path: /mnt/skills

memory:                   # 记忆配置
  enabled: true
  storage_path: memory.json

checkpointer:             # 状态持久化
  type: sqlite
  connection_string: checkpoints.db
```

---

## 十一、二开建议

### 11.1 扩展 Tool

在 `config.yaml` 添加自定义工具：

```yaml
tools:
  - name: my_tool
    use: my_package.my_module:my_tool_function
    group: custom
```

### 11.2 添加 MCP Server

在 `extensions_config.json` 配置：

```json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

### 11.3 自定义 Middleware

```python
from langchain.agents.middleware import AgentMiddleware

class MyCustomMiddleware(AgentMiddleware):
    async def after_agent(self, output, timeout):
        # 自定义处理逻辑
        return output
```

### 11.4 二开优先级建议

| 优先级 | 方向 | 说明 |
|--------|------|------|
| **P0** | Tool 扩展 | 添加业务相关的自定义工具 |
| **P0** | Skills 开发 | 封装领域知识为 Skill |
| **P1** | Middleware | 添加日志、监控等横切关注点 |
| **P1** | UI 定制 | 基于 UniClaw 品牌定制前端 |
| **P2** | 模型切换 | 支持更多 LLM Provider |

---

## 附录：关键文件索引

| 功能 | 文件路径 |
|------|----------|
| Agent 工厂 | `backend/packages/harness/deerflow/agents/factory.py` |
| Lead Agent | `backend/packages/harness/deerflow/agents/lead_agent/agent.py` |
| Thread State | `backend/packages/harness/deerflow/agents/thread_state.py` |
| Middleware 链 | `backend/packages/harness/deerflow/agents/middlewares/` |
| Sandbox | `backend/packages/harness/deerflow/sandbox/` |
| Subagent | `backend/packages/harness/deerflow/subagents/` |
| Skills | `backend/packages/harness/deerflow/skills/` |
| Memory | `backend/packages/harness/deerflow/agents/memory/` |
| Gateway API | `backend/app/gateway/routers/` |
| 前端 API | `frontend/src/core/api/` |
| LangGraph 配置 | `backend/langgraph.json` |
| 主配置 | `config.yaml` |
