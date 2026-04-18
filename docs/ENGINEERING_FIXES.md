# Pulse MVP 工程修复方案

**版本**: v1.2  
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
| E4 | Session timeout strategy 缺失 | P1 | ✅ |
| E5 | Retry strategy 缺失 | P2 | ✅ |
| E6 | Database indexes 缺失 | P2 | ✅ |
| E7 | Daily review failure isolation 缺失 | P2 | ✅ |

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

#### Failed State Recovery

```python
# src/recovery.py

from enum import Enum
from typing import Optional

class RecoveryAction(str, Enum):
    RETRY = "retry"           # Retry last operation
    RESTART = "restart"       # Restart session from collecting
    ABANDON = "abandon"       # Mark as failed, notify user
    ESCALATE = "escalate"     # Escalate to human support

# Recovery strategies by failure type
RECOVERY_STRATEGIES = {
    "llm_timeout": RecoveryAction.RETRY,
    "llm_rate_limit": RecoveryAction.RETRY,
    "llm_format_error": RecoveryAction.RESTART,
    "feishu_send_failed": RecoveryAction.RETRY,
    "db_error": RecoveryAction.RETRY,
    "extraction_failed": RecoveryAction.RESTART,
    "unknown_error": RecoveryAction.ESCALATE,
}

async def handle_failed_session(session: CheckinSession, error: Exception) -> RecoveryAction:
    """
    Determine recovery action for failed session.
    
    Returns action to take and updates session state.
    """
    error_type = classify_error(error)
    action = RECOVERY_STRATEGIES.get(error_type, RecoveryAction.ESCALATE)
    
    # Record failure
    session.failure_count = (session.failure_count or 0) + 1
    session.last_failure_at = datetime.utcnow()
    session.last_failure_reason = error_type
    
    if session.failure_count >= 3:
        # Too many failures, abandon
        action = RecoveryAction.ABANDON
    
    if action == RecoveryAction.RETRY:
        # Reset to previous state for retry
        session.status = session.previous_status or SessionStatus.COLLECTING
        session.retry_scheduled_at = datetime.utcnow() + timedelta(minutes=5)
    
    elif action == RecoveryAction.RESTART:
        # Restart from collecting
        session.status = SessionStatus.COLLECTING
        session.extraction_snapshot = None  # Clear partial data
    
    elif action == RecoveryAction.ABANDON:
        session.status = SessionStatus.FAILED
        session.failed_at = datetime.utcnow()
        await send_feishu_message(
            session.feishu_user_id,
            "周报提交遇到问题，请稍后重试或联系经理。"
        )
    
    elif action == RecoveryAction.ESCALATE:
        session.status = SessionStatus.FAILED
        session.needs_escalation = True
        await notify_admin(f"Session {session.id} needs escalation: {error}")
    
    await db.commit()
    return action


def classify_error(error: Exception) -> str:
    """Classify error type for recovery strategy."""
    error_str = str(error).lower()
    
    if "timeout" in error_str:
        return "llm_timeout"
    if "rate limit" in error_str:
        return "llm_rate_limit"
    if "format" in error_str or "json" in error_str:
        return "llm_format_error"
    if "feishu" in error_str:
        return "feishu_send_failed"
    if "database" in error_str or "db" in error_str:
        return "db_error"
    if "extraction" in error_str:
        return "extraction_failed"
    
    return "unknown_error"
```

#### Expired Session Handling

```python
# src/expiry.py

async def handle_expired_session(session: CheckinSession):
    """
    Handle session that has expired.
    
    Expired sessions can be:
    - Resurrected if user responds within grace period
    - Archived for analytics
    """
    session.status = SessionStatus.EXPIRED
    session.expired_at = datetime.utcnow()
    
    # Send notification
    await send_feishu_message(
        session.feishu_user_id,
        "本周周报已过期。如需补交，请回复"重新开始"。"
    )
    
    # Archive partial data for analytics
    await archive_session_data(session)
    
    await db.commit()


async def resurrect_session(session: CheckinSession) -> bool:
    """
    Attempt to resurrect expired session.
    
    Returns True if resurrection successful.
    """
    # Check grace period (7 days)
    if session.expired_at and datetime.utcnow() - session.expired_at > timedelta(days=7):
        return False
    
    # Create new session with same user
    new_session = await create_session_idempotent(
        session.feishu_user_id,
        get_current_period_key(),
        session.session_type
    )
    
    # Copy partial data
    if session.extraction_snapshot:
        new_session.extraction_snapshot = session.extraction_snapshot
    
    await db.commit()
    return True
```

---

## E2: Idempotency Strategy

### 问题

Feishu webhooks 可能重复投递消息，缺少去重机制。

### 解决方案

#### 2.1 消息去重 (Message Deduplication)

**Dedup Window**: 24 小时去重窗口，超过窗口的重复消息视为新消息。

```sql
-- Add unique constraint to message_log
ALTER TABLE message_log 
ADD CONSTRAINT uq_channel_message_id UNIQUE (channel_message_id);

-- Add dedup metadata
ALTER TABLE message_log
ADD COLUMN dedup_window_start TIMESTAMP;

CREATE INDEX idx_message_log_dedup 
ON message_log(channel_message_id, dedup_window_start);
```

**Upsert Pattern**:

```python
# src/idempotency.py

import hashlib
from datetime import datetime, timedelta
from typing import Optional
from sqlalchemy.dialects.postgresql import insert

DEDUP_WINDOW_HOURS = 24

async def deduplicate_message(
    session_id: str,
    channel_message_id: str,
    message_text: str
) -> tuple[bool, Optional[int]]:
    """
    Check if message is duplicate within dedup window.
    
    Returns:
        (is_duplicate, existing_message_id)
    """
    # Calculate dedup window
    window_start = datetime.utcnow() - timedelta(hours=DEDUP_WINDOW_HOURS)
    
    # Check for existing message
    existing = await db.execute(
        select(MessageLog)
        .where(MessageLog.channel_message_id == channel_message_id)
        .where(MessageLog.dedup_window_start >= window_start)
    )
    
    if existing:
        return (True, existing.id)
    
    return (False, None)


async def upsert_message(
    session_id: str,
    channel_message_id: str,
    message_text: str,
    sender_type: str
) -> MessageLog:
    """
    Insert message with idempotency.
    
    Uses PostgreSQL ON CONFLICT DO NOTHING for atomic dedup.
    """
    window_start = datetime.utcnow() - timedelta(hours=DEDUP_WINDOW_HOURS)
    
    stmt = insert(MessageLog).values(
        session_id=session_id,
        channel_message_id=channel_message_id,
        message_text=message_text,
        sender_type=sender_type,
        dedup_window_start=window_start,
        created_at=datetime.utcnow()
    ).on_conflict_do_nothing(
        constraint='uq_channel_message_id'
    ).returning(MessageLog)
    
    result = await db.execute(stmt)
    return result.scalar_one_or_none()  # None if duplicate
```

#### 2.2 Session 创建幂等

**Idempotency Key**: 使用 `period_key` + `feishu_user_id` 组合作为幂等键。

```sql
-- Add idempotency key column to checkin_session
ALTER TABLE checkin_session
ADD COLUMN idempotency_key VARCHAR(128);

CREATE UNIQUE INDEX idx_checkin_session_idempotency 
ON checkin_session(idempotency_key) 
WHERE idempotency_key IS NOT NULL;

-- Add constraint for one active session per user
CREATE UNIQUE INDEX idx_one_active_session_per_user 
ON checkin_session(feishu_user_id) 
WHERE status NOT IN ('confirmed', 'expired', 'failed');
```

**Session Creation Pattern**:

```python
def generate_idempotency_key(user_id: str, period_key: str, session_type: str) -> str:
    """Generate idempotency key for session creation."""
    return f"{session_type}:{user_id}:{period_key}"


async def create_session_idempotent(
    feishu_user_id: str,
    period_key: str,
    session_type: str
) -> tuple[CheckinSession, bool]:
    """
    Create session with idempotency.
    
    Returns:
        (session, is_new) - is_new=False if session already exists
    """
    idempotency_key = generate_idempotency_key(feishu_user_id, period_key, session_type)
    
    # Check for existing session
    existing = await db.execute(
        select(CheckinSession)
        .where(CheckinSession.idempotency_key == idempotency_key)
    )
    
    if existing:
        return (existing, False)
    
    # Create new session
    session = CheckinSession(
        feishu_user_id=feishu_user_id,
        period_key=period_key,
        session_type=session_type,
        idempotency_key=idempotency_key,
        status=SessionStatus.INITIATED,
        started_at=datetime.utcnow()
    )
    
    await db.commit()
    return (session, True)
```

#### 2.3 消息队列 (Message Queue)

**Per-Session Sequential Processing**: 每个 session 有独立的消息队列，保证顺序处理。

```python
# src/message_queue.py

import asyncio
from collections import defaultdict
from typing import Dict, Queue
from dataclasses import dataclass

@dataclass
class QueuedMessage:
    session_id: str
    channel_message_id: str
    message_text: str
    timestamp: datetime


class SessionMessageQueue:
    """
    Per-session message queue for sequential processing.
    
    Guarantees:
    - Messages for same session are processed sequentially
    - Concurrent sessions are processed in parallel
    - No race conditions on session state
    """
    
    def __init__(self):
        self._queues: Dict[str, asyncio.Queue] = defaultdict(asyncio.Queue)
        self._workers: Dict[str, asyncio.Task] = {}
        self._lock = asyncio.Lock()
    
    async def enqueue(self, message: QueuedMessage) -> bool:
        """Add message to session queue."""
        queue = self._queues[message.session_id]
        await queue.put(message)
        
        # Start worker if not running
        async with self._lock:
            if message.session_id not in self._workers:
                self._workers[message.session_id] = asyncio.create_task(
                    self._process_queue(message.session_id)
                )
        
        return True
    
    async def _process_queue(self, session_id: str):
        """Process messages for session sequentially."""
        queue = self._queues[session_id]
        
        while True:
            try:
                message = await asyncio.wait_for(queue.get(), timeout=300)
                await self._handle_message(message)
            except asyncio.TimeoutError:
                # No messages for 5 min, stop worker
                async with self._lock:
                    del self._workers[session_id]
                break
            except Exception as e:
                # Log error but continue processing
                logger.error(f"Error processing message: {e}")
                continue
    
    async def _handle_message(self, message: QueuedMessage):
        """Handle single message."""
        # 1. Deduplicate
        is_dup, _ = await deduplicate_message(
            message.session_id,
            message.channel_message_id,
            message.message_text
        )
        if is_dup:
            return
        
        # 2. Process message
        await process_session_message(
            message.session_id,
            message.message_text
        )


# Global instance
message_queue = SessionMessageQueue()
```

#### 2.4 Webhook Handler

```python
# src/webhook_handler.py

async def handle_feishu_webhook(event: FeishuEvent):
    """
    Handle Feishu webhook with idempotency.
    
    Guarantees:
    - Duplicate webhooks are ignored
    - Messages are queued for sequential processing
    - Response is fast (no blocking on processing)
    """
    # Extract message info
    channel_message_id = event.message_id
    user_id = event.sender_id
    message_text = event.content
    
    # Get or create session
    period_key = get_current_period_key()
    session, is_new = await create_session_idempotent(
        user_id, period_key, "weekly_checkin"
    )
    
    # Queue message for processing
    await message_queue.enqueue(QueuedMessage(
        session_id=session.id,
        channel_message_id=channel_message_id,
        message_text=message_text,
        timestamp=datetime.utcnow()
    ))
    
    # Return immediately (async processing)
    return {"status": "queued"}
```
---

## E3: LLM Error Handling

### 问题

LLM 提取可能失败（超时、格式错误、rate limit），缺少回退策略。

### 解决方案

#### 3.1 分层回退策略

```
1. Try LLM with structured output (highest quality)
   ↓ (timeout/malformed/rate-limit)
2. Fall back to regex rules (medium quality)
   ↓ (completeness < 30%)
3. Request manual input from user (lowest quality)
```

#### 3.2 完整回退实现

```python
# src/extraction/fallback.py

from enum import Enum
from typing import Optional
import re
import json

class ExtractionMethod(str, Enum):
    LLM = "llm"
    REGEX = "regex"
    MANUAL = "manual"

class ExtractionError(Exception):
    """Base extraction error."""
    pass

class LLMTimeoutError(ExtractionError):
    """LLM call timed out."""
    pass

class LLMRateLimitError(ExtractionError):
    """LLM rate limited."""
    pass

class LLMFormatError(ExtractionError):
    """LLM returned malformed response."""
    pass

# Regex patterns for fallback
REGEX_PATTERNS = {
    "progress": [
        r"(?:完成了?|完成工作|进展)[：:]\s*(.+?)(?=\n|$)",
        r"(?:本周|这周)[^。]*完成了?([^。]+)",
        r"(?:主要|重点)[^。]*是([^。]+)",
    ],
    "blocker": [
        r"(?:阻塞|卡点|障碍|困难|卡住)[：:]\s*(.+?)(?=\n|$)",
        r"(?:被|在)[^。]*(?:卡住|阻塞|阻塞了)([^。]+)",
        r"(?:等待|等)[^。]*(?:反馈|回复|确认)([^。]*)",
    ],
    "plan": [
        r"(?:计划|下一步|下周)[：:]\s*(.+?)(?=\n|$)",
        r"(?:下周|接下来)[^。]*(?:准备|打算|计划)([^。]+)",
    ],
    "support_needed": [
        r"(?:需要支持|需要帮助|需要资源)[：:]\s*(.+?)(?=\n|$)",
        r"(?:希望|需要)[^。]*(?:支持|帮助|协助)([^。]+)",
    ],
    "risk": [
        r"(?:风险|可能|担心)[：:]\s*(.+?)(?=\n|$)",
        r"(?:可能|担心|风险是)([^。]+)",
    ],
}

# Confidence thresholds
LLM_MIN_CONFIDENCE = 0.7
REGEX_MIN_COMPLETENESS = 0.3
MAX_LLM_RETRIES = 2


async def extract_structured_info(
    message_text: str,
    required_fields: list[str]
) -> ExtractionResult:
    """
    Extract structured info with full fallback chain.
    
    Fallback order:
    1. LLM with retry
    2. Regex patterns
    3. Manual input request
    """
    # Step 1: Try LLM with retry
    for attempt in range(MAX_LLM_RETRIES):
        try:
            result = await extract_with_llm(message_text, required_fields)
            if result.confidence >= LLM_MIN_CONFIDENCE:
                return result
        except (LLMTimeoutError, LLMRateLimitError) as e:
            logger.warning(f"LLM extraction failed (attempt {attempt+1}): {e}")
            if attempt < MAX_LLM_RETRIES - 1:
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
            continue
        except LLMFormatError as e:
            logger.warning(f"LLM format error: {e}")
            break  # No retry for format errors
    
    # Step 2: Try regex fallback
    result = extract_with_regex(message_text, required_fields)
    completeness = calculate_completeness(result.data, required_fields)
    
    if completeness >= REGEX_MIN_COMPLETENESS:
        return result
    
    # Step 3: Manual fallback - return with missing fields
    return ExtractionResult(
        method=ExtractionMethod.MANUAL,
        data=result.data,
        missing_fields=[f for f in required_fields if f not in result.data],
        confidence=0.0,
        raw_response=None
    )


def extract_with_regex(message_text: str, required_fields: list[str]) -> ExtractionResult:
    """Extract using regex patterns."""
    data = {}
    
    for field in required_fields:
        patterns = REGEX_PATTERNS.get(field, [])
        for pattern in patterns:
            match = re.search(pattern, message_text, re.IGNORECASE | re.DOTALL)
            if match:
                data[field] = match.group(1).strip()
                break
    
    return ExtractionResult(
        method=ExtractionMethod.REGEX,
        data=data,
        missing_fields=[f for f in required_fields if f not in data],
        confidence=calculate_regex_confidence(data, required_fields),
        raw_response=None
    )


def calculate_completeness(data: dict, required_fields: list[str]) -> float:
    """Calculate extraction completeness ratio."""
    if not required_fields:
        return 1.0
    extracted = sum(1 for f in required_fields if f in data and data[f])
    return extracted / len(required_fields)
```

#### 3.3 用户手动输入请求

```python
# src/extraction/manual.py

MANUAL_PROMPTS = {
    "progress": "请简要描述本周主要进展：",
    "blocker": "当前有什么阻塞或困难吗？",
    "plan": "下周的主要计划是什么？",
    "support_needed": "需要经理提供什么支持？",
    "risk": "有什么风险或担心需要提前说明吗？",
}

async def request_manual_input(session_id: str, missing_fields: list[str]) -> str:
    """Generate prompt for manual input."""
    if len(missing_fields) == 1:
        return MANUAL_PROMPTS.get(missing_fields[0], f"请提供{missing_fields[0]}：")
    
    prompts = [f"{i+1}. {MANUAL_PROMPTS.get(f, f)}" 
               for i, f in enumerate(missing_fields[:3])]
    
    return f"为了帮你整理完整的周报，还需要补充：\n" + "\n".join(prompts)
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

## E4: Session Timeout Strategy

### 问题

Session 可能卡在某个状态，需要超时机制。

### 解决方案

#### 4.1 状态超时定义

| 状态 | 超时时间 | 超时动作 | 通知消息 |
|------|----------|----------|----------|
| initiated | 24h | expire | "本周周报已过期，如需补交请联系经理。" |
| collecting | 48h | force_summary | "周报收集时间已到，我将根据已有信息生成摘要。" |
| risk_followup | 24h | continue | "风险追问超时，将继续生成摘要。" |
| summary_confirmation | 24h | auto_confirm | "摘要已自动确认。如需修改，请回复'修改'。" |
| failed | 72h | expire | "周报提交失败，请联系经理或重试。" |

#### 4.2 超时检查实现

```python
# src/scheduler/timeout_checker.py

from datetime import datetime, timedelta
from dataclasses import dataclass

@dataclass
class StateTimeout:
    status: str
    timeout: timedelta
    on_timeout: str  # expire/force_summary/auto_confirm/continue
    notification: str

STATE_TIMEOUTS = {
    "initiated": StateTimeout(
        status="initiated",
        timeout=timedelta(hours=24),
        on_timeout="expire",
        notification="本周周报已过期，如需补交请联系经理。"
    ),
    "collecting": StateTimeout(
        status="collecting",
        timeout=timedelta(hours=48),
        on_timeout="force_summary",
        notification="周报收集时间已到，我将根据已有信息生成摘要。"
    ),
    "risk_followup": StateTimeout(
        status="risk_followup",
        timeout=timedelta(hours=24),
        on_timeout="continue",
        notification="风险追问超时，将继续生成摘要。"
    ),
    "summary_confirmation": StateTimeout(
        status="summary_confirmation",
        timeout=timedelta(hours=24),
        on_timeout="auto_confirm",
        notification="摘要已自动确认。如需修改，请回复'修改'。"
    ),
    "failed": StateTimeout(
        status="failed",
        timeout=timedelta(hours=72),
        on_timeout="expire",
        notification="周报提交失败，请联系经理或重试。"
    ),
}

async def check_session_timeouts():
    """Periodic task to check and handle session timeouts. Run every 5 minutes."""
    now = datetime.utcnow()
    
    active_sessions = await db.execute(
        select(CheckinSession)
        .where(CheckinSession.status.in_([
            "initiated", "collecting", "risk_followup", 
            "summary_confirmation", "failed"
        ]))
    )
    
    for session in active_sessions:
        config = STATE_TIMEOUTS.get(session.status)
        if not config:
            continue
        
        elapsed = now - session.last_activity_at
        if elapsed > config.timeout:
            await handle_timeout(session, config)


async def handle_timeout(session: CheckinSession, config: StateTimeout):
    """Handle session timeout based on configured action."""
    
    if config.on_timeout == "expire":
        session.status = "expired"
        session.expired_at = datetime.utcnow()
    
    elif config.on_timeout == "force_summary":
        summary = await generate_summary(session.id)
        session.status = "summary_confirmation"
        session.summary_text = summary.text
    
    elif config.on_timeout == "auto_confirm":
        session.status = "confirmed"
        session.confirmed_at = datetime.utcnow()
        session.auto_confirmed = True
    
    elif config.on_timeout == "continue":
        session.status = "summary_confirmation"
    
    session.last_activity_at = datetime.utcnow()
    
    if config.notification:
        await send_feishu_message(session.feishu_user_id, config.notification)
    
    await db.commit()
```

#### 4.3 API 超时配置

```python
# src/constants.py

# LLM call timeouts
LLM_TIMEOUTS = {
    "extraction": 30,  # seconds
    "summary": 60,
    "risk_analysis": 30,
    "daily_review": 120,
}

# API call timeouts
API_TIMEOUTS = {
    "feishu_send": 10,
    "feishu_get_user": 5,
    "db_query": 5,
}
```

---

## E5: Retry Strategy

### 问题

Feishu API 和 LLM API 调用可能失败，缺少重试机制。

### 解决方案

#### 5.1 通用重试装饰器

```python
# src/retry.py

import random
from functools import wraps
from typing import Callable, Tuple, Type

def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    exponential_base: float = 2.0,
    exceptions: Tuple[Type[Exception], ...] = (Exception,)
):
    """
    Retry with exponential backoff + jitter.
    
    Delay = min(base_delay * (exponential_base ** attempt), max_delay)
    """
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_retries + 1):
                try:
                    return await func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    if attempt == max_retries:
                        raise
                    delay = min(base_delay * (exponential_base ** attempt), max_delay)
                    jitter = random.uniform(0.1, 0.3) * delay
                    await asyncio.sleep(delay + jitter)
            raise last_exception
        return wrapper
    return decorator
```

#### 5.2 Feishu API 重试

```python
# src/feishu/client.py

RETRYABLE_ERRORS = (FeishuRateLimitError, FeishuTimeoutError, asyncio.TimeoutError)

@retry_with_backoff(max_retries=3, base_delay=1.0, exceptions=RETRYABLE_ERRORS)
async def send_feishu_message(user_id: str, message: str) -> dict:
    """Send Feishu message with retry."""
    pass

@retry_with_backoff(max_retries=2, base_delay=0.5, exceptions=RETRYABLE_ERRORS)
async def get_feishu_user(user_id: str) -> dict:
    """Get Feishu user info with retry."""
    pass
```

#### 5.3 LLM API 重试 + Fallback

```python
# src/llm/client.py

RETRYABLE_LLM_ERRORS = (LLMRateLimitError, LLMTimeoutError, asyncio.TimeoutError)

# Fallback model chain
FALLBACK_MODELS = [
    "claude-3-sonnet",  # Primary
    "claude-3-haiku",   # Faster fallback
    "gpt-3.5-turbo",    # Last resort
]

@retry_with_backoff(max_retries=2, base_delay=2.0, exceptions=RETRYABLE_LLM_ERRORS)
async def call_llm(prompt: str, model: str = "claude-3-sonnet", timeout: int = 30) -> str:
    """Call LLM with retry and fallback to next model on rate limit."""
    try:
        return await _call_llm_internal(prompt, model, timeout)
    except LLMRateLimitError:
        # Try fallback model
        for fallback in FALLBACK_MODELS:
            if fallback != model:
                try:
                    return await _call_llm_internal(prompt, fallback, timeout)
                except LLMRateLimitError:
                    continue
        raise  # All models failed
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

## E6: Database Indexes

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

-- Foreign key constraints
ALTER TABLE checkin_session
ADD CONSTRAINT fk_checkin_session_user 
FOREIGN KEY (feishu_user_id) REFERENCES feishu_user(id) ON DELETE CASCADE;

ALTER TABLE checkin_session
ADD CONSTRAINT fk_checkin_session_manager 
FOREIGN KEY (manager_feishu_user_id) REFERENCES feishu_user(id) ON DELETE SET NULL;

ALTER TABLE message_log
ADD CONSTRAINT fk_message_log_session 
FOREIGN KEY (session_id) REFERENCES checkin_session(id) ON DELETE CASCADE;

ALTER TABLE extraction_snapshot
ADD CONSTRAINT fk_extraction_session 
FOREIGN KEY (session_id) REFERENCES checkin_session(id) ON DELETE CASCADE;

ALTER TABLE risk_item
ADD CONSTRAINT fk_risk_session 
FOREIGN KEY (session_id) REFERENCES checkin_session(id) ON DELETE CASCADE;

ALTER TABLE risk_item
ADD CONSTRAINT fk_risk_manager 
FOREIGN KEY (manager_feishu_user_id) REFERENCES feishu_user(id) ON DELETE SET NULL;

ALTER TABLE manager_brief
ADD CONSTRAINT fk_brief_manager 
FOREIGN KEY (manager_feishu_user_id) REFERENCES feishu_user(id) ON DELETE CASCADE;

ALTER TABLE manager_brief
ADD CONSTRAINT fk_brief_target 
FOREIGN KEY (target_user_id) REFERENCES feishu_user(id) ON DELETE SET NULL;
```

---

## E7: Daily Review Failure Isolation

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
| P1 | E2: Idempotency | 4h | 数据库迁移 |
| P1 | E4: Session Timeout | 2h | 无 |
| P2 | E3: LLM Error Handling | 4h | 无 |
| P2 | E5: Retry Strategy | 4h | 无 |
| P2 | E6: Database Indexes | 2h | 数据库迁移 |
| P2 | E7: Daily Review Isolation | 2h | 无 |

**总计**: ~22 小时

---

## 验收标准

### E1: State Machine
- [ ] 所有状态转换都有明确定义
- [ ] 并发消息不会导致状态混乱

### E2: Idempotency
- [ ] 重复消息被正确识别并忽略
- [ ] Session 创建幂等
- [ ] 数据库约束生效

### E3: LLM Error Handling
- [ ] LLM 超时自动回退到 regex
- [ ] 格式错误自动回退到 regex
- [ ] 所有方法失败时提示用户手动输入

### E4: Session Timeout
- [ ] 各状态超时时间已定义
- [ ] 超时检查任务每5分钟运行
- [ ] 超时动作正确执行

### E5: Retry Strategy
- [ ] Feishu API 失败自动重试
- [ ] 指数退避生效
- [ ] 最大重试次数限制

### E6: Database Indexes
- [ ] 所有索引已创建
- [ ] 查询性能符合预期

### E7: Daily Review Isolation
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
