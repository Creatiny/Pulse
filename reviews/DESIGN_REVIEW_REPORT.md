# Pulse MVP Design Review Report

**Review Date:** 2026-04-18
**Review Scope:** CEO + Engineering perspectives (Security and UX/UI excluded per project scope)
**Documents Reviewed:**
- `docs/manager_pulse_mvp_lean_architecture.md` (Architecture v1.1)
- `docs/manager_pulse_mvp_prd.md` (PRD)
- `docs/ENGINEERING_FIXES.md` (Engineering Fixes Spec)

---

## Executive Summary

| Dimension | Rating | Key Insight |
|-----------|--------|-------------|
| Product Strategy | 6/10 | Clear problem, weak differentiation |
| Business Value | 5/10 | Manager value defined, employee value vague |
| MVP Scope | 8/10 | Well-constrained, disciplined |
| Architecture Soundness | 7/10 | Good decomposition, missing contracts |
| Data Flow | 6/10 | Flow described, gaps in error paths |
| State Machine | 5/10 | States defined, transitions incomplete |
| Edge Cases | 4/10 | Many critical edge cases unaddressed |
| Idempotency | 3/10 | Critical gaps in webhook and session idempotency |
| Retry Strategy | 4/10 | Feishu and LLM retry strategies missing |
| Database Design | 6/10 | Tables defined, missing indexes and constraints |

**Overall Assessment:** Good scope discipline and build-vs-buy decisions. Critical gaps in idempotency, state machine completeness, and failure handling. ENGINEERING_FIXES.md addresses some issues but P0 items remain unresolved.

---

## P0 Issues (Must Fix Before Implementation)

| ID | Category | Issue | Impact | Recommendation |
|----|----------|-------|--------|----------------|
| CEO-1 | Strategy | No explicit kill criteria for MVP | Cannot declare failure, sunk cost escalation | Define hard stop: <50% completion rate after 3 weeks |
| CEO-2 | Validation | Employee engagement assumption unvalidated | Highest-risk assumption | Run pre-pilot survey or mock test with 3-5 employees |
| ENG-1 | Concurrency | Concurrent message handling undefined | Race conditions, corrupted state | Implement per-session message queue |
| ENG-2 | Idempotency | Duplicate Feishu webhook delivery not handled | Duplicate processing | Add unique constraint + upsert pattern |
| ENG-3 | Idempotency | Webhook idempotency incomplete | Same as ENG-2 | Complete idempotency_key implementation |

---

## P1 Issues (Should Fix Before Implementation)

### Product & Strategy (12 issues)

| ID | Issue | Recommendation |
|----|-------|----------------|
| CEO-3 | Vision lacks competitive moat | Define "unfair advantage" |
| CEO-4 | Target segment too broad | Pick ONE primary persona |
| CEO-5 | "Self-optimizing" value prop premature | Define minimum data threshold |
| CEO-6 | No monetization strategy | Define pilot pricing |
| CEO-7 | Value quantification missing baseline | Define pre-pilot measurement |
| CEO-8 | Daily auto-optimization is high-risk | Consider manual review first 2 weeks |
| CEO-9 | Manager trust unvalidated | Add confidence scores, citations |
| CEO-10 | No analysis of alternatives | Document switching rationale |
| CEO-11 | Metrics lack leading indicators | Add response rate, acceptance rate |
| CEO-12 | Employee value vague | Define measurable employee value |

### Engineering (15 issues)

| ID | Issue | Recommendation |
|----|-------|----------------|
| ENG-4 | Missing integration contracts | Define explicit API boundaries |
| ENG-5 | Extraction failure path incomplete | Complete fallback chain |
| ENG-6 | Summary rejection flow undefined | Define max 3 rejection cycles |
| ENG-7 | Failed state recovery missing | Add retry mechanism |
| ENG-8 | Timeout behavior incomplete | Define timeout per state |
| ENG-9 | Expired session handling undefined | Return graceful message |
| ENG-10 | Concurrent sessions undefined | Enforce one active session per user |
| ENG-11 | Session creation not idempotent | Add idempotency_key |
| ENG-12 | Feishu retry strategy missing | Implement exponential backoff |
| ENG-13 | LLM retry strategy missing | Implement backoff + fallback model |
| ENG-14 | Missing database indexes | Add indexes on key columns |
| ENG-15 | Missing foreign key constraints | Add FK constraints |

---

## Strengths

1. **Scope Discipline**: Excellent restraint on non-goals
2. **Build-vs-Buy**: Correct reuse of Feishu, OpenClaw
3. **Problem Definition**: Clear, validated pain points
4. **State Machine Foundation**: Status enum and transitions defined
5. **Data Model**: Core tables identified

---

## Recommended Next Steps

### Immediate (Before Implementation)

1. Define kill criteria in PRD
2. Validate employee engagement with mock test
3. Complete idempotency design
4. Define integration contracts
5. Add message queue for concurrency

### Short-term (Week 1-2)

6. Complete extraction failure fallback
7. Define timeout behavior
8. Add database indexes and FKs
9. Implement retry strategies
10. Define leading indicators
