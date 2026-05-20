# 项目面试追问清单

  

## 一、架构与设计思路

  

### 1. 你提到“把 LLM 作为意图规划器而非对话生成器”，为什么这样设计？如果直接让 LLM 端到端生成回复，会遇到哪些具体问题？

这样设计的核心原因是：这个项目本质上不是“陪聊机器人”，而是“带业务约束的电商客服系统”。  

在这种场景里，系统首先要做的是**判断该走哪条业务路径**，然后再由受控后端去执行，而不是让模型自己从理解到执行一路包办。

  

在这个项目里，LLM 主要负责两件事：

  

- 在 [turn_planner.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/plan/turn_planner.py) 里输出结构化 `TurnPlan`

- 在 `knowledge/chitchat/clarify/action_response` 这些模块里生成自然语言

  

也就是说，模型负责“想清楚往哪走”和“把话说自然”，不负责“直接决定系统做了什么”。

  

**为什么要把 LLM 作为意图规划器**

  

1. **业务执行要可控**

直接回复“我已经帮你提交退款申请了”和真正提交退款，是两回事。  

如果端到端让 LLM 自己决定回复，很容易出现“话术已经承诺，但后端并没执行”的问题。现在的设计是：

  

```text

LLM -> 产出 start_flow/refund_request

后端 -> 真正推进退款流程

Action -> 真正调业务接口

```

  

这样“说了什么”和“做了什么”是一致的。

  

2. **任务流程本身是状态机，不是单轮生成**

像订单查询、物流查询、退款申请都要经过：

  

- 收集槽位

- 校验参数

- 调外部接口

- 更新任务状态

- 可能中断和恢复

  

这些更像工作流，不像自由聊天。  

LLM 擅长语义理解，但不适合稳定维护这种细粒度状态机。

  

3. **需要强约束输出**

项目里通过 `TurnPlan` 把模型输出限制成固定结构，只允许：

  

- `task`

- `knowledge`

- `chitchat`

  

以及有限命令：

  

- `start_flow`

- `resume_flow`

- `cancel_flow`

- `set_slots`

  

这让后端可以校验、拒绝、澄清。  

如果端到端直接生成自然语言，后端几乎没法判断模型到底想执行什么。

  

4. **业务风险更低**

在客服场景里，错分流比“回复没那么自然”严重得多。  

比如：

  

- 把“查物流”理解成“查订单状态”

- 把“问退款政策”理解成“发起退款申请”

- 在没有订单对象时胡乱回答订单详情

  

现在的设计里，后端可以在 [validator.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/plan/validator.py) 做二次校验，缺对象就澄清，不会盲答。

  

**如果直接让 LLM 端到端生成回复，会遇到的具体问题**

  

1. **会出现“口头执行”**

模型可能回复：

  

> 已经为您提交退款申请

  

但实际上没有任何 `Action` 被调用，没有落库，也没有调业务 API。  

用户看到的是“系统承诺了”，后端却没做，线上很危险。

  

2. **难以稳定调用外部系统**

查订单、查物流、查商品信息都需要真实调用接口。  

如果让 LLM 端到端输出，它只能：

  

- 假装自己知道

- 或者输出一段“看起来像结果”的话

  

而不是可靠地经过 `action_lookup_order_status`、`action_lookup_logistics` 这些受控动作。

  

3. **多轮任务容易漂**

例如退款流程：

  

- 先收订单号

- 再收退款原因

- 再确认提交

  

端到端生成时，模型可能前后几轮自己改主意，或者忘记缺哪个槽位。  

现在的 flow 设计把这些状态放在 `DialogueState` 里，逻辑更稳。

  

4. **任务中断/恢复会很难做**

这个项目支持：

  

- 当前任务 `active_task`

- 暂停任务 `paused_tasks`

- 恢复任务 `resume_flow`

  

如果全部依赖 LLM 对话记忆，任务切换会很脆。  

模型可以“记得大概聊过什么”，但很难像状态机那样精确恢复到某个步骤。

  

5. **缺少可审计性**

现在后端可以知道本轮到底发生了什么：

  

- 模型选了哪个 `flow`

- 设置了哪些 `slots`

- 调了哪个 `action`

- 为什么进入澄清

  

如果端到端直接输出文本，日志里只剩一句回复，定位问题很难。

  

6. **测试和回归几乎没法做**

结构化规划可以测：

  

- “帮我查物流”是否产出 `start_flow(logistics_tracking)`

- “我想退款”是否进入 `refund_request`

  

端到端自然语言很难做稳定断言，因为文本变化很大，但业务路径才是关键。

  

7. **权限和边界不清**

像订单信息、退款、人工转接，本质上都是“业务动作”。  

这些动作应该由后端白名单能力执行，而不是让模型自由决定。  

否则模型很容易越过系统本来定义的边界。

  

**这不是不用 LLM，而是把它放在更合适的位置**

  

这个项目其实不是“削弱”LLM，而是把 LLM 放到它擅长的位置：

  

- 擅长的：语义理解、意图归类、自然语言表达

- 不擅长的：强状态一致性、业务执行保证、权限控制、可审计流程

  

所以更准确的说法是：

  

> LLM 负责理解和规划，后端负责校验和执行。

  

**面试里可以这样讲**

  

> 我们没有把大模型当成一个端到端聊天黑盒，而是把它设计成意图规划器。因为这个项目是电商客服系统，很多请求背后都对应真实业务动作，比如查订单、查物流、发起退款。  

> 如果直接让 LLM 端到端生成回复，会出现口头承诺和真实执行脱节、任务状态不稳定、外部接口调用不可控、缺少审计和测试抓手等问题。  

> 所以我们让模型先输出结构化 TurnPlan，再由后端状态机、任务流和 Action 系统去执行。这样既保留了 LLM 的理解能力，又保证了业务执行的可控性。

  

### 2. 项目的分层里有一层“应用服务层”，它的职责只是加载状态、调引擎、保存状态，看起来很薄。这一层存在的必要性是什么？能不能把它合并到 API 层或引擎层？

  

这一层看起来薄，但它不是多余层，它承担的是**应用边界**，不是业务算法本身。

  

在这个项目里，[dialogue_service.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/service/dialogue_service.py) 理论上的职责是：

  

- 根据 `sender_id` 加载 `DialogueState`

- 调用 [dialogue_engine.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/engine/dialogue_engine.py) 处理一轮消息

- 保存更新后的状态

- 把结果返回给 API 层

  

它“薄”是正常的，因为它处理的是**编排和边界**，不是细节业务规则。

  

**为什么这一层有必要**

  

1. **隔离传输协议和业务用例**

API 层关心的是 HTTP、`Request/Response`、依赖注入、状态码；引擎层关心的是对话状态机和任务分流。  

应用服务层把两边隔开，避免 HTTP 细节渗进引擎，也避免引擎直接依赖 Web 框架。

  

2. **把一次业务用例收口成事务边界**

“处理一条用户消息”本身就是一个完整用例：

  

```text

读状态 -> 执行业务 -> 写状态

```

  

这个边界放在 service 层最合理。以后如果要加：

  

- 异常兜底

- 重试

- 幂等

- 审计日志

- 埋点

- 事件发布

  

都应该先落在这一层，而不是散在 API 或引擎里。

  

3. **让引擎保持纯业务核心**

`DialogueEngine` 最理想的形态是：

  

```text

输入：state + user_message

输出：process_result

副作用：只修改内存中的 state

```

  

也就是“面向领域状态机”的纯业务对象。  

如果把加载数据库、保存数据库也塞进去，引擎就从业务核心变成了“业务 + 基础设施”的混合体，测试和复用都会变差。

  

4. **方便替换调用入口**

今天入口是 FastAPI，明天可能还有：

  

- WebSocket

- MQ 消费

- 定时补偿任务

- 内部批处理

- CLI 调试命令

  

这些入口都可以复用同一个 `DialogueService`。  

如果把逻辑合并到 API 层，就会把“处理消息”绑死在 HTTP 入口上。

  

**能不能合并到 API 层**

  

能，但不建议。

  

合并后 API 层会变成：

  

```text

解析请求

-> load_state

-> engine.process_message

-> save_state

-> 返回响应

```

  

短期看少一层，长期问题很直接：

  

- 路由层开始知道 repository 和领域状态

- HTTP 代码和业务用例耦合

- 以后换入口时要重复写一遍同样逻辑

- 日志、幂等、审计这类横切逻辑容易散落在多个路由里

  

换句话说，合并到 API 层可以跑，但架构会往“控制器越来越胖”方向走。

  

**能不能合并到引擎层**

  

也能，但问题更大。

  

如果引擎直接负责：

  

- 查数据库

- 存数据库

- 处理消息

  

那它就不再是单纯的对话引擎，而是“应用服务 + 引擎 + 仓储协调器”的混合体。这样会带来几个具体问题：

  

- 引擎依赖 repository / session，领域纯度下降

- 单元测试要么变重，要么需要大量 mock

- 引擎不能脱离数据库独立运行

- 后续如果想把状态存储从 MySQL 换到 Redis / 事件流，实现替换会更难

  

对这个项目来说，引擎最有价值的地方恰恰是它作为“可独立演化的对话状态机”。

  

**什么时候这层真的可以省**

  

如果项目非常小，满足以下条件，可以直接并到 API 层：

  

- 只有 1 个入口

- 没有 repository / 状态持久化

- 只是简单调用一个 handler

- 没有明显的事务边界和横切逻辑

  

但这个项目不是这种体量。它已经有：

  

- 对话状态持久化

- 引擎编排

- task / knowledge / chitchat / clarify 分流

- 后续明显会加日志、监控、失败处理

  

所以保留应用服务层是合理的。

  

**面试里可以这样回答**

  

> 这一层虽然薄，但它的价值不在于承载复杂业务，而在于定义“处理一条用户消息”这个应用用例的边界。它把 API 层的 HTTP 细节和引擎层的领域逻辑隔离开，同时统一承接读状态、调引擎、写状态这条事务链路。  

> 如果把它并到 API 层，控制器会逐渐变胖；如果并到引擎层，引擎就会混入数据库和基础设施依赖，失去领域核心的纯度。对这个项目来说，service 层的职责是薄但关键，它是编排层，不是冗余层。

  

### 3. `DialogueState` 同时维护了 `active_task`、`paused_tasks` 和 `active_system_task`，这三种 task 的优先级关系是怎样的？如果用户正在退款流程中被追问订单号，突然说“帮我查一下物流”，系统会怎么处理？

  

这三者的优先级关系，核心就一句话：

  

```text

active_system_task > active_task > paused_tasks

```

  

原因在 [src/domain/state.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/domain/state.py) 的 `current_task()`：

  

```python

def current_task(self):

    return self.active_system_task or self.active_task

```

  

以及 [src/task/flow/executor.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/task/flow/executor.py) 的执行入口：

  

```python

current_task = state.current_task()  # 系统任务优先

```

  

所以这三个状态的语义分别是：

  

- `active_task`

  当前正在办理的用户业务任务，比如退款、查物流、查订单。

- `paused_tasks`

  被中断后挂起的用户业务任务集合，本轮不会自动执行，只作为待恢复状态存在。

- `active_system_task`

  系统内部临时插入的控制任务，比如“开始某任务的提示”“任务被打断的提示”“收集槽位时的追问”。

  

**优先级怎么理解**

  

1. `active_system_task` 最高  

   只要它存在，执行器就先跑它。它像一个覆盖在用户任务之上的“系统前台提示流程”。

  

2. `active_task` 次之  

   当没有系统任务时，才继续推进当前用户业务流程。

  

3. `paused_tasks` 最低  

   它只是挂起态，不会被自动执行，只有明确触发 `resume_flow` 才会重新成为 `active_task`。

  

**你给的场景怎么处理**

  

场景是：

  

1. 用户正在 `refund_request` 退款流程中

2. 当前流程走到 `collect order_number`

3. 系统正在追问“请告诉我你的订单号”

4. 这时用户突然说：“帮我查一下物流”

  

这时系统会这么走：

  

**第一步：退款流程当前状态**

  

退款任务本身在 `active_task` 中，类似：

  

```text

active_task = refund_request

当前 step = ask_order_number（collect）

```

  

因为缺少 `order_number`，`FlowExecutor` 会启动一个系统收集流程：

  

```text

active_system_task = system_collect_information

```

  

所以这时真实优先执行的是 `active_system_task`，而不是直接执行退款 flow。

  

**第二步：用户说“帮我查一下物流”**

  

这是一条新的文本消息，进入 `DialogueEngine._handle_text_message()`。  

系统不会因为当前在追问订单号，就强制把这句话当成订单号填槽，而是先重新做一轮 `TurnPlanner` 规划。

  

如果 LLM 判断这是一个新任务请求，通常会产出：

  

```json

{

  "task": {

    "commands": [

      {"command": "start_flow", "flow": "logistics_tracking"}

    ]

  }

}

```

  

**第三步：启动新 flow 时如何处理旧任务**

  

进入 [src/task/command/processor.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/task/command/processor.py) 的 `_handle_start_flow()`：

  

- 先 `state.end_system_task()`  

  结束当前 `system_collect_information`

- 发现已有 `active_task = refund_request`

- 执行 `state.interrupt_active_task()`  

  把退款任务放进 `paused_tasks`

- 把 `logistics_tracking` 设置为新的 `active_task`

- 再启动一个新的 `active_system_task = system_task_interrupted`

  

这个系统 flow 的作用是先告诉用户：

  

```text

好的，我们先把退款申请放一放。

好的，我们先处理物流查询。

```

  

**第四步：后续执行**

  

由于 `active_system_task` 优先级最高，系统会先跑 `system_task_interrupted` 这段提示。  

提示结束后，`active_system_task` 清空，执行权回到新的：

  

```text

active_task = logistics_tracking

```

  

然后物流查询 flow 开始推进：

  

- 进入 `collect order_number`

- 如果没有订单号，继续追问订单号

- 如果用户提供订单号，再查物流

  

**最终状态会变成**

  

```text

active_task = logistics_tracking

paused_tasks = [refund_request]

active_system_task = None 或 system_task_interrupted 执行中

```

  

也就是说，退款流程不会丢，而是被挂起；物流查询成为当前主任务。

  

**这个设计的实际含义**

  

它体现的是“系统任务只是临时控制层，真正的业务中断和切换发生在用户任务层”。

  

所以在这个项目里：

  

- `active_system_task` 决定“此刻先说什么、先问什么”

- `active_task` 决定“当前主业务在办什么”

- `paused_tasks` 决定“之后还能恢复什么”

  

**面试表达可以这样说**

  

> `DialogueState` 里三种 task 的优先级是 `active_system_task > active_task > paused_tasks`。系统任务优先是因为很多时候需要先插入一段系统提示或收集动作，比如追问订单号、提示任务已中断。`active_task` 才是真正的当前业务任务，`paused_tasks` 只是挂起态，不会自动执行。  

>  

> 如果用户正在退款流程里被追问订单号，这时突然说“帮我查一下物流”，系统会重新做一轮意图规划。如果识别为新的物流查询任务，就会先结束当前系统追问 flow，把退款任务从 `active_task` 挂到 `paused_tasks`，再把物流查询设为新的 `active_task`，同时启动一个 `system_task_interrupted` 系统 flow 给用户提示“我们先把退款放一放，先处理物流查询”。这套机制本质上是在做多任务对话状态管理。

  
  

---

  

## 二、LLM 与 TurnPlan

  

### 4. LLM 生成 `TurnPlan` 后，你提到有一个 `TurnPlanValidator` 做校验。具体校验什么？如果校验不通过，系统怎么兜底？

  
  

`TurnPlanValidator` 的作用是把 LLM 的输出从“像是合理”再收紧成“可以安全执行”。  

也就是说，LLM 先负责理解，`TurnPlanValidator` 负责把关，避免模型输出一份语义上像对、执行上却有风险的计划。

  

当前校验逻辑在 [src/plan/validator.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/plan/validator.py)。

  

**它具体校验什么**

  

当前实现主要做了 3 类校验。

  

1. **轨道数量校验**

`TurnPlan` 顶层只允许三条轨道：

  

- `task`

- `knowledge`

- `chitchat`

  

校验器会先统计哪些轨道被命中了。

  

规则是：

  

- 一个都没命中：不通过，`MISSING_TRACK`

- 同时命中多个：不通过，`MULTIPLE_TRACKS`

- 只命中一个：继续细查

  

这一步是在防止模型输出“既像任务、又像知识、还像闲聊”的混合结果，系统不会直接猜。

  

2. **任务轨道校验**

如果命中的是 `task`，当前会检查：

  

- `task` 不能为空

- `task.commands` 不能为空

  

如果模型说这是任务，但没有给出任何可执行命令，比如没产出 `start_flow`、`set_slots`、`resume_flow` 之类，就会判定失败，原因是：

  

- `MISSING_TASK_COMMANDS`

  

这一步是在防止“知道是办业务，但不知道具体怎么办”。

  

3. **知识轨道校验**

如果命中的是 `knowledge`，当前会检查：

  

- `knowledge.intents` 不能为空

- 每个 intent 是否满足对象依赖

  

这里最关键的是“对象依赖”。

  

例如在 [src/knowledge/intents.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/knowledge/intents.py) 里：

  

- `product_info` 需要 `focused_object.type == product`

- `order_info` 需要 `focused_object.type == order`

  

所以如果用户问“这个商品是什么材质”，但当前并没有商品对象上下文，虽然 LLM 可能输出了 `product_info`，校验器还是会拦下来，返回：

  

- `MISSING_FOCUSED_OBJECT`

  

这一步是在防止模型没有足够上下文却硬答。

  

**当前已经定义的校验失败类型**

  

在 [src/plan/models.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/plan/models.py) 里，当前主要有这些失败原因：

  

- `MISSING_TRACK`

- `MULTIPLE_TRACKS`

- `MISSING_TASK_COMMANDS`

- `MISSING_KNOWLEDGE_INTENT`

- `MISSING_FOCUSED_OBJECT`

- `OBJECT_REQUIRES_INTENT`

  

其中前 5 个是 `TurnPlanValidator` 直接返回的，`OBJECT_REQUIRES_INTENT` 是对象消息分支里的另一类兜底场景。

  

**如果校验不通过，系统怎么兜底**

  

不会执行错误计划，也不会静默失败，而是进入澄清流程。

  

在 [src/engine/dialogue_engine.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/engine/dialogue_engine.py) 里：

  

```python

validation = self.turn_plan_validator.validate(...)

if not validation.valid:

    return await self.clarify_responder.respond(

        state=state,

        reason=validation.reason

    )

```

  

也就是说，校验失败后系统不再进入 `task` 或 `knowledge`，而是直接调用 `ClarifyResponder`。

  

**澄清流程怎么工作**

  

[clarify/responder.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/clarify/responder.py) 先根据失败原因生成一个基础澄清文案，再结合历史对话、用户当前消息、聚焦对象，用 LLM 改写成更自然的话术。

  

例如：

  

- `MISSING_TRACK`

  -> “你是想先处理业务问题，还是先咨询信息呢？”

- `MULTIPLE_TRACKS`

  -> “你这次同时提到了多个方向。我们先处理一个，您是想先处理哪一个任务呢？”

- `MISSING_TASK_COMMANDS`

  -> “你这次是想办理什么业务呢？比如查订单、查物流，或者申请退款。”

- `MISSING_KNOWLEDGE_INTENT`

  -> “你是想了解商品信息、订单信息，还是售后配送规则呢？”

- `MISSING_FOCUSED_OBJECT`

  -> “请先发送你想咨询的对象，我再继续帮你看。”

  

所以它的兜底策略不是报错，而是**转成一次受控追问**。

  

**为什么这层校验重要**

  

如果没有 `TurnPlanValidator`，直接执行 LLM 输出，具体风险会是：

  

- 一条消息同时被当成任务和知识来执行

- 模型说是任务，但没给出可执行命令

- 模型说查商品信息，但当前根本没有商品对象

- 后端为了“继续跑”只能自己猜，结果更不稳定

  

所以这层校验本质上是在把：

  

```text

模型理解结果

```

  

变成：

  

```text

可执行、可拒绝、可澄清的系统决策

```

  

**面试里可以这样回答**

  

> LLM 生成 TurnPlan 之后，我们不会直接执行，而是先经过 TurnPlanValidator。当前主要校验三类内容：第一，轨道是否合法，只能明确命中 task、knowledge、chitchat 中的一条；第二，task 轨道是否带了可执行 commands；第三，knowledge 轨道是否带了 intent，以及这个 intent 是否满足上下文依赖，比如查询商品信息必须先有商品对象。  

>  

> 如果校验不通过，系统不会冒险执行，而是进入 ClarifyResponder，根据失败原因生成澄清追问，让用户补充意图、对象或范围。这样做的目的是把 LLM 的输出从“看起来合理”进一步收敛成“可以安全执行”的计划。

  

### 5. 你让 LLM 做意图识别和规划，如果 LLM 返回了一个“看起来合理但实际上是错误的”规划，比如它把闲聊判断成了查订单，系统有没有办法发现并纠正？

  

有，但要分两层说：

  

**当前系统能发现的，是“结构性错误”和“上下文不成立的错误”；对“语义上看起来合理但其实分错了”的情况，只能部分纠正，不能完全自动识别。**

  

**现在这套代码里，能拦住的几类错误**

  

1. `TurnPlan` 结构不合法  

   例如同时命中多个轨道、没有命中任何轨道、`task` 没有 commands、`knowledge` 没有 intents。  

   这类会被 [validator.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/plan/validator.py) 拦下来，然后走澄清。

  

2. 上下文依赖不成立  

   例如模型判断成 `product_info`，但当前没有商品对象；判断成 `order_info`，但没有订单对象。  

   这类也会被拦下来，然后由 [clarify/responder.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/clarify/responder.py) 追问用户。

  

3. 任务执行过程中暴露出“不像这个任务”  

   比如模型把一句闲聊误判成“查订单”，系统会进入 `collect order_number`，回复“请告诉我你的订单号”。  

   如果用户接着说“我不是查订单，我只是随便问问”，下一轮规划就可能被重新纠正到 `chitchat` 或澄清轨道。

  

**但当前系统拦不住的一类问题**

  

如果 LLM 输出的是：

  

- 结构合法

- 上下文也成立

- 后端校验全部通过

  

但业务语义其实错了，比如把“随便聊聊”误判成 `order_status_query`，那**当前代码没有一个二次语义裁判**能直接判它错。

  

也就是说，当前系统更擅长防这种错误：

  

- “不能执行”

- “缺少对象”

- “计划不完整”

  

不太擅长自动识别这种错误：

  

- “计划完整，但方向选错了”

  

**它现在的纠正机制，本质上是交互式纠正**

  

不是系统自己百分百识别错分流，而是通过后续流程把问题暴露出来，再让用户把方向拉回来。

  

典型链路是：

  

```text

LLM 错判为 task

-> 后端开始该 flow

-> collect / action 暴露出和用户预期不一致

-> 用户否认或改口

-> 下一轮重新规划

-> 进入 chitchat / clarify / 正确 task

```

  

所以这套设计的重点是：

  

- 尽量不让错判直接造成危险执行

- 即使错判，也优先进入“可回退”的收集和澄清环节

- 通过多轮交互修正，而不是一次性拍板到底

  

**如果要更强地发现并纠正这种“看起来合理但其实错”的规划，通常会加几层东西**

  

1. 二次校验模型  

   让第二个 prompt 专门判断：`TurnPlan` 是否真的和用户最后一句一致。

  

2. 低风险确认  

   对高风险 flow 先确认一句：  

   “你是想查订单状态，还是只是随便问问？”

  

3. 规则对冲  

   对明显闲聊语气、寒暄词、无业务实体的输入，给 `chitchat` 更高先验权重。

  

4. 执行前置信度门槛  

   如果模型对任务规划置信度不高，就先走澄清，而不是直接 `start_flow`。

  

5. 用户拒绝信号回流  

   如果用户频繁说“不是这个意思”，就把该轮标记为 `clarification_rejected`，强制重规划。

  

**面试里可以这么答**

  

> 当前系统能很好地拦住结构性错误和上下文不成立的错误，比如没有对象却想查订单详情、命中了多个轨道、task 没有可执行命令等，这些都会被 TurnPlanValidator 拦下并转成澄清追问。  

> 但如果 LLM 给出的是一份结构合法、上下文也成立、只是语义上分错的规划，比如把闲聊误判成查订单，当前系统没有完全自动识别这种错误的能力。它的主要纠正方式是交互式纠偏，也就是先进入一个低风险的 collect 或提示环节，让用户下一轮把方向拉回来。  

> 所以这套设计的重点不是假设 LLM 永远不会错，而是即使错了，也尽量让错误暴露在可澄清、可回退的阶段，而不是直接进入不可控执行。

  
  

### 6. 四个轨道（`task` / `knowledge` / `chitchat` / `clarify`）中，`clarify` 是兜底。如果 LLM 频繁产出需要 `clarify` 的结果，比如每次都判断为意图不明确，你会怎么优化？这是 prompt 问题、模型能力问题，还是 flow 设计问题？

  

这类问题不能只归因到一个点，通常是 **`prompt + 模型能力 + 候选 flow/intent 设计 + 上下文供给`** 共同造成的。  

如果 `clarify` 频繁触发，我不会先拍脑袋说“模型不行”，而是先把问题拆成“模型为什么不敢选”或者“模型为什么选了也过不了校验”。

  

先给结论：

  

> 大多数情况下，频繁进入 `clarify`，第一嫌疑是 `turn_plan` prompt 和候选定义不够清晰；第二嫌疑是 flow / intent 颗粒度和边界设计不合理；第三才是模型能力本身。

  

**我会先区分 clarify 是在哪一层触发的**

  

这个项目里 `clarify` 不是单一来源，至少有两类：

  

1. `TurnPlanner` 本身没有给出明确可执行结果  

   例如输出空轨道、多轨道、task 没 commands、knowledge 没 intents。

  

2. `TurnPlanValidator` 把结果打回来了  

   例如缺少 `focused_object`，或者知识意图依赖上下文但当前上下文不满足。

  

这两类的优化方向不同。  

如果是第一类，更像是 `prompt / 模型` 问题；如果是第二类，更像是 `flow / intent / 上下文设计` 问题。

  

**如果是 prompt 问题，我会怎么改**

  

当前 [turn_plan.jinja2](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/prompts/jinja2/turn_plan.jinja2) 已经在做结构化约束，但还可以继续收紧：

  

1. **增加更强的正反例**

不要只告诉模型“可用 flow 有哪些”，还要给它示例：

  

- “帮我查订单状态” -> `order_status_query`

- “物流到哪了” -> `logistics_tracking`

- “这个商品是什么材质” -> `knowledge.product_info`

- “你好呀” -> `chitchat`

  

模型在这种任务里，few-shot 例子往往比抽象规则更有效。

  

2. **明确默认策略**

如果用户语义偏业务，但缺少参数，不应该直接 `clarify`，而应该优先选一个 task flow，再由 `collect` 去收参数。  

比如“帮我查物流”本来就不需要在规划层澄清，而应该直接进 `logistics_tracking`，后面收订单号。

  

3. **减少 prompt 里的歧义空间**

如果 prompt 同时强调“真实意图”“可多轨输出”“后续系统会澄清”，模型会更倾向保守。  

要是业务目标是减少 clarify，可以把策略改成：

  

```text

优先选最可能的一条轨道；

只有在确实无法区分时才输出导致澄清的结果。

```

  

很多时候 clarify 过多，是 prompt 把模型训练得过度谨慎了。

  

**如果是模型能力问题，我会怎么处理**

  

模型问题通常表现为：

  

- 对短句、口语、模糊表达泛化差

- 对 task 和 knowledge 边界判断不稳定

- 同一句话多次结果波动大

  

这种情况我会做几件事：

  

1. **先看是否真需要更强模型**

如果当前模型在中文客服意图分类上稳定性差，直接升级模型通常比堆复杂规则更划算。

  

2. **把规划任务变简单**

让模型只做轨道选择和 flow/intent 选择，不要在同一步里做太多推理。  

必要时拆成两步：

  

- 第一步：选 `task / knowledge / chitchat`

- 第二步：如果是 task，再选具体 flow

  

3. **加入规则前置分类**

像“你好”“谢谢”“在吗”这种高频闲聊，没必要次次进 LLM。  

可以先用轻规则或词典把明显的 chitchat 截走，减少模型在边界样本上的压力。

  

**如果是 flow / intent 设计问题，我会怎么改**

  

这类问题其实很常见，而且经常被误判成“模型不行”。

  

1. **flow 粒度过细，模型不敢选**

如果多个 flow 描述很接近，比如：

  

- `order_status_query`

- `order_info`

- `logistics_tracking`

  

模型会犹豫，因为“查订单”和“查物流”在自然语言里经常混着说。  

这时应该考虑：

  

- 合并上层 flow，后续用步骤再细分

- 或者强化 flow description，让边界更可区分

  

2. **knowledge intent 和 task flow 边界不清**

比如“订单信息咨询”和“订单状态查询”本质上就很近。  

如果定义不清，模型频繁多轨或保守澄清是正常的。  

这时应该重新定义：

  

- task：需要办理或推进流程

- knowledge：只做解释或查询，不推进业务状态

  

3. **缺少过渡型 flow**

很多 clarify 其实不是用户真不明确，而是系统没有“接得住”的中间态。  

例如用户说“我想退这个”，系统如果只有“退款申请”和“缺对象澄清”，就容易卡住。  

这时可以设计一个更宽的 flow 或 system flow，先承接意图，再逐步收上下文。

  

**如果是上下文供给问题，我会怎么改**

  

`TurnPlanner` 依赖：

  

- 历史对话

- active task

- interrupted tasks

- focused object

- available flows

- knowledge intents

  

如果这些上下文喂得不完整，模型就容易保守。

  

我会重点检查：

  

1. `focused_object` 是否稳定维护  

   很多本来该走 knowledge 的请求，因为没对象上下文，被 validator 打回 clarify。

  

2. 历史对话是否太长或太噪  

   历史太多会稀释当前意图，模型反而更不敢选。

  

3. flow 描述是否真的可读  

   如果传给模型的是一堆结构字段而不是高质量业务描述，它不容易稳定匹配。

  

**真正落地时，我会怎么排查**

  

我不会直接改 prompt，而是先做一个小型错误分析：

  

1. 抽样最近 100 条进入 `clarify` 的请求

2. 按原因分桶：

   - `MISSING_TRACK`

   - `MULTIPLE_TRACKS`

   - `MISSING_TASK_COMMANDS`

   - `MISSING_KNOWLEDGE_INTENT`

   - `MISSING_FOCUSED_OBJECT`

3. 看每类占比

4. 再决定改哪一层

  

例如：

  

- `MISSING_FOCUSED_OBJECT` 多  

  说明对象上下文设计有问题，不是模型主因。

- `MISSING_TRACK` 多  

  先改 prompt 和示例。

- `MULTIPLE_TRACKS` 多  

  说明 flow / intent 边界重叠。

- `MISSING_TASK_COMMANDS` 多  

  说明模型没学会输出规范 JSON，可能是 prompt 或模型能力问题。

  

**我会优先做的优化顺序**

  

1. 改 `turn_plan` prompt：补示例，明确默认策略，降低过度保守。

2. 重新审视 flow / intent 边界：合并过细类别，强化 description。

3. 完善上下文供给：特别是 `focused_object` 和 active task。

4. 对高频闲聊和固定表达加规则前置。

5. 如果效果仍差，再升级模型或拆分规划步骤。

  

**面试里可以这样答**

  

> 如果 clarify 频繁触发，我不会简单归因为模型能力差，而是先区分问题发生在规划层还是校验层。大多数情况下，第一嫌疑是 prompt 和候选 flow/intent 定义不够清晰，第二是 flow 粒度和边界设计不合理，第三才是模型本身。  

>  

> 我会先做 clarify 样本分桶，看是 `MISSING_TRACK` 多、`MULTIPLE_TRACKS` 多，还是 `MISSING_FOCUSED_OBJECT` 多。前两类更偏 prompt 和 flow 边界问题，后者更偏上下文供给问题。优化上，我会先补 few-shot 示例、明确默认路由策略、收紧 flow/intents 的边界，再看是否需要引入规则前置或升级模型。核心目标不是一味减少 clarify，而是让 clarify 只出现在真正需要用户补充信息的时候，而不是因为系统本身设计得太保守。

  

---

  

## 三、任务流程引擎

  

### 7. 任务流程是通过 YAML 配置的，`collect -> action -> end` 的节点模型看起来像一个简单的状态机。如果一个 flow 需要在 collect 失败时重试三次，或者超时自动取消，你的节点模型能支持吗？如果不能，会怎么改？

  
  

当前这套节点模型，**能支持一部分“collect 校验失败后的分支跳转”**，但**不能完整支持“重试三次”**，也**不能原生支持“超时自动取消”**。

  

先拆开说。

  

**当前能支持什么**

  

从 [src/task/flow/steps.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/task/flow/steps.py) 和 [src/task/flow/executor.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/task/flow/executor.py) 看，`collect` 节点已经有两种基础能力：

  

- 收集槽位

- 校验失败后返回一段 `failure_response`

  

也就是说，像“用户输入了一个格式不对的订单号，提示他重新输入”这种单次失败反馈，模型是能做的。

  

但它的限制也很明显：

  

- 没有“失败次数计数器”

- 没有“达到上限后走另一条分支”的内建机制

- 没有“等待超时”的时间语义

- 没有后台调度器或定时任务去触发取消

  

所以严格说，**当前模型是一个轻量流程状态机，不是完整工作流引擎**。

  

**为什么现在不支持“重试三次”**

  

因为 `collect` 节点当前只知道：

  

- 这个槽位有没有值

- 如果有值，校验是否通过

- 如果不通过，返回什么提示

  

但它不知道：

  

- 这是第几次失败

- 前两次和第三次失败要不要不同处理

- 超过次数后是取消任务、转人工，还是换别的 flow

  

要支持“最多重试三次”，至少要让 `DialogueState` 或 `TaskContext` 里保存这类运行时元数据，比如：

  

```python

retry_counters = {

  "order_number": 2

}

```

  

然后 `FlowExecutor` 在 collect 校验失败时：

  

1. 递增该槽位失败次数

2. 判断是否超过阈值

3. 未超过则继续回到 collect

4. 超过则跳到 `cancel` / `handoff` / `clarify` 分支

  

**为什么现在不支持“超时自动取消”**

  

因为当前流程推进是“消息驱动”的，不是“时间驱动”的。

  

也就是说，现在只有用户发消息时，系统才会：

  

- 加载状态

- 跑引擎

- 推进 flow

  

系统本身没有一个后台机制去检查：

  

- 这个任务已经卡了 10 分钟

- 这个 collect 已经等待太久

- 应该自动取消或关闭

  

所以“超时自动取消”不是加一个 YAML 字段就够了，它需要额外的基础设施：

  

- 在状态里记录等待开始时间

- 有定时扫描器 / scheduler / worker

- 能找到超时任务并触发状态迁移

- 最好还能发通知或写审计日志

  

**如果要改，我会怎么设计**

  

我不会推翻现有模型，而是做增量扩展。

  

**1. 扩展 `collect` 节点定义**

  

给 YAML 增加重试和超时配置，比如：

  

```yaml

- id: ask_order_number

  type: collect

  slot_name: order_number

  response:

    text: "请告诉我你的订单号。"

  validation:

    condition: "..."

    failure_response:

      text: "订单号格式不对，请重新输入。"

  retry:

    max_attempts: 3

    exceeded_next: cancel_or_handoff

  timeout:

    seconds: 600

    on_timeout: cancel_or_handoff

  next: lookup_order_status

```

  

这样 flow 设计者可以声明策略，而不是把策略硬编码在 Python 里。

  

**2. 扩展运行时状态**

  

在 `TaskContext` 或 `DialogueState` 里增加流程运行元数据，例如：

  

- `slot_attempts`

- `waiting_since`

- `deadline_at`

  

例如：

  

```python

runtime = {

  "slot_attempts": {"order_number": 2},

  "waiting_since": {"order_number": 1710000000},

  "deadline_at": 1710000600

}

```

  

这样引擎才能知道“当前已经失败几次”“从什么时候开始等”。

  

**3. 扩展 `FlowExecutor`**

  

让 `FlowExecutor` 在 `collect` 校验失败时不只是返回 `failure_response`，还要：

  

- 累加失败次数

- 判断是否超过 `max_attempts`

- 超过后跳转到 `exceeded_next`

  

这部分仍然是状态机逻辑，适合留在执行器里。

  

**4. 引入时间驱动的超时处理器**

  

超时不能只靠 `FlowExecutor`，因为用户不发消息时它根本不会运行。  

所以需要新增一层，例如：

  

- 定时任务扫描数据库中的 `DialogueState`

- 找出超时的 `active_task`

- 将其切换到取消状态，或者注入一个 `cancel_flow` / `timeout_flow`

  

这部分更像应用层或调度层能力，不应该硬塞进当前对话引擎同步链路里。

  

**5. 保持节点模型简单，但支持钩子化能力**

  

我不会把 YAML 做成完整 BPMN，那会过重。  

更实际的方向是：保留 `start / collect / action / end` 这套轻量模型，再给 `collect` 增加少量高价值控制项：

  

- `retry.max_attempts`

- `retry.exceeded_next`

- `timeout.seconds`

- `timeout.on_timeout`

  

这样复杂度还在可控范围内。

  

**面试里可以这样答**

  

> 当前这套 YAML flow 本质上是一个轻量状态机，适合表达 collect、action、end 这种消息驱动流程。它可以支持简单的 collect 校验失败提示，但不原生支持“失败三次后切换分支”，也不支持“超时自动取消”，因为这两件事都需要额外的运行时状态和时间语义。  

>  

> 如果要扩展，我会保留现有节点模型，只对 `collect` 节点增加 `retry` 和 `timeout` 配置，同时在 `TaskContext` 里补充失败计数和等待时间等元数据。对于重试上限，放在 `FlowExecutor` 里做状态迁移；对于超时自动取消，则需要引入独立的调度器或后台扫描任务，因为它不是消息驱动能解决的问题。这样可以在不推翻现有模型的前提下，把它从“简单状态机”升级成“可控的轻量工作流引擎”。

  
  

### 8. 任务暂停和恢复（interrupt / resume）的数据一致性怎么保证？比如退款任务被暂停时 slots 里已经收集了两个字段，恢复后会不会丢失？暂停期间用户改了 slots 怎么办？

  

当前这套设计里，**暂停/恢复的一致性主要靠 `TaskContext` 整体保存 + `DialogueState` 整体持久化**，所以像“退款任务暂停时已经收集了两个 slots，恢复后会不会丢”这种情况，按设计是**不会丢**的。

  

**它为什么不会丢**

  

暂停时不是只记一个 `flow_id`，而是把整个 `active_task` 放进 `paused_tasks`。  

`TaskContext` 里本身就带着：

  

- `flow_id`

- `step_id`

- `slots`

  

定义在 [state.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/domain/state.py) 和 [contexts.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/domain/contexts.py)。

  

暂停逻辑在 [state.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/domain/state.py)：

  

```python

def interrupt_active_task(self):

    self.paused_tasks.append(self.active_task)

    self.active_task = None

```

  

恢复逻辑也是把原来的 `TaskContext` 直接拿回来：

  

```python

def resume_task(self, flow_id: str):

    for task in self.paused_tasks:

        if task.flow_id == flow_id:

            self.active_task = task

            self.paused_tasks.remove(task)

            break

```

  

所以恢复后回到的不是“同一个 flow 的开头”，而是**暂停前那个 step 和 slots 快照**。  

如果退款任务在暂停前已经收集了：

  

```text

order_number

refund_reason

```

  

恢复后这两个字段还在，因为它们就保存在那个被挂起的 `TaskContext.slots` 里。

  

**为什么跨请求也不会丢**

  

因为每轮消息结束后，整个 `DialogueState` 会被序列化存到数据库 `dialogue_states.state_json`，见 [dialogue_state_repository.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/repository/dialogue_state_repository.py)。

  

也就是说，暂停任务不是只挂在内存里，而是跟着整份状态一起落库：

  

```text

active_task

paused_tasks

active_system_task

focused_object

sessions

```

  

所以只要保存成功，恢复时会从数据库把同一份 `paused_tasks` 读回来。

  

**暂停期间用户改 slots 怎么办**

  

这里要分两种情况。

  

**第一种：改的是当前正在执行的新任务的 slots**  

这不会影响被暂停的旧任务。  

因为当前 `set_slots` 只会更新 `active_task`，见 [processor.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/task/command/processor.py)：

  

```python

def _handle_set_slots(self, command: SetSlotsCommand, state: DialogueState):

    if state.active_task:

        state.set_slots(command.slots)

```

  

而 `state.set_slots()` 也是直接写当前 `active_task.slots`。  

所以如果退款任务被挂到 `paused_tasks`，用户去做物流查询时填的 `order_number`，默认是写到新的物流任务，不会反向污染退款任务。

  

**第二种：用户明确想修改被暂停任务的数据**  

当前模型**没有显式支持“修改指定 paused task 的 slots”**。  

也就是说，系统现在支持：

  

- 暂停

- 恢复

- 恢复后继续在原任务上改 slots

  

但不支持这种更细的操作：

  

```text

退款任务暂停中

用户说：把刚才退款原因改成质量问题

系统直接修改 paused_tasks 里的那个退款任务

```

  

因为当前 `set_slots` 没有“目标 task id”这个维度，只能写当前 `active_task`。

  

所以如果用户在暂停期间想改旧任务字段，合理做法应该是：

  

1. 先 `resume_flow(refund_request)`

2. 把退款任务恢复成 `active_task`

3. 再通过 `set_slots` 或重新 collect 去更新字段

  

这时数据是一致的，因为修改发生在恢复后的主任务上。

  

**这套一致性保证的边界**

  

这套方案在“单用户、单线程、串行处理消息”的前提下是成立的，但它也有边界。

  

1. **优点**

- 暂停时保存的是完整任务上下文，不只是任务名

- 恢复时直接拿回原 step 和 slots

- 每轮处理后整份状态落库，跨请求能续上

  

2. **局限**

- 没有版本号或乐观锁

- 没有细粒度地修改 paused task 的能力

- 如果同一用户并发发两条消息，当前实现可能有“后写覆盖前写”的风险

  

最后这一点很关键：  

当前仓储层是“读整份 state -> 改 -> 写整份 state”的模式。如果同一 `sender_id` 的两条请求并发进入，没有额外并发控制，就可能出现最后一次保存覆盖前一次结果的问题。这个属于更高层的数据一致性问题，不是 `interrupt/resume` 逻辑本身能单独解决的。

  

**面试里可以这样答**

  

> 任务暂停恢复的一致性，主要靠 `TaskContext` 整体保存来保证。暂停时不是只记一个 flow_id，而是把包含 `flow_id`、`step_id` 和 `slots` 的完整任务上下文放进 `paused_tasks`；恢复时再原样取回，所以已经收集好的字段不会丢。并且每轮消息处理后，整个 `DialogueState` 会序列化落库，所以跨请求也能恢复到暂停前状态。  

>  

> 当前实现下，`set_slots` 只会修改当前 `active_task`，不会直接改 `paused_tasks` 里的任务，因此暂停期间处理新任务不会污染旧任务。但它也意味着系统暂时不支持“在任务暂停状态下直接修改某个 paused task 的 slots”，如果要改，应该先恢复该任务再更新。更严格的一致性，比如并发请求下避免最后写覆盖前写，还需要在仓储层增加版本控制或串行化处理。

  
  

### 9. YAML 里写死了回复文案，比如“请告诉我你的订单号”。你怎么处理多语言、多文案 A/B 测试，或者运营想随时调整话术的场景？

  

当前这套写法，`YAML` 里直接写死文案，优点是简单直观，但它更适合原型期，不适合多语言、A/B 测试和运营高频调话术。  

所以如果面试里被问到，我会先明确说：**现有方案能跑，但扩展性一般，后续要把“流程定义”和“文案管理”拆开。**

  

**先说当前方案的问题**

  

把 `"请告诉我你的订单号"` 直接写在 flow 里，会带来几个具体限制：

  

- 多语言会复制整份 flow，配置膨胀

- A/B 测试很难做，除非在 YAML 里硬分叉

- 运营改话术要改配置甚至发版

- 文案和流程逻辑耦合，维护成本高

- 很难统一管理版本、灰度、回滚

  

所以我的做法不会是继续把文案堆在 YAML 里，而是把它改成“**文案 key + 渲染参数**”模式。

  

**我会怎么改**

  

第一步，把 YAML 里的文本从“字面量”改成“引用”。

  

比如从这样：

  

```yaml

response:

  text: "请告诉我你的订单号。"

```

  

改成这样：

  

```yaml

response:

  message_key: "order.ask_order_number"

```

  

或者：

  

```yaml

args:

  template_key: "order.status.result"

```

  

这样 flow 只表达“这里要说什么类型的话”，不直接保存最终文案。

  

第二步，引入独立的话术中心。  

可以是数据库表、配置中心，或者至少是独立的文案文件。典型结构是：

  

- `message_key`

- `locale`

- `variant`

- `content`

- `status`

- `version`

  

例如：

  

| message_key | locale | variant | content |

|---|---|---|---|

| `order.ask_order_number` | `zh-CN` | `A` | 请告诉我你的订单号。 |

| `order.ask_order_number` | `zh-CN` | `B` | 麻烦提供一下订单号，我来帮你查。 |

| `order.ask_order_number` | `en-US` | `A` | Please share your order number. |

  

第三步，在 `action_response` 或响应渲染层统一解析。  

也就是 flow 执行到这里时：

  

```text

message_key + locale + experiment bucket + context

-> 文案服务查出具体内容

-> 再做模板渲染

-> 返回给用户

```

  

这样多语言、A/B、运营改文案都集中在一层解决。

  

**多语言怎么处理**

  

多语言不应该复制 flow，而应该让 flow 保持不变，只切换文案资源。

  

核心思路是：

  

```text

同一个 flow

-> 根据用户 locale 取不同语言文案

```

  

比如：

  

- `zh-CN` -> “请告诉我你的订单号。”

- `en-US` -> “Please share your order number.”

  

如果有对象属性、订单号、商品名这些变量，再走模板渲染：

  

```text

"订单{{ slots.order_number }}当前状态是：{{ slots.order_status }}"

```

  

但模板本身也应该按语言存储，而不是写死在 YAML 里。

  

**A/B 测试怎么处理**

  

A/B 不应该通过写两套 flow 来做，那样会把流程逻辑和实验策略绑死。

  

更合理的是：

  

- 用户进入系统后根据 `sender_id` 或实验平台分桶

- 同一个 `message_key` 根据 bucket 返回 A 或 B 文案

- flow 完全不变

  

例如：

  

```text

order.ask_order_number

-> bucket A: 请告诉我你的订单号。

-> bucket B: 麻烦提供一下订单号，我帮你查物流进度。

```

  

这样你可以只测“话术效果”，不动业务流程。

  

**运营随时改话术怎么处理**

  

如果运营要频繁改，文案最好不要跟代码部署强耦合。  

更现实的做法是：

  

1. flow 配置中只放 `message_key`

2. 文案存在数据库或配置平台

3. 文案服务支持后台修改、生效、回滚

4. 应用侧做短时缓存

  

这样运营改文案不需要改 Python 代码，也不需要改任务 flow 结构。

  

**哪些文案还可以继续放 YAML**

  

不是所有文本都必须抽出去。  

我会这样分：

  

适合留在 YAML 的：

  

- 很稳定的内部流程描述

- 调试期临时文案

- 不面向最终用户的说明字段

  

应该抽到文案中心的：

  

- 面向用户展示的话术

- 高频调整的话术

- 多语言文案

- 参与实验的话术

- 需要统一运营管理的话术

  

**面试里可以这样回答**

  

> 当前项目里 YAML 直接写文案，适合原型阶段，因为配置直观、开发快。但如果进入多语言、A/B 测试和运营高频改话术场景，这种做法扩展性不够。我会把 flow 定义和文案资源拆开，让 YAML 里只保留 `message_key` 或 `template_key`，真正的话术放到独立的文案中心，根据用户语言、实验分桶和上下文动态取回，再做模板渲染。这样 flow 仍然负责业务状态机，文案系统负责多语言、灰度和运营管理，两者职责会更清晰。  

  

如果你需要，我可以把这个问题整理成一版更像面试现场口语回答的短答案，适合 30 秒直接说。

  

---

  

## 四、状态持久化

  

### 10. 你选择了 JSON 快照式持久化，每次处理消息时整体序列化、整体写回数据库。当 `DialogueState` 变大，比如历史 turns 很多，每次读写整个 JSON 的性能会怎么样？你会怎么优化？

  

这种方案在项目早期是合理的，但当 `DialogueState` 变大之后，性能和可维护性问题会比较明显。  

核心问题不是“JSON 不能用”，而是**整份状态按快照读写的成本会随着历史对话线性增长**。

  

**性能会怎么变差**

  

当前仓储层是这种模式：

  

```text

load_state(sender_id)

-> 读整条 state_json

-> 反序列化成 DialogueState

  

process_message(...)

-> 改内存中的 state

  

save_state(state)

-> 整体序列化

-> 整体写回 state_json

```

  

所以一旦历史 `turns` 很多，会有几个直接问题：

  

1. **读写成本线性上升**  

   每来一条消息，都要把整个 JSON 读出来、反序列化、再整体写回去。历史越长，单次请求越重。

  

2. **序列化/反序列化开销变大**  

   不是只改一个 `slot` 或追加一个 turn，而是整棵对象都要重新编码。

  

3. **数据库写放大**  

   明明本轮可能只是新增一条 bot message，结果整条 `state_json` 都要更新。

  

4. **并发覆盖风险更高**  

   这是“读整份 -> 改整份 -> 写整份”的典型问题。state 越大，写入窗口越长，并发请求越容易发生最后写覆盖前写。

  

5. **查询能力弱**  

   如果想查“某用户最近 20 条消息”或“退款任务平均停留在哪一步”，JSON 快照不适合做分析。

  

**我会怎么优化**

  

我一般不会一步到位全拆，而是按阶段做。

  

**第一阶段：保留快照，但控制体积**

  

这是成本最低的优化。

  

1. **截断历史 turns**

只在 `DialogueState` 里保留最近 N 轮，比如最近 20 轮或 50 轮；更早的历史不再放入热状态。

  

2. **对会话做摘要**

老历史不保留全文，而是定期压缩成 summary，供 LLM 继续参考。

  

3. **把大字段从热状态里剥离**

例如完整历史消息单独存表，`DialogueState` 只保留：

- `active_task`

- `paused_tasks`

- `active_system_task`

- `focused_object`

- 最近若干 turns

- 会话摘要

  

这样 `state_json` 仍然能承担“运行时状态快照”的角色，但不会无限膨胀。

  

**第二阶段：冷热分离**

  

这是我更推荐的长期方案。

  

把“运行时控制状态”和“历史消息”拆开：

  

1. `dialogue_state` 表  

   只存热状态，体积小，更新频繁：

   - `sender_id`

   - `active_task`

   - `paused_tasks`

   - `active_system_task`

   - `focused_object`

   - `current_session_id`

   - 少量 runtime metadata

  

2. `dialogue_turns` 表  

   单独存每一轮消息历史，按 append-only 方式写入：

   - `sender_id`

   - `session_id`

   - `turn_id`

   - `user_message`

   - `bot_messages`

   - `created_at`

  

这样每次处理消息时：

  

```text

读小状态表

-> 执行业务

-> 更新小状态表

-> 追加一条 turn 记录

```

  

而不是重写整份大 JSON。

  

**第三阶段：事件化或增量化**

  

如果流量更大，我会进一步把状态更新改成“增量事件”而不是“全量快照”。

  

例如记录：

  

- `task_started`

- `slot_set`

- `task_interrupted`

- `task_resumed`

- `turn_committed`

  

然后由事件流重建状态，或者异步生成快照。  

这更复杂，但适合高并发和审计需求强的场景。

  

**如果只允许小改动，我会优先做什么**

  

优先级我会这样排：

  

1. 把历史 `turns` 从 `DialogueState` 里裁短

2. 增加会话摘要，替代长历史

3. 把 `turns` 独立成消息表，状态表只留运行时状态

4. 在状态表增加版本号，做乐观锁，防止并发覆盖

  

前两步解决性能问题最快，第三步解决架构问题最彻底，第四步解决一致性问题。

  

**面试里可以这样答**

  

> JSON 快照式持久化适合项目早期，因为实现简单、状态恢复直接。但它的代价是每次处理消息都要整份读写 `DialogueState`，所以当历史 turns 变多后，读写成本、序列化开销和数据库写放大会线性增长，同时并发下还会有整份状态覆盖的问题。  

>  

> 我的优化思路是分阶段做。短期先控制热状态体积，比如只保留最近 N 轮对话，把更早历史做摘要；中期把运行时状态和历史消息拆表，状态表只存 `active_task`、`paused_tasks` 这类热数据，消息历史走 append-only；如果业务再上量，可以进一步演进为事件化或增量快照方案。这样既能保留当前设计的简单性，也能逐步解决大状态下的性能瓶颈。

  

### 11. 如果两个请求同时到达，比如用户快速连发两条消息，当前的“读 -> 处理 -> 写”模式会出现竞态条件吗？怎么解决？

  

会，**当前模式存在典型竞态条件**，而且是比较标准的“lost update”问题。

  

当前链路本质上是：

  

```text

请求 A: load_state -> process -> save_state

请求 B: load_state -> process -> save_state

```

  

如果 A 和 B 几乎同时到达，可能发生：

  

1. A、B 同时从数据库读到同一份旧 `DialogueState`

2. A 基于旧状态处理，写回新状态 `S1`

3. B 也基于同一份旧状态处理，写回新状态 `S2`

4. `S2` 把 `S1` 覆盖掉

  

因为当前仓储层是“整份 JSON 快照 upsert”，见 [src/repository/dialogue_state_repository.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/repository/dialogue_state_repository.py)，保存时没有版本检查，只是：

  

- 读整份 `state_json`

- 改内存对象

- 整体 `on_duplicate_key_update`

  

所以它不能防止并发覆盖。

  

**具体会出什么问题**

  

比如用户连续两条：

  

- 第一条：`帮我查物流`

- 第二条：`订单号是 A20260410001`

  

如果两条请求并发：

  

- 第一条可能把任务切到 `logistics_tracking`

- 第二条可能还是基于“旧状态”处理，根本不知道第一条已经创建了物流任务

- 最后写回顺序不对，就会出现：

  - 任务状态丢失

  - `slots` 丢失

  - `paused_tasks` 丢失

  - turn 历史丢一轮

  

这不是 LLM 问题，是状态持久化并发控制问题。

  

**怎么解决**

  

我会分成三层，按成本从低到高讲。

  

**1. 最直接的业务级方案：按 `sender_id` 串行处理**

同一用户的消息，不允许并发进入引擎。

  

做法可以是：

  

- 单机内存锁：`sender_id -> asyncio.Lock`

- 分布式锁：Redis 锁

- 更稳妥的是队列/actor 模型：同一用户消息进同一个串行消费者

  

这是最符合这个项目形态的方案，因为对话状态本来就是“单用户顺序状态机”。

  

优点：

  

- 逻辑最符合业务语义

- 不需要大改状态模型

- 最能避免乱序处理

  

缺点：

  

- 要处理锁超时、异常释放

- 分布式部署时单机锁不够

  

**2. 数据库级方案：乐观锁**

给 `dialogue_states` 增加一个 `version` 字段。

  

流程改成：

  

1. `load_state()` 时读出 `version`

2. `save_state()` 时带条件更新：

  

```sql

update dialogue_states

set state_json = ?, version = version + 1

where sender_id = ? and version = ?

```

  

如果更新行数是 0，说明有并发写入，当前请求应该：

  

- 重新加载最新状态

- 重新执行处理

- 或返回重试

  

这是最常见的并发控制方式。

  

优点：

  

- 不阻塞数据库行

- 适合这种“读多写少、冲突可重试”的模式

  

缺点：

  

- 要支持重放 `process_message`

- LLM 调用会变复杂，因为失败重试意味着可能重复调用模型或外部接口

  

所以如果引擎里有副作用动作，乐观锁要配合幂等设计一起做。

  

**3. 数据库级方案：悲观锁**

在读状态时对该用户记录加行锁，例如：

  

```sql

select ... for update

```

  

同一事务内完成：

  

```text

读状态 -> 处理 -> 写状态 -> 提交

```

  

这样第二个请求会被数据库阻塞，直到第一个事务提交。

  

优点：

  

- 一致性直接

- 不需要重试逻辑那么复杂

  

缺点：

  

- 会把 LLM 调用和外部 API 调用都包在事务里，不合适

- 锁持有时间长，吞吐差

- 高延迟下会明显拖慢系统

  

所以这个项目里，我**不推荐**把整个对话处理过程放进悲观锁事务。

  

**这个项目更合理的解法**

  

如果是这类“单用户多轮对话状态机”系统，我会优先这样做：

  

1. **同一 `sender_id` 串行化**

2. **数据库再加 `version` 做兜底**

3. **关键动作做幂等**

  

也就是：

  

```text

应用层保证大多数情况下不并发

数据库层防止极端情况下覆盖

动作层防止重试带来副作用重复

```

  

这是更工程化的组合。

  

**面试里可以这样答**

  

> 会，当前“读状态 -> 处理 -> 整体写回”的 JSON 快照模式存在典型的 lost update 风险。两个请求如果同时读到同一份旧状态，后写入的请求会把前一个请求的结果整体覆盖掉，导致任务状态、slots 或对话轮次丢失。  

>  

> 这类系统最合理的做法是按 `sender_id` 串行化处理，因为对话本身就是有序状态机；在此基础上，再给状态表加 `version` 做乐观锁，防止极端情况下的并发覆盖。如果只靠数据库悲观锁把整条链路包住，会把 LLM 和外部 API 调用都拖进事务，锁持有时间太长，不适合这个场景。

  

### 12. 你在文档里提到后续可以拆分出“消息表、任务表、事件表”。如果要拆，你会拆成什么样的表结构？拆完之后对恢复对话状态有什么影响？

  

我会把现在这份“大 JSON 快照”拆成三类数据：**消息事实、任务状态、领域事件**。核心原则是把“运行时热状态”和“历史审计数据”分开，不再让一张表同时承担全部职责。

  

**我会怎么拆**

  

第一类是 `message` 表，保存对话事实，按追加写。

  

可以拆成一张 `dialogue_messages`，也可以按轮次拆成 `dialogue_turns + turn_messages`。如果追求简单，我更倾向先上一张消息表：

  

| 字段 | 说明 |

|---|---|

| `id` | 主键 |

| `sender_id` | 用户 ID |

| `session_id` | 会话 ID |

| `turn_id` | 对话轮次 ID |

| `role` | `user` / `bot` / `system` |

| `message_type` | `text` / `object` |

| `text` | 文本内容 |

| `object_json` | 对象消息内容 |

| `created_at` | 创建时间 |

  

如果希望保留“轮次”概念更强，可以再补一张 `dialogue_turns`：

  

| 字段 | 说明 |

|---|---|

| `turn_id` | 主键 |

| `sender_id` | 用户 ID |

| `session_id` | 会话 ID |

| `started_at` | 本轮开始时间 |

| `completed_at` | 本轮结束时间 |

  

第二类是 `task` 表，保存运行时任务状态。

  

这张表替代当前 `active_task / paused_tasks / active_system_task` 的大部分内容：

  

| 字段 | 说明 |

|---|---|

| `id` | 主键 |

| `sender_id` | 用户 ID |

| `session_id` | 会话 ID |

| `flow_id` | flow 标识 |

| `task_type` | `user` / `system` |

| `status` | `active` / `paused` / `completed` / `canceled` |

| `step_id` | 当前步骤 |

| `slots_json` | 当前槽位 |

| `priority` | 可选，区分系统任务优先级 |

| `started_at` | 开始时间 |

| `updated_at` | 更新时间 |

| `resumed_from_task_id` | 可选，恢复链路 |

| `version` | 乐观锁版本号 |

  

如果后续要支持更强能力，还可以把 `slots_json` 再拆成 `task_slots` 表，但第一步没必要过度拆。

  

第三类是 `event` 表，保存状态变化事件，偏审计和可回放。

  

| 字段 | 说明 |

|---|---|

| `id` | 主键 |

| `sender_id` | 用户 ID |

| `session_id` | 会话 ID |

| `task_id` | 关联任务 |

| `turn_id` | 关联轮次 |

| `event_type` | `task_started` / `slot_set` / `task_paused` / `task_resumed` / `task_completed` / `clarify_triggered` 等 |

| `payload_json` | 事件明细 |

| `created_at` | 事件时间 |

  

如果要再补一张“轻状态表”，我会加 `dialogue_state_runtime`，只保存热状态指针：

  

| 字段 | 说明 |

|---|---|

| `sender_id` | 主键 |

| `current_session_id` | 当前会话 |

| `active_task_id` | 当前用户任务 |

| `active_system_task_id` | 当前系统任务 |

| `focused_object_json` | 当前聚焦对象 |

| `summary_text` | 历史摘要 |

| `version` | 乐观锁版本 |

  

**拆完之后，恢复对话状态会怎么变**

  

影响是：恢复状态不再靠“读一坨 JSON”，而是靠“读热状态 + 补最近历史 + 必要时回放事件”。

  

我会分两层恢复。

  

第一层是**请求处理时的快速恢复**，只恢复运行必需状态：

  

1. 从 `dialogue_state_runtime` 取：

   - `current_session_id`

   - `active_task_id`

   - `active_system_task_id`

   - `focused_object`

   - `summary_text`

  

2. 从 `tasks` 表取：

   - 当前 `active` / `paused` 的任务

   - 当前 step 和 slots

  

3. 从 `messages` 表取最近 N 条消息，供 LLM 规划

  

这样就够支撑一次正常对话，不需要每次把全量历史都装进内存。

  

第二层是**审计/修复型恢复**，当热状态损坏或要回放时：

  

1. 从 `events` 表按时间顺序拉取

2. 重建某个任务或某个 session 的状态

3. 必要时重新生成快照或修复 runtime 表

  

也就是说，拆分后恢复会从“全量快照恢复”变成“两级恢复”：

  

- 正常路径：读热状态快

- 异常路径：事件回放准

  

**这样拆的收益**

  

- `messages` 走 append-only，历史越长越适合存储和查询

- `tasks` 只保留热业务状态，更新轻很多

- `events` 提供可审计、可追踪、可回放能力

- 状态恢复更灵活，不必每次整份反序列化

- 后续支持统计分析也更自然，比如：

  - 哪个 flow 最常中断

  - 哪个 collect 节点失败最多

  - 哪类澄清最常见

  

**代价也很明确**

  

- 写入链路从“一次写快照”变成“多表写”

- 需要处理事务一致性

- `DialogueState` 不再是天然单对象，要做组装

- 代码复杂度会上升

  

所以我的建议是：**不要一步拆成完整事件溯源系统**。比较现实的演进顺序是：

  

1. 先拆 `messages` 表

2. 再拆 `tasks` 表

3. 最后补 `events` 表做审计和回放

  

**面试里可以这样答**

  

> 如果要拆，我会把当前的 `DialogueState` 分成三类持久化对象：消息表保存对话事实，任务表保存运行时 flow 状态，事件表保存状态迁移记录。这样消息是 append-only 的，任务是小而热的，事件则用于审计和回放。  

>  

> 拆完以后，对话状态恢复不再是每次读整份 JSON，而是优先读取一份轻量 runtime 状态，再补当前 active/paused task 和最近若干条消息；只有在修复或追溯时才需要回放事件。这会让在线恢复更快，但实现复杂度也更高，所以我会按“先拆消息、再拆任务、最后补事件”的顺序渐进演进。

  

---

  

## 五、知识检索与 Provider 机制

  

### 13. 四种 Provider（ProductAPI、OrderAPI、FAQ、RAG）的结果格式和可信度不同。比如 OrderAPI 返回的是真实数据，RAG 返回的是检索结果，可能有误差。`KnowledgeResponder` 在生成回复时怎么区分对待？

  

当前这套实现里，`KnowledgeResponder` **基本没有区分对待**不同 Provider 的可信度和类型，它把它们统一当成“知识片段文本”来喂给 LLM。

  

证据很直接：

  

- [src/knowledge/providers.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/knowledge/providers.py) 里的 `KnowledgeChunk` 只有一个字段：`content: str`

- [src/knowledge/handler.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/knowledge/handler.py) 只是把多个 provider 返回的 `chunks` 收集起来

- [src/knowledge/responder.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/knowledge/responder.py) 直接把这些 `chunk.content` 用 `"\n\n".join(...)` 拼成 `knowledge_content`，然后统一交给 LLM 生成回复

  

也就是说，当前系统里：

  

- `OrderAPIProvider` 返回的真实订单数据

- `ProductAPIProvider` 返回的商品数据

- `FAQProvider` 返回的 FAQ 文本

- `RAGProvider` 返回的检索结果

  

在 `KnowledgeResponder` 看来，本质上都只是文本，没有额外的：

  

- `source_type`

- `confidence`

- `trust_level`

- `citation`

- `is_ground_truth`

  

所以严格回答，这个项目**目前没有在生成层显式区分“强事实数据”和“弱检索结果”**。

  

**这会带来什么问题**

  

问题不是“不能回答”，而是“回答时缺少来源策略”。  

例如：

  

- 订单状态这类强事实，本来应该直接按 API 数据回答

- RAG 检索结果本来应该带保守措辞，比如“根据当前检索结果”

- FAQ 和 RAG 冲突时，应该优先 FAQ 或官方规则

- API 数据和 RAG 文本冲突时，应该以 API 为准

  

但当前实现没有这层机制，所以最终回复质量会比较依赖 prompt 和模型自身判断。

  

**如果我要优化，会怎么改**

  

我会先从 `KnowledgeChunk` 结构入手，不会只保留一个 `content` 字段，而是改成带元信息的统一知识对象，比如：

  

```python

@dataclass

class KnowledgeChunk:

    content: str

    source_type: str          # api / faq / rag

    source_id: str = ""

    trust_level: str = "medium"   # high / medium / low

    score: float | None = None

    citation: str | None = None

```

  

这样四类 Provider 可以显式标注：

  

- `ProductAPI` / `OrderAPI`

  - `source_type = "api"`

  - `trust_level = "high"`

- `FAQ`

  - `source_type = "faq"`

  - `trust_level = "high"` 或 `medium`

- `RAG`

  - `source_type = "rag"`

  - `trust_level = "low"` 或 `medium`

  - 可附带检索分数 `score`

  

然后 `KnowledgeResponder` 就不该只是简单拼接文本，而应该按来源分层组织 prompt，例如：

  

```text

High-trust facts:

- 订单状态来自订单 API

- 物流状态来自物流 API

  

Reference materials:

- FAQ 内容

- RAG 检索片段

  

Answer policy:

- 如果 API 与 RAG 冲突，以 API 为准

- 如果只有 RAG，没有确定事实，用保守措辞

- 不要把低置信度检索结果说成确定事实

```

  

**更合理的回复策略**

  

我会把生成策略分成三档：

  

1. `API` 数据优先直接回答  

   像订单状态、物流进度、商品详情，这类强结构化事实，应该优先走“事实回答”，甚至可以弱化 LLM，自然语言只是包装层。

  

2. `FAQ` 作为规则型权威来源  

   平台规则、退款政策、配送政策，如果 FAQ 是官方整理的，就应该高于 RAG。

  

3. `RAG` 作为补充，不作为硬事实  

   RAG 更适合补背景、补解释、补扩展说明，不适合单独承诺“订单已经怎样”这类确定事实。

  

**面试里可以这样答**

  

> 当前实现里，`KnowledgeResponder` 对四类 Provider 并没有做显式区分，因为它接收到的只是统一的 `KnowledgeChunk.content` 文本，最后直接拼接后交给 LLM 生成回复。所以从代码层面看，真实 API 数据和 RAG 检索结果在生成阶段还没有建立可信度分层。  

>  

> 如果要把这块做得更工程化，我会先给 `KnowledgeChunk` 增加来源类型、可信度、分数和引用信息，然后在 `KnowledgeResponder` 的 prompt 里明确规则：API 数据优先级最高，FAQ 作为规则型权威来源，RAG 只作为补充参考；如果来源冲突，以高可信来源为准；如果只有低可信检索结果，就要求模型使用保守措辞。这样能把“事实回答”和“检索性回答”分开处理。

  

### 14. 如果用户问“这个商品和那个商品哪个好”，这可能需要同时调用 `ProductAPIProvider` 和 `RAGProvider`。你的 Provider 机制支持多 Provider 联合查询吗？

  

支持，但要分“机制层面支持”还是“当前意图定义是否已经覆盖”来回答。

  

**从机制层面看，是支持多 Provider 联合查询的。**

  

关键在 [src/knowledge/handler.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/knowledge/handler.py)：

  

- 一个 `intent` 可以映射多个 `provider_id`

- `KnowledgeHandler` 会先把所有 intent 对应的 `provider_id` 收集出来

- 然后逐个调用 provider，把结果合并成 `chunks`

- 最后统一交给 `KnowledgeResponder`

  

这段逻辑本质上就是：

  

```text

intents

-> provider_ids

-> 多个 provider.retrieve(state)

-> chunks 合并

-> LLM 汇总回答

```

  

在 [src/knowledge/intents.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/knowledge/intents.py) 里其实已经有多 Provider 的例子，比如：

  

- `refund_policy`

  -> `faq.default` + `rag.default`

- `shipping_policy`

  -> `faq.default` + `rag.default`

  

所以严格说，**Provider 机制本身已经支持“一次知识问答联合多个 Provider”**。

  

**但当前“商品对比”这个具体场景，还不算完整支持。**

  

因为当前系统还有几个限制。

  

1. **`DialogueState` 只有一个 `focused_object`**

现在状态里只有一个当前聚焦对象，见 [src/domain/state.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/domain/state.py)。

  

这意味着系统天然更适合处理：

  

- “这个商品怎么样”

- “这个订单到哪了”

  

而不适合直接表达：

  

- “这个商品和那个商品哪个好”

  

因为这里其实需要两个对象上下文，而不是一个。

  

2. **当前没有“商品对比”意图**

现有 intent 里有：

  

- `product_info`

- `order_info`

- `refund_policy`

- `shipping_policy`

  

但没有类似：

  

- `product_comparison`

- `product_recommendation_reasoning`

  

所以当前即使机制支持多 Provider，也缺少业务入口定义。

  

3. **当前 Provider 输入模型偏单对象**

`ProductAPIProvider` 现在默认从 `state.focused_object.id` 取一个商品 ID。  

如果要比较两个商品，Provider 或者上层上下文模型要能拿到：

  

- `product_a`

- `product_b`

  

或者一个对象列表，而不是单个 `focused_object`。

  

**如果要支持“这个商品和那个商品哪个好”，我会怎么改**

  

我不会改 Provider 机制本身太多，因为它已经具备“联合多个 Provider”的基本形态。  

我会主要改三层。

  

**第一层：扩展意图定义**

  

新增一个 intent，例如：

  

```text

product_comparison

```

  

它可以绑定多个 provider：

  

- `api.product`

- `rag.default`

  

必要时还能加 FAQ 或评论摘要类 provider。

  

**第二层：扩展上下文模型**

  

把当前单一的 `focused_object` 扩展成更适合比较场景的结构，比如：

  

- `focused_objects: list[FocusedObject]`

- 或在 `pending_turn` / task context 里保存比较对象集合

  

这样系统才能明确知道“这个”和“那个”分别是谁。

  

**第三层：让 Provider 支持多对象输入**

  

例如 `ProductAPIProvider` 不再只支持一个商品，而是支持：

  

```text

retrieve(state) -> 根据上下文拉两个商品详情

```

  

或者更干净一点，给 `KnowledgeProvider` 的接口增加参数，不只依赖全局 state。

  

**面试里可以这样答**

  

> 这套 Provider 机制本身支持多 Provider 联合查询，因为一个 knowledge intent 可以绑定多个 provider_id，`KnowledgeHandler` 会把多个 provider 的结果聚合成 knowledge chunks，再统一交给 `KnowledgeResponder`。像当前的退款政策、配送政策，其实已经在同时使用 FAQ 和 RAG。  

>  

> 但“这个商品和那个商品哪个好”这个具体场景，当前代码还没有完整支持，因为现在只有一个 `focused_object`，没有“双商品比较”的上下文模型，也没有专门的 `product_comparison` intent。也就是说，机制上支持多 Provider 聚合，业务上还需要补 intent、上下文结构和多对象 Provider 输入能力。

  
  

---

  

## 六、工程与可靠性

  

### 15. 你的外部依赖有 LLM、MySQL、商城 API。如果商城 API 超时了，对话还能继续吗？你会在哪个层级做降级处理？

  

能继续，但要分“当前代码已经具备的降级”和“更完整的工程化降级”两层说。

  

**当前代码下，对话大概率还能继续，但会退化成失败提示，不会把整个会话直接打崩。**

  

原因是商城 API 主要出现在两类地方：

  

- `knowledge` 里的 Provider，比如 `ProductAPIProvider`、`OrderAPIProvider`

- `task` 里的自定义 Action，比如查订单、查物流

  

当前实现里，`task/action/custom/shared.py` 这类调用已经做了异常兜底：请求异常时返回 `None`。随后上层 Action 会把它转成可读的失败结果，比如：

  

- “暂时无法查到该订单信息，请稍后再试”

- “暂时无法查到物流信息，请稍后再试”

  

所以从用户体验上看，**商城 API 超时通常不会让整轮对话崩掉，而是退化成一个业务失败回复**。  

这属于“局部降级”。

  

但如果你问工程上应该怎么设计，我会把降级分层处理，而不是只在一个地方硬 catch。

  

**我会在哪个层级做降级**

  

1. **基础设施层**

在 HTTP Client 这一层做通用的超时、重试、连接池和熔断策略。

  

这一层负责：

- 设置合理 timeout

- 对幂等 GET 请求做有限重试

- 区分超时、连接失败、4xx、5xx

- 统一抛出受控异常或返回标准错误结果

  

这一层不该决定回复用户什么，只负责把“底层失败”标准化。

  

2. **Provider / Action 层**

这是最关键的业务降级层。  

因为这里只有它最清楚：这次失败影响的是“订单查询”“物流查询”还是“商品信息咨询”。

  

比如：

- `OrderAPIProvider` 超时 -> 返回“订单信息暂时不可用”

- `LookupLogisticsAction` 超时 -> 返回“物流查询失败，请稍后重试”

  

这一层应该把技术错误翻译成业务可理解结果。

  

3. **Engine / Service 层**

这里负责保证整轮对话别崩，以及状态一致性。

  

比如：

- 这轮虽然外部查询失败，但 `pending_turn` 仍然要正常提交

- 当前任务是否继续保留，还是停在当前步骤等待用户稍后重试

- 是否记录失败事件、埋点、告警

  

这一层更像“事务边界和整体兜底”。

  

4. **API 层**

这里只做最后一层全局异常兜底。  

如果前面没处理住，API 层至少要返回统一错误格式，而不是 500 堆栈直接暴露给前端。

  

**我会怎么定降级策略**

  

不是所有商城 API 超时都用同一种降级。

  

- **知识查询类**

  可以继续对话，回复“我这边暂时没查到，稍后再试”，必要时引导用户换个问题。

- **任务执行类**

  如果是查订单、查物流这种只读动作，失败后可以保留当前任务，让用户稍后重试。

- **写操作类**

  如果以后接入真实退款提交、工单创建，这类失败要更谨慎，不能口头说“已提交”，必须明确说“提交未成功”。

  

所以降级不仅是“catch exception”，更重要的是**不能让系统在失败时做出错误承诺**。

  

**面试里可以这样答**

  

> 商城 API 超时后，对话 ideally 应该还能继续，但会降级成业务失败提示，而不是把整个会话打崩。当前代码里已经有一部分这种处理，比如自定义 Action 调商城 API 失败时会返回 `None`，上层再转成“暂时无法查询”的回复。  

>  

> 如果做得更完整，我会分层降级：基础设施层处理 timeout、重试和统一异常；Provider 或 Action 层把技术失败翻译成业务语义；Service/Engine 层保证整轮状态提交和任务一致性；API 层做最后的全局异常兜底。这样既能保证对话不中断，也能避免系统在外部依赖失败时给出错误承诺。

  

### 16. `DialogueState.to_dict()` 把所有字段序列化到 JSON。如果某天改了 `TaskContext` 的字段名，数据库中旧版 JSON 反序列化就会失败。你怎么做数据迁移和向前兼容？

  

### 17. 文档中说后续要“增加单元测试和集成测试”，但你现在还没写。如果现在让你给核心模块写测试，你会先测哪个？为什么？

  

我会先测 **`DialogueEngine + TaskHandler` 这条主链路**，准确说是先从 **“核心状态迁移”** 下手，而不是先测 LLM prompt 或 HTTP 路由。

  

原因很直接：这个项目最核心、最容易出真实业务问题的地方，不是“模型说得好不好”，而是**状态有没有被正确推进**。一旦这里出错，会直接影响：

  

- 任务是否被正确启动

- `active_task / paused_tasks / active_system_task` 是否一致

- `slots` 会不会丢

- interrupt / resume 会不会乱

- 本轮 turn 会不会正确提交

  

这些问题的风险远高于单纯话术不自然。

  

**如果按优先级排，我会这样测**

  

1. **`DialogueEngine` 的分流与状态提交**

   先测 [dialogue_engine.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/engine/dialogue_engine.py)。

  

   重点断言：

   - 文本消息能否正确分流到 `task / knowledge / chitchat / clarify`

   - `pending_turn` 是否正确生成并提交

   - 当 `turn_plan` 非法时是否走 `clarify`

   - 当进入 `knowledge` 或 `chitchat` 时，是否会正确中断当前任务

  

   这层测试价值最高，因为它是总调度中心。

  

2. **`TaskHandler / CommandProcessor / FlowExecutor`**

   这是第二优先级，也是最值得做细单测的地方。

  

   重点断言：

   - `start_flow` 是否正确创建 `active_task`

   - 已有任务时再 `start_flow`，是否把旧任务放进 `paused_tasks`

   - `resume_flow` 是否恢复原 `step_id` 和 `slots`

   - `collect` 缺参时是否启动 `system_collect_information`

   - `set_slots` 后流程是否能继续推进到下一个 step

   - `end` 是否正确结束 system task 或 active task

  

   这是项目的“状态机核心”，比测 API 更重要。

  

3. **`TurnPlanValidator`**

   这是最适合先补的纯单元测试模块，成本低、收益高。

  

   重点断言：

   - `MISSING_TRACK`

   - `MULTIPLE_TRACKS`

   - `MISSING_TASK_COMMANDS`

   - `MISSING_KNOWLEDGE_INTENT`

   - `MISSING_FOCUSED_OBJECT`

  

   这个模块规则明确、无外部依赖，很容易快速建立测试保护网。

  

4. **Repository 持久化测试**

   测 [dialogue_state_repository.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/repository/dialogue_state_repository.py)。

  

   重点断言：

   - `DialogueState -> JSON -> DialogueState` 是否无损

   - 保存后再读取，`active_task / paused_tasks / sessions / turns` 是否一致

   - upsert 是否生效

  

   这是为了防止“状态逻辑没问题，但落库后恢复错了”。

  

**我不会最先测什么**

  

我不会先测：

  

- `FastAPI router`

- prompt 文案

- LLM 生成文本内容本身

  

因为这些虽然也要测，但它们不是系统最脆弱的点。  

这个项目最容易出事故的是“状态错乱”，不是“回复措辞不够自然”。

  

**如果只允许我今天先写一组测试**

  

我会先写一组围绕“退款流程被中断再恢复”的测试，因为它能同时覆盖最多核心逻辑：

  

- `start_flow(refund_request)`

- collect 槽位

- `interrupt_active_task()`

- `paused_tasks`

- `resume_flow(refund_request)`

- slots 保留

- step 恢复

  

这组测试一旦通过，说明项目最有特色的多任务对话状态管理基本站住了。

  

**面试里可以这样答**

  

> 如果现在开始补测试，我不会先测 API 层，也不会先测 prompt 文案，而是先测对话引擎和任务状态机。因为这个项目的核心风险不在于模型回复是否自然，而在于 `active_task`、`paused_tasks`、`active_system_task`、slots 和 turn 提交这些状态迁移是否正确。  

>  

> 具体上我会先补 `DialogueEngine` 的分流测试、`TaskHandler/FlowExecutor` 的状态推进测试，以及 `TurnPlanValidator` 的规则测试。这样可以先把最关键的业务骨架保护住，再往外补 repository、API 和集成测试。

---

  

## 七、追问与压力题

  

### 18. 你的系统假设用户消息已经通过某个渠道，比如 Web 或 App，发到了 API。如果真的接入一个电商平台，比如日均 10 万咨询量，你认为当前架构的第一个瓶颈会在哪里？

第一个瓶颈我会判断在 **同步链路里的 LLM 调用**，不是 MySQL，也不是 FastAPI 本身。

  

原因很直接：当前每条文本消息进入 [dialogue_engine.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/engine/dialogue_engine.py) 后，基本都会先经过 [turn_planner.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/plan/turn_planner.py) 做一次 LLM 调用。之后如果命中：

  

- `knowledge`

  还要再调一次 LLM 生成回答

- `chitchat`

  还要再调一次 LLM

- `clarify`

  还要再调一次 LLM

- 某些 `action_response`

  也可能再调 LLM 做改写

  

也就是说，很多请求不是 1 次模型调用，而是 **1 次规划 + 1 次回复生成**，甚至更多。  

在日均 10 万咨询量场景下，真正先吃满的通常是：

  

- LLM 延迟

- LLM QPS 限额

- LLM 成本

- 外部网络抖动带来的尾延迟

  

而不是 Python Web 层。

  

**为什么不是数据库先炸**

  

当前 MySQL 也有问题，但它更像“第二瓶颈”。

  

确实存在这些风险：

  

- 每次读整份 `state_json`

- 每次写整份 `state_json`

- 同用户并发下有覆盖问题

  

但从吞吐角度看，日均 10 万咨询量并不算特别高，平均下来 QPS 不高。  

如果没有明显峰值，数据库在短期内通常还能扛一阵，尤其是表结构目前还简单。

  

相反，LLM 是每条消息几乎必走的高延迟外部依赖，而且链路是同步阻塞的，这会更早成为首要瓶颈。

  

**更准确地说，瓶颈会分阶段出现**

  

如果让我排序，我会这样看：

  

1. **第一瓶颈：LLM 同步调用链路**

   - 规划必调

   - 回复常常还要再调

   - 延迟和成本最高

   - 外部依赖最不稳定

  

2. **第二瓶颈：JSON 快照式状态读写**

   - 整体读、整体写

   - state 变大后越来越重

   - 并发时有 lost update 风险

  

3. **第三瓶颈：商城 API**

   - 订单、物流、商品信息查询会依赖外部接口

   - 高峰时尾延迟和失败率会上升

   - 尤其是知识问答和任务 action 会放大外部依赖成本

  

4. **第四瓶颈：单机同步处理模型**

   - 当前架构是强同步 request-response

   - 一旦某些链路慢，请求堆积会拖垮整体吞吐

  

**如果真接 10 万日咨询，我会先优化什么**

  

第一波我不会先拆数据库，而是先控 LLM 流量。

  

1. **减少不必要的 LLM 调用**

   - 高频闲聊、固定问候走规则直出

   - 简单任务入口走轻规则分类

   - 一些静态话术不要再让 `action_response` 二次改写

  

2. **把规划和生成解耦优化**

   - 对简单路径做 shortcut

   - 不要所有消息都先过完整 LLM planner

  

3. **做缓存和降级**

   - FAQ / 规则类问题缓存

   - 常见知识问答模板化

   - LLM 超时后快速 fallback

  

4. **再处理状态存储**

   - 缩小 `DialogueState`

   - 拆消息表和状态表

   - 加串行化或乐观锁

  

**面试里可以这样答**

  

> 如果真的接到日均 10 万咨询量，我认为当前架构的第一个瓶颈会先出现在同步链路里的 LLM 调用。因为文本消息基本都会先经过一次 TurnPlanner 规划，后续 knowledge、chitchat、clarify 这些轨道通常还会再调用一次 LLM 生成回复，所以很多请求实际上是两次模型调用起步。相比之下，FastAPI 和 MySQL 虽然也有压力，但短期内通常不会比 LLM 更早成为首要瓶颈。  

>  

> 第二阶段的瓶颈才会是 JSON 快照式状态读写和商城 API 调用。优化顺序上，我会先减少不必要的 LLM 调用，把简单路径规则化或模板化，再逐步拆分状态存储和增强外部依赖的降级能力。

  

### 19. 如果有人跟你说“这个系统太复杂了，现在 LangChain + Agent 框架几行代码就能搞定客服，没必要自己写状态机和流程引擎”，你怎么回应？

  

我会先承认一半：**如果目标只是做一个能聊、能演示、能跑通简单问答的 Demo，那确实没必要写这么多，Agent 框架几行代码就能起一个原型。**  

但如果目标是“电商客服生产系统”，我会很明确地反驳：**几行 Agent 代码能解决的是 demo 复杂度，不是业务复杂度。**

  

这个项目之所以要有状态机和流程引擎，不是因为我想把系统写复杂，而是因为业务本身就有这些约束：

  

- 多轮槽位收集

- 任务中断和恢复

- 订单、物流、退款这类真实业务动作

- 状态持久化

- 澄清和回退

- 可审计、可测试、可控执行

  

这些不是 Agent 框架“自动调用工具”就天然能解决的。

  

**我会怎么回应这个质疑**

  

第一，**Agent 擅长的是开放式推理，不擅长强约束业务流程。**  

客服里很多事情不是“回答得像不像”，而是“状态是不是对、动作是不是可控”。

  

比如退款申请不是一句：

  

> 我已经帮你提交了退款

  

就算完成了。  

它要求系统明确知道：

  

- 当前是不是退款流程

- 订单号有没有收齐

- 退款原因有没有收齐

- 有没有中途被别的任务打断

- 最终到底有没有真的调用提交动作

  

这类场景更接近状态机，不是纯 Agent 自由推理。

  

第二，**Agent 能编排工具，但不等于天然具备稳定状态管理。**  

这个项目最有价值的部分，其实不是“会不会调用 API”，而是：

  

- `active_task`

- `paused_tasks`

- `active_system_task`

- `slots`

- `focused_object`

  

这些结构把多轮对话从“上下文聊天”变成“可恢复的业务状态”。  

很多 Agent Demo 能把工具调起来，但一旦问到“中断后怎么恢复”“两个任务怎么切换”“老状态怎么持久化”，就开始模糊了。

  

第三，**生产系统要可解释、可审计、可测试。**  

自己写状态机和流程引擎的一个核心价值是：系统每一步都能解释。

  

例如可以明确说：

  

- 这一轮被规划成 `refund_request`

- 当前卡在 `collect order_number`

- 因为用户切换到“查物流”，退款任务被放进 `paused_tasks`

- 当前 `system_task_interrupted` 正在提示用户任务切换

  

这类解释能力，直接关系到：

  

- 问题排查

- 线上运维

- 业务验收

- 合规审计

- 自动化测试

  

而如果全部交给通用 Agent 自由运行，很多时候你只能看到“它好像是这么想的”，但很难精确定位状态。

  

第四，**复杂度不是被消灭了，只是被隐藏了。**  

用 LangChain/Agent 框架把代码写短，不代表系统真的简单了。  

很多时候只是把复杂度从显式代码，转移到了：

  

- prompt

- tool description

- memory 配置

- agent loop 行为

- 框架内部状态

  

短期看很快，长期看可能更难调试。  

尤其是客服这种强业务约束场景，隐藏复杂度通常比显式复杂度更危险。

  

**我不会全盘否定 Agent**

  

我会说，Agent 不是不能用，而是要放对位置。  

比如在这个项目里，LLM/Agent 更适合做：

  

- 意图规划

- 话术生成

- 知识汇总

- 澄清追问

  

而不适合直接接管：

  

- 多轮任务状态机

- 业务动作提交

- 状态持久化一致性

- 高风险流程控制

  

更现实的做法不是“全手写”或“全 Agent”二选一，而是：

  

```text

LLM / Agent 负责理解和生成

状态机 / 流程引擎负责约束和执行

```

  

这才是比较稳的组合。

  

**面试里可以这样答**

  

> 如果只是做 Demo，我同意，用 LangChain 或 Agent 框架几行代码就能把客服原型跑起来。但这个项目面对的是电商客服业务，不只是问答，还包括订单查询、物流查询、退款申请、多轮槽位收集、任务中断恢复和状态持久化。  

>  

> 这些问题本质上不是“会不会调用工具”，而是“业务状态能不能稳定管理、执行过程能不能可控、系统是否可审计和可测试”。Agent 框架可以帮我快速搭原型，但不能替代状态机和流程引擎本身。否则复杂度并没有消失，只是被藏进 prompt、memory 和 agent loop 里了。  

>  

> 所以我的取舍是：让 LLM 负责意图理解、知识汇总和自然语言生成，让状态机和流程引擎负责业务约束、状态迁移和可控执行。这样既能利用大模型，又不会把生产系统的核心控制权完全交给不确定的推理链。

  

---

  

**这些问题从上到下逐步深入：前 4 题侧重于理解设计意图，5-12 题考察对系统细节和边界的掌握程度，13-17 题测试工程能力和扩展思考，最后两题看技术判断力和大局观。**