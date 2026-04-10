# Corpus / Organ / Nerve 约束文档（AI 可执行版）
status: draft
version: 0.1
source: Corpus.md
audience:
  - LLM-agent
  - tool-agent
  - developer
purpose:
  - 把“设计思想”转成可被 AI 与工程同时遵守的硬约束与接口契约
  - 作为 Organ 接入与 Corpus 实现的验收基线

## 0. 规范性用语
- MUST：必须满足，否则不兼容
- MUST NOT：禁止，否则不兼容
- SHOULD：强烈建议；偏离需要明确理由与替代控制
- MAY：可选

## 0.1 适用范围与非目标
- 本文档约束的是跨项目稳定的角色边界、系统不变量与协议语义能力。
- 本文档不规定：具体字段名、序列化格式、传输协议、具体技术栈、具体业务流程与领域模型。
- 若子项目需要字段级强约束，应在子项目内提供补充规范（例如 JSON Schema/IDL/OpenAPI），并声明其与本文档的映射关系。

## 0.2 兼容性与演进
- 子项目在扩展 Nerve 时 MUST 保持后向兼容：新增语义能力不得破坏既有调用方的基础路径。
- 子项目若引入破坏性变更 MUST 提升主版本并提供迁移策略（例如双写、桥接、灰度）。
- 子项目 SHOULD 提供能力协商或特性声明机制，以支持新旧 Organ 并存与渐进演进。

## 1. 角色与边界（不可替代定义）
### 1.1 Corpus（本体/中枢）
- Corpus MUST 作为全局状态与决策的唯一真值中心（Single Source of Truth）。
- Corpus MUST 负责：状态跳转、资源仲裁、安全审计、失败处理的最终裁决。
- Corpus MUST NOT 直接实现具体业务能力（例如模型推理、检索、外部系统调用）；这些 MUST 由 Organ 承担。

### 1.2 Organ（器官/枢翼）
- Organ MUST 是可替换的执行单元；更换 Organ 不得要求更换 Corpus 的核心逻辑。
- Organ SHOULD 运行在隔离边界内（独立进程或等价隔离）；其崩溃 MUST NOT 导致 Corpus 不可用。
- Organ MUST 仅通过 Nerve 与 Corpus 通信。

### 1.3 Nerve（神经/协议）
- Nerve MUST 是 Corpus 与 Organ 的唯一通信通道。
- Nerve MUST 可机器解析、可版本化、无歧义。
- Nerve MUST 支持：请求/响应、健康检查、注册/发现、错误语义、审计与追踪字段。

## 2. 系统不变量（AI 与工程共同遵守）
### 2.1 主权与真值
- 全局状态变更 MUST 由 Corpus 发起或确认。
- 任何 Organ 的输出 MUST 被视为“建议/证据/候选结果”，除非 Corpus 明确采纳并产生可审计的决策记录。

### 2.2 确定性与可追溯
- Corpus MUST 将概率性/不确定性输出转化为确定性行为：要么采纳（commit），要么拒绝（reject），要么请求补充证据（need_more_evidence）。
- 每次采纳或拒绝 MUST 生成决策记录（Decision Record），可用于回放与审计。

### 2.3 隔离与容错
- Organ 失败（超时、异常、崩溃）时，Corpus MUST 保持自身可用，并执行明确定义的降级/回退路径。
- Corpus SHOULD 对 Organ 实施：超时、重试策略、熔断、并发/资源配额。

### 2.4 安全与最小权限
- Organ MUST 具有身份（organ_id + 版本 + 可选签名材料）。
- Corpus MUST 实施最小权限：Organ 只能调用其被授权的 capability 与数据范围。
- 审计日志 MUST 不记录敏感明文（密钥、令牌、个人敏感信息）；必要时记录哈希或脱敏摘要。

## 3. Nerve 消息契约（语义约束）
> **说明：** 本节约束的是**语义能力**，不规定具体字段名、序列化格式或传输协议。子项目 MAY 使用 JSON、Protobuf、Avro 或任何等价格式。下方 JSON 结构均为**参考示例**，仅用于说明需要承载哪些语义，不构成字段级强制约束。

### 3.1 统一信封（Envelope）
每条消息 MUST 在信封层承载以下语义能力；具体字段命名与结构由子项目自行决定。

```json
{
  "nerve_version": "1.0",
  "message_type": "invoke|register|heartbeat|health|shutdown",
  "request_id": "uuid-or-snowflake",
  "trace_id": "w3c-traceparent-compatible-or-uuid",
  "timestamp_ms": 0,
  "from": { "kind": "corpus|organ", "id": "string" },
  "to": { "kind": "corpus|organ", "id": "string" },
  "auth": { "scheme": "mTLS|token|sigv1", "credential_ref": "string" },
  "body": {},
  "constraints": {
    "timeout_ms": 0,
    "idempotency_key": "string",
    "priority": "low|normal|high"
  }
}
```

语义约束（下方字段名仅供参考）：
- 调用唯一标识 MUST 全局唯一，并贯穿一次业务调用链。
- 分布式追踪标识 SHOULD 支持跨服务关联。
- 幂等键 SHOULD 在会触发外部副作用的调用中提供。

### 3.2 调用（invoke）
```json
{
  "capability": "string",
  "input": {},
  "context": {
    "tenant_id": "string",
    "user_id": "string",
    "locale": "string"
  },
  "evidence_requirements": {
    "need_citations": true,
    "need_intermediate_steps": false
  }
}
```

### 3.3 响应（invoke result）
```json
{
  "status": "ok|error|declined",
  "output": {},
  "confidence": 0.0,
  "evidence": {
    "citations": [],
    "artifacts": []
  },
  "usage": { "latency_ms": 0 },
  "error": {
    "code": "string",
    "message": "string",
    "retryable": false
  }
}
```

语义约束（下方字段名仅供参考）：
- 置信度 MUST 在 [0, 1] 区间。
- 当响应为错误时，错误信息 MUST 存在。
- 当响应为成功时，输出内容 MUST 存在（可以为空）。

### 3.4 注册（register）
Organ 启动后 SHOULD 先 register，再进入 heartbeat/health。

```json
{
  "organ": {
    "organ_id": "string",
    "organ_version": "semver",
    "capabilities": [
      {
        "name": "string",
        "input_schema_ref": "string",
        "output_schema_ref": "string",
        "side_effects": "none|external",
        "slo": { "timeout_ms": 0 }
      }
    ],
    "resources": {
      "cpu": "string",
      "memory": "string"
    }
  }
}
```

### 3.5 健康与心跳（health / heartbeat）
- Corpus MUST 能区分：存活（heartbeat）与就绪（health）。
- Organ SHOULD 上报：版本、负载、队列长度、最近错误摘要（脱敏）。

## 4. Corpus 的“确定性转化”约束（决策器最小语义）
Corpus MUST 对每次 Organ 输出生成决策记录（Decision Record）。以下 JSON 为**参考示例**，子项目可用等价结构替代，不要求字段名一致。

```json
{
  "decision_id": "string",
  "request_id": "string",
  "capability": "string",
  "inputs_digest": "hash",
  "organ_id": "string",
  "organ_version": "string",
  "organ_output_digest": "hash",
  "confidence": 0.0,
  "decision": "commit|reject|need_more_evidence|fallback",
  "policy_version": "string",
  "reasons": ["string"],
  "timestamp_ms": 0
}
```

语义约束（下方字段名仅供参考）：
- Corpus MUST NOT 以"默认采纳"方式吞掉不确定性；每次采纳或拒绝 MUST 有可审计的理由记录。
- 决策所依据的策略版本 MUST 可追溯（例如配置版本、策略表版本）。

## 5. 失败语义（统一处理）
- 超时：当超过约定超时时限，Corpus MUST 终止等待，并进入 fallback 或 reject。
- 重试：仅当 Organ 明确标记可重试且调用无外部副作用时，Corpus MAY 自动重试。
- 熔断：当某 Organ 在窗口期内持续失败，Corpus SHOULD 暂停向其派发并切换备用 Organ。
- 降级：当所有候选 Organ 不可用，Corpus MUST 返回可预期的降级结果，并记录审计事件。

## 6. 安全与合规（最小要求）
- Corpus MUST 有 Organ 白名单/信任策略（至少基于 organ_id + 版本约束）。
- Nerve MUST 支持认证（mTLS/token/signature 任选一种或组合），并可在审计中引用可追溯的凭证标识（不记录凭证明文）。
- 所有审计事件 MUST 脱敏；PII/密钥/令牌 MUST NOT 落盘。

## 7. 可观测性（调试与运营必需字段）
- 每条消息 MUST 带 request_id；SHOULD 带 trace_id。
- 每次决策 MUST 产生审计事件：开始、调用、结果、裁决、状态变更（可选）。

## 8. Organ 接入验收清单（可自动化）
- 仅使用 Nerve 通信；无旁路调用（MUST）。
- 提供 register + health + heartbeat（SHOULD）。
- capability 输入输出 SHOULD 可机器验证（如 JSON Schema、IDL 或等价机制）。
- 输出 SHOULD 包含置信度与溯源证据（当业务需要可追溯时 MUST，按 Corpus 策略启用）。
- 支持超时/取消（SHOULD）。

## 9. 合规矩阵（机器可读）
```yaml
conformance:
  - id: CORP-SSOT-001
    subject: Corpus
    level: MUST
    rule: 全局状态与决策的唯一真值中心
    evidence:
      - type: decision_record
        note: 决策记录中可推导出“由 Corpus 产出/确认”的决策事实
      - type: state_transition_log
        note: 状态跳转由 Corpus 统一记录或对外发布
  - id: CORP-ARBITRATION-001
    subject: Corpus
    level: MUST
    rule: 状态跳转/资源仲裁/安全审计/失败处理具备最终裁决权
    evidence:
      - type: decision_record
        note: 记录裁决结果与理由
      - type: audit_event
        note: 记录仲裁/安全/失败处理事件（脱敏）
  - id: CORP-NO-BIZ-001
    subject: Corpus
    level: MUST_NOT
    rule: 直接承载具体业务能力实现（模型推理/检索/外部系统调用等）
    evidence:
      - type: module_boundary_doc
        note: 子项目提供模块边界说明，证明业务能力实现位于 Organ 或等价边界外
  - id: ORG-ISOLATION-001
    subject: Organ
    level: SHOULD
    rule: 具备隔离边界且故障不影响 Corpus 可用性
    evidence:
      - type: deployment_boundary
        note: 进程/DLL/容器/沙箱等隔离边界的说明或配置（具体形式由子项目决定）
      - type: fault_test_record
        note: 注入故障后 Corpus 仍可对外服务的验证记录（可为自动化或人工演练）
  - id: NERVE-ONLY-001
    subject: Nerve
    level: MUST
    rule: Corpus 与 Organ 的唯一通信通道
    evidence:
      - type: nerve_message
        note: 所有跨边界交互均可在 Nerve 消息或其等价载体中追溯
      - type: audit_event
        note: 旁路调用检测或禁止策略的审计事件（若子项目实现）
  - id: NERVE-SEMANTIC-001
    subject: Nerve
    level: MUST
    rule: 可机器解析/可版本化/无歧义的语义契约
    evidence:
      - type: protocol_spec
        note: 子项目提供字段级/IDL 级规范，并声明与本文档语义能力的映射
      - type: capability_manifest
        note: Organ 能力清单可被机器读取（字段名可不同）
  - id: TRACE-REQ-001
    subject: Nerve
    level: MUST
    rule: 每次调用具备全局唯一标识且贯穿调用链
    evidence:
      - type: nerve_message
        note: 消息中存在可关联的调用唯一标识与追踪标识（字段名可不同）
  - id: DECISION-REC-001
    subject: Corpus
    level: MUST
    rule: 对 Organ 输出生成可审计决策记录（采纳/拒绝/补证/回退之一）
    evidence:
      - type: decision_record
        note: 至少包含决策类型、理由、策略版本或等价可追溯信息
  - id: SECURITY-LEAST-001
    subject: Corpus
    level: MUST
    rule: 对 Organ 实施最小权限与可追溯审计（不记录敏感明文）
    evidence:
      - type: authorization_policy
        note: 权限策略或等价机制的存在性证明（不要求具体实现）
      - type: audit_event
        note: 审计事件脱敏并可关联到 Organ 身份或凭证标识（不记录凭证明文）
```

## 10. 证据类型字典（跨项目通用）
本节定义可用于合规验证的证据类型名称；子项目 MAY 用不同结构承载，但 SHOULD 能映射到下列类型之一。

```yaml
evidence_types:
  - name: decision_record
    meaning: Corpus 对一次调用/一次候选结果的裁决记录（commit/reject/need_more_evidence/fallback 等）
  - name: audit_event
    meaning: 安全/仲裁/失败处理等审计事件（必须脱敏）
  - name: nerve_message
    meaning: Nerve 通道上的请求/响应/心跳/健康等消息或其等价载体
  - name: state_transition_log
    meaning: 状态跳转日志或事件流（由 Corpus 统一产出/确认）
  - name: capability_manifest
    meaning: Organ 能力清单与其声明（版本、能力名、约束、SLO 等）
  - name: protocol_spec
    meaning: 子项目的字段级/IDL 级协议规范与其到本文档语义的映射说明
  - name: authorization_policy
    meaning: 最小权限策略或等价访问控制机制的声明与可追溯标识
  - name: module_boundary_doc
    meaning: 模块边界与职责划分说明，用于证明 Corpus 不直接承载业务能力实现
  - name: deployment_boundary
    meaning: 隔离边界的实现说明或配置（进程/容器/沙箱/插件隔离等）
  - name: fault_test_record
    meaning: 故障注入/演练记录，用于证明 Organ 故障不导致 Corpus 不可用
```
