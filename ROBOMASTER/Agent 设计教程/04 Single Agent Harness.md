---
标题: Single Agent Harness
修改日期: 2026-09-07
状态: 完成
编写人: 郑邦合
来源: 上海大学SRM战队RM培训视频
所属: Agent 设计教程笔记
---
# 四、经典 Single Agent Harness

> 本节回答：文档解决了 Agent 应该知道什么，接下来要解决的是——这些信息怎么进入模型、模型提出的工具调用又由谁真正执行。这就是 Harness 的职责。

## 4.1 核心定义

**Harness = 模型 + 工具 + 状态 + Loop。** 它是包围模型的一整套运行环境，把一次次的模型调用、工具执行和状态更新，连接成一个受控的 Agent Loop。

**主讲人观点（关键）：**
> Harness 并不是什么新概念，它是对已有的东西进行一个比较好的综合。很多人往往只做一个 ReAct，但这是不够的。而 Harness 有一个完整的 loop，包含了如何 build context、去控制 permission、checkpoint，以及 tool router（工具路径），是一个很好的架构。

> 可以理解为一个操作系统——可以安装不同软件，通过 harness 进一步执行，将这些状态重新连接成一个受控的 agent loop。

**也就是说：** ReAct 只是"模型怎么思考下一步"这一个环节；而 Harness 是把思考、取上下文、查权限、调工具、存状态、评估整套都管起来的运行时外壳。

## 4.2 解决了什么问题

**本节结论：** 模型只负责提出下一步行动；Harness 负责组装上下文、执行工具、保存状态、控制权限和判断何时停止。

**核心价值：** 设计一个 Single Agent 架构，可以参考这个来设计。这样就把完全黑盒的模型，变成一个可控、可观察的系统行为。

## 4.3 一次完整运行（流程图+九环节）

用一张流程图概括：

```text
用户目标 ─→ Context Builder ─→ LLM ─→ Validation ─→ Permission Controller ─→ Tool Router
(需求/约束)    │ 项目规则/相关代码/     │     工具是否存在?    用户是否授权?    执行搜索/文件/
               │ 任务状态/必要历史      │     参数符合 Schema? 是否高风险?     数据库/外部API
               ↑                      │                     是否需人工确认?         │
               │ 恢复/下一轮            │ Tool Call                允许             │
               │                       ↓                                          ↓
         State/Checkpoint ←─ 保存 ── Agent Loop ←─ 决策 ──                Observation
         (当前步骤/任务状态/      Continue/Finish/                       (工具结果/错误/测试证据)
          已执行动作/恢复位置)     Retry/Wait for Approval                     │ 反馈
                                      │ 记录                                   └─→ 回到 LLM
                                      └─→ Trace/Evaluation
                                          (模型调用/工具调用/Token耗时/成功标准)
```

**各环节职责：**

**1. 用户目标 → Context Builder（组装上下文）**
用户目标要有需求、约束、授权范围，**用户写得越清晰越好，不能只给一小段话**。Context Builder 加载：
- 项目规则（CLAUDE.md 的稳定规则）
- PRD 相关部分
- 当前任务 PROGRESS.md 的进度代码
- 最近有用的工具结果（result）

结合成一个 context，扔给 LLM。

**2. Context Builder → LLM（判断）**
LLM 收到 context 后，判断工具是否足够、需不需要额外工具。比如查天气，已装了查天气的 tool，判断是否需要生成 tool call。LLM 输出两类：**生成 Final Answer（直接回答）** 或 **Tool Call（调用工具）**。

**3. LLM → Validation（校验，第一个安全关口）**
如果是 Tool Call，走向 Validation，校验：
- 工具是否存在？
- LLM 传的参数是否符合格式要求（Schema）？

这就是类似一个校验、一个安全保障。

**4. Validation → Permission Controller（权限/人工关口）**
检验通过后，如果属于有风险权限、用户不让你这么做，就需要人工校验（Human-in-the-loop），进入 Permission Controller：
- 用户是否授权？
- 是否高风险？
- 是否需要人工确认？

**5. Permission → Tool Router（真正执行）**
经过一系列检查之后，才会变成一个调用文件、数据库的方法。Tool Router 负责执行：搜索、文件、数据库、外部 API（工具路径）。

**6. Tool Router → Observation → 反馈回 LLM**
工具执行后产生 Observation（工具结果 / 错误 / 测试证据），反馈回 LLM，进入下一轮。

**7. State/Checkpoint & Agent Loop（状态/停止）**
- State / Checkpoint：保存当前步骤、任务状态、已执行动作、恢复位置（这就是"恢复/下一轮"的来源）。
- Agent Loop：决定 Continue / Finish / Retry / Wait for Approval。

**8. Trace/Evaluation（记录/评估）**
记录模型调用、工具调用、Token 耗时、成功标准——这是"可观察"的落点。

**9. Harness 硬边界（兜底/限制）**
图右侧红色的硬边界限制整个循环：
- MAX_TURNS（最大轮数）
- Timeout · Budget（超时/预算）
- Sandbox · Allowlist（沙箱/允许白名单）
- Human-in-the-loop（关键时刻人工介入）

**与本课程前面的衔接**

- Context Builder 把项目文档（CLAUDE 规则 + PRD + PROGRESS + 工具结果）组装成 context——这正是前面学的上下文工程/按需加载的落地。
- Validation / Permission 做安全和权限把关——呼应"CLAUDE.md 只是软指令，不是真权限系统，真正挡危险的是硬机制"。
- Agent Loop / Checkpoint 管状态和停止——呼应 PROGRESS.md（存档）和"真正运行时持久化靠数据库"。
- Tool Router 真正执行——回答过渡句"模型提出的工具调用由谁真正执行"。

**课程脉络：** 先教怎么"喂"（文档/上下文工程）→ 再教怎么"控制"（三种思考模式）→ 现在教怎么"包住"运行时（Harness 把这一切连成受控的 Loop）。
## 4.4 一次完整运行（概念伪代码）

**定位：** 上一节的流程图是"长什么样"，这节的伪代码把流程一步步用代码走一遍。主讲人开头：Harness 把模型不完全可靠的行动意图，转换为受控、可观察、可恢复的系统行为。

**概念伪代码（完整）：**

```text
state = load_or_create_state(task_id)

while not state.finished:
    assert state.turns < MAX_TURNS

    context = build_context(
        project_rules,
        task,
        relevant_files,
        state,
        recent_observations
    )

    response = model.generate(context, tool_schemas)

    if response.is_final:
        state.finish(response.output)
        break

    tool_call = validate(response.tool_call)
    decision = permission_check(user_authority, tool_call)

    if decision.requires_approval:
        state.wait_for_user(tool_call)
        break

    observation = execute_with_timeout(tool_call)
    state.append(observation)

    checkpoint(state)
```

**运行步骤对照：**

| 步骤 | 伪代码对应 |
| --- | --- |
| 1 用户提出目标 | `state = load_or_create_state(task_id)` |
| 2 Context Builder 加载规则/任务/相关代码和状态 | `context = build_context(project_rules, task, relevant_files, state, recent_observations)` |
| 3 LLM 返回文本或 Tool Call | `response = model.generate(context, tool_schemas)` |
| 4 Harness 校验 Tool Call | `tool_call = validate(response.tool_call)` |
| 5 Permission Controller 判断是否允许 | `decision = permission_check(user_authority, tool_call)` |
| 6 Tool Router 执行工具 | `observation = execute_with_timeout(tool_call)` |
| 7 Observation 回写任务上下文 | `state.append(observation)` |
| 8 Agent Loop 判断继续/结束/重试/待确认 | `while not state.finished:` ... |
| 9 State Store 保存检查点 | `checkpoint(state)` |
| 10 Trace 记录完整运行轨迹 | 日志记录 |

**关键点：**
- `while not state.finished` + `assert state.turns < MAX_TURNS` —— 硬边界兜底（最多跑 MAX_TURNS 轮，防死循环）。
- `model.generate(context, tool_schemas)` —— 给模型的输入是"构建好的 context"，不是把整个对话全塞进去。
- 三个阶段把关：validate（工具/Schema）→ permission_check（授权/风险/人工）→ execute_with_timeout（带超时执行）。
- `checkpoint(state)` —— 每轮结束存检查点，可恢复。

## 4.5 Harness 的核心组件

**组件表（重要）：**

| 组件 | 职责 | 典型失败 |
| --- | --- | --- |
| Context Builder | 选择并组织本轮上下文 | 加载过多、过期或冲突信息 |
| Agent Loop | 控制模型与工具的迭代 | 死循环、无法停止 |
| Tool Registry | 描述和注册工具 | 工具描述不清、Schema 错误 |
| Tool Router | 执行工具并返回 Observation | 超时、异常、非法返回 |
| State Store | 保存任务状态和检查点 | 状态丢失、并发覆盖 |
| Permission Controller | 校验授权和副作用 | 权限扩大、提示疲劳 |
| Trace / Evaluation | 记录和评估执行轨迹 | 无法复现、只看到最终答案 |

**Context Policy（主讲人补充，关键）：** 一个对话有很多 context——用户的输入、模型的返回、调用的事实等。我们写入后，读取什么、怎么读取、哪些取舍需要写出来。主讲人说"和 AI 讨论，他不展开"。这其实是在说 Context Builder 不是随便拼，而是有一套明确的 Context Policy（上下文策略）：哪些该读、哪些该丢、怎么取舍。它决定"这轮到底该带什么进上下文"。

**Harness 结构树：**

```text
Harness
├── Context Policy        读取哪些MD、代码、Memory和State
├── Execution Loop        模型和工具怎样反复运行
├── Tool Runtime          工具注册、参数校验和执行
├── State Machine         任务状态与Checkpoint
├── Safety Enforcement    权限、Sandbox、预算和人工确认
└── Observability         Trace、日志、成本和评估
```

**组件定位：**
- Context Policy：读取哪些 MD、代码、Memory 和 State（把取舍写出来）。
- Execution Loop：模型和工具怎样反复运行（Loop 怎么设计）。
- Tool Runtime：工具注册、参数校验和执行。
- State Machine：任务状态与 Checkpoint。
- Safety Enforcement：权限、Sandbox、预算和人工确认。
- Observability：Trace、日志、成本和评估。

## 4.6 怎么用（搭建顺序建议）

**主讲人建议的搭建顺序（7 步）：**

1. 第一步：定义任务状态与停止条件。
2. 第二步：定义模型输入输出和 Tool Schema。
3. 第三步：实现 Tool Registry 与 Router。
4. 第四步：实现最小 Agent Loop。
5. 第五步：加入超时、重试、最大轮次。
6. 第六步：加入权限确认、Sandbox。
7. 第七步：加入 Checkpoint、Trace 与评估。

**主讲人忠告：** 不要从"支持十种 Agent 模式"开始。先让一个 Agent 能够可靠完成一类任务。从最小可用的 loop 起步，逐步加边界和安全。

## 4.7 安全边界

**核心观点：** 模型说"我要清理项目、清理旧文件"，需要 prompt 做软约束，但这不是硬约束。因此要用 harness 做硬约束，用代码强行按住。所以：模型负责提出意图，harness 决定行为能不能执行、如何执行，且状态可追踪。

**安全边界的典型流程（LLM 提出"删除旧文件"）：**

```text
LLM 提出：删除旧文件
   ↓
Harness 检查：
   - 用户是否授权？
   - 工具是否允许删除？
   - 路径是否在工作区？
   - 是否属于高风险操作？
   - 是否需要人工确认？
   ↓
允许 / 拒绝 / 请求确认
```

**关键结论：** Prompt 中的"不要删除文件"是软约束；工具权限、Sandbox 和审批机制才是硬边界。

## 4.8 局限是什么

**被经常忽略但很重要的三个组件（右侧 Canvas 明确标注）：**

- **State 与 Checkpoint**：负责保存任务进行到哪一步、执行了哪个动作、下一步应该从哪里开始。（对应图上 State/Checkpoint）
- **Loop**：决定应该继续、重试，还是等用户确认。它不能完全替模型说"我还没做完"，它会有一个时间限制。
- **Trace**：记录每次工具调用参数、Token 消耗、耗时状态。一旦失败，需要用 Trace 拉出来判断——是工具错误、参数错误、模型超时、网络问题，还是权限问题。这个需要清晰可视化出来，而不是完全黑盒。这就是 Harness 的意义所在。

**局限清单：**

- 所有工作集中在一个上下文，长任务容易 Context Rot。
- 独立子任务通常串行执行。
- 一个错误假设可能影响后续所有步骤。
- 能力越多，Tool Selection 越困难。
- 长任务需要更复杂的 Checkpoint 和恢复。
- 单 Agent 已经足够时，继续增加自主性只会增加风险。

**课堂互动（如果 Agent 连续十次调用同一个搜索工具，应该在哪一层解决？）**

主讲人解答：当然可以用 prompt 提醒他不要重复，但这是软约束。更可靠的是 Harness 限制它的最大轮次（MAX_TURNS）、Token 上限（TOKEN_MAX），通过一系列参数把它牢牢框住——这是硬性限制。

---

## 过渡句 → Multi-Agent

当一个任务包含多个互相独立、可以并行探索的方向时，单 Agent 的串行执行和单一上下文会成为瓶颈。此时才需要考虑 Multi-Agent。
---