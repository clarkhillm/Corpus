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

