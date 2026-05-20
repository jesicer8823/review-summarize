# 面试问题


## 一、架构与设计思路
  
### 1. 你提到"把 LLM 作为意图规划器而非对话生成器"，为什么这样设计？如果直接让 LLM 端到端生成回复，会遇到哪些具体问题？

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

  
  

### 2. 项目的分层里有一层"应用服务层"，它的职责只是加载状态→调引擎→保存状态，看起来很薄。这一层存在的必要性是什么？能不能把它合并到 API 层或引擎层？

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

  
  

### 3. DialogueState 同时维护了 active_task、paused_tasks 和 active_system_task，这三种 task 的优先级关系是怎样的？如果用户正在退款流程中被追问订单号，突然说"帮我查一下物流"，系统会怎么处理？

  

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

  

## 二、LLM 与 TurnPlan

###　4. LLM 生成 TurnPlan 后，你提到有一个 TurnPlanValidator 做校验。具体校验什么？如果校验不通过，系统怎么兜底？

  

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

  

### 5. 你让 LLM 做意图识别 + 规划，如果 LLM 返回了一个"看起来合理但实际上是错误的"规划（比如它把闲聊判断成了查订单），系统有没有办法发现并纠正？

  

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

  

### 6.  四个轨道（task / knowledge / chitchat / clarify）中，clarify 是兜底。如果 LLM 频繁产出需要 clarify 的结果（比如每次都判断为意图不明确），你会怎么优化？这是 prompt 问题、模型能力问题、还是 flow 设计问题？

  

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

  

## 三、任务流程引擎

### 7. 任务流程是通过 YAML 配置的，collect → action → end 的节点模型看起来像一个简单的状态机。如果一个 flow 需要在 collect 失败时重试三次、或者超时自动取消，你的节点模型能支持吗？如果不能，会怎么改？

  

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

  

### 9. YAML 里写死了回复文案（比如 "请告诉我你的订单号"）。你怎么处理多语言、多文案 A/B 测试、或者运营想随时调整话术的场景？

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