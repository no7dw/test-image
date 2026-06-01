# maybe-gateway-overlay 基于 Hermes 的分阶段实施方案

这份文档记录 `maybe-gateway-overlay` 在 **Hermes 路线** 下的实施计划。

当前目标明确限定为：

- 先做 **POC**
- 先验证业务闭环，不先做重平台
- 暂时忽略或延后：
  - 海量高并发调度
  - 金融级安全与不出域
  - 私有化部署复杂形态
- DAG 工作流编排以 **Skill 驱动** 为主
- 企业级技能中心使用 `https://github.com/iflytek/skillhub`
- 失控任务熔断优先使用 **配置 + hook**
- 可审计、可追溯优先使用 **hook**

---

## 1. 目标与边界

### 1.1 本期目标

POC 阶段要验证的是：

- 现有 `Chat UI / App UI` 能接入 Hermes
- 用户请求能稳定映射到 Hermes session
- 业务任务能通过 `Skill` 驱动执行
- 轻量 DAG 能跑通
- 运行过程可审计、可中断、可追溯
- 技能能从企业技能中心下发到运行时

### 1.2 本期不解决的问题

以下能力明确不作为 POC 成功标准：

- 海量并发任务调度平台
- 多地域容灾
- 金融级密钥隔离与出域治理
- 企业级统一权限中台
- 通用工作流引擎
- 完整可视化编排平台

---

## 2. 总体架构

建议的 POC 架构如下：

```text
Chat UI / App UI
  -> maybe-gateway-overlay（薄接入层）
  -> Hermes gateway
  -> Hermes platform plugin: maybe_gateway_overlay
  -> AIAgent / skills / tools / memory
  -> hooks（audit / trace / breaker）
  -> SkillHub（技能发布与治理，不进入在线关键路径）
```

核心原则：

- `maybe-gateway-overlay` 先做薄接入层，不做大而全平台
- Hermes 负责 agent runtime、tools、skills、memory、gateway
- SkillHub 负责技能发布、版本、治理
- hooks 负责审计、追踪、熔断

---

## 3. 目录建议

如果按 Hermes 方案实施，建议目录分成三层：

### 3.1 Hermes 平台接入层

```text
hermes-agent/plugins/platforms/maybe_gateway_overlay/
  plugin.yaml
  adapter.py
```

职责：

- 接收来自 overlay/BFF 的消息
- 映射到 Hermes gateway 会话
- 负责平台侧收发消息

### 3.2 Hermes 审计与治理插件

```text
hermes-agent/plugins/observability/maybe_audit/
  plugin.yaml
  plugin_api.py
```

职责：

- 注册 hooks
- 记录 tool call / llm call / session lifecycle
- 做轻量熔断与风险阻断

### 3.3 技能目录

运行时本地技能目录：

```text
~/.hermes/skills/
```

企业技能中心导出目录：

```text
/opt/maybe/skillhub-export/
```

Hermes 通过 `skills.external_dirs` 扫描只读技能目录。

---

## 4. 分阶段方案

## Phase 0：边界收敛与最小骨架

### 目标

先把 POC 的边界锁死，避免一开始做成泛化平台。

### 要做的事

1. 确定统一入口协议
2. 确定用户标识与会话标识
3. 确定技能的来源与同步策略
4. 确定最小数据模型

### 建议结论

- `user_id`：使用你现有业务用户 ID
- `conversation_id`：映射到 Hermes session
- `tenant`：POC 可先弱化，不先做强租户体系
- `skills`：SkillHub 为发布源，Hermes 本地为运行副本

### 最小数据模型

建议至少准备这几张表：

```text
users
user_agent_binding
user_token_binding
task_runs
audit_events
```

### 本阶段交付物

- 一页架构图
- 数据模型草案
- 请求链路说明
- 最小目录骨架

---

## Phase 1：先打通入口链路

### 目标

让 `Chat UI / App UI` 能稳定进 Hermes，并能保持 session 连续性。

### 推荐策略

POC 优先使用“薄 BFF + Hermes”模式，不先强行做完整平台适配器体系。

链路如下：

```text
Chat UI / App UI
  -> maybe-gateway-overlay / BFF
  -> Hermes gateway
  -> session
```

### 本阶段职责划分

#### maybe-gateway-overlay / BFF

- 鉴权
- 提取 `user_id`
- 映射 `conversation_id`
- 统一请求格式
- 转发到 Hermes

#### Hermes

- 维护会话
- 调模型
- 调工具
- 执行技能

### 这一阶段不要做的事

- 不先做一人一个完整 Hermes profile
- 不先做多实例调度
- 不先做复杂租户资源隔离

### 本阶段交付物

- UI 发消息到 Hermes 成功
- 同一用户多轮对话能命中同一 session
- 能带入 `user_id` 上下文
- 能返回标准响应给 UI

---

## Phase 2：Skill 驱动业务流程

### 目标

把核心业务逻辑从“代码编排”切到“Skill 驱动”。

### 设计原则

不先做通用 DAG 引擎，而是把工作流拆成：

- `Skill`：表达流程规则
- `execute_code`：表达多步逻辑
- `delegate_task`：表达子任务并行或拆分

### 技能目录示例

```text
~/.hermes/skills/workflows/customer-onboarding/
  SKILL.md
  references/
  templates/
  scripts/
```

### 每类文件职责

- `SKILL.md`：流程定义、节点顺序、分支条件、人审点、失败回退
- `references/`：业务规则、SOP、接口约束
- `templates/`：固定输出格式
- `scripts/`：结构化辅助逻辑

### POC 阶段只支持两类流程

- 顺序流程
- 少量条件分支流程

### 状态持久化建议

先不引入复杂编排引擎，使用轻量持久化：

```text
task_runs 表
.hermes/workflows/<run-id>.json
```

### 本阶段交付物

- 至少 2 个业务 Skill 跑通
- 一个顺序型流程
- 一个带分支流程
- 能记录流程状态与输出

---

## Phase 3：接 SkillHub 做企业技能中心

### 目标

让技能具备企业级的发布、版本、治理能力。

### 关键原则

SkillHub 做 **发布源**，Hermes 做 **运行时消费方**。

不要让在线请求链路依赖 SkillHub 实时可用。

### 推荐分层

```text
SkillHub
  -> 技能审批 / 发布 / 版本管理
  -> 导出技能到只读目录
  -> Hermes 扫描该目录
```

### 推荐路径

```text
/opt/maybe/skillhub-export/
  workflows/
  finance/
  customer-service/
```

Hermes 配置：

```yaml
skills:
  external_dirs:
    - /opt/maybe/skillhub-export
```

### 建议的同步策略

1. SkillHub 中批准版本
2. 同步器导出到只读目录
3. Hermes 扫描只读目录
4. 本地 `~/.hermes/skills/` 只保留临时或运行时技能

### 本阶段交付物

- SkillHub 到 Hermes 的单向同步链路
- 技能版本号可追踪
- 技能来源可区分：本地 / SkillHub

---

## Phase 4：审计、追踪、熔断

### 目标

让系统具备最小企业可用性：

- 可审计
- 可追踪
- 可中断
- 可止损

### 建议使用 Hermes hooks

重点使用：

- `pre_tool_call`
- `post_tool_call`
- `pre_llm_call`
- `post_llm_call`
- `on_session_start`
- `on_session_end`

### 审计记录建议字段

```text
trace_id
run_id
parent_run_id
user_id
conversation_id
session_id
skill_name
tool_name
model_name
latency_ms
token_usage
status
error_code
```

### 熔断策略建议

POC 阶段先采用规则式熔断，不做复杂调度器：

1. 单次运行超时
2. 连续工具调用次数超限
3. 同一工具重复失败 N 次
4. 同一 Skill 在一个 run 中反复自旋
5. 人工 `/stop` 或外部 stop 指令

### 建议实现方式

#### 配置层

- `max_iterations`
- provider `request_timeout_seconds`
- provider `stale_timeout_seconds`
- tool `timeout`

#### hook 层

- `pre_tool_call` 统计调用次数
- `post_tool_call` 统计失败率
- 发现风险时直接 block

### 可视化追踪

如果需要快速看到 trace，可优先启用现成插件：

```text
plugins/observability/langfuse
```

### 本阶段交付物

- 每轮 agent 运行有 trace
- 每次 tool call 可追溯
- 熔断规则可生效
- 能人工终止任务

---

## Phase 5：再考虑平台化与扩容

### 目标

在 POC 已经证明业务价值后，再处理真正重的平台问题。

### 这一阶段再做的事情

- 多实例运行
- 更强租户隔离
- 队列与 worker
- 高并发调度
- 私有化部署形态
- 更严格的安全与出域控制

### 为什么放到最后

因为这些问题的代价很高，而 Hermes 路线最先需要验证的是：

- Skill 驱动流程是否适合你的业务
- hooks + config 是否足够支撑治理
- SkillHub + Hermes 的协作模式是否顺手

如果这三点没验证清楚，先建重平台只会放大错误方向。

---

## 5. POC 成功标准

POC 成功不看“架构是否完美”，只看是否跑通最关键闭环。

建议成功标准：

1. 现有 UI 能接入 Hermes
2. 同一用户多轮对话 session 稳定
3. 至少 2 个 Skill 驱动业务流程跑通
4. 至少 1 个轻 DAG 任务跑通
5. 审计日志可查
6. 任务可手动停止
7. SkillHub 能把技能下发给 Hermes

---

## 6. 推荐的实际开工顺序

建议按下面顺序推进：

1. 建入口链路
2. 建 session 绑定
3. 建 1 个核心 Skill 流程
4. 建 audit hook
5. 建 breaker hook
6. 接 SkillHub 导出目录
7. 补第二个业务流程

---

## 7. 当前推荐结论

对于本项目当前阶段，建议结论如下：

- 路线：先基于 Hermes 做 POC
- 重点：Skill 驱动业务流程，而不是先做编排平台
- 技能中心：SkillHub 做发布与治理，Hermes 做运行时消费
- 审计与熔断：优先用 hooks + config
- 调度与强安全：明确延后

一句话总结：

> 先把 `Hermes + Skill + Hook + SkillHub` 这条链路跑通，再决定是否值得往更重的平台方向演进。
