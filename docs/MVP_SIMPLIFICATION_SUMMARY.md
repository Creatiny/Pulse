# Pulse MVP 简化建议汇总

**版本**: v1.0  
**日期**: 2026-04-18  
**目标**: 快速上线验证，最小可行产品

---

## 核心结论

| 维度 | 原方案 | 简化后 | 节省 |
|------|--------|--------|------|
| 开发周期 | 27 天 | 16 天 | 41% |
| 工程工时 | ~28 小时 | ~10 小时 | 64% |
| 试点周期 | 6 周 | 4 周 | 33% |
| 数据表 | 8 张 | 5 张 | 38% |
| 状态机 | 6 状态 | 4 状态 | 33% |

---

## 一、产品视角简化

### 1.1 可砍掉的功能 (延后到 v1.1)

| 功能 | 理由 | 节省时间 |
|------|------|----------|
| 团队摘要 (team_digest) | 二级输出，先跑通个人周报 | 2 天 |
| 每日复盘 → 每周复盘 | MVP 数据量小，每周足够 | 1 天 |
| 风险追踪独立流程 | 合并到周报采集，追问即可 | 1 天 |
| 员工负面反馈率监控 | 改用定性访谈 | 0.5 天 |

### 1.2 可简化的流程

**周报采集字段: 6 → 4**

| 原字段 | 简化后 | 理由 |
|--------|--------|------|
| progress | ✅ 保留 | 核心 |
| blocker | ✅ 保留 | 核心 |
| plan | ✅ 保留 | 核心 |
| support_needed | ✅ 保留 | 核心 |
| mood | ❌ 砍掉 | 非核心 |
| highlight | ❌ 砍掉 | 非核心 |

**状态机: 6 状态 → 4 状态**

```
原方案:
initiated → collecting → risk_followup → summary_confirm → confirmed
                                              ↓
                                           expired/failed

简化后:
initiated → collecting → confirmed
                ↓
            expired
```

- 砍掉 `risk_followup`: 风险追问合并到 collecting
- 砍掉 `summary_confirm`: 直接生成摘要，用户可修改但不强制确认
- 砍掉 `failed`: 失败直接 expired，用户重试

### 1.3 可减少的数据表

| 原表 | 处理 | 理由 |
|------|------|------|
| checkin_session | ✅ 保留 | 核心 |
| message_log | ✅ 保留 | 核心 |
| extraction_snapshot | ❌ 砍掉 | 合并到 session |
| risk_item | ✅ 保留 | 核心 |
| manager_brief | ❌ 砍掉 | MVP 先不发 brief |
| daily_review_report | ❌ 砍掉 | 改为每周人工复盘 |
| optimization_suggestion | ❌ 砍掉 | 延后 |
| feishu_user | ✅ 保留 | 核心 |

**简化后 5 张表:**
1. `feishu_user` - 用户
2. `checkin_session` - 会话
3. `message_log` - 消息
4. `risk_item` - 风险
5. `period_config` - 周期配置

### 1.4 最小验证指标

**必须监控 (2 个):**
1. 周报完成率 = confirmed / initiated ≥ 70%
2. 经理采纳率 = briefs viewed / briefs sent ≥ 60%

**辅助指标 (2 个):**
3. 平均对话轮数 3-8 轮
4. 平均完成时长 ≤ 5 分钟

---

## 二、工程视角简化

### 2.1 工程修复简化

| ID | 原方案 | 简化后 | 节省 |
|----|--------|--------|------|
| E1 State Machine | 4h | 2h (核心转换) | 2h |
| E2 Idempotency | 4h | 1h (简单去重) | 3h |
| E3 LLM Error | 4h | 2h (单层回退) | 2h |
| E4 Timeout | 2h | 0.5h (统一超时) | 1.5h |
| E5 Retry | 4h | 0h (延后) | 4h |
| E6 Indexes | 2h | 1h (核心索引) | 1h |
| E7 Daily Review | 2h | 1h (简化评分) | 1h |

**总计节省: ~15 小时**

### 2.2 可延后的工程项

| 项目 | 延后理由 |
|------|----------|
| 并发锁机制 | MVP 用户量小，并发概率低 |
| Session 复活机制 | 过期就过期，用户重新开始 |
| 多模型回退 | 单一模型 + Regex 足够 |
| Outbox Pattern | 失败直接记录日志 |
| 指数退避 + jitter | 固定重试 2 次即可 |
| 外键约束 | 先用逻辑关联 |
| 详细评分维度 | 只统计完成率 + 失败原因 |

### 2.3 简化后的工程实现

**E1 State Machine (2h):**
```python
# 只实现核心转换
VALID_TRANSITIONS = {
    "initiated": ["collecting", "expired"],
    "collecting": ["confirmed", "expired"],
    "confirmed": [],
    "expired": [],
}

# 统一 48h 超时
TIMEOUT = timedelta(hours=48)
```

**E2 Idempotency (1h):**
```python
# 只加唯一约束
ALTER TABLE message_log 
ADD CONSTRAINT uq_channel_message_id UNIQUE (channel_message_id);
```

**E3 LLM Error (2h):**
```python
# 单层回退
async def extract(message):
    try:
        return await llm_extract(message)
    except:
        return regex_extract(message)
```

**E4 Timeout (0.5h):**
```python
# 统一超时检查
async def check_timeouts():
    for session in get_active_sessions():
        if session.last_activity_at < now - 48h:
            session.status = "expired"
```

**E5 Retry - 延后**

**E6 Indexes (1h):**
```sql
-- 只建核心索引
CREATE INDEX idx_session_user ON checkin_session(feishu_user_id);
CREATE INDEX idx_session_status ON checkin_session(status);
CREATE INDEX idx_message_session ON message_log(session_id);
```

**E7 Daily Review (1h):**
```python
# 简化评分：只统计完成率
async def daily_review():
    stats = {
        "total": count_sessions(),
        "completed": count_sessions(status="confirmed"),
        "expired": count_sessions(status="expired"),
    }
    log.info(f"完成率: {stats['completed'] / stats['total']:.1%}")
```

### 2.4 最小监控

**3 个指标:**
1. Session 完成率
2. LLM 成功率
3. 每日活跃用户数

**2 条告警:**
1. LLM 失败率 > 30% → 通知开发者
2. Session 完成率 < 50% → 记录日志

---

## 三、简化后的 MVP 实施清单

### Phase 1: 核心流程 (8h)

- [ ] 数据库建表 (5 张)
- [ ] 核心索引
- [ ] 状态转换 (4 状态)
- [ ] 消息去重
- [ ] LLM 提取 + Regex 回退
- [ ] 统一 48h 超时

### Phase 2: 飞书集成 (4h)

- [ ] 飞书 webhook 接收
- [ ] 消息发送
- [ ] 用户信息获取

### Phase 3: 周报生成 (4h)

- [ ] 摘要生成
- [ ] 风险抽取
- [ ] 简单追问

### Phase 4: 监控 (2h)

- [ ] 3 个指标打点
- [ ] 2 条告警规则

**总计: ~18 小时 (2-3 天)**

---

## 四、延后到 v1.1 的功能

| 功能 | 延后理由 | 预计工时 |
|------|----------|----------|
| 团队摘要 | 先跑通个人周报 | 2 天 |
| 经理 Brief | 先验证员工端 | 1 天 |
| 每日复盘 | 每周足够 | 1 天 |
| 并发锁 | 用户量小 | 2h |
| 多模型回退 | 单模型足够 | 2h |
| Outbox Pattern | 失败率低 | 4h |
| 外键约束 | 逻辑关联足够 | 1h |
| 详细监控 | 先跑通流程 | 2h |

---

## 五、风险与建议

### 5.1 简化风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 无并发锁 | 可能状态混乱 | MVP 用户量小，概率低 |
| 无 Session 复活 | 用户需重新开始 | 提示用户重试 |
| 无多模型回退 | 单点故障 | Regex 兜底 |
| 无外键约束 | 数据不一致 | 应用层校验 |

### 5.2 建议

1. **先跑通核心流程** - 员工周报采集 → 摘要生成
2. **快速验证** - 2 周试点，收集反馈
3. **迭代优化** - 根据反馈决定 v1.1 优先级
4. **监控先行** - 上线前先部署监控

---

## 六、决策点

需要确认:

1. **是否砍掉经理 Brief?** - MVP 只验证员工端
2. **是否砍掉每日复盘?** - 改为每周人工复盘
3. **是否统一 48h 超时?** - 不区分状态
4. **是否延后外键约束?** - 先用逻辑关联

建议: **全部 Yes**，目标是 2-3 天内上线验证。
