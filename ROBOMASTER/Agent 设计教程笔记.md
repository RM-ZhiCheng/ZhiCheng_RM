---
标题: Agent 设计教程笔记
修改日期: 2026-09-07
状态: 修改中
编写人: 郑邦合
来源: 上海大学SRM战队RM培训视频
---
Agent 设计教程笔记
---


## 📚 章节目录

> 点击跳转到本笔记对应章节。

1. [[#一、总框架：Agent 的三种思考模式|01 · 三种思考模式]] —— ReAct / Plan-and-Execute / Reflection
2. [[#2.1 Prompt Engineering —— 怎么把指令说清楚|02 · Prompt 与上下文工程]] —— Prompt / 上下文工程 / Context Rot
3. [[#三、Vibe Coding 项目级规范文档|03 · 项目级规范文档]] —— PRD / APP_FLOW / 前端 / 后端 / CLAUDE / PROGRESS
4. [[#四、经典 Single Agent Harness|04 · Single Agent Harness]] —— 完整循环 / 组件 / 伪代码 / 安全边界
5. [[#五、Claude 式 Multi-Agent|05 · Multi-Agent]] —— Orchestrator-Worker / 三层Memory / 子任务合同
6. [[#六、WebSocket 与飞书通信|06 · WebSocket 与飞书通信]] —— 四种通信机制 / 飞书接入流程

## 一、总框架：Agent 的三种思考模式

同一个 Agent 可以用三种不同方式去"思考/控制"它干活。它们**不是三套互斥的架构，而是可以组合使用的控制模式**。

分三层理解：

| 层          | 内容                                       |
| ---------- | ---------------------------------------- |
| 第一层：系统基础设施 | Model · Tools · Memory · State · Harness |
| 第二层：组织架构   | Single-Agent / Multi-Agent               |
| 第三层：控制模式   | ReAct / Plan-and-Execute / Reflection    |

**一句话记住三者分工：**
> ReAct 是"走一步看一步"，Plan-and-Execute 是"先规划再干、偏了就重规划"，Reflection 是"干完检查、找差距、再修正"。

---

## 1.1 ReAct —— 边推理边行动

**核心定义：** ReAct 是 Reasoning + Acting。模型根据当前信息进行判断，选择一个行动，观察行动结果，再决定下一步。

**流程公式：** `Reason → Act → Observe → Reason → Act → Observe → Finish`

| 环节 | 干什么 |
| --- | --- |
| Reason | 基于当前信息推理判断 |
| Act | 执行一个动作（调用工具/查资料/跑代码） |
| Observe | 观察动作返回的结果 |
| → 循环 | 用观察结果再推理，决定下一步 |
| Finish | 目标达成，结束 |

**解决了什么问题：** 那些"没有实际反馈就没法往下走"的任务——比如查资料、跑代码、读文件，必须真的执行一步、看到结果，才知道下一步该干嘛。

**什么时候用：** 探索性强、中途信息会变的场景。

**局限：** 没有全局计划容易走弯路；如果每一步反馈都很贵（费时间/费钱），纯 ReAct 会来回折腾。

---

## 1.2 Plan-and-Execute —— 先规划，再执行

**核心定义：** 先制定一个高层计划，再逐项执行；执行结果与预期不符时，允许重新规划。

**流程公式：** `Goal → Plan → Execute Step → Check → Replan or Continue → Finish`

| 步骤                 | 干什么                   |
| ------------------ | --------------------- |
| Goal               | 明确目标                  |
| Plan               | 制定高层计划（先定大方向，不必细化到每步） |
| Execute Step       | 执行当前这一步               |
| Check              | 检查执行结果和预期是否一致         |
| Replan or Continue | 一致 → 继续下一步；不一致 → 重新规划 |
| Finish             | 完成收尾                  |

**和 ReAct 的核心区别：**
- ReAct：没有预先计划，`Reason→Act→Observe` 循环，边走边决定。
- Plan-and-Execute：规划在前，`Goal→Plan→Execute`，关键在 `Check→Replan` 这个纠偏环节。

**类比：**
- ReAct 像开车没导航，看到路况再拐。
- Plan-and-Execute 像开车有导航，先设目的地，走错就重新规划路线。

**局限（重要）：**
- 初始计划可能建立在错误假设上。
- 计划过细 → 维护成本高；环境变化快 → 计划很快过期。
- 模型可能机械执行错误计划。
- 关键：必须有明确的 Replan 条件，不能只在开头规划一次（这就是 `Check→Replan` 存在的意义）。

**什么时候用：** 目标明确、能提前拆分步骤的任务（大型工程、多阶段开发）。
**什么时候用 ReAct：** 目标模糊、需边探索边定（查资料、调试未知 bug）。实际常组合——先有大计划，执行中卡住就临时切 ReAct 应变。

---

## 1.3 Reflection —— 做完检查、找差距、再修正

**核心定义：** Agent 对阶段结果进行检查，找出与目标、规则或验收标准不一致的地方，再进行修正。

**流程公式：** `Generate → Evaluate → Find Gap → Revise → Verify`

| 环节 | 干什么 |
| --- | --- |
| Generate | 生成当前结果 |
| Evaluate | 评估结果 |
| Find Gap | 找出与目标/验收标准的差距 |
| Revise | 修正差距 |
| Verify | 验证修正后是否符合 |

**解决了什么问题：**
- 第一版结果经常遗漏边界条件。
- 代码能运行 ≠ 满足业务需求。
- 多步骤任务里，早期错误会向后传播。
- 交付前需要检查质量和完整性。

**怎么用（关键）：** 不要只靠模型"再想一遍"，要依赖外部证据。

```text
result    = 执行当前任务
evidence  = 运行测试 + 静态检查 + 验收清单
gaps      = reviewer(result, evidence, acceptance_criteria)
if gaps 非空 且 未超过修改预算:
    修正(gaps)
```

**有效的 Reflection 输入（外部证据）包括：**
- PRD 中的验收标准
- 单元测试和集成测试结果
- Lint、类型检查、构建结果
- 独立 Reviewer 的审查意见
- 真实工具返回的数据

**局限：**
- 只是"再想一遍"往往发现不了自己的错。
- 没有评价标准时，Reflection 只是反复改写、原地打转。
- 如果 Reviewer 和生成者用同一套错误假设，会一起出错。
- 必须设置最大修改轮次和停止条件，不能无限改。

---

## 2.1 Prompt Engineering —— 怎么把指令说清楚

**核心定义：** 通过角色、目标、约束、示例、输出格式、成功标准，让模型正确理解当前指令。

**解决了什么问题：**
- 需求表达模糊（AI 不清楚你要啥）。
- 输出结构不稳定（每次格式不一样）。
- 模型不知道哪些内容不能做。
- 不同人对"完成"的理解不一致。

**一个可执行的 Prompt 通常包含：**

| 要素 | 回答什么 |
| --- | --- |
| 目标 | 要完成什么 |
| 背景 | 为什么要做 |
| 范围 | 包含什么 / 不包含什么 |
| 约束 | 必须遵守什么 |
| 输入 | 当前有哪些信息 |
| 输出 | 以什么格式交付 |
| 验收 | 怎样判断完成 |

**局限（很多人忽略）：**
- Prompt 无法替代真正的权限控制（写命令、联网要靠工具权限）。
- Prompt 无法补齐不存在的业务信息（你不给，它就猜）。
- 指令越长 ≠ 效果越好。
- 冲突、过期、重复的指令会降低遵循效果。
- 只优化措辞，解决不了错误上下文或错误工具结果。

---

## 2.2 上下文工程（Context Engineering）—— 喂进去的必须"有用"

**核心思路：** 不要一股脑把项目里所有东西都丢给模型，而是要"组装"一份精简的上下文。

**怎么用（伪代码）：**

```text
context = []
context += 加载稳定规则（CLAUDE.md）
context += 加载当前需求（PRD 中相关部分）
context += 检索相关代码和文档
context += 加载当前任务状态（PROGRESS.md）
context += 最近且必要的工具结果
context = 去重、排序、压缩（context）
```

**每一步都是在"精挑"：**

| 加载项 | 是"挑"还是"全" |
| --- | --- |
| 稳定规则（CLAUDE.md） | 挑 → 只取现在的任务相关的规范 |
| 当前需求（PRD 相关部分） | 挑 → 不是整本 PRD，是相关章节 |
| 相关代码和文档 | 挑 → 检索出来的，不是全仓库 |
| 当前任务状态（PROGRESS.md） | 挑 → 当前进度这一段 |
| 最近且必要的工具结果 | 挑 → 只留最近、必要的，不是全部日志 |
| 最后一步 | 去重 + 排序 + 压缩，再交给模型 |

**解决了什么问题：**
- 模型不知道项目约束。
- 大量无关信息挤占注意力。
- 新会话无法继续旧任务。
- 过期信息与最新事实冲突。
- 工具结果太多，核心目标被淹没。

**局限（要清醒）：**
- "相关"本身需要判断，检索可能漏掉关键内容。
- 摘要会损伤细节。
- 外部 Memory 可能过期。
- 上下文设计不当时，会把错误信息稳定地传给每一次调用。
- 上下文窗口变大 ≠ 模型会同等重视所有内容。

---

## 2.3 上下文腐烂（Context Rot）

**核心定义：** 上下文虽然可能还没超过长度限制，但因为无关、重复、过期、冲突的信息不断累积，模型的判断质量逐渐下降。

**注意：** 这个问题不是"没塞满"才出现，而是"塞了但没管理"才出现。重点是"挑什么进去"，而不是"塞得多不多"。它是问题/症状的名称（一组现象），不是解法。

**用来识别这些现象：**
- 忘记最初目标（做着做着跑偏了）。
- 使用被否决的旧方案。
- 重复实现已有模块（重新造轮子）。
- 前后命名和约束不一致。
- 只关注最近一条消息，忽略整体要求。
- 修复一个问题时，破坏之前已经完成的功能。

**怎么管理（核心策略）：**
1. **只加载当前任务真正需要的文件** —— 不要像把整份 CLAUDE.md 或全部历史倒进去，按自己的范围取舍，筛出对当前任务有用的部分。**最重要的一条。**
2. 稳定规则写进规范文档，而不是依赖聊天历史。
3. 每完成一个阶段就做结果摘要。
4. 大产物写入文件，只在上下文中保留引用。
5. 明确标记 Deprecated 内容。
6. 定期清理重复和冲突指令。
7. 新会话用 PROGRESS.md 恢复，而不是加载全部旧聊天。

**重点强调（这条是灵魂）：**
> 不要把项目里所有东西（整份 CLAUDE.md、整个仓库、全部历史聊天）一股脑塞进上下文。要按自己的任务范围取舍，只筛选出当前任务有用的那部分进去。而且这不是一次性动作——写文档、维护上下文的过程中要持续做"途中处理"：该删的删、该合并的合并、该标记废弃的标记、该摘要的摘要。这是一套需要主动管理的好策略，不是"写一次就完事"。

**局限（管理不等于解决）：**
- 无法完全消除，只能管理。
- 过度压缩也会丢失关键上下文。
- 自动摘要可能把错误结论写成"事实"（要小心）。
- 长任务仍然需要外部状态 + 人工复核。

**过渡句：** 如果 Agent 的表现高度依赖上下文，就不能每次靠聊天重新解释项目。因此，项目需要一套项目级的外部记忆与协作契约——即"Vibe Coding 项目级规范文档"（PRD / CLAUDE / APP_FLOW / FRONTEND_GUIDELINE / BACKEND_STRUCTURE / PROGRESS）。

---

## 2.4 小结（这几个怎么串起来）

- Prompt Engineering 决定"指令清不清楚、有没有验收标准"（对应 PRD.md、AGENTS.md）。
- Plan-and-Execute 决定"先列计划再动手"。
- ReAct 决定"执行时边走边看结果"。
- Reflection 决定"做完自检、跑测试、看验收"。

**所以"用 Markdown 当外挂大脑"和"三种思考模式 + Prompt 模板"是配套的：** 文档给 Agent 提供目标/验收/边界（Prompt 要素 + 外部证据），Agent 再按 Plan → Act → Reflect 的循环执行。
---

# 三、Vibe Coding 项目级规范文档

> 主讲人核心判断：这些文档是除了 Memory 之外最重要的东西。
> 本节结论：这些文档不是为了增加形式（不是搞仪式感），而是为了分别保存产品事实、用户流程、工程边界、Agent 协作规则、当前工作状态。**不要把所有的内容写进一个文件**，划分这些文档的目的，是为了建立清晰的 "Source of Truth"（唯一事实来源）。

## 3.1 文档职责总览

| 文档 | 回答的问题 | 稳定性 |
| --- | --- | --- |
| PRD.md | 为什么做、为谁做、做什么、怎样验收？ | 中高 |
| APP_FLOW.md | 用户怎样完成任务、异常时发生什么、状态怎样变化？ | 中高 |
| FRONTEND_GUIDELINE.md | 前端怎样表现、组件遵守什么规则？ | 高 |
| BACKEND_STRUCTURE.md | 后端怎样分层、数据怎样流动？ | 高 |
| CLAUDE.md | Agent 在仓库里应该怎样工作？ | 高 |
| PROGRESS.md | 当前做到哪里、下一步是什么？ | 低，持续变化 |

**按稳定性分两类：**
- 稳定的（高/中高）：PRD / APP_FLOW / FRONTEND_GUIDELINE / BACKEND_STRUCTURE / CLAUDE —— 长期事实，基本不变，适合放"长期指令"。
- 持续变化的（低）：PROGRESS.md —— 当前状态，几乎每次都变，属于"当前状态"。

**最重要的一条：不要把所有内容写进一个文件。** 拆开是为了让每个角色/层面各有一个明确的"真相来源"（Source of Truth）。全塞进一个大文档，Agent 每次都得翻半天、分不清轻重——正是"上下文腐烂"要避免的。

**每个文档写什么：**
- **PRD.md（产品需求）**：产品需求的唯一事实来源，描述问题、用户、目标、范围、业务规则、验收标准。回答"为什么做、为谁做、做什么、怎么算做完"。
- **APP_FLOW.md（用户流程）**：用户从入口到结果的完整旅程，正常流程、异常分支、权限、页面/消息状态。回答"用户怎样完成任务、异常时发生什么、状态怎样变化"。
- **FRONTEND_GUIDELINE.md（前端要求）**：前端怎样表现、组件遵守什么规则，定视觉、组件、交互、异常状态、响应式、可访问性。
- **BACKEND_STRUCTURE.md（后端限制）**：后端怎样分层、数据怎样流动，定工程边界：模块划分、接口/数据流、报错/测试。
- **CLAUDE.md（Agent 协作规则，总纲最关键）**：Agent 在仓库里应该怎样工作，规定 Agent 怎么干活、项目命令与规范、安全与验证要求。这是直接给 AI 看的"行为守则"，对应项目里的 AGENTS.md。
- **PROGRESS.md（进度管理，动态状态）**：当前做到哪里、下一步是什么。像一份实时存档：记录当前目标、已完成、下一步、决策、阻塞、测试。**换新 Agent / 隔很久重启，只要读它就能立刻知道"进行到哪了"**，不用从头翻聊天记录。对应"途中处理"和"新会话用 PROGRESS.md 恢复"。

## 3.2 PRD.md：产品需求

**核心定义：** PRD 是产品需求的唯一事实来源，描述问题、用户、目标、范围、业务规则和验收标准。

**解决了什么问题：**
- Agent 不再一边写代码一边猜需求（边写边改，越改越乱）。
- 产品范围和非目标明确（知道哪些不做）。
- 开发与验收用同一套标准（做和判用一把尺）。
- 新会话可以快速理解产品价值（换人/换 Agent 也能上手）。

**怎么用（主讲人推荐模板，共 11 部分）：**

```markdown
# 产品名称
## 文档状态
Status: Active
Owner:
Last Updated:

# 1. 背景与问题        → 当前发生了什么？为什么值得解决？
# 2. 目标用户与使用场景 → 谁在什么情况下使用？
# 3. 产品目标          → 完成后希望产生什么结果？
# 4. 非目标            → 本版本明确不做/不做什么？
# 5. 核心功能          → 按 P0 / P1 / P2 描述
# 6. 业务规则          → 哪些规则不能违反？
# 7. 核心状态          → 有哪些状态？状态怎么转换？
# 8. 异常场景          → 超时、重复、权限不足、依赖失败怎么办？
# 9. 验收标准          → 使用 Given / When / Then 或清晰检查项
# 10. 约束             → 时间、成本、平台、合规和技术约束
# 11. 未决问题         → 仍需负责人决策的问题
```

**第 5 和第 9 是最重要的两块：**
- **#5 核心功能（P0/P1/P2）**：按优先级排，让 Agent 知道"先做什么、必须做什么"。P0=不做项目就废，P1=重要但不致命，P2=锦上添花。
- **#9 验收标准（Given/When/Then）**：**最重要的第一条**。用"在…情况下，当…发生时，应该…"格式把"怎么算做完"写死。没有它，Agent 不知怎么才算交付完成（对应 Reflection 里"要有外部证据/验收标准"）。

**关键提醒（主讲人特别强调）：**
> PRD 不能规定所有的技术实现。PRD 说的是"要什么"（what），不是"怎么做"（how）。具体用哪个框架、哪种数据结构、哪条 API 由工程在 BACKEND_STRUCTURE 决定。写出技术实现细节，反而会把实现方案错误地固化成为产品要求，限制方案自由度。

**局限：**
- PRD 不能替代架构设计（管需求，不管技术实现）。
- 写得太细会把实现方案错误地固化成为产品要求（把"怎么做"当成"要什么"）。
- 需求变化后不更新，反而会成为错误上下文（PRD 过期比没有更糟——Context Rot 的一个来源）。
- "支持消息处理"这种模糊描述，不能作为验收标准（要具体到可判断，否则等于没说）。

## 3.3 APP_FLOW.md：用户流程

**核心定义：** 描述用户从入口到结果的完整旅程，包括正常流程、异常分支、权限和页面/消息状态。

**解决了什么问题：**
- PRD 里孤立的功能点，被连接成一条真实体验（PRD 讲"有哪些功能"，APP_FLOW 讲"用户走一遍会经历什么"）。

> 这一节承接 PRD：PRD 告诉 Agent"有哪些事要做"，APP_FLOW 告诉 Agent"这些事串起来是什么体验"。后续还将展开异常分支、权限、状态流转。
> **核心区分（主讲人）：** PRD 是一个个"功能点"（有哪些事要做），APP_FLOW 是把它们连结成用户实际经历的过程。它回答的不是"有什么功能"，而是"用户走一遍会经历什么、每一步遇到情况会怎样"。

**例子（用户发消息的场景）：** 用户发了消息之后，系统要做一连串校验——这个需求是否重复、是否超出权限，最后把结果返回给用户。**但同时要划出分支**：切屏了怎么办？超时怎么办？高风险操作要不要"human in loop"（人在环里，即需要人工确认）？

**主讲人重点：成熟的 Agent 不只是架构上做得好，错误处理更关键。** 如果错误分支没想全，Agent 一跑就崩。APP_FLOW 的价值就在于把"正常路径 + 异常分支"都提前画出来。

**怎么用（主讲人推荐模板）：**

```markdown
## 角色与入口
## 流程：用户向飞书 Agent 提问
## 触发条件
   用户 在允许的会话中发送文本消息。
## 前置条件
   Bot 已安装，用户有使用权限。
## 正常流程
   1. 飞书发送消息事件。
   2. 系统校验事件并进行鉴权等检查。
   3. 创建 Agent Task。
   4. 返回"正在处理"。
   5. Agent 执行并回传结果。
## 异常流程
   - 非法签名：拒绝请求并记录日志。
   - 重复事件：返回已有任务，不重复执行。
   - Agent 超时：更新失败状态并通知用户。
   - 高风险操作：等待用户确认。
## 状态变化
   RECEIVED → QUEUED → RUNNING → SUCCEEDED / FAILED / CANCELLED
## 成功标准
   用户收到与原消息关联的最终答复。
```

**模板结构：** 角色/入口 → 触发条件 → 前置条件 → 正常流程（主路径）→ 异常流程（分支，critical！）→ 状态变化（状态机）→ 成功标准。

**局限：**
- 它描述体验，不负责决定所有后端实现。
- 只画正常流程会隐藏真实复杂度（异常路径才考验功力）。
- 流程图过大时需要拆分子流程（别一张图画到死）。
- 如果没有状态定义，流程容易停留在"页面跳转说明"（要落到状态，不只是页面切换）。

## 3.4 FRONTEND_GUIDELINE.md：前端规范

**核心定义：** 前端视觉、交互、组件、状态、响应式、可访问性和代码组织的长期约束。

**主讲人核心观点：** 前端开发创意是匮乏的，不管用什么 skill 还是别的。建议直接"抄袭"——找一个好看的前端，直接拿来主义抄过来。别费劲从零设计视觉，找做得好的页面照着搬，比自己凭空造高效得多。

**解决了什么问题：**
- 不同 Agent 生成的页面仍然像同一个产品（风格统一）。
- 避免每个页面重新发明颜色、间距和组件（不重复造轮子）。
- 不遗漏异常状态和可访问性。
- 减少局部实现破坏整体设计系统。

**局限：**
- 不能替代真实组件库和设计稿（它不是组件库本身）。
- 过度规定会限制具体场景的合理变化（太死板）。
- 只有规则没有组件示例时，模型可能理解不一致（要配示例）。
- 技术规范与视觉规范混杂太多时难以维护（分开写）。

## 3.5 BACKEND_STRUCTURE.md：后端结构

**核心定义：** 描述后端的分层、模块职责、依赖方向、数据模型、接口契约和关键工程机制。

**主讲人核心观点：** 后端是 Agent 最重要的边界。它要明确告诉 Agent：系统有哪些层、每层负责什么、不负责什么、调用谁、如果进入怎么校验、在哪里保存、错误和重置怎么处理。

**例子（宏观架构）：** 简单架构是一个总代理，叫 orchestrator（总代理），下面有 tool_handler（管理工具的）、管理 memory 的等，层次要清晰规定好。开始项目前就要精确地写好。否则 AI 自娱自乐、无法及时看懂，就会造屎山（代码混乱成一团）。

**模块职责建议写成表格：**

| 模块 | 负责 | 不负责 |
| --- | --- | --- |
| Channel Adapter | 飞书协议转换、签名校验 | Agent 推理 |
| Task Service | 创建任务、状态转换 | 直接调用模型 |
| Orchestrator | 调度 Agent 和工具 | 直接执行 SQL |
| Tool Registry | 注册与校验工具 | 决定产品目标 |
| LLM Provider | 封装厂商模型接口 | 保存业务状态 |

> Channel Adapter = 通道与 SDK 转换，负责把外部平台（飞书等）的协议转换成系统内部能处理的格式，并做签名校验。

**解决了什么问题：**
- Agent 知道新功能应该接入哪一层（不用乱猜）。
- 避免 Route、Service、Repository 职责混乱（分层清晰）。
- 避免重复创建 Provider、工具和状态存储（不重造轮子）。
- 让调用链、异常边界与数据流可追踪。

**局限：**
- 文档不能替代代码中的接口和类型约束（文档终究是文档）。
- 目录树不等于架构说明（光列文件夹没用，要说清职责）。
- 过早设计复杂分层会造成过度工程（别过度设计）。
- 架构发生变化却不更新文档，会误导后续 Agent（又是 Context Rot）。

## 3.6 CLAUDE.md：Agent 项目指令

**核心定义：** 提供给编码 Agent 的长期稳定项目说明和协作规则。它告诉 Agent 在这个仓库里应该怎样工作。

**主讲人核心观点：** 每一个项目都要有一份 CLAUDE.md。它告诉编码 Agent 在这个项目里应该怎么做。

**通常包括哪些内容：**
- 项目介绍（这是什么项目）。
- 必须阅读的文档（开工前要看哪几份）。
- 常用安装命令（怎么装依赖）。
- 目录说明（各个目录是干嘛的）。
- 禁止修改的地方（红线，别碰）。
- 可以读取需求、根据需求说明代码方案（开始写之前先讲思路）。
- 测试改动范围（改哪些要测哪些）。
- 修改后必须运行测试（改完必须跑测试）。
- 高风险操作必须确认（危险操作要先问人）。

**解决了什么问题：**
- 每个新会话不用重复解释基本规则（不用每次重讲）。
- Agent 知道项目命令、目录和验证方式。
- 统一修改范围、安全边界和代码风格（让所有 Agent 遵守同一套）。
- 引导 Agent 先读需求和代码，再开始修改（养成"先看懂再动手"的习惯）。

**怎么写：** 有推荐模板（CLAUDE.md 推荐模板）。

**局限（主讲人强调"非常重要"）：**
1. **它是指令和上下文，不是真正的权限系统** —— 这是第一点也是最重要的一点。
2. **不应该复制整份 PRD 或架构文档**（比如把整个 BACKEND_STRUCTURE 或 PRD 复制进来）。
3. **内容过长会占满每次会话的上下文** —— 所以别塞太大。
4. **临时任务进度放在这里会迅速过期** —— 别把"当前进度"塞 CLAUDE 里。
5. **不同工具可能使用不同的项目指令文件名，需要建立对应入口** —— 比如有的用 CLAUDE.md，有的用 AGENTS.md。

**关键提醒（点睛之笔）：** 因为 CLAUDE.md 是"软指令"，不是真正的 sandbox 或权限系统。所以不要把整份架构（如 BACKEND_STRUCTURE）或 PRD 复制到 CLAUDE.md 里。它能引导 Agent，但不能真正阻止 Agent 做危险操作。真正能挡住的，是文件系统权限、沙箱、审批这些硬机制。所以 CLAUDE.md 里只放指引（命令、目录、红线、流程、测试要求），别把大段的 PRD/架构/整个项目塞进去。

**它的定位：** 前面总览表里的 CLAUDE.md（长期指令，最关键的总纲）。直接规定"Agent 在这个仓库该怎么干活"。

---

## 3.7 这几节串起来（主讲人的逻辑顺序）

这套文档的顺序是有逻辑的：

1. PRD：有什么功能点（要什么）。
2. APP_FLOW：用户怎么走一遍（体验/流程）。
3. FRONTEND_GUIDELINE：前端长什么样（视觉/交互）。
4. BACKEND_STRUCTURE：后端怎么分层（边界/架构）—— 这是 Agent 最重要的边界。
5. CLAUDE.md：Agent 在这个仓库怎么干活（行为规范，总纲）。

**核心警告：** 架构边界一定要在开始项目前就精确写好。否则 AI 自娱自乐，没人看懂结构，AI 就会造屎山。这就是为什么 BACKEND_STRUCTURE 和 CLAUDE.md 这么重要。
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