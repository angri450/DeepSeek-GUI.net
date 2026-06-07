# NanoBot Runtime 集成方案

本文记录 `DeepSeek-GUI.net` 这个 fork 的下一阶段方向：桌面 GUI 继续保留现有
Electron + React 工作台体验，但底层 agent 能力逐步接入 `NanoBot.net`。目标不是
在界面上增加第二套运行时，而是把 NanoBot 暴露成 Kun 兼容的单一 HTTP/SSE 边界。

## 目标

- `DeepSeek-GUI.net` 作为桌面工作台，负责会话、文件树、工具详情、审批、设置和中英文界面。
- `NanoBot.net` 作为 .NET 8 agent runtime，负责模型调用、工具循环、记忆、MCP、Nong bridge、WebSocket/Web API 和本地工作区。
- 默认模型路线统一为 DMX 中转的 DeepSeek V4 Pro，模型 ID 为 `deepseek-v4-pro-guan`。
- 保持本地优先：配置、会话、记忆、工作区和工具执行都在本机落地，不引入强制云控平面。

## 不做

- 不在仓库、文档、测试或示例里写真实 API key。
- 不新增 GUI 可见的运行时切换器、诊断面板或第二套 provider 入口。
- 不复制外部项目源码、素材或专有命名，只借鉴交互结构和信息密度。
- 不把 NanoBot WebUI 直接嵌进 Electron；桌面端应通过协议调用 NanoBot 能力。

## 总体架构

```text
Renderer (React + Zustand)
  Chat / Write / Workspace / Tool detail / Settings
        |
        | preload IPC: runtimeRequest / startSse / stopSse
        v
Main process
  RuntimeHost
  Kun-compatible NanoBot adapter
        |
        | HTTP + SSE
        v
NanoBot Runtime Host (.NET 8)
  Sessions / Streaming turns / Events / Approvals
  Nanobot.Core Agent loop
  Providers / Tools / Memory / MCP / Nong
        |
        v
DMX OpenAI-compatible API
  baseUrl: https://www.dmxapi.cn/v1/
  model: deepseek-v4-pro-guan
```

当前 `docs/AGENTS.zh-CN.md` 要求 GUI 只保留 Kun 这一个运行时边界。因此落地方式是：

1. 短期保留前端 `kun-runtime.ts` / `kun-mapper.ts` 的调用形状。
2. 在 main process 下新增 NanoBot adapter，让它对 GUI 呈现 Kun 兼容 API。
3. NanoBot 侧补齐 HTTP/SSE 端点，优先复用 `Nanobot.Core` 已有 agent、tool、memory、MCP、Nong 能力。
4. 最终用户在 Settings -> Agent 只看到一套运行时配置，内部实现可以从 Kun 逐步切到 NanoBot。

## 协议映射

| GUI/Kun 能力 | NanoBot 对应能力 | P 阶段 |
| --- | --- | --- |
| 创建/列出会话 | `.webui` session store 或新的 runtime session store | P1 |
| 发送 turn | `Agent.RunStreamingAsync` / `Agent.RunAsync` | P1 |
| SSE 流式输出 | `RuntimeEventBus` + response delta | P1 |
| 工具调用时间线 | tool started/result/error events | P1 |
| 文件树 | workspace-bounded file browser | P1 |
| 记忆 | `FileMemoryStore`、`MEMORY.md`、session history | P2 |
| MCP 工具 | `McpClientFactory` + `McpToolProvider` | P2 |
| Nong bridge | `NongTool` argument-array bridge | P2 |
| 审批/拒绝 | approval gate，先做 API 契约再接 UI | P2 |
| fork/resume/search/archive | session store 扩展 | P3 |
| usage/cost/cache | provider response usage + runtime aggregation | P3 |
| Write inline completion | 复用同一 NanoBot provider 配置，后续单独适配 | P3 |

## NanoBot HTTP/SSE 边界

建议在 NanoBot 侧新增或扩展一个 runtime host，而不是让 Electron 直接调用 CLI：

```text
GET  /health
GET  /v1/runtime/status
GET  /v1/threads?limit=&search=&include_archived=
POST /v1/threads
GET  /v1/threads/{id}
POST /v1/threads/{id}/turns
GET  /v1/threads/{id}/events?since=
POST /v1/threads/{id}/interrupt
GET  /v1/workspace/tree?path=
GET  /v1/workspace/file?path=
GET  /v1/memory
GET  /v1/tools
POST /v1/approvals/{id}
POST /v1/user-inputs/{id}
```

SSE 事件建议统一成 append-only event log，GUI 只做映射和展示：

```json
{
  "seq": 12,
  "type": "tool.completed",
  "threadId": "thread_...",
  "runId": "run_...",
  "timestamp": "2026-06-07T00:00:00Z",
  "payload": {}
}
```

首批事件类型：

- `turn.started`
- `content.delta`
- `content.completed`
- `tool.started`
- `tool.completed`
- `tool.failed`
- `memory.updated`
- `approval.requested`
- `user_input.requested`
- `run.failed`

## 模型定制

NanoBot 默认模型列表使用 DMX 中转的 DeepSeek V4 Pro：

```json
{
  "providers": {
    "dmx": {
      "kind": "openai-compatible",
      "apiKey": "",
      "apiBase": "https://www.dmxapi.cn/v1/",
      "defaultModel": "deepseek-v4-pro-guan",
      "models": [
        {
          "id": "deepseek-v4-pro-guan",
          "apiModelId": "deepseek-v4-pro-guan",
          "supportsStreaming": true,
          "supportsTools": true
        }
      ]
    }
  },
  "agents": {
    "defaults": {
      "model": "dmx::deepseek-v4-pro-guan",
      "fallbackModels": ["dmx::deepseek-v4-pro-guan"]
    }
  }
}
```

注意：用户给出的接口是 `https://www.dmxapi.cn/v1/chat/completions`，但 NanoBot 的
OpenAI-compatible SDK 会自己拼接 `chat/completions`，所以配置里应写 base URL：
`https://www.dmxapi.cn/v1/`。

本地测试只允许使用环境变量：

```powershell
$env:DMX_API_KEY = "<local-secret>"
$env:DMX_API_BASE = "https://www.dmxapi.cn/v1/"
$env:DMX_MODEL = "deepseek-v4-pro-guan"
```

## 阶段计划

### P0：方案与配置统一

- 在 `DeepSeek-GUI.net` 固定本方案。
- 在 `NanoBot.net` onboard 配置、README 和环境变量里加入 `dmx` provider。
- 移除新手文档中的旧示例模型，把默认路径统一成 `deepseek-v4-pro-guan`。
- 验证 `dotnet test` 不破坏旧 OpenAI-compatible fallback。

### P1：NanoBot Runtime Host

- NanoBot 暴露 `/health`、runtime status、threads、turns、events、workspace tree。
- 支持流式输出、会话持久化、文件树、工具调用详情，覆盖当前 WebUI P2 能力。
- DeepSeek-GUI main process 能启动/连接 NanoBot host，并把事件映射到现有 Kun UI。

### P2：完整 agent 能力接出

- 接出 memory、MCP、Nong、web fetch/search、GitHub、shell allowlist。
- 做工具审批 gate 和 user input gate。
- GUI 右侧详情面板展示 tool args/result/error、memory 更新、Nong structured output。

### P3：桌面工作台增强

- 会话搜索、归档、fork、resume。
- usage/token/model 统计。
- Write inline completion 和选中文本助手接入同一 NanoBot provider 配置。
- 计划/todo/diff/approval 面板与 NanoBot event log 对齐。

## 验证矩阵

| 项目 | 命令或检查 |
| --- | --- |
| DeepSeek-GUI 类型检查 | `npm run typecheck` |
| DeepSeek-GUI 测试 | `npm test` |
| DeepSeek-GUI 打包 | `npm run build` |
| NanoBot 构建 | `dotnet build` |
| NanoBot 测试 | `dotnet test` |
| DMX 本地集成 | 设置 `DMX_API_KEY` 后跑 real integration tests |
| 手工冒烟 | 创建会话、流式输出、工具调用、文件树、记忆读写、Nong 调用 |

## 风险

- DeepSeek-GUI 当前 Kun 协议较厚，NanoBot adapter 要先做兼容层，避免一次性重写 UI。
- 工具审批和 user input 需要 durable gate，否则桌面端刷新后会丢 pending 状态。
- DMX 是 OpenAI-compatible 中转，真实返回字段可能和官方 SDK 有细节差异，需要用流式和工具调用分别实测。
- 真实 API key 只能放用户本机环境变量或 `~/.nanobot/config.json`，不能进入 git。
