# Pulse MVP Engineering Review Report

**Review Date:** 2026-04-18  
**Review Type:** Engineering Review (gstack /plan-eng-review)  
**Documents Reviewed:**
- docs/manager_pulse_mvp_lean_architecture.md (Architecture Design)
- docs/manager_pulse_mvp_prd.md (PRD)
- docs/ENGINEERING_FIXES.md (Proposed Fixes)

**Focus Areas:** Architecture soundness, Data flow, State management, Edge cases, Failure modes, State machine completeness, Idempotency, Retry strategy, Database design

---

## Executive Summary

| Dimension | Rating | Key Insight |
|-----------|--------|-------------|
| Architecture Soundness | 7/10 | Good decomposition but missing integration contracts |
| Data Flow | 6/10 | Flow described but gaps in error paths |
| State Machine | 5/10 | States defined, transitions incomplete |
| Edge Cases | 4/10 | Many critical edge cases unaddressed |
| Idempotency | 3/10 | Missing for Feishu webhooks and session creation |
| Retry Strategy | 4/10 | Mentioned but not specified |
| Database Design | 6/10 | Tables defined, missing indexes and constraints |

**Overall:** The architecture demonstrates good scope discipline and correct build-vs-buy decisions. However, critical engineering gaps remain in state machine completeness, idempotency guarantees, and failure handling. The ENGINEERING_FIXES.md document addresses some but not all issues.

---

## 1. Architecture Soundness

### 1.1 Module Decomposition (Rating: 7/10)

**What's Good:**
- Clear 4-module decomposition: Check-in Orchestrator, Extract and Risk Analyzer, Summary/Brief Generator, Daily Review Job
- Correct decision to not build: user auth, org management, notification system, web backend
- Smart reuse of Feishu for identity and OpenClaw for channel/agent orchestration

**Findings:**

| ID | Severity | Confidence | Issue |
|----|----------|------------|-------|
| A1 | P1 | 8/10 | No integration contracts between modules |
| A2 | P2 | 7/10 | OpenClaw integration points undefined |
| A3 | P2 | 6/10 | LLM workflow integration not specified |

#### A1: Missing Integration Contracts (P1)

The architecture defines 4 modules but does not specify:
- API contracts between modules (function signatures, request/response types)
- Error propagation between modules
- Synchronous vs asynchronous boundaries

**Impact:** Implementation will have inconsistent interfaces, making testing and debugging difficult.

**Recommendation:** Define explicit contracts for each module interface.

#### A2: OpenClaw Integration Points Undefined (P2)

Architecture states OpenClaw is the channel and Agent orchestration base but does not specify:
- How Team Pulse Agent registers with OpenClaw
- Message routing configuration
- Context management integration

---

## 2. Data Flow and State Management

### 2.1 Check-in Session Flow (Rating: 6/10)

**Findings:**

| ID | Severity | Confidence | Issue |
|----|----------|------------|-------|
| D1 | P0 | 9/10 | Concurrent message handling undefined |
| D2 | P1 | 8/10 | Extraction failure path missing |
| D3 | P1 | 7/10 | Summary rejection flow incomplete |
| D4 | P2 | 6/10 | Manager brief generation trigger undefined |

#### D1: Concurrent Message Handling (P0)

**Scenario:** User sends 3 messages rapidly before bot responds.

**Current State:** Architecture does not address this. ENGINEERING_FIXES.md proposes session_lock but implementation details are incomplete.

**Failure Mode:**
1. Message 1 arrives, extraction starts
2. Message 2 arrives, extraction starts again
3. Message 3 arrives, extraction starts again
4. Race condition on session status update
5. Summary generated from partial/wrong messages

**Recommendation:** Implement per-session message queue with sequential processing using async locks.

#### D2: Extraction Failure Path (P1)

**Scenario:** LLM extraction fails (timeout, rate limit, malformed response).

**Current State:** ENGINEERING_FIXES.md proposes fallback to regex, but:
- No specification of what happens if regex also fails
- No user notification strategy
- No retry vs abort decision criteria

**Recommendation:** Complete fallback chain with explicit user notification at each stage.

#### D3: Summary Rejection Flow (P1)

**Scenario:** User rejects summary and provides corrections.

**Current State:** State machine shows summary_confirm to collecting transition, but:
- How are corrections merged with existing extraction?
- Is previous extraction discarded or amended?
- How many rejection cycles before forced completion?

---

## 3. State Machine Completeness

### 3.1 Session Status State Machine (Rating: 5/10)

**What's Good:**
- Status enum defined: initiated, collecting, risk_followup, summary_confirmation, confirmed, expired
- ENGINEERING_FIXES.md adds state transition diagram and VALID_TRANSITIONS table
- Terminal states identified: confirmed, expired, failed

**Findings:**

| ID | Severity | Confidence | Issue |
|----|----------|------------|-------|
| S1 | P1 | 9/10 | Missing failed state recovery path |
| S2 | P1 | 8/10 | Timeout behavior incomplete |
| S3 | P2 | 7/10 | No state recovery mechanism after crash |
| S4 | P2 | 6/10 | period_key uniqueness not enforced |

#### S1: Failed State Recovery (P1)

ENGINEERING_FIXES.md adds failed status but does not specify:
- Can failed sessions be retried by user?
- What data is preserved for debugging?
- How is user notified?

#### S2: Timeout Behavior Incomplete (P1)

ENGINEERING_FIXES.md defines timeouts:
- initiated: 24h
- collecting: 2h between messages
- risk_followup: 1h
- summary_confirm: 1h

**Missing:**
- What happens when timeout triggers? (auto-expire? notify user?)
- Who enforces timeouts? (background job? lazy check on access?)
- How are timeout events logged for analytics?

#### S3: No State Recovery After Crash (P2)

**Scenario:** Service crashes while processing message.

**Current State:** No crash recovery mechanism defined.

**Failure Mode:**
1. Session in collecting state
2. Service crashes during LLM extraction
3. Service restarts
4. Session stuck in collecting with no activity
5. Eventually times out (2h later)

**Recommendation:** Implement crash recovery with background job to detect and recover stuck sessions.

---

## 4. Edge Cases and Failure Modes

### 4.1 Edge Case Analysis (Rating: 4/10)

**Findings:**

| ID | Severity | Confidence | Issue |
|----|----------|------------|-------|
| E1 | P0 | 9
/10 | Duplicate Feishu webhook delivery not handled |
| E2 | P1 | 8/10 | User sends message to expired session |
| E3 | P1 | 7/10 | Manager changes during active session |
| E4 | P1 | 8/10 | Same user multiple concurrent sessions |
| E5 | P2 | 6/10 | Feishu user deleted/deactivated mid-session |
| E6 | P2 | 7/10 | Empty or whitespace-only message |
| E7 | P2 | 6/10 | Extremely long message (10KB+) |

#### E1: Duplicate Feishu Webhook Delivery (P0)

**Scenario:** Feishu delivers the same message 3 times (network retry).

**Current State:** ENGINEERING_FIXES.md proposes unique constraint on channel_message_id but:
- No specification of idempotency key format
- No handling for out-of-order delivery
- No dedup window specification

**Failure Mode:**
1. Feishu sends message A (id: msg-123)
2. Network timeout, Feishu retries
3. Message A delivered again (same id: msg-123)
4. System processes twice, creates duplicate extraction
5. Summary contains duplicate information

**Recommendation:** Add unique constraint and upsert pattern in application code.

#### E4: Multiple Concurrent Sessions (P1)

**Scenario:** User has active weekly checkin, then daily pulse triggers.

**Current State:** Not addressed. period_key exists but uniqueness not enforced.

**Failure Mode:**
1. Weekly session in collecting state
2. Daily pulse job fires
3. Creates second session for same user
4. User confused by two parallel conversations

**Recommendation:** Enforce one active session per user, or merge contexts.

---

## 5. Idempotency and Retry Strategy (Rating: 3/10)

**Findings:**

| ID | Severity | Confidence | Issue |
|----|----------|------------|-------|
| I1 | P0 | 9/10 | Feishu webhook idempotency incomplete |
| I2 | P1 | 8/10 | Session creation not idempotent |
| I3 | P1 | 7/10 | Brief generation not idempotent |
| I4 | P2 | 6/10 | Daily review job not idempotent |

#### I1: Feishu Webhook Idempotency (P0)

ENGINEERING_FIXES.md proposes:
- Unique constraint on channel_message_id
- Upsert pattern

**Missing:**
- Deduplication window (how long to track?)
- Out-of-order message handling
- Message ID format validation

**Recommendation:**
```
ALTER TABLE message_log 
ADD CONSTRAINT uq_channel_message_id UNIQUE (channel_message_id);
```

#### I2: Session Creation Not Idempotent (P1)

**Scenario:** Daily pulse job fires twice (scheduler bug).

**Failure Mode:**
1. Job creates session for user A
2. Job runs again (duplicate trigger)
3. Creates second session for user A
4. User receives two check-in prompts

**Recommendation:** Add idempotency_key to checkin_session with unique constraint.

---

## 6. Retry Strategy (Rating: 4/10)

**Findings:**

| ID | Severity | Confidence | Issue |
|----|----------|------------|-------|
| R1 | P1 | 8/10 | Feishu API retry not specified |
| R2 | P1 | 7/10 | LLM API retry not specified |
| R3 | P2 | 6/10 | Database retry not specified |
| R4 | P2 | 7/10 | No circuit breaker pattern |

#### R1: Feishu API Retry (P1)

**Scenario:** Feishu send_message returns 429 (rate limit) or 503 (service unavailable).

**Current State:** Not addressed.

**Recommendation:** Implement exponential backoff with jitter:

```
max_retries: 3
base_delay: 1s
max_delay: 30s
jitter: true
retry_on: [429, 503, 502, timeout]
```

#### R2: LLM API Retry (P1)

**Scenario:** LLM API returns 429 or times out.

**Current State:** ENGINEERING_FIXES.md mentions fallback to regex but no retry before fallback.

**Recommendation:** Retry LLM before falling back:

```
1. Try LLM (timeout: 10s)
2. On retryable error, retry with backoff (max 2 retries)
3. On exhaustion, fall back to regex
4. On regex failure, ask user for clarification
```

---

## 7. Database Design (Rating: 6/10)

**What's Good:**
- Core tables defined: checkin_session, message_log, extraction_snapshot, daily_brief
- JSONB for flexible extraction data
- Timestamps and soft delete pattern

**Findings:**

| ID | Severity | Confidence | Issue |
|----|----------|------------|-------|
| DB1 | P1 | 9/10 | Missing critical indexes |
| DB2 | P1 | 8/10 | No foreign key constraints defined |
| DB3 | P2 | 7/10 | extraction_snapshot JSONB schema not validated |
| DB4 | P2 | 6/10 | No partitioning strategy for message_log |

#### DB1: Missing Critical Indexes (P1)

**Required indexes:**

```sql
-- Session lookup by user
CREATE INDEX idx_checkin_session_user_status 
ON checkin_session(feishu_user_id, status) 
WHERE status NOT IN ('confirmed', 'expired', 'failed');

-- Message log by session
CREATE INDEX idx_message_log_session 
ON message_log(session_id, created_at);

-- Brief lookup by manager and period
CREATE INDEX idx_daily_brief_manager_period 
ON daily_brief(manager_feishu_user_id, period_key);

-- Extraction by session
CREATE INDEX idx_extraction_snapshot_session 
ON extraction_snapshot(session_id);
```

#### DB2: No Foreign Key Constraints (P1)

**Current State:** Tables reference each other but no FK constraints defined.

**Impact:**
- Orphaned records possible
- No cascade delete behavior
- Data integrity not enforced

**Recommendation:**

```sql
ALTER TABLE message_log 
ADD CONSTRAINT fk_message_log_session 
FOREIGN KEY (session_id) REFERENCES checkin_session(id) 
ON DELETE CASCADE;

ALTER TABLE extraction_snapshot 
ADD CONSTRAINT fk_extraction_session 
FOREIGN KEY (session_id) REFERENCES checkin_session(id) 
ON DELETE CASCADE;
```

#### DB3: JSONB Schema Not Validated (P2)

**Current State:** extraction_snapshot.extracted_data is JSONB with no schema validation.

**Risk:**
- Malformed LLM output stored directly
- Query errors on unexpected structure
- Migration pain if schema evolves

**Recommendation:** Add check constraint or validate in application layer.

---

## 8. Summary of Critical Issues (P0)

| ID | Issue | Impact | Recommendation |
|----|-------|--------|----------------|
| D1 | Concurrent message handling | Race conditions, corrupted state | Per-session async lock |
| E1 | Duplicate webhook delivery | Duplicate processing, confused users | Unique constraint + upsert |
| I1 | Webhook idempotency incomplete | Same as E1 | Complete idempotency pattern |

**Action Required:** These P0 issues must be resolved before implementation.

---

## 9. Summary of High-Priority Issues (P1)

| ID | Issue | Impact |
|----|-------|--------|
| A1 | Missing integration contracts | Inconsistent interfaces |
| D2 | Extraction failure path | User confusion, abandoned sessions |
| D3 | Summary rejection flow | Incomplete user corrections |
| S1 | Failed state recovery | No retry path for users |
| S2 | Timeout behavior | Sessions stuck indefinitely |
| E2 | Message to expired session | Error or confusion |
| E4 | Multiple concurrent sessions | User confusion |
| I2 | Session creation not idempotent | Duplicate sessions |
| R1 | Feishu API retry | Message delivery failures |
| R2 | LLM API retry | Extraction failures |
| DB1 | Missing indexes | Slow queries, scalability issues |
| DB2 | No foreign keys | Data integrity issues |

---

## 10. Recommendations Summary

### Must Fix Before Implementation (P0)

1. **Concurrent Message Handling:** Implement per-session async lock with message queue
2. **Webhook Idempotency:** Add unique constraint on channel_message_id, implement upsert pattern
3. **Session Creation Idempotency:** Add idempotency_key to checkin_session

### Should Fix Before Implementation (P1)

1. Define integration contracts between modules
2. Complete extraction failure fallback chain
3. Define summary rejection merge strategy
4. Add failed state recovery path
5. Implement timeout enforcement (background job)
6. Add critical database indexes
7. Add foreign key constraints
8. Implement retry strategies for Feishu and LLM APIs

### Can Defer (P2)

1. OpenClaw integration specification
2. Crash recovery mechanism
3. JSONB schema validation
4. Circuit breaker pattern
5. Message log partitioning

---

## Appendix: State Machine Diagram

```
                    +------------+
                    | initiated  |
                    +-----+------+
                          |
                    user sends message
                          |
                          v
                    +------------+
                    | collecting |
                    +-----+------+
                          |
        +-----------------+-----------------+
        |                                   |
  extraction complete              risks detected
        |                                   |
        v                                   v
  +------------+                    +--------------+
  |  summary   |                    | risk_followup |
  |   confirm  |                    +------+-------+
  +-----+------+                           |
        |                             followup done
        |                                   |
        +-----------------+-----------------+
                          |
                          v
                    +------------+
                    | confirmed  |  (terminal)
                    +------------+

        Any state --[timeout]--> expired (terminal)
        Any state --[error]---> failed (terminal)
```

---

## Appendix: Data Flow Diagram

```
User Message Flow:
==================

Feishu --> Webhook Handler --> Message Log (append)
                                  |
                                  v
                            Session Lock
                                  |
                                  v
                            Load Session
                                  |
                                  v
                            State Machine Router
                                  |
        +-------------------------+-------------------------+
        |                         |                         |
   collecting              risk_followup              summary_confirm
        |                         |                         |
        v                         v                         v
   Extract/Risk            Process Followup          Generate Summary
        |                         |                         |
        v                         |                         v
   Update Snapshot              |                    Show to User
        |                         |                         |
        +-------------------------+-------------------------+
                                  |
                                  v
                            Save Session
                                  |
                                  v
                            Send Response --> Feishu
```

---

**Report Generated:** 2026-04-18  
**Review Methodology:** gstack plan-eng-review skill  
**Next Steps:** Address P0 issues before implementation, review P1 issues with team
