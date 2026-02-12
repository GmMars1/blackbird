# AI Agent Security SDK v0.1 — Module Structure Blueprint

This document turns the v0.1 system blueprint into a concrete Python SDK layout that can be implemented incrementally while preserving one hard rule:

> The model is stochastic. The control plane must be deterministic.

## Recommendation

Start with the **module structure** first, then write the threat model against those concrete interfaces in parallel. In practice, security reviews land faster when reviewers can point to exact classes, request/response contracts, and enforcement points.

---

## Package Layout

```text
agent_security_sdk/
  __init__.py
  adapters/
    __init__.py
    base.py
    langchain.py
    autogen.py
  proposals/
    __init__.py
    schema.py
    normalize.py
  policy/
    __init__.py
    engine.py
    dsl.py
    matcher.py
    reason_codes.py
  proxy/
    __init__.py
    gateway.py
    validators.py
    wrappers/
      __init__.py
      files.py
      http.py
      email.py
  runtime/
    __init__.py
    sandbox.py
    docker_exec.py
    network_policy.py
  audit/
    __init__.py
    logger.py
    sink_jsonl.py
    sink_otlp.py
    hashers.py
  identity/
    __init__.py
    context.py
    rbac.py
  labels/
    __init__.py
    taxonomy.py
    flow_control.py
  redteam/
    __init__.py
    harness.py
    scenarios.py
    metrics.py
  compliance/
    __init__.py
    report.py
  exceptions.py
  settings.py
```

---

## Core Interfaces (Deterministic Contracts)

### 1) Proposal Schema (`proposals/schema.py`)

```python
@dataclass(frozen=True)
class ToolProposal:
    tool: str
    arguments: dict[str, Any]
    reasoning: str
    session_id: str
    user_id: str
    turn_id: str
    context_hash: str
```

Design notes:
- `context_hash` stores a stable hash of prompt/tool context, not raw prompt text.
- `frozen=True` prevents mutation after capture.

### 2) Adapter Interface (`adapters/base.py`)

```python
class AgentAdapter(Protocol):
    def intercept(self, raw_call: Any, context: AgentContext) -> ToolProposal: ...
```

Responsibilities:
- Convert framework-specific tool calls to `ToolProposal`.
- No side effects, no execution.

### 3) Policy Engine (`policy/engine.py`)

```python
@dataclass(frozen=True)
class PolicyDecision:
    effect: Literal["ALLOW", "DENY"]
    reason_code: str
    matched_rule_id: str | None

class PolicyEngine:
    def evaluate(self, proposal: ToolProposal, ctx: SecurityContext) -> PolicyDecision: ...
```

Rules:
- Deterministic matching order.
- Total function: always returns a decision.
- Safe default: implicit deny.

### 4) Proxy Gateway (`proxy/gateway.py`)

```python
class ToolProxyGateway:
    def execute(self, proposal: ToolProposal, ctx: SecurityContext) -> ToolExecutionResult: ...
```

Pipeline:
1. Validate schema
2. Evaluate policy
3. If denied -> return blocked result
4. If allowed -> route to typed wrapper
5. Emit audit event

### 5) Tool Wrappers (`proxy/wrappers/*.py`)

```python
class SafeReadFile:
    def run(self, path: str, ctx: SecurityContext) -> str: ...
```

Requirements:
- Strict type validation
- Canonical path checks
- Allowlisted domains/paths/methods
- No raw shell command passthrough

### 6) Runtime Sandbox (`runtime/sandbox.py`)

```python
class SandboxRuntime:
    def run(self, action: Callable[[], T], limits: RuntimeLimits) -> T: ...
```

Controls:
- Filesystem mount restrictions
- Network egress policy
- CPU/memory/time limits
- Read-only root when possible

### 7) Audit Layer (`audit/logger.py`)

```python
@dataclass(frozen=True)
class AuditEvent:
    timestamp: datetime
    session_id: str
    user_id: str
    proposal: ToolProposal
    decision: PolicyDecision
    result: dict[str, Any]
```

Properties:
- Append-only events
- Stable event schema
- Export sinks (JSONL, OTLP)

---

## Default Policy DSL (v0.1)

```yaml
version: "1"
default_effect: DENY
rules:
  - id: read_docs
    tool: read_file
    effect: ALLOW
    constraints:
      path_prefix: "/workspace/docs/"

  - id: outbound_email_company_only
    tool: send_email
    effect: ALLOW
    constraints:
      recipient_domains: ["company.com"]

  - id: deny_shell
    tool: execute_shell
    effect: DENY
```

Determinism requirements:
- Rules evaluated top-to-bottom with first-match semantics.
- Invalid policy fails closed (engine load error -> deny all).

---

## 90-Day Plan Mapped to Modules

### Days 1-30 (Containment)
- `adapters/*`: capture LangChain tool calls.
- `policy/*`: static DSL + evaluator.
- `proxy/wrappers/{files,http,email}.py`: safe wrappers.
- `runtime/*`: Docker-backed execution limits.
- `audit/*`: structured immutable logs.

Exit criteria:
- No direct tool execution path bypassing `ToolProxyGateway`.
- 100% of proposals produce an audit event.

### Days 31-60 (Adversarial Harness)
- `redteam/scenarios.py`: injection/escalation/exfiltration suites.
- `redteam/harness.py`: deterministic scenario runner.
- `redteam/metrics.py`: unauthorized proposal rate, bypass rate, time-to-violation, false positives.

Exit criteria:
- Policy bypass rate remains zero across CI runs.

### Days 61-90 (Enterprise Hardening)
- `identity/rbac.py`: role-scoped permissions.
- `labels/*`: PUBLIC/INTERNAL/SENSITIVE flow checks.
- `compliance/report.py`: control mapping + violation reports.
- Audit anomaly queries over event stream.

Exit criteria:
- One-click compliance artifact export for customer reviews.

---


## Double-Check Findings (v0.1 Readiness Review)

A second pass against the architecture yields the following findings that should be tracked before implementation starts:

- **Policy completeness risk**: every registered tool needs an explicit policy rule test, or the system can drift into accidental deny/allow confusion.
- **Adapter bypass risk**: framework maintainers can introduce new execution paths that skip interception unless adapter contract tests lock this down.
- **Wrapper drift risk**: if wrappers expose "escape hatch" arguments (for example raw command strings), determinism degrades immediately.
- **Audit integrity risk**: append-only storage must be enforced at storage level, not only application logic.
- **Sandbox mismatch risk**: local development and CI sandbox settings must match production defaults to prevent false confidence.

Recommended mitigation gates for v0.1:

1. Contract tests proving every tool invocation path emits a `ToolProposal`.
2. Golden tests for policy evaluation order and reason codes.
3. Negative tests for path traversal, domain bypass, and argument type confusion.
4. Tamper-evidence checks on audit events (hash chaining or signed batches).
5. CI profile that runs with production-equivalent sandbox/network restrictions.

---

## v0.1 Non-Negotiables

- No probabilistic policy adjudication.
- No raw shell tool in default registry.
- No mutable audit trail.
- No execution without a prior policy decision.

If any of the above are violated, v0.1 is not containment-complete.
