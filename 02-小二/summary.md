# 电商智能客服对话平台项目总结

  

## 1. 项目概述

  

本项目是一个面向电商客服场景的智能对话后端系统，主要用于支持用户通过自然语言完成订单查询、物流查询、退款申请、商品咨询、平台规则咨询、闲聊和澄清追问等能力。

  

项目的重点不是简单调用大模型生成回复，而是将大模型能力放入一个可控的后端业务编排框架中。系统通过对话状态管理、任务流程引擎、知识检索、业务 Action 执行和数据库持久化，实现了更接近真实客服系统的多轮对话能力。

  

技术栈主要包括：

  

- FastAPI：提供异步 HTTP API 服务

- SQLAlchemy Async：实现异步数据库访问

- MySQL：持久化用户对话状态

- LangChain：封装 LLM 调用

- httpx：调用外部商城 API

- Jinja2：管理 prompt 模板和动态回复模板

- YAML：配置任务流程

  

## 2. 项目整体架构

  

项目后端主要位于 `src/` 目录，整体采用分层架构：

  

```text

API 层

  -> 应用服务层

    -> 对话引擎层

      -> 任务 / 知识 / 闲聊 / 澄清模块

        -> 基础设施层

```

  

各层职责如下：

  

| 层级  | 主要职责 |
| --- | ---- |
| API 层 | 接收请求、参数校验、响应封装、依赖注入 |
| 应用服务层 | 加载对话状态、调用对话引擎、保存对话状态 |
| 对话引擎层 | 管理一轮对话的完整生命周期，进行业务分流 |
| 任务模块 | 处理查订单、查物流、退款等多轮任务流程 |
| 知识模块 | 处理商品信息、订单信息、平台规则、售后政策等问答 |
| 闲聊模块 | 处理普通聊天和寒暄 |
| 澄清模块 | 在意图不明确、对象缺失或规划冲突时进行追问 |
| 基础设施层 | 封装数据库、HTTP Client、LLM、配置等底层能力 |
  

这种结构使项目具备较好的可维护性和扩展性，各模块职责清晰，避免将所有逻辑堆在一个接口中。

  

## 3. 核心调用流程

  

用户发送消息后，系统完整处理流程如下：

  

```text

用户请求

  -> FastAPI 路由接收 ChatRequest

  -> 构造 UserMessage

  -> DialogueService 加载 DialogueState

  -> DialogueEngine 处理本轮消息

      -> 准备会话 session

      -> 开启 pending_turn

      -> 判断消息类型

      -> 调用 TurnPlanner 生成结构化 TurnPlan

      -> 调用 TurnPlanValidator 校验规划结果

      -> 分流到 task / knowledge / chitchat / clarify

      -> 生成 BotMessage

      -> 提交本轮 turn

  -> DialogueService 保存 DialogueState

  -> API 层封装 ChatResponse 返回

```

  

应用服务层可以理解为系统的事务边界：

  

```python

async def process_message(self, user_message):

    state = await repository.load_state(user_message.sender_id)

    result = await dialogue_engine.process_message(state, user_message)

    await repository.save_state(state)

    return result

```

  

也就是说，服务层本身不直接做意图识别和业务判断，而是负责组织状态读写和引擎调用。

  

## 4. 对话引擎设计

  

对话引擎是项目的核心，负责将一条用户消息转换成一组机器人回复，并更新用户对话状态。

  

它主要完成以下工作：

  

1. 管理会话生命周期  

   如果用户没有当前会话，则创建新会话；如果会话超时，则关闭旧会话并重置运行时状态。

  

2. 维护多轮对话状态  

   系统会维护当前 session、历史 turns、当前活跃任务、暂停任务、系统任务、槽位信息和聚焦对象。

  

3. 使用 LLM 生成结构化规划  

   系统不会直接让 LLM 自由回复，而是让 LLM 根据上下文生成结构化的 `TurnPlan`，再由后端进行校验。

  

4. 进行业务轨道分流  

   校验后的请求会被分流到四类轨道：

  

| 轨道 | 说明 |
|---|---|
| task | 任务型业务办理，例如查订单、查物流、申请退款 |
| knowledge | 知识问答，例如商品信息、退款政策、配送规则 |
| chitchat | 普通闲聊 |
| clarify | 意图不明确或上下文缺失时进行澄清 |

  

这一设计的优势是：LLM 主要负责理解和规划，真正的业务执行仍由后端可控逻辑完成，从而降低大模型幻觉和误操作风险。

  

## 5. 任务流程模块

  

任务模块负责处理有明确业务步骤的多轮对话，例如：

  

- 订单状态查询

- 物流查询

- 退款申请

- 相似商品推荐

- 转人工客服

  

任务流程通过 `flow_config/*.yml` 配置，典型步骤包括：

  

```text

start -> collect -> action -> response -> end

```

  

以物流查询为例：

  

```text

用户：帮我查物流

系统：请告诉我订单号

用户：A20260410001

系统：调用物流查询 Action

系统：返回物流公司、物流单号和当前进度

```

  

任务模块内部主要由以下组件组成：

  

| 组件 | 职责 |
|---|---|
| CommandProcessor | 处理 start_flow、set_slots、cancel_flow、resume_flow 等命令 |
| FlowExecutor | 推进流程节点，处理 collect、action、end 等步骤 |
| ActionRunner | 根据 action 名称执行具体业务动作 |
| ActionRegistry | 注册内置 Action 和自定义 Action |

  

任务模块的一个重要特点是支持任务中断和恢复。例如用户正在申请退款，中途询问订单信息，系统可以暂停当前退款任务，先回答订单问题，后续再恢复原任务。

  

## 6. 知识问答模块

  

知识模块负责处理非流程型的咨询问题，例如：

  

- 商品信息咨询

- 订单信息咨询

- 退款政策咨询

- 退货政策咨询

- 配送政策咨询

- 平台规则咨询

  

知识问答流程如下：

  

```text

KnowledgeHandler

  -> 根据 intent 找到 provider_id

  -> 通过 KnowledgeProviderRegistry 获取 Provider

  -> Provider 检索商品、订单、FAQ 或 RAG 信息

  -> KnowledgeResponder 结合知识片段和对话历史调用 LLM 生成回复

```

  

项目中通过 Provider 机制解耦知识来源：

  

| Provider | 作用 |
|---|---|
| ProductAPIProvider | 调用商品 API 获取商品信息 |
| OrderAPIProvider | 调用订单和物流 API 获取订单信息 |
| FAQProvider | 提供 FAQ 类知识 |
| RAGProvider | 提供外部知识库检索能力 |

  

这种设计便于后续扩展更多知识来源，例如接入向量数据库、搜索引擎、客服知识库或内部文档系统。

  

## 7. 闲聊与澄清模块

  

闲聊模块用于处理普通非业务对话。它会参考用户当前消息和历史上下文，调用 LLM 生成自然回复，但不会推进任务流程，也不会修改业务槽位。

  

澄清模块用于处理系统无法安全判断用户意图的情况，例如：

  

- 缺少明确意图

- 同时命中多个意图

- 判断为任务但缺少任务命令

- 判断为知识问答但缺少知识意图

- 需要商品或订单对象但当前没有聚焦对象

- 用户只发送了订单或商品卡片，但没有说明要做什么

  

澄清模块的价值在于避免系统在不确定的情况下直接执行错误操作，特别适合客服、订单、售后这类对准确性要求较高的业务场景。

  

## 8. 状态持久化设计

  

系统通过 `DialogueState` 维护完整对话状态，包括：

  

- 用户 ID

- 当前会话

- 历史消息轮次

- 当前活跃任务

- 被暂停任务

- 当前系统任务

- 聚焦对象

- 任务槽位

  

数据库中使用 `dialogue_states` 表保存状态：

  

| 字段 | 说明 |
|---|---|
| sender_id | 用户 ID，主键 |
| state_json | 序列化后的完整 DialogueState |

  

每次处理用户消息时：

  

```text

load_state(sender_id)

  -> 反序列化 DialogueState

  -> 调用 DialogueEngine 修改状态

save_state(state)

  -> 序列化为 JSON

  -> upsert 到数据库

```

  

这种 JSON 快照式持久化方案实现简单，适合项目前期快速落地。后续如果需要做更细粒度的数据分析、审计和检索，可以进一步拆分出消息表、任务表和事件表。

  

## 9. 基础设施与中间件设计

  

基础设施层主要提供运行时资源：

  

| 模块 | 作用 |
|---|---|
| config | 从 `.env` 读取 LLM、数据库、商城 API、服务端口等配置 |
| database | 初始化 SQLAlchemy 异步数据库连接 |
| http_client | 初始化全局 httpx 异步客户端 |
| llm | 初始化 LangChain LLM 客户端 |

  

应用启动时通过 FastAPI lifespan 初始化资源：

  

```text

启动：

  init_http_client()

  init_dialogue_engine()

  init_db_engine()

  

关闭：

  close_db_engine()

  close_http_client()

```

  

当前项目中没有复杂的自定义中间件，但已经通过 FastAPI 的依赖注入和生命周期管理完成了基础设施编排。后续可以继续扩展：

  

- CORS 中间件

- 请求日志中间件

- Trace ID 中间件

- 全局异常处理中间件

- 鉴权中间件

- 限流中间件

- LLM 和外部 API 超时降级机制

  

## 10. 项目特点与亮点

  

### 10.1 不是简单 ChatBot，而是业务型对话系统

  

项目并不是简单地把用户问题转发给大模型，而是围绕真实电商客服业务设计了任务流、状态管理、知识检索和业务接口调用。

  

### 10.2 LLM 结构化规划，后端可控执行

  

系统让 LLM 生成结构化 `TurnPlan`，再由后端校验并执行。这样既利用了大模型的自然语言理解能力，又保证业务流程由后端规则控制。

  

### 10.3 支持多轮任务状态管理

  

项目维护 active task、paused tasks、slots、system task 等状态，能够支持槽位收集、任务中断、任务恢复等复杂多轮对话场景。

  

### 10.4 任务流程配置化

  

订单查询、物流查询、退款申请等流程通过 YAML 配置，新增业务流程时可以主要通过配置和 Action 扩展完成，降低代码侵入。

  

### 10.5 支持对象上下文理解

  

系统引入 focused object 机制，可以识别用户当前关注的订单或商品。例如用户发送订单卡片后，再问“这个什么时候到”，系统可以结合订单对象继续处理。

  

### 10.6 知识来源可扩展

  

知识问答模块通过 Provider 机制接入商品 API、订单 API、FAQ 和 RAG，后续可以方便扩展向量数据库、搜索服务或企业知识库。

  

### 10.7 模块边界清晰，易于扩展和维护

  

API、Service、Engine、Task、Knowledge、Infrastructure 各层职责明确，后续无论是新增业务流程、替换 LLM、扩展知识库还是增加中间件，都有清晰的扩展点。

  

## 11. 面试讲解版本

  

面试中可以这样介绍：

  

> 这个项目是一个电商智能客服后端，主要解决用户通过自然语言办理订单查询、物流查询、退款申请和商品咨询等问题。

>

> 项目的核心不是简单调用大模型，而是把大模型作为意图规划器，结合后端的状态机、任务流引擎和业务接口调用，完成可控的多轮业务对话。

>

> 用户消息进入系统后，应用服务层会先根据 sender_id 加载用户的对话状态，然后交给对话引擎。对话引擎会结合历史上下文、当前任务、可用流程和知识意图，让 LLM 生成结构化的 TurnPlan，再经过后端校验，最后分流到任务、知识问答、闲聊或澄清模块。

>

> 任务型能力通过 YAML 配置 flow，通过 ActionRunner 执行业务动作；知识型能力通过 Provider 机制接入商品、订单、FAQ 和 RAG 数据源。系统还维护了 active task、paused tasks、slots 和 focused object，因此可以支持多轮槽位收集、任务中断恢复以及订单或商品对象上下文理解。

>

> 我认为这个项目的特色在于，它没有把 LLM 当作黑盒聊天接口，而是把 LLM 放进一个可控的业务编排框架中，既能利用自然语言理解能力，又能保证后端流程、状态和业务接口调用是可管理、可扩展的。

  

## 12. 后续优化方向

  

后续可以从以下方向继续增强项目：

  

- 修复并完善服务层、状态模型和流程执行中的字段一致性问题

- 增加单元测试和集成测试，覆盖任务流、知识问答和状态持久化

- 增加全局异常处理和降级回复

- 增加请求日志、Trace ID 和调用链监控

- 将消息、任务、事件从 JSON 快照中拆分出来，支持更细粒度分析

- 接入真实 RAG 知识库和向量数据库

- 引入权限校验，保护订单和用户隐私数据

- 对 LLM 输出增加更严格的 schema 校验和重试机制

  
  
  
  

## flow

从项目当前设计看，`flow` 可以分成 **2 大类**：

  

1. `user flow`

   面向用户业务办理，定义在 [flow_config/user_flows.yml](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/flow_config/user_flows.yml)

  

2. `system flow`

   面向系统内部控制和提示，定义在 [flow_config/system_flows.yml](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/flow_config/system_flows.yml)

  

**当前 `user flow` 包括 6 个：**

  

- `onboarding`

- `order_status_query`

- `logistics_tracking`

- `refund_request`

- `similar_product_recommendation`

- `human_handoff`

  

**当前 `system flow` 包括 6 个：**

  

- `system_task_started`

- `system_task_resumed`

- `system_completed`

- `system_cannot_handle`

- `system_collect_information`

- `system_task_interrupted`

- `system_task_canceled`

  

严格说，文件里实际可见的是 7 个 `system flow`，所以项目当前总数是 **13 个 flow**。

  

如果你问的是“flow 内部的步骤类型”，那是另一层概念，当前步骤类型有 4 种：

  

- `start`

- `action`

- `collect`

- `end`

  

定义在 [src/task/flow/steps.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/task/flow/steps.py)。

  
  

`flow` 的步骤类型可以理解为任务流程里的“节点类型”。当前项目里定义了 **4 种步骤类型**，在 [src/task/flow/steps.py](D:/01_Study/01_Python/02_SGG/26_05_Platform_Operations/src/task/flow/steps.py)：

  

- `start`

- `action`

- `collect`

- `end`

  

它们分别解决不同阶段的问题。

  

**1. `start`**

表示一个 flow 的起点。

  

作用很简单：标记流程从哪里开始，然后立即跳到下一个步骤。它本身通常不做业务处理，只是流程入口。

  

例如：

  

```yaml

- id: start

  type: start

  next: ask_order_number

```

  

这表示流程启动后，先进入 `ask_order_number`。

  

**2. `collect`**

表示“收集槽位信息”，也就是向用户要参数。

  

这是任务流里最关键的一类步骤，常用于收集：

  

- 订单号 `order_number`

- 商品 ID `product_id`

- 退款原因 `refund_reason`

  

如果当前槽位没有值，系统会先向用户追问；如果已经有值，就继续往下走。

  

例如：

  

```yaml

- id: ask_order_number

  type: collect

  slot_name: order_number

  response:

    text: "请告诉我你的订单号。"

  next: lookup_logistics

```

  

意思是：

  

- 如果没有 `order_number`，就回复“请告诉我你的订单号”

- 如果已经拿到 `order_number`，就进入 `lookup_logistics`

  

**3. `action`**

表示执行一个动作。

  

动作可能是两类：

  

- 调用业务逻辑，比如查订单、查物流、查商品

- 生成回复，比如直接返回一段话术

  

例如：

  

```yaml

- id: lookup_logistics

  type: action

  action: action_lookup_logistics

  next: show_logistics

```

  

这个步骤会调用注册好的 `action_lookup_logistics`，查到物流信息后把结果写回槽位，再继续到 `show_logistics`。

  

再比如：

  

```yaml

- id: show_logistics

  type: action

  action: action_response

  args:

    text: "订单{{ slots.order_number }}由{{ slots.logistics_company }}配送..."

  next: end

```

  

这里的 `action` 不是查数据，而是把已有槽位渲染成用户可读的话术。

  

**4. `end`**

表示 flow 结束。

  

走到这个节点，说明当前流程处理完成了。对于用户任务流来说，一般意味着当前任务结束；对于系统 flow 来说，意味着这段系统提示流程结束。

  

例如：

  

```yaml

- id: end

  type: end

  next: []

```

  

**它们怎么配合工作**

  

一个典型流程通常像这样：

  

```text

start

-> collect

-> action

-> action

-> end

```

  

以“物流查询”为例：

  

```text

start

-> ask_order_number      collect

-> lookup_logistics      action

-> show_logistics        action

-> end                   end

```

  

含义就是：

  

1. 流程启动

2. 收集订单号

3. 调物流查询动作

4. 组织回复话术

5. 流程结束

  

**补充一点**

步骤类型和 flow 类型不是一回事：

  

- `flow` 类型：`user flow`、`system flow`

- `step` 类型：`start`、`collect`、`action`、`end`

  

前者是在说“这条流程属于什么用途”，后者是在说“流程中的某个节点干什么”。

  

如果你愿意，我可以接着把 `start -> collect -> action -> end` 结合“订单查询 flow”画成一张更直观的执行时序图。

  
  

以 `order_status_query` 这条 flow 为例，可以把 `start -> collect -> action -> end` 的执行过程理解成下面这张时序图。

  

```text

用户                  DialogueEngine / TaskHandler            FlowExecutor                 ActionRunner / 外部系统

 |                              |                                |                                 |

 |  "帮我查订单状态"             |                                |                                 |

 | ---------------------------> |                                |                                 |

 |                              |  start_flow(order_status_query) |                                 |

 |                              | ------------------------------> |                                 |

 |                              |                                |  进入 start 节点                |

 |                              |                                |  -> 跳到 ask_order_number       |

 |                              |                                |                                 |

 |                              |                                |  进入 collect 节点              |

 |                              |                                |  检查 slots.order_number        |

 |                              |                                |  发现没有值                     |

 |                              |                                |  启动 system_collect_information |

 |                              |                                |  返回 action_response           |

 |                              | <------------------------------ |                                 |

 |  "请告诉我你的订单号"         |                                |                                 |

 | <--------------------------- |                                |                                 |

 |                              |                                |                                 |

 |  "A20260410001"              |                                |                                 |

 | ---------------------------> |                                |                                 |

 |                              |  set_slots(order_number=...)   |                                 |

 |                              | ------------------------------> |                                 |

 |                              |                                |  再次进入 collect 节点          |

 |                              |                                |  发现 order_number 已有值       |

 |                              |                                |  -> 跳到 lookup_order_status    |

 |                              |                                |                                 |

 |                              |                                |  进入 action 节点               |

 |                              |                                |  调用 action_lookup_order_status|

 |                              |                                | ------------------------------> |

 |                              |                                |                                 | 查询订单接口

 |                              |                                | <------------------------------ |

 |                              |                                |  回填 slots.order_status        |

 |                              |                                |  回填 slots.order_summary       |

 |                              |                                |  -> 跳到 show_order_status      |

 |                              |                                |                                 |

 |                              |                                |  进入 action 节点               |

 |                              |                                |  调用 action_response           |

 |                              |                                | ------------------------------> |

 |                              |                                | <------------------------------ |

 |  "订单A20260410001当前状态是..."|                               |                                 |

 | <--------------------------- |                                |                                 |

 |                              |                                |  进入 end 节点                  |

 |                              |                                |  结束当前 flow                  |

 |                              |                                |                                 |

```

  

再压缩成节点视角，就是这样：

  

```text

start

  -> collect(order_number)

      没有订单号:

        回复“请告诉我你的订单号”

      有订单号:

        进入下一步

  -> action(action_lookup_order_status)

      查询订单状态

      写回 slots.order_status / slots.order_summary

  -> action(action_response)

      把 slots 渲染成最终回复

  -> end

```

  

对应到 `order_status_query` 的业务含义：

  

- `start`：启动“订单状态查询”流程

- `collect`：先收集 `order_number`

- `action`：调订单查询能力，拿到状态和摘要

- `action`：把结果组织成用户可读的话术

- `end`：结束本次任务

  

如果你是面试讲解，可以直接说：

  

> 订单查询 flow 本质上是一个小状态机。用户先触发 `start_flow(order_status_query)`，流程进入 `collect` 节点收集订单号；如果缺少订单号，就先追问用户；拿到订单号后进入 `action` 节点调用订单查询接口，把查询结果写回槽位；再经过一个 `action_response` 节点把槽位渲染成最终回复，最后到 `end` 节点结束流程。这样每个任务 flow 都是按统一的节点模型运行的。