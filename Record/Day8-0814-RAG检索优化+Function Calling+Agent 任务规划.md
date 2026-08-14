## RAG检索优化

```text
candidateTopK：第一阶段最多保留多少个候选，主要控制覆盖机会与后续成本
threshold：拒绝低于最低相关性要求的候选，主要控制噪声入口
rerank：用第二套、更细的 query-chunk 相关性信号重新排序候选
finalTopK：重排后最多送入上下文多少条，主要控制证据密度与 Token 成本
```

实现流程
```
query
  → 第一阶段打分与候选召回（candidateTopK）
  → 相似度门槛（threshold）
  → 第二阶段相关性重排（rerank）
  → 最终截断（finalTopK / context budget）
  → 空结果程序化拒答，或构造 context
```

相关注意问题：
```
Rerank只看第一阶段已经查询过的候选Key个数据块，如果真正相关的数据块因为候选key太小或者阈值太高被丢弃，Rerank就没办法查询到真正相关的数据

一次两个阶段的检索一般都取宽候选集，在重排序到最后的数据库，候选级通常越宽，召回机会通常越高，但是重排序相关的延迟也会变高，成本也会变高

```

#### 召回率和准确率

```

Precision@K = 前 K 条中相关结果数 / 实际返回结果数
Recall@K    = 前 K 条中相关结果数 / 该查询全部应相关结果数
```

## Function Calling

大模型根据用户问题调用工具的流程
```
```text
1. 应用把用户消息和允许使用的工具定义一起发给模型
2. 模型决定直接回答，或返回 tool call 请求
3. 后端只在白名单中解析工具名，并校验参数
4. 后端结合 JWT / 服务端上下文做权限检查
5. Dispatcher 调用对应 Service，捕获超时和业务异常
6. 后端把结构化 ToolResult 回传给模型
7. 模型使用工具结果生成最终回答；必要时还可能申请下一次调用
```

#### 大模型和后端的功能

模型可以负责 | 后端必须负责 |
|---|---|
| 根据语义选择候选工具 | 决定本次请求实际暴露哪些工具 |
| 按 schema 生成候选参数 | 反序列化、类型校验和业务校验 |
| 根据工具结果组织回答 | 身份、权限、额度、幂等和确认机制 |
| 在结果不足时提出下一步 | 真正访问数据库、缓存、文件或外部 API |

#### Function Calling 和 Agent

```
Function Calling：模型与应用之间交换“调用哪个工具、使用哪些参数”的结构化协议/能力。
Agent：围绕目标维护状态，可能反复进行观察、规划、工具调用、结果判断和停止决策的系统。
```

## Agent最小的执行模型

根据用户传来的消息解析用户的目标然后根据目标来进行相应的操作
```
Goal
  → 读取 State
  → Decide：选择下一动作
  → Act：回答、追问或请求工具
  → Observation：接收真实结果
  → Update State：记录事实、进度、失败与预算
  → 检查 Stop Condition
      ├─ 满足：Final / Ask / Refuse / Fail
      └─ 未满足：进入下一轮 Decide
```
各部分的职责
部分 | 要回答的问题 | 面试教练示例 |
|---|---|---|
| Goal | 最终要达成什么 | 判断用户的 Java 集合薄弱点并给出练习建议 |
| State | 已知什么、做过什么、还差什么 | 已完成诊断；HashMap 得分低；还未给针对性反馈 |
| Decide | 现在最合适的下一步是什么 | 调用题库工具找一道 HashMap 追问题 |
| Act | 执行哪个受控动作 | `searchQuestion(keyword="HashMap")` |
| Observation | 环境真实返回了什么 | 找到 3 题，或成功但结果为空，或工具超时 |
| Update | 哪些事实和进度需要写回 | 记录候选题、失败次数、剩余轮次 |
| Stop | 为什么结束 | 目标完成、需要用户输入、越权拒绝、达到最大轮次

#### Workflow 和 Agent

```
Workflow：步骤和分支主要由代码预先规定，适合路径清晰、稳定性要求高的任务。
Agent：模型根据目标、状态和 Observation 动态选择下一动作，适合难以预先确定步骤的任务。
```

#### AgentState

```text
goal             用户已确认的目标
constraints      时间、主题、权限和禁止事项
status           RUNNING / WAITING_USER / SUCCEEDED / FAILED / REFUSED
completedSteps   已完成的可验证步骤
currentStep      当前准备执行的动作
observations     工具返回的结构化事实摘要
retryCounts      各动作已重试次数
lastError        最近一次稳定错误分类
turn             当前轮次
maxTurns         最大轮次
stopReason       本次结束的明确原因
```

#### 停止条件

```
SUCCESS                目标已达到，输出最终结果
NEED_USER_INPUT        缺少只能由用户决定的信息
REFUSED                目标或动作越权、危险或不允许
MAX_TURNS_REACHED      达到最大轮次，防止无限循环
BUDGET_EXHAUSTED       时间、Token、费用或工具额度耗尽
NON_RETRYABLE_FAILURE  出现不可重试失败
CANCELLED              用户或系统取消
```

#### 失败后的执行策略
Observation | 正确方向 | 不应直接做什么 |
|---|---|---|
| 参数缺失或类型错误 | 修正参数；无法推断时询问用户 | 用同样参数原样重试 |
| 搜索成功但结果为空 | 改关键词、换工具或说明无结果 | 标记为工具执行失败 |
| 短暂超时/限流 | 在预算内退避重试；先检查幂等性 | 无限快速重试 |
| 越权或危险动作 | 拒绝并停止该动作 | 通过换工具绕过权限 |
| 写工具结果未知 | 先查幂等键或操作状态 | 盲目重试并制造重复副作用 |
| 目标含糊 | 追问决定路线的关键信息 | 擅自假设用户偏好 |


1. planner 只决定下一动作，Dispatcher 仍掌握真实工具执行边界。
2. Observation 无论成功或失败都要写回 State，下一轮才能调整。
3. 追问、拒绝、成功和失败都是合法停止出口。
4. maxTurns 是最后一道机械保险，不应代替业务完成条件。
5. retryPolicy 不能只看“失败”，还要看错误类型、幂等性和预算。
