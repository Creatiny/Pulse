# Pulse MVP 工程修复方案

**版本**: v1.1  
**日期**: 2026-04-18  
**范围**: 核心工程问题修复（排除安全和 UI/UX）  
**依据**: DESIGN_REVIEW_REPORT.md

---

## 修复清单

| ID | 问题 | 优先级 | 状态 |
|----|------|--------|------|
| E1 | Session state machine 缺失 | P1 | ✅ |
| E2 | Idempotency 策略缺失 | P1 | ✅ |
| E3 | LLM error handling 缺失 | P2 | ✅ |
| E4 | Retry strategy 缺失 | P2 | ✅ |
| E5 | Database indexes 缺失 | P2 | ✅ |
| E6 | Daily review failure isolation 缺失 | P2 | ✅ |

---

## E1: Session State Machine

### 问题

架构文档定义了 status 枚举，但未明确：
- 有效状态转换
- 超时行为
- 并发消息处理

### 解决方案

#### 状态转换图

```
                    ┌──────────────────────────────────────────────────┐
                    │                                                  │
                    ▼                                                  │
┌───────────┐    ┌─────────────┐    ┌───────────────┐    ┌─────────────────┐
│ initiated │───▶│ collecting  │───▶│ risk_followup │───▶│ summary_confirm │
└───────────┘    └─────────────┘    └───────────────┘    └─────────────────┘
      │                │                    │                    │
      │                │                    │                    │
      │                ▼                    ▼                    ▼
      │          ┌─────────────┐     ┌───────────┐        ┌───────────┐
      │          │  (extraction │     │ (followup │        │  confirmed│
      │          │   complete)  │     │   done)   │        └───────────┘
      │          └─────────────┘     └───────────┘              │
      │                                                          │
      └──────────────────────────────────────────────────────────┘
                         (user rejects summary)
```

#### 状态转换规则

```python
# src/state_machine.py

from enum import Enum
from datetime import datetime, timedelta
from typing import Optional

class SessionStatus(str, Enum):
    INITIATED = "initiated"           # Bot sent initial message, waiting for user
    COLLECTING = "collecting"         # User responding, extracting info
    RISK_FOLLOWUP = "risk_followup"   # Asking follow-up on detected risks
    SUMMARY_CONFIRM = "summary_confirmation"  # Showing summary for confirmation
    CONFIRMED = "confirmed"           # User confirmed, session complete
    EXPIRED = "expired"               # Timeout, session abandoned
    FAILED = "failed"                 # Unrecoverable error

# Valid transitions: {from_status: [allowed_to_statuses]}
VALID_TRANSITIONS = {
    SessionStatus.INITIATED: [
        SessionStatus.COLLECTING,     # User sends first message
        SessionStatus.EXPIRED,        # Timeout before user responds
    ],
    SessionStatus.COLLECTING: [
        SessionStatus.RISK_FOLLOWUP,  # Risk detected, need follow-up
        SessionStatus.SUMMARY_CONFIRM, # Extraction complete, show summary
        SessionStatus.FAILED,         # Unrecoverable extraction error
    ],
    SessionStatus.RISK_FOLLOWUP: [
        SessionStatus.SUMMARY_CONFIRM, # Follow-up complete
        SessionStatus.FAILED,         # Unrecoverable error
    ],
    SessionStatus.SUMMARY_CONFIRM: [
        SessionStatus.CONFIRMED,      # User confirms
        SessionStatus.COLLECTING,     # User rejects, re-collect
        SessionStatus.EXPIRED,        # Timeout waiting for confirmation
    ],
    SessionStatus.CONFIRMED: [],      # Terminal state
    SessionStatus.EXPIRED: [],        # Terminal state
    SessionStatus.FAILED: [],         # Terminal state
}

# Timeout configuration
TIMEOUTS = {
    SessionStatus.INITIATED: timedelta(hours=24),      # 24h to respond
    SessionStatus.COLLECTING: timedelta(hours=2),      # 2h between messages
    SessionStatus.RISK_FOLLOWUP: timedelta(hours=1),   # 1h for follow-up
    SessionStatus.SUMMARY_CONFIRM: timedelta(hours=1), # 1h to confirm
}

def can_transition(from_status: SessionStatus, to_status: SessionStatus) -> bool:
    """Check if transition is valid."""
    return to_status in VALID_TRANSITIONS.get(from_status, [])

def get_timeout(status: SessionStatus) -> Optional[timedelta]:
    """Get timeout for status."""
    return TIMEOUTS.get(status)

def is_expired(status: SessionStatus, last_activity: datetime) -> bool:
    """Check if session has expired."""
    timeout = get_timeout(status)
    if timeout is None:
        return False
    return datetime.utcnow() - last_activity > timeout

def is_terminal(status: SessionStatus) -> bool:
    """Check if status is terminal (no further transitions)."""
    return len(VALID_TRANSITIONS.get(status, [])) == 0
```

#### 并发处理

```python
# src/session_lock.py

import asyncio
from contextlib import asynccontextmanager
from typing import Dict

# Per-session locks to handle concurrent messages
_session_locks: Dict[str, asyncio.Lock] = {}

@asynccontextmanager
async def session_lock(session_id: str):
    """Acquire lock for session to handle concurrent messages."""
    if session_id not in _session_locks:
        _session_locks[session_id] = asyncio.Lock()
    
    async with _session_locks[session_id]:
        yield

async def handle_message(session_id: str, message):
    """Handle incoming message with session lock."""
    async with session_lock(session_id):
        # 1. Load session
        # 2. Check if expired
        # 3. Validate transition
        # 4. Process message
        # 5. Update status
        pass
```

---

## E2: Idempotency Strategy

### 问题

Feishu webhooks 可能重复投递消息，缺少去重机制。

### 解决方案

#### 数据库约束

```sql
-- Add unique constraint to message_log
ALTER TABLE message_log 
ADD CONSTRAINT uq_channel_message_id UNIQUE (channel_message_id);

-- Add idempotency key column to checkin_session for dedup
ALTER TABLE checkin_session
ADD COLUMN idempotency_key VARCHAR(64);

CREATE INDEX idx_checkin_session_idempotency 
ON checkin_session(idempotency_key) 
WHERE idempotency_key IS NOT NULL;
```

#### 消息处理模式

```python
# src/idempotency.py

import hashlib
from typing import Optional
from sqlalchemy.dialects.postgresql import insert
---

## E3: LLM Error Handling

### 问题

LLM 提取可能失败（超时、格式错误、rate limit），缺少回退策略。

### 解决方案

#### 分层回退策略

```
1. Try LLM with structured output (highest quality)
   ↓ (timeout/malformed/rate-limit)
2. Fall back to regex rules (medium quality)
   ↓ (still fails)
3. Mark all fields as missing, prompt user for manual input
```

#### Regex Fallback Patterns

```python
# src/extraction.py

REGEX_PATTERNS = {
    "progress": r"(?:完成了?|完成工作|进展)[：:]\s*(.+?)(?=\n|$)",
    "blocker": r"(?:阻塞|卡点|障碍|困难)[：:]\s*(.+?)(?=\n|$)",
    "plan": r"(?:计划|下一步|下周)[：:]\s*(.+?)(?=\n|$)",
    "support_needed": r"(?:需要支持|帮助|资源)[：:]\s*(.+?)(?=\n|$)",
}

async def extract_structured_info(message_text: str, required_fields: list[str]):
    # Step 1: Try LLM
    try:
        result = await extract_with_llm(message_text, required_fields)
        if result.confidence > 0.7:
            return result
    except (TimeoutError, JSONDecodeError, RateLimitError):
        pass  # Continue to fallback
    
    # Step 2: Try regex fallback
    result = extract_with_regex(message_text, required_fields)
    if result.data:
        return result
    
    # Step 3: Manual fallback
    return ExtractionResult(
        method="manual",
        data={},
        missing_fields=required_fields,
        confidence=0.0
    )
```

---

## E4: Retry Strategy

### 问题

Feishu API 调用可能失败，缺少重试机制。

### 解决方案

#### 指数退避重试

```python
# src/retry.py

def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    exponential_base: float = 2.0,
    exceptions: tuple = (Exception,)
):
    """
    Retry with exponential backoff + jitter.
    
    Delay = min(base_delay * (exponential_base ** attempt), max_delay)
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_retries + 1):
                try:
                    return await func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_retries:
                        raise
                    delay = min(base_delay * (exponential_base ** attempt), max_delay)
                    jitter = random.uniform(0.1, 0.3) * delay
                    await asyncio.sleep(delay + jitter)
        return wrapper
    return decorator

# Usage
@retry_with_backoff(max_retries=3, exceptions=(FeishuAPIError, TimeoutError))
async def send_feishu_message(user_id: str, message: str):
    pass
```

#### 消息发送队列 (Outbox Pattern)

```sql
-- Store messages in database before sending
-- Background worker retries failed sends

CREATE TABLE message_outbox (
    id SERIAL PRIMARY KEY,
    message_id VARCHAR(64) UNIQUE,
    recipient_id VARCHAR(64),
    message_type VARCHAR(32),
    payload JSONB,
    status VARCHAR(32) DEFAULT 'pending',  -- pending/sent/failed
    attempts INT DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMP,
    sent_at TIMESTAMP
);
```

---

## E5: Database Indexes

### 问题

架构文档未定义索引规范。

### 解决方案

```sql
-- Primary lookup indexes
CREATE INDEX idx_checkin_session_user ON checkin_session(feishu_user_id);
CREATE INDEX idx_checkin_session_manager ON checkin_session(manager_feishu_user_id);
CREATE INDEX idx_checkin_session_status ON checkin_session(status);
CREATE INDEX idx_checkin_session_period ON checkin_session(period_key);
CREATE INDEX idx_checkin_session_created ON checkin_session(created_at DESC);

-- Composite indexes for common queries
CREATE INDEX idx_checkin_session_user_period ON checkin_session(feishu_user_id, period_key);
CREATE INDEX idx_checkin_session_manager_status ON checkin_session(manager_feishu_user_id, status);

-- Message log indexes
CREATE INDEX idx_message_log_session ON message_log(session_id);
CREATE INDEX idx_message_log_created ON message_log(created_at DESC);
CREATE UNIQUE INDEX idx_message_log_channel_id ON message_log(channel_message_id);

-- Risk item indexes
CREATE INDEX idx_risk_item_session ON risk_item(session_id);
CREATE INDEX idx_risk_item_manager ON risk_item(manager_feishu_user_id);
CREATE INDEX idx_risk_item_severity ON risk_item(severity);
CREATE INDEX idx_risk_item_status ON risk_item(status);

-- Manager brief indexes
CREATE INDEX idx_manager_brief_manager ON manager_brief(manager_feishu_user_id);
CREATE INDEX idx_manager_brief_period ON manager_brief(period_key);

-- Daily review indexes
CREATE INDEX idx_daily_review_date ON daily_review_report(review_date DESC);
```

---

## E6: Daily Review Failure Isolation

### 问题

每日复盘任务处理所有 session，一个失败可能影响整体。

### 解决方案

#### 批处理 + 隔离

```python
# src/daily_review.py

async def run_daily_review():
    """
    Daily review with failure isolation.
    
    Each session is processed independently.
    Failures are logged but don't stop the batch.
    """
    sessions = await get_sessions_for_review()
    
    results = {
        "total": len(sessions),
        "success": 0,
        "failed": 0,
        "errors": []
    }
    
    for session in sessions:
        try:
            score = await score_session(session)
            results["success"] += 1
        except Exception as e:
            results["failed"] += 1
            results["errors"].append({
                "session_id": session.id,
                "error": str(e)
            })
            # Continue processing other sessions
            continue
    
    # Generate report even if some failed
    report = await generate_review_report(results)
    return report
```

#### 超时保护

```python
async def score_session_with_timeout(session, timeout_seconds=30):
    """Score session with timeout protection."""
    try:
        return await asyncio.wait_for(
            score_session(session),
            timeout=timeout_seconds
        )
    except asyncio.TimeoutError:
        raise ReviewTimeoutError(f"Session {session.id} scoring timed out")
```

---

## 实施优先级

| 优先级 | 问题 | 预计工时 | 依赖 |
|--------|------|----------|------|
| P1 | E1: State Machine | 4h | 无 |
| P1 | E2: Idempotency | 2h | 数据库迁移
| P2 | E3: LLM Error Handling | 4h | 无 |
| P2 | E4: Retry Strategy | 4h | 无 |
| P2 | E5: Database Indexes | 2h | 数据库迁移 |
| P2 | E6: Daily Review Isolation | 2h | 无 |

**总计**: ~18 小时

---

## 验收标准

### E1: State Machine
- [ ] 所有状态转换都有明确定义
- [ ] 超时行为已实现并测试
- [ ] 并发消息不会导致状态混乱

### E2: Idempotency
- [ ] 重复消息被正确识别并忽略
- [ ] Session 创建幂等
- [ ] 数据库约束生效

### E3: LLM Error Handling
- [ ] LLM 超时自动回退到 regex
- [ ] 格式错误自动回退到 regex
- [ ] 所有方法失败时提示用户手动输入

### E4: Retry Strategy
- [ ] Feishu API 失败自动重试
- [ ] 指数退避生效
- [ ] 最大重试次数限制

### E5: Database Indexes
- [ ] 所有索引已创建
- [ ] 查询性能符合预期

### E6: Daily Review Isolation
- [ ] 单个 session 失败不影响其他
- [ ] 超时保护生效
- [ ] 错误日志完整

---

## 附录：完整状态转换表

| From | To | Trigger | Action |
|------|-----|---------|--------|
| initiated | collecting | User sends first message | Start extraction |
| initiated | expired | 24h timeout | Mark as expired |
| collecting | risk_followup | Risk detected | Ask follow-up questions |
| collecting | summary_confirm | Extraction complete | Show summary |
| collecting | failed | Unrecoverable error | Log error, notify |
| risk_followup | summary_confirm | Follow-up done | Show summary |
| risk_followup | failed | Unrecoverable error | Log error, notify |
| summary_confirm | confirmed | User confirms | Complete session |
| summary_confirm | collecting | User rejects | Re-collect info |
| summary_confirm | expired | 1h timeout | Mark as expired |
