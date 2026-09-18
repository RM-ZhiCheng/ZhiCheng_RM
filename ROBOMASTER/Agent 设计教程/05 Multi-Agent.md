---
标题: Claude 式 Multi-Agent
修改日期: 2026-09-07
状态: 完成
编写人: 郑邦合
来源: 上海大学SRM战队RM培训视频
所属: Agent 设计教程笔记
---
# 五、Claude 式 Multi-Agent

> 本节讲 Multi-Agent。定位：假设我们的任务需要调研三个独立不同的方向，让一个 Agent 串行做会很麻烦（单 Agent 的串行执行 + 单一上下文成为瓶颈），这正是需要 Multi-Agent 架构的原因。主讲人以 Claude 为例讲 Multi-Agent 的架构。

**口径说明：** 这里讲的是 Anthropic 公开介绍的 Claude Research 多 Agent 系统——Lead Researcher 使用 Orchestrator-Worker 模式创建并协调 Subagent。不要描述成所有 Claude 产品都用完全相同的内部实现。

**本节结论：** Orchestrator 通过受控的 Subagent Tool，创建拥有独立上下文和有限权限的 Worker；Worker 返回压缩后的结构化结果或 Artifact 引用。

## 5.1 核心定义（Orchestrator-Worker 模式）

**Claude Research 公开架构采用 Orchestrator-Worker 模式**，分五个角色：

1. Lead Agent 理解目标、规划和分解任务。
2. 通过工具创建多个独立 Subagent。
3. Subagent 在独立上下文中使用工具完成子任务。
4. Lead Agent 收集结果、查漏补缺并综合。
5. 必要时再交给 Citation 或 Review Agent 做专项处理。

**主讲人核心观点：** 不管是总的 Claude 还是什么，一般都有一个 Orchestrator（脏的 orchestration），类似一个总的经理，会管理你的 loop 如何实现。

## 5.2 解决了什么问题

主要针对适合广度优先探索的复杂任务：

- 多个独立方向可以并行。
- 每个方向拥有干净的独立上下文。
- 大量搜索细节不必全部进入 Lead Agent（Lead AI 不用背所有中间过程）。
- 可以为不同任务分配不同工具、Prompt 和预算。
- 专项 Agent 可以承担引用、审查等后处理。

## 5.3 Orchestrator 做什么（Lead Agent 的职责）

**Lead Agent 要做：**
- 理解目标。
- 判断复杂度（Lead Agent 判断这个问题是否复杂）。
- 制定总体计划。
- 保存计划到 Memory（然后根据 Memory 进行审查，以及反向的 evaluation 评估）。
- 拆分不重叠的子任务。
- 创建 Subagent。
- 跟踪状态与预算。
- 汇总结果。
- 判断是否需要下一轮。
- 停止并输出。

**任务拆分（关键）：** Lead Agent 要把任务边界拆分清楚，子任务拆分不重复。合格的子任务要用目标、范围、上下文、允许的工具和输出格式标准。

**流程走向：** 用户目标（复杂、开放式、可拆分任务）→ Lead Agent / Orchestrator → spawn subagent Tool → 三个 Subagent A/B/C（各自独立 Context / Agent Loop / Tools）→ 外部工具（Web Search / Page Reader / Files / Database / MCP / APIs）→ Artifact Store / Structured Results → Citation / Review Agent → 最终结果。

## 5.4 Subagent 怎么创建（核心机制）

**主讲人核心观点：** 我们可以把创建 subagent 这个本身作为工具封装起来。Lead Agent 会生成一个 subagent 调用，里面带有：任务的合同要求、允许的工具、预算、输出的格式。

**这就是 spawn subagent Tool（图右侧紫色块）：** 它包含 Task Contract（任务合同）、Context、Tool Allowlist（允许的工具）、Budget、Output Schema（输出格式）。

**接收后：** Multi-Agent 会为这些 worker 创建独立的 system prompt：每个 subagent 都会对应一个单独的 prompt，比如"你是写代码的""你是校验的"。然后都会有新的独立 context、独立 loop、独立 allowlist、独立权限。

**点睛总结：** Multi-Agent 类似一个总 Harness，去管理其他受限的 Harness。

**图右侧结构：**
- 安全边界：Subagent 权限 < Lead 权限 + 用户授权；Handle 级回检查；Sandbox · Budget · Trace。这是降权限的设计，Subagent 权限被限制在 Lead 之下。
- 每个 Subagent 独立：独立 Context、独立 Agent Loop、独立 Tools（Tools for A/B/C）。
- 外部工具：Web Search、Page Reader、Files、Database、MCP、APIs。
- Artifact Store：谁做、代码、数据、来源索引、Checkpoint。
- Structured Results：Summary · Evidence · Artifact Reference · Open Questions。
- Citation / Review Agent：做引用、对照验收标准、专项处理。
**Worker 内部也走完整思考循环（补充）：**

**关键：** 在 worker 内部，每一个 agent 都会自己先 plan execute，然后 react、reflection 类似这种流程。

**含义：** Multi-Agent 不是"换一套思考方式"，而是把前面学过的 Single Agent Harness 那套完整循环（Plan-and-Execute / ReAct / Reflection）复制到每一个 Subagent 内部。每个 Subagent 都是一个完整的受限 Harness，自己有一套"规划→执行→自检"的循环。这正是"Multi-Agent 是管理其他受限 Harness 的总 Harness"的含义——总 Harness（Lead）在外面管，每个子 Harness（Worker）内部也在跑自己的 Loop。

**与前面章节的关联：** 三种思考模式（ReAct / Plan-and-Execute / Reflection）本来就装在每个 Agent 里。到了多 Agent，只是把它们分层——Lead Agent 负责拆任务、分派、汇总，Worker 各管一段独立上下文里跑自己的循环。

**一个合格的子任务合同（模板）：**

**主讲人核心观点：** Orchestrator 不应该亲自吞下所有细节。它的核心能力是正确委派和综合。也就是说，Lead Agent 不自己干完所有事，而是把每个子任务写成一份清晰的"合同"交给 Subagent。

**子任务合同模板：**

```text
目标：调研 LangGraph 是否适合构建飞书 Agent

范围：
- 状态持久化
- Tool Use
- Human-in-the-loop
- 部署复杂度

不需要：
- 公司历史
- 与当前项目无关的教程

可用工具：
- Web Search
- Page Reader

输出：
- 核心能力
- 优缺点
- 适用性结论
- 证据来源
- 未确认问题

预算：最多 8 次搜索、5 分钟
停止：证据足够或预算耗尽
```

**委派公式：**

```text
Subtask = Objective + Scope + Context + Tools
       + Output Schema + Budget + Stop Condition
```

**为什么"不需要"和"停止条件"很重要：**
- "不需要"（Negatives）：明确告诉 Subagent 别碰哪些，防止它跑偏、浪费搜索。
- "预算"（Budget）+ "停止"（Stop Condition）：限定最多 8 次搜索、5 分钟，证据足够或预算耗尽就停——这是硬约束，防止 Worker 无限烧 token（呼应 Harness 的 MAX_TURNS / TOKEN_MAX）。

## 5.5 Memory 怎么设计（分三层）

**核心：** Agent 会分为几层。

**第一层：Lead Agent Context（Lead 的上下文）**
保存当前需要参与综合的信息：
- 用户目标
- 总体计划
- 子任务状态
- Subagent 精炼后的结论
- 未解决问题

**第二层：Subagent Private Context（Worker 的私有上下文）**
保存某一个子任务的详细过程：
- 子任务合同
- 相关资料
- 搜索结果
- 工具 Observation
- 阶段性判断

**第三层：External Memory / Artifact Store（外部记忆）**
保存跨上下文的大结果：
- 总体计划
- Checkpoint
- 报告和代码
- 数据文件
- 来源索引

**核心机制：** 每一次我们需要去 context 的时候，就会结合 memory 去进行一个拼凑，加上 prompt 就是每一轮喂给 agent 的东西。context 的组合过程是由 orchestrator 来实现的。

### 返回 JSON（Subagent 不回传全部历史）

**关键规则：** Subagent 不应把全部历史复制回 Lead Agent，而应返回精简的结构化结果（JSON）：

```json
{
  "status": "completed",
  "summary": "LangGraph 适合需要持久状态和人工确认的工作流。",
  "evidence": ["source-1", "source-2"],
  "artifact": "artifacts/langgraph-analysis.md",
  "open_questions": ["当前部署框架下的运维成本尚未验证"]
}
```

**核心意义：** Multi-Agent 并不是让他们更聪明，而是把大量的上下文放在 worker 内部，只把有价值的、关键的内容进行传递。Subagent 可能有自己独立的 memory，但我们不会把它看出来，只会把关键的东西输出出来——这就是它的意义。

## 5.6 安全边界

**核心：** 多 Agent 会有更复杂的权限问题，每个 agent 都需要单独配置。

**权限关系必须满足：**

```text
Subagent 权限 ≤ Lead Agent 权限 ≤ 用户授权
```

**每个 Subagent 单独限制：**
- Tool Allowlist
- 可访问文件和网络
- 最大运行时间
- Token 与调用预算
- 最大子 Agent 数量
- 是否允许继续创建 Agent
- 哪些行为必须人工确认

**交接也是安全边界：**
- Lead → Subagent：检查委派任务是否超出用户授权。
- Subagent → Lead：检查结果、来源和执行轨迹是否被 Prompt Injection 污染。

## 5.7 怎么搭建一个简化版本

**搭建顺序（8 步）：**
1. 先拥有可靠的 Single Agent Harness。
2. 将创建 Worker 封装为 spawn_subagent Tool。
3. 为 Worker 创建独立 Context 和 Agent Loop。
4. 定义结构化子任务合同。
5. 定义统一结果 Schema。
6. 增加并发、超时、取消和预算。
7. 增加任务状态表与 Artifact Store。
8. 增加权限、Handoff 检查和 Trace。

**概念伪代码：** 先用代码写出一个 harness，分出一个 harness 类，根据这个类制造新的 worker，可以批量制造 worker，要给每个 worker 定义子任务合同。

```text
plan = lead_agent.plan(user_goal)
memory.save(plan)

subtasks = lead_agent.decompose(plan)

results = parallel_map(subtasks, task =>
    spawn_subagent(
        task = task,
        allowed_tools = tools_for(task),
        budget = budget_for(task),
        output_schema = ResearchResult
    )
)

gaps = lead_agent.find_gaps(results)

if gaps 非空 and budget 足够:
    results += 运行第二轮 Subagent(gaps)

draft = lead_agent.synthesize(results)
final = citation_or_review_agent(draft, artifacts)
```

**关键：** 在返回结果后，用一个评测 agent 进行综合评价，进行最后一个处理，包括 review 进行一个总结之类。对应伪代码里的 gap 检查 + 第二轮 + citation_or_review_agent。

## 5.8 局限是什么

**核心观点：** 局限形式是成本增加、过慢的错误。Orchestrator 的拆分错误会出现工作重复和一系列混乱情况。因此可以用单 Agent 的问题就不要用多 Agent。

**局限清单：**
- Token 和模型调用成本明显增加。
- 并行不代表零延迟，最慢 Worker 仍可能阻塞一批任务。
- Orchestrator 拆分错误会产生重复工作或覆盖缺口。
- Agent 间状态一致性和失败传播更复杂。
- 结果多次压缩可能损失关键细节。
- Prompt Injection 和权限问题会跨 Agent 传播。
- 紧密依赖、必须顺序推理的任务不一定适合拆分。
- 单 Agent 能解决的问题不应为了"高级感"强行多 Agent 化。

## 5.9 什么时候才应该使用多 Agent

**核心判断：** 值得用的是任务边界清楚、任务依赖少，就可以用多 Agent；因为这些情况下就是会有局限。

**同时满足越多，越适合：**
- 子任务能够明确拆分。
- 子任务之间依赖较少。
- 存在真实并行收益。
- 单一上下文已经成为瓶颈。
- 每个 Worker 可以输出清晰结果。
- 任务价值足以覆盖额外成本。

**课堂互动：** "调查三个框架并比较"和"追踪一个复杂并发 Bug 的因果链"，哪个更适合并行多 Agent？

**答案：** 前面（调查三个框架并比较）适合多 Agent——三个方向独立、可并行、边界清楚、依赖少。后面（追踪复杂并发 Bug 的因果链）适合单 Agent——它依赖顺序推理、每一步都依赖前一步结果，紧密耦合，必须串联，拆开反而乱。
---