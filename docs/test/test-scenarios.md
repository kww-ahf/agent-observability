# 🧪 测试场景设计：Agent 可观测系统

> 关联文档：  
> PRD: [agent-tracing-system-prd.md](../prd/agent-tracing-system-prd.md)  
> Design: [agent-tracing-system-design.md](../design/agent-tracing-system-design.md)  
> 更新时间: 2026-03-19

---

## 1. 测试设计方法

本测试设计综合运用以下测试方法：

| 方法 | 应用场景 |
|------|----------|
| 等价类划分 | 查询参数有效/无效分组、trace_id 格式校验 |
| 边界值分析 | 分页大小、时间范围、并发量上限 |
| 判定表 | Schema 校验规则组合、错误码映射 |
| 状态转换 | Trace 生命周期（写入 → 可查 → 过期）、Collector pipeline 处理流 |
| 场景法 | 端到端业务场景、用户故事覆盖 |
| 错误猜测 | 存储不可用、网络中断、字段缺失等异常场景 |

---

## 2. 测试场景总览

| 场景编号 | 测试场景 | 关联需求 | 优先级 |
|----------|----------|----------|--------|
| TS-01 | 四层链路数据写入与完整性 | FR-1 | P0 |
| TS-02 | Trace 详情查询 | FR-2 / FR-3 | P0 |
| TS-03 | Agent / Conversation / 对话历史查询 | FR-2 | P0 |
| TS-04 | 异常与问题定位 | FR-4 | P1 |
| TS-05 | Schema 校验与兼容治理 | FR-5 | P1 |
| TS-06 | Collector pipeline 处理 | FR-1 / FR-5 | P1 |
| TS-07 | 性能与容量 | NFR-性能 | P1 |
| TS-08 | 可用性与容错 | NFR-可用性 | P1 |
| TS-09 | 数据安全与隔离 | NFR-安全 | P2 |

---

## 3. 测试场景详细描述

### TS-01：四层链路数据写入与完整性

**目标：** 验证业务系统和 Agent Runtime 按 OTel 规范产出的四层链路数据能被正确接收、处理和存储。

**覆盖面：**
- 请求链路（`agent.trace.type=request`）root span 创建
- 推理链路（`agent.trace.type=reasoning`）子 span 记录
- 工具链路（`agent.trace.type=tool`）子 span 记录
- 知识链路（`agent.trace.type=knowledge`）子 span 记录
- 四层链路通过 `trace_id` / `span_id` / `parent_span_id` 建立父子关联
- Events（`tool_invocation_started`、`knowledge_retrieved`、`response_composed`）记录
- 大字段摘要化存储（prompt、response、证据正文）

**前置条件：**
- OTel Collector 和 Data Prepper 正常运行
- OpenSearch 索引可写入
- 业务系统已集成 OTel SDK

**测试设计方法：**
- 等价类划分：完整数据 / 部分缺失 / 全部缺失
- 边界值：span 嵌套层级上限、attributes 数量上限

---

### TS-02：Trace 详情查询

**目标：** 验证 `GET /api/v1/traces/{trace_id}` 接口能正确返回符合 OTel TraceData 结构的完整链路数据。

**覆盖面：**
- 合法 `trace_id` 查询返回完整 `resourceSpans → scopeSpans → spans` 结构
- Span 层级关系正确（root span、child spans）
- 返回的 `gen_ai.*` 和 `agent.*` 属性字段正确
- Events 数据完整且时间戳正确
- 不存在的 `trace_id` 返回合理响应
- 非法格式 `trace_id` 返回 `400 INVALID_ARGUMENT`
- 查询超时返回 `504 QUERY_TIMEOUT`
- Partial trace 包含不完整状态标记

**前置条件：**
- 至少已写入一条完整 trace 数据
- Agent Observability Service 正常运行

**测试设计方法：**
- 等价类划分：有效 trace_id / 无效格式 / 不存在的 trace_id
- 边界值：包含最大/最小 span 数量的 trace
- 错误猜测：数据延迟、存储部分不可用

---

### TS-03：Agent / Conversation / 对话历史查询

**目标：** 验证已有接口（Agent 列表、Conversation 列表、对话历史消息查询）在可观测系统上下文中能正常工作，且查询结果与 trace 数据正确关联。

**覆盖面：**
- Agent 列表筛选（名称、业务线、时间范围）
- Conversation 列表分页与排序
- 对话历史按轮次展示
- 对话轮次与 `trace_id` 关联正确
- 分页参数校验（page_size 默认 20，最大 200）
- 时间范围限制（最大 168 小时）
- 排序逻辑正确

**前置条件：**
- 已有 Agent、Conversation 和对话数据
- 查询服务正常运行

**测试设计方法：**
- 等价类划分：有数据 / 无数据 / 大数据量
- 边界值：page_size=1 / page_size=200 / page_size=201；时间范围=0 / =168h / >168h
- 判定表：多条件组合筛选

---

### TS-04：异常与问题定位

**目标：** 验证系统能正确识别并高亮失败步骤、超时环节和可疑决策节点。

**覆盖面：**
- `status=error` 的 span 高亮标记
- 耗时超阈值的 span 识别与标记
- 错误节点包含错误码、错误消息、发生阶段和时间戳
- 链路不完整时显示数据缺口提示
- 日志或证据缺失时的降级展示
- 按工具维度聚合失败率（一期不支持聚合查询接口）

**前置条件：**
- 已写入包含错误和高耗时节点的 trace 数据
- SchemaGuard 标记的 `partial_trace` 数据存在

**测试设计方法：**
- 场景法：用户从失败请求列表 → trace 详情 → 定位根因
- 错误猜测：多层嵌套错误传播、部分子链路缺失

---

### TS-05：Schema 校验与兼容治理

**目标：** 验证 SchemaGuard 对链路数据的校验规则、`partial_trace` 标记和历史字段兼容映射。

**覆盖面：**
- 请求链路必填字段校验：`trace_id`、`span_id`、`agent_id`、`conversation_id`、`turn_id`、`agent.trace.type=request`
- 子链路必填 `parent_span_id` 校验
- `agent.trace.type` 枚举值校验（`request`、`reasoning`、`tool`、`knowledge`）
- 缺失关键字段时标记为 `partial_trace` 并记录 `schema_validation_failed` 事件
- 历史字段别名映射（如 `session_id` → `agent.session.id`）
- 兼容表版本化管理
- `schema_version` 字段正确写入

**前置条件：**
- 准备合规和不合规数据样本
- 配置历史字段别名映射

**测试设计方法：**
- 判定表：字段有/无 × 类型对/错 × 值合规/不合规
- 等价类划分：合规数据 / 缺少1个必填字段 / 缺少多个必填字段 / 全部缺失

---

### TS-06：Collector Pipeline 处理

**目标：** 验证 OTel Collector 的接收、批处理、属性增强、采样和导出流程正确性。

**覆盖面：**
- OTLP Receiver 正确接收 trace / log / metric 三类数据
- Attributes Processor 注入环境、租户、服务元数据
- Transform Processor 标准化公共字段和兼容历史别名
- Batch Processor 批量发送行为正确
- Memory Limiter 防止突发流量打爆 Collector
- Tail Sampling 规则生效（错误链路全量保留、成功链路按比例采样）
- Exporter 正确导出到 Data Prepper
- 导出失败时重试且不静默丢数

**前置条件：**
- Collector 已配置完整 pipeline
- 采样规则已下发

**测试设计方法：**
- 状态转换：数据从 Receiver → Processor → Exporter 的状态流转
- 边界值：批量大小上限、内存使用上限
- 错误猜测：下游不可用时的重试和反压

---

### TS-07：性能与容量

**目标：** 验证系统在目标负载下满足性能指标要求。

**覆盖面：**

| 指标 | 目标值 | 验证方式 |
|------|--------|----------|
| 单条 trace 精确查询 P95 | ≤ 1s | 负载测试 |
| Trace 详情首屏查询 P95 | ≤ 2s | 负载测试 |
| 查询吞吐基线 | 200 QPS | 压测 |
| 新数据 5 分钟内可查 | ≥ 95% | 端到端时效性测试 |
| 超长时间范围查询保护 | 限制生效 | 边界值测试 |

**前置条件：**
- 环境灌入足量 trace 数据（建议至少 100 万条）
- 压测工具就绪

**测试设计方法：**
- 边界值：QPS 从 1 递增到 200 再到 300
- 场景法：混合查询场景（列表 + 详情 + 聚合）

---

### TS-08：可用性与容错

**目标：** 验证系统在组件故障和异常条件下的降级与恢复能力。

**覆盖面：**
- 查询服务 SLA ≥ 99.9%
- 数据接收链路可用性 ≥ 99.95%
- OpenSearch 单节点不可用时查询降级行为
- Collector 副本 failover
- Vega 数据虚拟化超时降级
- Agent Observability Service 重启后恢复
- 配置回滚（Collector ConfigMap 恢复）
- 聚合分析失败时保留 trace 明细查询能力

**前置条件：**
- K8s 环境可模拟故障注入
- 监控告警配置就绪

**测试设计方法：**
- 错误猜测：各组件单点故障、级联故障
- 场景法：故障注入 → 观测降级行为 → 恢复 → 验证数据完整性

---

### TS-09：数据安全与隔离

**目标：** 验证跨租户数据隔离和敏感数据脱敏。（注：一期不含 RBAC / 审计日志 / 字段级ACL，仅验证基本安全措施）

**覆盖面：**
- 跨租户数据隔离（租户 A 无法查询租户 B 的 trace）
- 敏感字段脱敏存储与展示
- 大字段不直接进入索引（仅保存摘要和引用）

**前置条件：**
- 多租户 trace 数据已写入
- 脱敏规则已配置

**测试设计方法：**
- 等价类划分：本租户数据 / 其他租户数据 / 敏感字段 / 非敏感字段
- 错误猜测：通过伪造 trace_id 尝试跨租户访问

---

## 4. 测试环境要求

| 项目 | 要求 |
|------|------|
| K8s 集群 | ≥ 3 节点，支持多副本部署 |
| OpenSearch | ≥ 3 节点集群，配置 agent-traces/logs/metrics 索引模板 |
| OTel Collector | 多副本 Deployment，完整 pipeline 配置 |
| Data Prepper | 独立部署，支持索引路由 |
| Agent Observability Service | 部署完成，连通 ADP / Vega |
| 测试数据 | 性能测试 ≥ 100 万条 trace；功能测试覆盖正常 / 异常 / 边界数据 |
| 压测工具 | k6 / Locust / JMeter |
| 故障注入 | Chaos Mesh 或等效工具 |

---

## 5. 测试退出标准

| 类别 | 标准 |
|------|------|
| P0 测试场景 | 100% 通过 |
| P1 测试场景 | ≥ 95% 通过，剩余有规避方案 |
| P2 测试场景 | ≥ 80% 通过 |
| 性能指标 | 全部达标 |
| 严重缺陷 | 0 未关闭 |
| 一般缺陷 | 已有修复计划 |
