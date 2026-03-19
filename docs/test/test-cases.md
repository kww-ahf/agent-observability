# 🧪 测试用例设计：Agent 可观测系统

> 关联文档：  
> 测试场景: [test-scenarios.md](./test-scenarios.md)  
> PRD: [agent-tracing-system-prd.md](../prd/agent-tracing-system-prd.md)  
> Design: [agent-tracing-system-design.md](../design/agent-tracing-system-design.md)  
> 更新时间: 2026-03-19

---

## 用例编号规则

`TC-{场景编号}-{序号}`，例如 `TC-01-001` 表示场景 TS-01 的第 1 条用例。

优先级说明：P0 = 必须通过 | P1 = 应当通过 | P2 = 建议通过

---

## TS-01：四层链路数据写入与完整性

### TC-01-001：完整四层链路写入验证

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | OTel Collector、Data Prepper、OpenSearch 正常运行；业务系统已集成 OTel SDK |
| 测试步骤 | 1. 业务系统发起一次包含推理、工具调用和知识检索的 Agent 请求<br>2. 等待 5 分钟<br>3. 通过 `GET /api/v1/traces/{trace_id}` 查询该请求 |
| 预期结果 | 1. 返回 200，结构符合 OTel TraceData（`resourceSpans → scopeSpans → spans`）<br>2. Root span 的 `agent.trace.type = request`<br>3. 子 span 中分别包含 `reasoning`、`tool`、`knowledge` 类型<br>4. 所有 span 通过 `span_id` / `parent_span_id` 正确关联<br>5. `trace_id` 全链路一致 |
| 测试方法 | 场景法 |

### TC-01-002：请求链路 root span 字段完整性

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 同 TC-01-001 |
| 测试步骤 | 1. 发起 Agent 请求<br>2. 查询生成的 trace<br>3. 检查 root span 属性 |
| 预期结果 | root span 包含以下必填字段：<br>- `trace_id`（非空、格式正确）<br>- `span_id`（非空）<br>- `gen_ai.operation.name = invoke_agent`<br>- `agent.trace.type = request`<br>- `gen_ai.agent.name`（非空）<br>- `gen_ai.conversation.id`（非空）<br>- `agent.request.id`（非空）<br>- `parentSpanId` 为空（root span） |
| 测试方法 | 等价类划分 |

### TC-01-003：推理链路子 span 记录

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 同 TC-01-001 |
| 测试步骤 | 1. 发起触发模型推理的 Agent 请求<br>2. 查询 trace 详情<br>3. 筛选 `agent.trace.type = reasoning` 的 span |
| 预期结果 | 1. 推理 span 的 `parentSpanId` 指向 root span<br>2. 包含 `gen_ai.operation.name = chat`<br>3. 包含 `gen_ai.provider.name` 和 `gen_ai.request.model`<br>4. 包含 `agent.reasoning.step`（递增整数）<br>5. `startTimeUnixNano` < `endTimeUnixNano` |
| 测试方法 | 等价类划分 |

### TC-01-004：工具链路子 span 记录

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 同 TC-01-001 |
| 测试步骤 | 1. 发起触发工具调用的 Agent 请求<br>2. 查询 trace 详情<br>3. 筛选 `agent.trace.type = tool` 的 span |
| 预期结果 | 1. 工具 span 的 `parentSpanId` 正确指向调用方 span<br>2. 包含 `gen_ai.operation.name = execute_tool`<br>3. 包含 `gen_ai.tool.name` 和 `gen_ai.tool.type`<br>4. 包含 `agent.tool.status`（`ok` 或错误码）<br>5. 包含 `agent.tool.latency_ms`<br>6. Events 中包含 `tool_invocation_started` |
| 测试方法 | 等价类划分 |

### TC-01-005：知识链路子 span 记录

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 同 TC-01-001 |
| 测试步骤 | 1. 发起触发知识检索的 Agent 请求<br>2. 查询 trace 详情<br>3. 筛选 `agent.trace.type = knowledge` 的 span |
| 预期结果 | 1. 知识 span 的 `parentSpanId` 正确指向上层 span<br>2. 包含 `gen_ai.operation.name = retrieval`<br>3. 包含 `gen_ai.retrieval.query.text`<br>4. 包含 `gen_ai.knowledge.source_id`<br>5. Events 中包含 `knowledge_retrieved`，且 `agent.knowledge.hit` 为 bool |
| 测试方法 | 等价类划分 |

### TC-01-006：大字段摘要化存储

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 发起含长 prompt（>10KB）的请求 |
| 测试步骤 | 1. 发起包含超长 prompt 的 Agent 请求<br>2. 查询 trace 详情<br>3. 检查 prompt/response 相关字段 |
| 预期结果 | 1. prompt 和 response 正文不直接出现在 span attributes 中<br>2. 仅保存摘要或结构化引用<br>3. 不影响 trace 查询延迟 |
| 测试方法 | 边界值分析 |

### TC-01-007：多步骤推理链路顺序

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 发起触发多步推理的复杂 Agent 请求 |
| 测试步骤 | 1. 发起包含多步推理（≥3 步）的请求<br>2. 查询 trace<br>3. 按 `agent.reasoning.step` 排序 |
| 预期结果 | 1. reasoning span 的 `agent.reasoning.step` 从 1 递增<br>2. 每一步的时间戳按顺序递增<br>3. 步骤间的因果关系通过 span 层级正确表达 |
| 测试方法 | 状态转换 |

---

## TS-02：Trace 详情查询

### TC-02-001：正常 trace_id 查询

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 至少存在一条已完成写入的 trace 数据 |
| 测试步骤 | 1. 获取一个已知的有效 `trace_id`<br>2. 调用 `GET /api/v1/traces/{trace_id}` |
| 预期结果 | 1. 返回 200<br>2. 响应体包含 `resourceSpans` 数组<br>3. 内含 `resource.attributes`（包括 `service.name`）<br>4. `scopeSpans[].scope` 包含 `name` 和 `version`<br>5. `spans` 数组非空 |
| 测试方法 | 等价类划分（有效输入） |

### TC-02-002：不存在的 trace_id 查询

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 确保目标 trace_id 不存在于存储中 |
| 测试步骤 | 1. 构造一个格式合法但不存在的 `trace_id`<br>2. 调用 `GET /api/v1/traces/{trace_id}` |
| 预期结果 | 返回 404 或空的 `resourceSpans` 数组（取决于实现约定），不返回 500 |
| 测试方法 | 等价类划分（无效输入） |

### TC-02-003：非法格式 trace_id

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 无 |
| 测试步骤 | 1. 分别使用以下 trace_id 调用接口：<br>  - 空字符串<br>  - 特殊字符（`!@#$%`）<br>  - 超长字符串（>128字符）<br>  - SQL 注入 payload |
| 预期结果 | 返回 `400 INVALID_ARGUMENT`，包含可读错误信息，不暴露内部实现 |
| 测试方法 | 等价类划分 + 错误猜测 |

### TC-02-004：Trace 查询超时

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 模拟底层 OpenSearch 响应延迟（>1s） |
| 测试步骤 | 1. 注入存储层延迟<br>2. 调用 `GET /api/v1/traces/{trace_id}` |
| 预期结果 | 返回 `504 QUERY_TIMEOUT`，包含查询耗时和过滤条件摘要 |
| 测试方法 | 错误猜测 |

### TC-02-005：Partial trace 查询

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 存在被标记为 `partial_trace` 的数据 |
| 测试步骤 | 1. 查询一条已知 partial_trace 的 trace_id<br>2. 检查返回结构 |
| 预期结果 | 1. 返回 200（不丢弃数据）<br>2. 在异常 span 或 event 中标记不完整状态<br>3. 已有的 span 数据正常返回 |
| 测试方法 | 场景法 |

### TC-02-006：Span 层级结构正确性

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 存在包含 ≥4 层嵌套的 trace |
| 测试步骤 | 1. 查询该 trace<br>2. 遍历所有 span 验证父子关系 |
| 预期结果 | 1. 有且仅有一个 root span（`parentSpanId` 为空）<br>2. 所有非 root span 的 `parentSpanId` 指向已存在的 span<br>3. 不存在孤儿 span（除 root 外） |
| 测试方法 | 状态转换 |

### TC-02-007：时间戳一致性

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 存在完整 trace |
| 测试步骤 | 1. 查询 trace<br>2. 验证所有 span 的时间戳 |
| 预期结果 | 1. 所有 span 的 `startTimeUnixNano` < `endTimeUnixNano`<br>2. 子 span 的 `startTimeUnixNano` ≥ 父 span 的 `startTimeUnixNano`<br>3. 子 span 的 `endTimeUnixNano` ≤ 父 span 的 `endTimeUnixNano`<br>4. 时间戳为纳秒精度 |
| 测试方法 | 边界值分析 |

---

## TS-03：Agent / Conversation / 对话历史查询

### TC-03-001：Agent 列表查询

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 存在多个已注册的 Agent |
| 测试步骤 | 1. 调用 Agent 列表接口，不带筛选条件 |
| 预期结果 | 1. 返回 Agent 列表<br>2. 每条记录至少包含名称、时间、状态<br>3. 默认分页大小 20 |
| 测试方法 | 等价类划分 |

### TC-03-002：Conversation 列表分页

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 某 Agent 下有 >20 条 Conversation |
| 测试步骤 | 1. 查询 Conversation 列表，page_size=10<br>2. 翻页查询第 2 页<br>3. 查询 page_size=200<br>4. 查询 page_size=201 |
| 预期结果 | 1. 第 1 页返回 10 条，含翻页信息<br>2. 第 2 页数据与第 1 页不重复<br>3. page_size=200 合法返回<br>4. page_size=201 返回 400 或自动截取为 200 |
| 测试方法 | 边界值分析 |

### TC-03-003：时间范围查询边界

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 存在跨 7 天以上的数据 |
| 测试步骤 | 1. 查询时间范围 = 168 小时（7天）<br>2. 查询时间范围 = 169 小时<br>3. 查询时间范围 = 0<br>4. 查询 start_time > end_time |
| 预期结果 | 1. 168h 正常返回<br>2. 169h 返回 400 或自动截取为 168h<br>3. 范围=0 返回合理结果或 400<br>4. start>end 返回 400 |
| 测试方法 | 边界值分析 |

### TC-03-004：对话轮次与 trace_id 关联

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | Conversation 中有多轮对话 |
| 测试步骤 | 1. 查询某 Conversation 的对话历史<br>2. 获取某轮对话的 `trace_id`<br>3. 用该 `trace_id` 查询 trace 详情 |
| 预期结果 | 1. 每轮对话均有唯一 `trace_id`<br>2. 通过 `trace_id` 能查到完整链路<br>3. trace 中的 `gen_ai.conversation.id` 与 Conversation 一致 |
| 测试方法 | 场景法 |

### TC-03-005：空结果查询

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 无 |
| 测试步骤 | 1. 按不存在的 Agent 名称查询<br>2. 按未来时间范围查询 |
| 预期结果 | 返回空列表（非 null），HTTP status 200，附带分页元信息 |
| 测试方法 | 等价类划分 |

### TC-03-006：多条件组合筛选

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 存在多种 Agent、时间段和状态的数据 |
| 测试步骤 | 筛选条件组合：<br>1. Agent名称 + 时间范围<br>2. Agent名称 + 状态<br>3. 时间范围 + 状态<br>4. 三者同时指定 |
| 预期结果 | 每种组合返回的结果集均满足所有筛选条件，无遗漏和多余 |
| 测试方法 | 判定表 |

---

## TS-04：异常与问题定位

### TC-04-001：失败节点高亮

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 存在工具调用失败的 trace |
| 测试步骤 | 1. 查询包含失败 span 的 trace<br>2. 检查返回的 span status |
| 预期结果 | 1. 失败 span 的 `status` 包含 `code=ERROR`<br>2. 包含错误码和错误消息<br>3. 包含发生阶段（`agent.trace.type`）<br>4. 包含精确时间戳 |
| 测试方法 | 场景法 |

### TC-04-002：高耗时节点识别

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 存在某 span 耗时明显高于总耗时 50% 的 trace |
| 测试步骤 | 1. 查询该 trace<br>2. 计算每个 span 的耗时和占比 |
| 预期结果 | 1. 可从 `startTimeUnixNano` 和 `endTimeUnixNano` 计算出每步耗时<br>2. 高耗时 span 能被准确识别<br>3. `agent.tool.latency_ms` 值与时间戳差值一致 |
| 测试方法 | 边界值分析 |

### TC-04-003：链路不完整提示

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 存在 partial_trace（某些子链路 span 缺失） |
| 测试步骤 | 1. 查询 partial_trace<br>2. 检查响应中的不完整标记 |
| 预期结果 | 1. 返回的数据中存在 `schema_validation_failed` 事件<br>2. 缺失的子链路有明确的不完整标记<br>3. 已有的 span 仍然正常展示 |
| 测试方法 | 错误猜测 |

### TC-04-004：错误传播场景

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 工具调用失败导致整个请求失败的 trace |
| 测试步骤 | 1. 查询该 trace<br>2. 追踪错误从工具 span → 父 span → root span 的传播 |
| 预期结果 | 1. 工具 span 标记 ERROR<br>2. 父 span 状态反映子调用失败<br>3. root span 最终状态为 ERROR<br>4. 错误链路可从 root 到具体工具完整追溯 |
| 测试方法 | 场景法 |

---

## TS-05：Schema 校验与兼容治理

### TC-05-001：合规数据通过校验

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 准备一条包含所有必填字段的 trace |
| 测试步骤 | 1. 发送合规 trace 数据<br>2. 查询该 trace |
| 预期结果 | 1. 数据正常写入和查询<br>2. 不包含 `partial_trace` 标记<br>3. 不产生 `schema_validation_failed` 事件 |
| 测试方法 | 等价类划分 |

### TC-05-002：缺少 trace_id 的数据

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 准备缺少 `trace_id` 的 trace 数据 |
| 测试步骤 | 1. 通过 OTLP 发送缺少 `trace_id` 的数据<br>2. 检查 SchemaGuard 校验结果 |
| 预期结果 | 1. 数据不被丢弃<br>2. 标记为 `partial_trace`<br>3. 产生 `schema_validation_failed` 事件<br>4. 告警被触发 |
| 测试方法 | 判定表 |

### TC-05-003：缺少 agent.trace.type 的数据

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 准备缺少 `agent.trace.type` 属性的 trace 数据 |
| 测试步骤 | 1. 发送数据<br>2. SchemaGuard 校验 |
| 预期结果 | 标记为 `partial_trace`，产生告警，数据不丢弃 |
| 测试方法 | 判定表 |

### TC-05-004：agent.trace.type 非法枚举值

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 准备 `agent.trace.type = invalid_type` 的数据 |
| 测试步骤 | 1. 发送 `agent.trace.type` 为非枚举值的数据<br>2. SchemaGuard 校验 |
| 预期结果 | 标记为 `partial_trace`，记录非法值，数据不丢弃 |
| 测试方法 | 等价类划分 |

### TC-05-005：历史字段别名映射

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 启用兼容别名（`schema.compat_alias.enabled = true`） |
| 测试步骤 | 1. 发送使用旧字段名 `session_id` 的数据<br>2. 通过查询验证映射 |
| 预期结果 | 1. `session_id` 正确映射为 `agent.session.id`<br>2. 查询使用 `agent.session.id` 可查到数据<br>3. 原始字段不产生校验告警 |
| 测试方法 | 场景法 |

### TC-05-006：Schema 版本字段

| 项目 | 内容 |
|------|------|
| 优先级 | P2 |
| 前置条件 | 配置 `schema.version = v1` |
| 测试步骤 | 1. 发送 trace 数据<br>2. 查询 OpenSearch 确认 `schema_version` 字段 |
| 预期结果 | 写入的文档包含 `schema_version = v1` |
| 测试方法 | 等价类划分 |

---

## TS-06：Collector Pipeline 处理

### TC-06-001：OTLP 三类信号接收

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | Collector OTLP Receiver 已启动 |
| 测试步骤 | 1. 分别通过 OTLP 发送 trace、log、metric 数据<br>2. 验证 Collector 接收日志 |
| 预期结果 | 三类信号均被正确接收，无拒绝、无丢失 |
| 测试方法 | 等价类划分 |

### TC-06-002：Attributes Processor 元数据注入

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | Attributes Processor 已配置环境和租户元数据 |
| 测试步骤 | 1. 发送不含环境、租户标签的 trace<br>2. 在 OpenSearch 中查询落地数据 |
| 预期结果 | 落地数据中自动补充了环境标签和租户元数据 |
| 测试方法 | 场景法 |

### TC-06-003：Tail Sampling 错误链路全量保留

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 采样配置：`error_keep_ratio = 1.0`，`success_keep_ratio = 0.2` |
| 测试步骤 | 1. 发送 100 条错误 trace 和 100 条成功 trace<br>2. 查询 OpenSearch 落地数量 |
| 预期结果 | 1. 100 条错误 trace 全部保留<br>2. 成功 trace 保留约 20 条（允许统计误差） |
| 测试方法 | 边界值分析 |

### TC-06-004：Batch Processor 行为

| 项目 | 内容 |
|------|------|
| 优先级 | P2 |
| 前置条件 | Batch Processor 配置 batch_size 和 send_interval |
| 测试步骤 | 1. 快速发送少于 batch_size 的数据<br>2. 等待 send_interval 超时<br>3. 快速发送超过 batch_size 的数据 |
| 预期结果 | 1. 少量数据在 send_interval 后被发送<br>2. 大量数据在达到 batch_size 时立即发送 |
| 测试方法 | 边界值分析 |

### TC-06-005：Memory Limiter 保护

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | Memory Limiter 配置上限 |
| 测试步骤 | 1. 发送超大量突发数据使 Collector 内存接近上限<br>2. 观察 Collector 行为 |
| 预期结果 | 1. Collector 不 OOM<br>2. 触发反压或拒绝新数据<br>3. 内存降低后自动恢复接收 |
| 测试方法 | 错误猜测 |

### TC-06-006：Exporter 导出失败重试

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | Data Prepper 暂时不可用 |
| 测试步骤 | 1. 关闭 Data Prepper<br>2. 发送 trace 数据<br>3. 等待重试<br>4. 恢复 Data Prepper |
| 预期结果 | 1. 不静默丢失数据<br>2. 输出错误日志<br>3. Data Prepper 恢复后数据补发成功 |
| 测试方法 | 错误猜测 |

---

## TS-07：性能与容量

### TC-07-001：单条 trace 查询延迟

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | OpenSearch 中已有 ≥100 万条 trace，查询服务运行正常 |
| 测试步骤 | 1. 随机选取 1000 个 trace_id<br>2. 并发调用 `GET /api/v1/traces/{trace_id}`<br>3. 记录响应时间 |
| 预期结果 | P95 响应时间 ≤ 1 秒 |
| 测试方法 | 负载测试 |

### TC-07-002：查询吞吐量

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 同 TC-07-001 |
| 测试步骤 | 1. 使用压测工具逐步增加 QPS：50 → 100 → 200 → 300<br>2. 监控成功率和延迟 |
| 预期结果 | 1. 200 QPS 下成功率 ≥ 99%<br>2. P95 延迟 ≤ 2 秒<br>3. 300 QPS 下系统不崩溃（可降级） |
| 测试方法 | 边界值分析 |

### TC-07-003：数据时效性

| 项目 | 内容 |
|------|------|
| 优先级 | P0 |
| 前置条件 | 端到端链路正常 |
| 测试步骤 | 1. 记录发送时间 T0，发送 100 条 trace<br>2. 每 30 秒查询一次<br>3. 记录首次可查到的时间 T1 |
| 预期结果 | ≥ 95% 的 trace 在 T0 + 5 分钟内可查询 |
| 测试方法 | 场景法 |

### TC-07-004：超长时间范围查询保护

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | `query.max_time_range_hours = 168` |
| 测试步骤 | 1. 发起时间范围 = 200 小时的查询<br>2. 发起不指定时间范围的查询 |
| 预期结果 | 1. 超范围查询被拒绝或截取<br>2. 不指定范围时使用默认限制<br>3. 不因大范围查询导致服务雪崩 |
| 测试方法 | 边界值分析 |

---

## TS-08：可用性与容错

### TC-08-001：OpenSearch 单节点故障

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | OpenSearch 3 节点集群 |
| 测试步骤 | 1. 关闭 1 个 OpenSearch 节点<br>2. 执行 trace 查询<br>3. 发送新 trace 数据 |
| 预期结果 | 1. 查询服务仍可用<br>2. 数据写入仍成功<br>3. 节点恢复后数据同步 |
| 测试方法 | 错误猜测 |

### TC-08-002：Collector Pod 重启

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | Collector 多副本部署 |
| 测试步骤 | 1. 滚动重启 Collector Pod<br>2. 持续发送数据<br>3. 检查数据丢失率 |
| 预期结果 | 1. 至少一个 Pod 可用<br>2. 数据丢失率趋近于 0<br>3. 重启完成后恢复全量处理 |
| 测试方法 | 场景法 |

### TC-08-003：Vega 数据虚拟化超时

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | Vega 模拟超时 |
| 测试步骤 | 1. 注入 Vega 延迟（>2s）<br>2. 调用查询接口 |
| 预期结果 | 1. 返回 504 超时错误<br>2. 不阻塞其他查询请求<br>3. Vega 恢复后查询正常 |
| 测试方法 | 错误猜测 |

### TC-08-004：查询服务重启恢复

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | Agent Observability Service Deployment |
| 测试步骤 | 1. 重启 Agent Observability Service Pod<br>2. 检查服务就绪时间<br>3. 立即执行查询 |
| 预期结果 | 1. 服务在合理时间内（<30s）就绪<br>2. 就绪后查询功能正常<br>3. 采集链路不受影响 |
| 测试方法 | 场景法 |

### TC-08-005：存储不可用时错误返回

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | OpenSearch 完全不可用 |
| 测试步骤 | 1. 关闭 OpenSearch 集群<br>2. 调用查询接口 |
| 预期结果 | 1. 返回 `503 STORAGE_UNAVAILABLE`<br>2. 触发告警<br>3. 不返回 500 或不可读错误 |
| 测试方法 | 错误猜测 |

---

## TS-09：数据安全与隔离

### TC-09-001：跨租户数据隔离

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 租户 A 和租户 B 各有 trace 数据 |
| 测试步骤 | 1. 以租户 A 身份查询<br>2. 尝试使用租户 B 的 trace_id 查询 |
| 预期结果 | 租户 A 无法查询到租户 B 的数据 |
| 测试方法 | 等价类划分 |

### TC-09-002：敏感字段脱敏

| 项目 | 内容 |
|------|------|
| 优先级 | P2 |
| 前置条件 | trace 中包含敏感信息（用户输入、模型输出等） |
| 测试步骤 | 1. 发送包含敏感内容的 trace<br>2. 查询 trace 详情 |
| 预期结果 | 1. 敏感字段已脱敏展示<br>2. OpenSearch 中存储的也是脱敏后数据 |
| 测试方法 | 场景法 |

### TC-09-003：SQL 注入防护

| 项目 | 内容 |
|------|------|
| 优先级 | P1 |
| 前置条件 | 无 |
| 测试步骤 | 在查询参数中注入 SQL/DSL payload：<br>1. `trace_id = "'; DROP TABLE traces;--"`<br>2. Agent 名称筛选传入 DSL 语句 |
| 预期结果 | 1. 返回 400 参数错误<br>2. 不执行任何注入的操作<br>3. 不泄露内部错误信息 |
| 测试方法 | 错误猜测 |

---

## 用例覆盖矩阵

| 需求 | P0 用例数 | P1 用例数 | P2 用例数 | 总计 |
|------|-----------|-----------|-----------|------|
| FR-1 四层链路写入 | 5 | 2 | 0 | 7 |
| FR-2 / FR-3 Trace 查询 | 4 | 3 | 0 | 7 |
| FR-2 列表查询 | 2 | 3 | 1 | 6 |
| FR-4 异常定位 | 1 | 3 | 0 | 4 |
| FR-5 Schema 治理 | 2 | 3 | 1 | 6 |
| Collector Pipeline | 2 | 2 | 2 | 6 |
| 性能 | 3 | 1 | 0 | 4 |
| 可用性 | 0 | 5 | 0 | 5 |
| 安全 | 0 | 2 | 1 | 3 |
| **合计** | **19** | **24** | **5** | **48** |
