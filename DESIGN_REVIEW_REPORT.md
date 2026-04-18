# Team Pulse MVP Architecture — Design Review Report

**Review Date:** 2026-04-18  
**Review Type:** Multi-dimensional Design Review (gstack)  
**Documents Reviewed:**
- `manager_pulse_mvp_lean_architecture.md` (Architecture Design)
- `manager_pulse_mvp_prd.md` (PRD)

**Reviewers:** CEO Review, Engineering Review, Design Review, Security Audit (CSO)

---

## Executive Summary

| Dimension | Rating | Key Insight |
|-----------|--------|-------------|
| CEO / Strategic | 8.25/10 | Strong MVP with excellent scope discipline; missing competitive analysis |
| Engineering | 7/10 | Solid foundation but critical gaps in state machine, idempotency, error handling |
| Design / UX | 3/10 | Severely under-specified user experience; backend-focused, not user-focused |
| Security | 6/10 | Medium-high risk; missing encryption, auth validation, authorization model |

**Overall Verdict:** The architecture demonstrates excellent scope discipline and correct "build vs buy" decisions. However, it is significantly under-specified in user experience design and has critical security gaps that must be addressed before handling real employee data.

---

## Part 1: CEO / Strategic Review

### Ratings

| Dimension | Rating | Summary |
|-----------|--------|---------|
| Product Strategy & Business Value | 8/10 | Clear problem-solution fit, strong value proposition |
| Market Fit & Target User Alignment | 7/10 | Well-defined ICP, but competitive landscape analysis missing |
| MVP Scope Appropriateness | 9/10 | Excellent scope discipline, correctly identifies what NOT to build |
| Build vs Buy Decisions | 9/10 | Smart leverage of Feishu and OpenClaw, minimal reinvention |

### Critical Gaps

1. **No competitive landscape analysis** — What Feishu native tools or third-party apps already solve this?
2. **No pre-build user validation** — Has anyone talked to target managers?
3. **LLM quality uncertainty** — Has baseline extraction quality been validated?
4. **Missing operational concerns** — Error handling, rate limiting, data retention

### Top Recommendations (Priority 1)

| # | Recommendation | Effort |
|---|----------------|--------|
| 1 | Conduct competitive audit of Feishu ecosystem | 2-3 days |
| 2 | Interview 5-10 target managers | 1-2 days |
| 3 | Run 10-20 mock conversations manually to validate LLM quality | 1 day |
| 4 | Define error handling strategy | 2-4 hours |
| 5 | Specify database indexes | 1-2 hours |

---

## Part 2: Engineering Review

### Architecture Findings

| ID | Priority | Confidence | Issue |
|----|----------|------------|-------|
| 1.1 | P1 | 9/10 | Session state machine needs explicit transition rules |
| 1.2 | P1 | 8/10 | No idempotency strategy for Feishu message delivery |
| 1.3 | P2 | 7/10 | Missing error handling for LLM extraction failures |
| 1.4 | P2 | 8/10 | No retry strategy for Feishu API failures |
| 1.5 | P2 | 7/10 | Daily review job has no failure isolation |
| 1.6 | P3 | 6/10 | Missing observability hooks |

### Critical Issues

#### 1. Session State Machine (P1)

The status enum lists states but doesn't define:
- Valid transitions (can you go from `collecting` back to `initiated`?)
- Timeout behavior (when does `initiated` become `expired`?)
- Concurrent message handling (what if user sends 3 messages rapidly?)

**Recommendation:** Add explicit state transition diagram:

```
initiated --[first_user_message]--> collecting
collecting --[extraction_complete]--> risk_followup OR summary_confirmation
risk_followup --[followup_done]--> summary_confirmation
summary_confirmation --[user_confirms]--> confirmed
summary_confirmation --[user_rejects]--> collecting (loop back)
ANY --[timeout: 24h]--> expired
```

#### 2. Idempotency (P1)

Feishu webhooks can deliver duplicate messages. The design mentions `channel_message_id` but doesn't specify deduplication.

**Recommendation:**
```sql
-- message_log should have unique constraint
UNIQUE(channel_message_id)
-- Or use upsert pattern
| INSERT INTO message_log (...) ON CONFLICT (channel_message_id) DO NOTHING
| ```

---

## Part 3: Design / UX Review

### Overall Rating: 3/10

The plan describes backend architecture well but severely under-specifies the user-facing experience.

### Dimension Ratings

| Dimension | Rating | Gap |
|-----------|--------|-----|
| User Experience Flow | 3/10 | No user journey, no first-time experience, no progress indication |
| Information Architecture | 2/10 | No message card layouts, no visual hierarchy |
| Interaction Design | 3/10 | No interaction states, no timeout/expiration UX, no error recovery |
| Conversational UX | 4/10 | No persona/tone definition, no off-topic handling |

### Missing UX Specifications

1. **No first-time user experience** — What does the user see when the bot initiates a check-in?
2. **No progress indication** — How does the user know where they are in the conversation?
3. **No error states** — What happens when extraction fails? When the user is confused?
4. **No conversation design** — Tone, persona, handling of off-topic responses

---

## Part 4: Security Audit (CSO)

### Overall Risk: MEDIUM-HIGH

### Critical Security Findings

| ID | Severity | Confidence | Category | Issue |
|----|----------|------------|----------|-------|
| S1 | HIGH | 9/10 | Information Disclosure | Employee conversation data not encrypted at rest |
| S2 | HIGH | 9/10 | Auth Failure | Single point of auth failure (Feishu-only) |
| S3 | HIGH | 9/10 | Broken Access Control | Missing authorization model |
| S4 | MEDIUM | 8/10 | Information Disclosure | Manager brief exposure risk |
| S5 | MEDIUM | 8/10 | Privacy | Feishu user cache over-collection |

### Critical Issues

#### S1: Employee Conversation Data Privacy (HIGH)

`message_log.message_text` stores employee conversations without encryption-at-rest specification.

**Impact:** Database breach exposes all historical employee conversations, sensitive work situations, performance concerns.

**Recommendation:**
1. Encrypt `message_text` and `summary_text` columns using AES-256-GCM
2. Use envelope encryption with KMS for key management
3. Document data retention policy and implement automated purging

#### S2: Single Point of Authentication Failure (HIGH)

Architecture relies entirely on Feishu for authentication. No secondary validation.

**Exploit Scenario:** Attacker compromises Feishu bot token → can impersonate any user.

**Recommendation:**
1. Implement Feishu webhook signature verification (HMAC-SHA256)
2. Validate `open_id` against Feishu API before sensitive operations
3. Log all authentication events

#### S3: Missing Authorization Model (HIGH)

Design states "No separate complex permission system" but lacks basic authorization checks:
- Can user X view session Y?
- Can manager M view brief for team T?

**Recommendation:**
1. Implement ownership check: user can only access their own sessions
2. Implement manager-team relationship check for brief access
3. Log all authorization failures

---

## Consolidated Recommendations

### Priority 1 (Before Implementation)

| # | Area | Recommendation | Effort |
|---|------|----------------|--------|
| 1 | Security | Add column-level encryption for message_text and summary_text | 1 day |
| 2 | Security | Implement Feishu webhook signature verification | 4 hours |
| 3 | Security | Add basic authorization checks (session ownership, team membership) | 1 day |
| 4 | Engineering | Define session state machine transitions and timeouts | 4 hours |
| 5 | Engineering | Add idempotency key (channel_message_id unique constraint) | 2 hours |
| 6 | CEO | Conduct competitive audit of Feishu ecosystem | 2-3 days |
| 7 | CEO | Interview 5-10 target managers | 1-2 days |

### Priority 2 (During Implementation)

| # | Area | Recommendation | Effort |
|---|------|----------------|--------|
| 8 | Engineering | Add LLM extraction fallback (regex rules) | 4 hours |
| 9 | Engineering | Add message outbox pattern for retry | 1 day |
| 10 | Design | Define conversation persona and tone | 2 hours |
| 11 | Design | Create message card mockups | 1 day |
| 12 | Security | Add access logging for brief retrieval | 4 hours |

### Priority 3 (Post-MVP)

| # | Area | Recommendation | Effort |
|---|------|----------------|--------|
| 13 | Engineering | Add observability (completion rates, latency metrics) | 2 days |
| 14 | Security | Implement brief expiration and watermarking | 1 day |
| 15 | CEO | Define business model hypothesis | 1 day |

---

## Summary

**What's Good:**
- Excellent scope discipline — correctly cuts platform-building
- Smart leverage of Feishu and OpenClaw — minimal reinvention
- Clear value proposition and target user definition
- 4-step implementation sequence with validation gates

**What Needs Work:**
- **Security (Critical):** Encryption, auth validation, authorization model missing
- **Engineering (Important):** State machine, idempotency, error handling gaps
- **Design (Significant):** UX severely under-specified — this is a chat product with no conversation design
- **Validation (Important):** No competitive analysis or pre-build user research

**Recommendation:** Address Priority 1 security and engineering items before handling real employee data. Add UX specifications before implementation begins.

#### 3. LLM Failure Handling (P2)

What happens when LLM returns malformed JSON or times out?

**Recommendation:**
```
1. Try LLM extraction with structured output
2. If timeout/malformed: fall back to regex rule extraction
3. If still fails: mark extraction_snapshot.missing_fields_json with all fields
4. Prompt user for manual input on missing fields
```

### Data Model Issues

| ID | Issue | Recommendation |
|----|-------|----------------|
| 2.1 | `feishu_user_cache` has no sync strategy | Add TTL-based refresh or webhook-triggered update |
| 2.2 | `message_log` will grow quickly | Add partitioning strategy by created_at |
| 2.3 | No index specifications | Define indexes before implementation |

---

## Part 3: Design / UX Review

### Overall Rating: 3/10

The plan describes backend architecture well but severely under-specifies the user-facing experience.

### Dimension Ratings

| Dimension | Rating | Gap |
|-----------|--------|-----|
| User Experience Flow | 3/10 | No user journey, no first-time experience, no progress indication |
| Information Architecture | 2/10 | No message card layouts, no visual hierarchy |
| Interaction Design | 3/10 | No interaction states, no timeout/expiration UX, no error recovery |
| Conversational UX | 4/10 | No persona/tone definition, no off-topic handling |

### What a 10 Looks Like

#### User Experience Flow (Target: 10/10)

```
STEP | USER SEES                    | USER FEELS      | DESIGN SPEC
-----|------------------------------|-----------------|----------------
1    | Bot greeting + question card | Curious/safe    | "Hi [Name], time for your weekly check-in..."
2    | Progress indicator           | Guided          | "Question 1 of 3: What did you accomplish?"
3    | Acknowledgment + next prompt | Heard           | "Got it. Any blockers?"
4    | Summary card for review      | In control      | Interactive card with Edit/Confirm buttons
5    | Confirmation message         | Complete        | "Thanks! Your manager will see this."
```

#### Information Architecture (Target: 10/10)

```
MESSAGE CARD HIERARCHY (Summary Confirmation):
┌─────────────────────────────────────────┐
│ PRIMARY: Your Weekly Summary            │  <- Bold,
