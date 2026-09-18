---
标题: WebSocket 与飞书通信
修改日期: 2026-09-07
状态: 完成
编写人: 郑邦合
来源: 上海大学SRM战队RM培训视频
所属: Agent 设计教程笔记
---
# 六、WebSocket 与飞书通信

> 本节讲 Agent 怎么和用户交流。定位：前面讲了 Agent 内部怎么思考、怎么组织、怎么协作，这章讲对外接口层——消息从飞书进来、结果回到飞书。本节结论：Agent 架构解决"系统怎样思考和执行"；WebSocket 等通信机制解决"消息怎样在系统之间传输"。

## 6.1 四种通信机制

| 机制 | 核心定义 | 适合场景 | 局限 |
| --- | --- | --- | --- |
| HTTP | 一次请求对应一次响应 | 普通 API、短任务 | 不适合持续推送进度 |
| Webhook | 外部平台主动调用你的 HTTP 接口 | 飞书事件通知 | 仍是单次请求，需处理重试和幂等 |
| SSE | 服务端通过一个 HTTP 连接持续向客户端推送 | Token 流、单向状态更新 | 客户端不能在同一通道双向发送 |
| WebSocket | 建立全双工长连接，双方均可主动发送消息 | 实时对话、进度、取消、协作状态 | 连接、心跳、重连和扩容更复杂 |

**使用判断：**
- 外部平台（飞书）主动调用我们 HTTP 接口 → 适合被动接收通知（Webhook）。
- HTTP 适合普通短请求。
- SSE 是服务端单向持续推送，比如只推送 token 使用量信息。只要服务端往浏览器推 token，SSE 反而更简单。
- WebSocket 是双工长连接，适合进度读取、双向交互、确认这些。

**关键提醒：** 不是只要看到一些输出就要用 WebSocket。如果只是一些服务端向浏览器推送 token，SSE 反而更简单；WebSocket 会出现反向压力、心跳等问题。

## 6.2 WebSocket 解决了什么问题

- Agent 任务运行时间较长，普通 HTTP 容易超时。
- 前端需要实时看到 Token、工具和子任务进度。
- 用户可能需要暂停、取消或确认操作（双向交互）。
- 服务端需要主动推送状态变化（不能等前端来问）。

## 6.3 怎么用

**核心：建议传递事件，而不是随意发送字符串。**

```json
{
  "type": "tool_started",
  "task_id": "task_123",
  "sequence": 17,
  "timestamp": "...",
  "payload": {
    "tool": "web_search"
  }
}
```

**典型事件：**

```
task_created
agent_started
token_delta
tool_started
tool_finished
subagent_created
task_progress
approval_required
task_completed
task_failed
```

**说明：** 传递事件（结构化 JSON），而不是随意发字符串——这样前端和系统都能按 type 处理，可扩展。

## 6.4 飞书场景中的位置

**完整流程图（飞书 → 系统 → 回发）：**

1. **飞书 ──事件──→ 事件接入**：用户消息 / 事件订阅 / Bot 回复。通过 Webhook 或平台长连接，做签名校验、解密。
2. **事件接入 ──→ Channel Adapter**：平台协议 → Domain Event（平台协议转成内部领域事件），做去重、用户映射、消息格式转换。
3. **Channel Adapter ──Domain Event──→ Task Service**：创建任务、幂等检查、状态转换、持久化。
4. **Task Service ──→ Orchestrator → Agent**：总管把任务交给 Agent 执行。
5. **执行后回发**：飞书接口往回发 → 回复用户。
6. **调式前端 → 取消/确认**：双向交互。

**分层原则（重要）：**
- Agent Runtime 负责执行任务。
- WebSocket Gateway 负责连接。
- Event Bus 解耦两者。

**说明：** 只要在后端写一个接口，把接口和飞书连接起来。飞书有一个控制平台，机器人会有对应权限，比如编辑表格、可创建一个企业。

**流程串讲：** 飞书消息进入系统 → 通过 webhook 进行尝试性接入 → 进入某个 channel → 达成平台协议校验、签名转换、去重 → 转换成系统内部事件 → 根据 TaskService 创建任务、保存状态 → 总 Orchestrator 把任务交给 Agent 执行 → 再通过飞书接口往回发。

## 6.5 局限是什么

- 需要心跳、断线重连和连接清理。
- 多实例部署需要共享 Pub/Sub 或消息总线。
- 事件可能重复、乱序或丢失，需要序列号和幂等。
- 鉴权不能只在首次连接时考虑。
- 飞书接入不一定需要自己使用 WebSocket，取决于平台事件接入方式和你的前端需求。
- 如果只是服务端向浏览器推送 Token，SSE 可能更简单。

**常见误区（重要）：** WebSocket 不是 Agent 的 Memory，也不是多 Agent 的调度器。它只是通信通道。别把通信（WebSocket）和记忆（Memory）、调度（Orchestrator）混为一谈。WebSocket 只负责"传消息"，不负责"记东西"或"派活"。