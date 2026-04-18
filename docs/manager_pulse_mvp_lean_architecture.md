     1|# Team Pulse MVP 极简架构方案
     2|
     3|版本：v1.0  
     4|状态：MVP 极简方案  
     5|日期：2026-04-17  
     6|目标：最快完成真实团队验证
     7|
     8|## 1. 先回答你的问题
     9|
    10|是，会太复杂。
    11|
    12|如果当前目标是：
    13|
    14|- 尽快上线
    15|- 在真实团队里验证
    16|- 用最少工程量跑通闭环
    17|
    18|那就不应该先按“完整 SaaS 平台”来设计。
    19|
    20|对 Team Pulse MVP 来说，正确思路应该是：
    21|
    22|**能复用飞书的，复用飞书。能复用 OpenClaw 的，复用 OpenClaw。只有飞书和 OpenClaw 都不能可靠承担的状态，才自己存。**
    23|
    24|所以第一版不要做：
    25|
    26|- 自建完整组织架构系统
    27|- 自建复杂权限系统
    28|- 自建大而全的团队管理后台
    29|- 自建过多中间服务
    30|- 自建大量通用化数据模型
    31|
    32|第一版必须做的，只有 4 个核心机制：
    33|
    34|1. `异步引导式对话`
    35|2. `结构化摘要与风险提取`
    36|3. `经理可用输出`
    37|4. `每日自动复盘与优化`
    38|
    39|## 2. 极简架构原则
    40|
    41|### 2.1 飞书是组织与身份源
    42|
    43|直接复用飞书里的：
    44|
    45|- 用户身份
    46|- 部门/团队组织结构
    47|- 上下级关系
    48|- 群和私聊入口
    49|
    50|MVP 不单独维护一套完整 `organization/team/user/membership` 主数据。
    51|
    52|本地只缓存最少映射：
    53|
    54|- 飞书用户 ID
    55|- 飞书 open_id
    56|- 用户显示名
    57|- 直属经理 ID
    58|- 所属团队标识
    59|
    60|这些数据只作为运行时缓存，不作为“主系统”。
    61|
    62|### 2.2 OpenClaw 是渠道与 Agent 编排底座
    63|
    64|直接复用 OpenClaw：
    65|
    66|- Feishu Channel 接入
    67|- 消息路由
    68|- Agent 执行入口
    69|- 基础上下文管理
    70|
    71|Team Pulse 自己只补业务逻辑，不重复造一层聊天平台。
    72|
    73|### 2.3 本地数据库只存“不可替代”的业务状态
    74|
    75|第一版本地数据库只存 8 类数据：
    76|
    77|- check-in 会话
    78|- 对话消息
    79|- 抽取结果
    80|- 风险项
    81|- 已确认摘要
    82|- 1:1 brief
    83|- 反馈
    84|- 每日复盘结果
    85|
    86|也就是说：
    87|
    88|**不存组织全量主数据，只存业务闭环需要的最小状态。**
    89|
    90|### 2.4 不做后台，先做飞书内闭环
    91|
    92|MVP 的用户界面只有：
    93|
    94|- 飞书私聊机器人
    95|- 飞书消息卡片
    96|- 飞书群消息
    97|- 可选飞书文档输出
    98|
    99|先不做 Web 管理后台。
   100|
   101|### 2.5 不做自动策略自修改，只做“自动发现问题 + 生成优化建议”
   102|
   103|为了降低风险，第一版的“自优化”不要自动改 prompt。
   104|
   105|第一版只做：
   106|
   107|- 自动复盘
   108|- 自动打分
   109|- 自动找失败模式
   110|- 自动生成优化建议
   111|
   112|然后由你人工决定是否更新 prompt。
   113|
   114|这已经足够验证“系统能不能越来越好”。
   115|
   116|## 3. 极简系统架构
   117|
   118|```text
   119|飞书用户
   120|  ->
   121|OpenClaw Feishu Channel
   122|  ->
   123|Team Pulse Agent / App
   124|  ->
   125|PostgreSQL
   126|  ->
   127|每日复盘任务
   128|```
   129|
   130|就这么简单。
   131|
   132|如果再展开一层：
   133|
   134|```text
   135|Feishu
   136|  ->
   137|OpenClaw
   138|  ->
   139|1. Check-in Orchestrator
   140|2. Extract & Risk Analyzer
   141|3. Summary/Brief Generator
   142|4. Daily Review Job
   143|  ->
   144|PostgreSQL
   145|```
   146|
   147|第一版不要拆微服务，不要引入复杂事件总线。
   148|
   149|推荐部署形态：
   150|
   151|- 一个 API 服务
   152|- 一个 Worker/定时任务进程
   153|- 一个 PostgreSQL
   154|
   155|可选：
   156|
   157|- Redis，用于队列和幂等
   158|
   159|## 4. 哪些能力直接复用飞书
   160|
   161|飞书直接承担：
   162|
   163|- 用户登录和身份识别
   164|- 部门和团队关系
   165|- 私聊触达
   166|- 群通知
   167|- 交互卡片确认
   168|- 文档输出载体
   169|- 可能的日历 1:1 提醒
   170|
   171|所以第一版不要做：
   172|
   173|- 用户注册登录
   174|- 组织架构后台
   175|- 团队管理页面
   176|- 单独通知系统
   177|
   178|## 5. 哪些能力直接复用 OpenClaw
   179|
   180|OpenClaw 直接承担：
   181|
   182|- 飞书渠道接入
   183|- Agent 路由
   184|- 消息处理入口
   185|- 基础对话上下文拼接
   186|
   187|所以 Team Pulse 不再单独实现：
   188|
   189|- 渠道适配框架
   190|- 通用消息总线层
   191|- 通用 Agent 平台壳
   192|
## 6. 第一版真正要自己做的模块

只做 4 个模块。

### 6.0 模块集成契约 (Integration Contracts)

各模块间通过明确的 API 边界交互，确保松耦合和可测试性。

#### CheckinOrchestrator API

```python
# src/orchestrator/contracts.py

@dataclass
class ProcessResult:
    """Result of processing a user message."""
    success: bool
    new_status: SessionStatus
    response_message: Optional[str]
    extraction_result: Optional[ExtractionResult] = None
    error: Optional[str] = None

async def handle_message(
    session_id: str,
    message: str,
    sender_type: str  # "user" | "bot"
) -> ProcessResult:
    """
    Process incoming message and update session state.
    
    Responsibilities:
    - Validate message against current session state
    - Determine if extraction is needed
    - Trigger follow-up questions if needed
    - Update session status
    
    Errors:
    - SessionNotFoundError: session_id invalid
    - InvalidStateTransitionError: message not allowed in current state
    """
    pass

async def initiate_session(
    user_id: str,
    period_key: str,
    session_type: str
) -> CheckinSession:
    """
    Start new check-in session.
    
    Idempotent: returns existing session if already created.
    """
    pass
```

#### ExtractAnalyzer API

```python
# src/extraction/contracts.py

@dataclass
class ExtractionResult:
    """Result of extracting structured info from message."""
    method: str  # "llm" | "regex" | "manual"
    data: dict
    missing_fields: list[str]
    confidence: float  # 0.0 - 1.0
    raw_response: Optional[str] = None

@dataclass
class RiskItem:
    """Detected risk from message."""
    title: str
    description: str
    severity: str  # "low" | "medium" | "high"
    needs_followup: bool
    evidence: list[str]

async def extract(
    message_text: str,
    required_fields: list[str]
) -> ExtractionResult:
    """
    Extract structured info from natural language.
    
    Fallback chain:
    1. Try LLM with structured output
    2. Fall back to regex patterns
    3. Return empty result with missing_fields
    
    Errors:
    - LLMTimeoutError: LLM call timed out
    - LLMRateLimitError: LLM rate limited
    """
    pass

async def analyze_risks(
    extraction_result: ExtractionResult
) -> list[RiskItem]:
    """
    Identify risk signals from extraction result.
    
    Returns empty list if no risks detected.
    """
    pass
```

#### SummaryGenerator API

```python
# src/summary/contracts.py

@dataclass
class Summary:
    """Generated summary for a session."""
    text: str
    structured_json: dict
    source_message_ids: list[str]
    confidence: float

@dataclass
class Brief:
    """Manager brief for 1:1 or team digest."""
    brief_type: str  # "one_on_one" | "team_digest"
    manager_id: str
    target_user_id: Optional[str]
    text: str
    suggested_questions: list[str]

async def generate_summary(
    session_id: str
) -> Summary:
    """
    Generate summary from session messages and extractions.
    
    Prerequisites:
    - Session must be in collecting or risk_followup state
    - At least one extraction must exist
    
    Errors:
    - InsufficientDataError: not enough data to summarize
    """
    pass

async def generate_brief(
    manager_id: str,
    target_user_id: Optional[str],
    period_key: str,
    brief_type: str
) -> Brief:
    """
    Generate manager brief from confirmed sessions.
    
    For team_digest: target_user_id is None
    For one_on_one: target_user_id is required
    """
    pass
```

#### DailyReviewJob API

```python
# src/review/contracts.py

@dataclass
class SessionScore:
    """Score for a single session."""
    session_id: str
    completion_score: float  # 0.0 - 1.0
    info_completeness: float
    friction_score: float
    summary_accuracy: float
    overall_score: float

@dataclass
class ReviewReport:
    """Daily review report."""
    review_date: date
    total_sessions: int
    completed_sessions: int
    high_friction_sessions: int
    top_failure_modes: list[str]
    optimization_suggestions: list[str]

async def score_session(
    session_id: str
) -> SessionScore:
    """
    Score a single session for quality.
    
    Metrics:
    - completion_score: did session reach confirmed state
    - info_completeness: were all required fields extracted
    - friction_score: how many追问, how long
    - summary_accuracy: did user modify summary
    """
    pass

async def run_daily_review(
    review_date: date
) -> ReviewReport:
    """
    Run daily review for all sessions.
    
    Failure isolation: single session failure does not stop batch.
    Timeout protection: each session scored with 30s timeout.
    """
    pass
```

#### 模块依赖关系

```
CheckinOrchestrator
    ├── ExtractAnalyzer (extract, analyze_risks)
    └── SummaryGenerator (generate_summary)

SummaryGenerator
    └── (reads from DB: message_log, extraction_snapshot)

DailyReviewJob
    └── (reads from DB: checkin_session, message_log, extraction_snapshot)
```

#### 错误传播规则

1. **Orchestrator → Extract**: 如果 extract 失败，Orchestrator 决定是否重试或回退
2. **Extract → LLM**: LLM 错误由 Extract 内部处理，返回 fallback 结果
3. **Summary → DB**: DB 错误向上传播，由调用方决定重试策略
4. **DailyReview → Session**: 单个 session 失败不影响其他 session

---

## 6.1 Check-in Orchestrator
   198|
   199|职责：
   200|
   201|- 发起周报/日报对话
   202|- 维护当前对话阶段
   203|- 决定继续追问还是结束
   204|- 触发摘要生成
   205|
   206|这是核心中的核心。
   207|
   208|## 6.2 Extract & Risk Analyzer
   209|
   210|职责：
   211|
   212|- 从自然语言中抽取进展、阻塞、计划、支持诉求
   213|- 识别风险信号
   214|- 生成结构化风险项
   215|
   216|第一版建议规则 + LLM 混合，不要纯模型黑盒。
   217|
   218|## 6.3 Summary / Brief Generator
   219|
   220|职责：
   221|
   222|- 生成员工摘要
   223|- 生成团队摘要
   224|- 生成 1:1 brief
   225|
   226|注意：
   227|
   228|第一版团队摘要和 1:1 brief 都可以直接基于已确认摘要生成，不需要再额外做复杂知识图谱。
   229|
   230|## 6.4 Daily Review Job
   231|
   232|职责：
   233|
   234|- 每晚扫描所有已结束对话
   235|- 对每条对话打分
   236|- 汇总失败模式
   237|- 生成一份优化建议日报
   238|
   239|第一版最关键的是“看见问题”，不是“自动修自己”。
   240|
   241|## 7. 极简数据模型
   242|
   243|第一版建议把表数直接砍到最少，够用就行。
   244|
   245|## 7.1 feishu_user_cache
   246|
   247|用途：缓存飞书用户最少信息，不做主数据中心。
   248|
   249|建议字段：
   250|
   251|- `id`
   252|- `feishu_user_id`
   253|- `open_id`
   254|- `name`
   255|- `manager_feishu_user_id`
   256|- `team_key`
   257|- `raw_profile_json`
   258|- `updated_at`
   259|
   260|说明：
   261|
   262|- `team_key` 可以先直接用飞书部门 ID 或业务自定义团队标识
   263|- 不单独拆 `organization/team/membership`
   264|
   265|## 7.2 checkin_session
   266|
   267|用途：记录一次员工 check-in 主流程。
   268|
   269|建议字段：
   270|
   271|- `id`
   272|- `feishu_user_id`
   273|- `manager_feishu_user_id`
   274|- `team_key`
   275|- `session_type` (`daily_pulse` / `weekly_checkin`)
   276|- `period_key`
   277|- `status` (`initiated` / `collecting` / `risk_followup` / `summary_confirmation` / `confirmed` / `expired`)
   278|- `started_at`
   279|- `last_message_at`
   280|- `confirmed_at`
   281|- `summary_text`
   282|- `summary_json`
   283|- `prompt_version`
   284|
   285|说明：
   286|
   287|- 先把 summary 直接挂在 session 上，减少单独 summary 主表复杂度
   288|- 后期再拆表
   289|
   290|## 7.3 message_log
   291|
   292|用途：记录对话消息。
   293|
   294|建议字段：
   295|
   296|- `id`
   297|- `session_id`
   298|- `sender_type`
   299|- `channel_message_id`
   300|- `message_text`
   301|- `message_json`
   302|- `sequence_no`
   303|- `created_at`
   304|
   305|## 7.4 extraction_snapshot
   306|
   307|用途：记录某次抽取结果。
   308|
   309|建议字段：
   310|
   311|- `id`
   312|- `session_id`
   313|- `source_message_id`
   314|- `structured_json`
   315|- `missing_fields_json`
   316|- `confidence_json`
   317|- `created_at`
   318|
   319|## 7.5 risk_item
   320|
   321|用途：记录识别出的风险。
   322|
   323|建议字段：
   324|
   325|- `id`
   326|- `session_id`
   327|- `feishu_user_id`
   328|- `manager_feishu_user_id`
   329|- `team_key`
   330|- `title`
   331|- `description`
   332|- `severity`
   333|- `needs_manager_attention`
   334|- `status`
   335|- `evidence_json`
   336|- `created_at`
   337|- `updated_at`
   338|
   339|## 7.6 manager_brief
   340|
   341|用途：记录团队摘要或 1:1 brief。
   342|
   343|建议字段：
   344|
   345|- `id`
   346|- `brief_type` (`team_digest` / `one_on_one`)
   347|- `manager_feishu_user_id`
   348|- `target_feishu_user_id` null
   349|- `team_key`
   350|- `period_key`
   351|- `brief_text`
   352|- `brief_json`
   353|- `source_session_ids_json`
   354|- `created_at`
   355|
   356|说明：
   357|
   358|- 团队摘要和 1:1 brief 先共用一张表
   359|- MVP 不必拆成两张
   360|
   361|## 7.7 feedback_log
   362|
   363|用途：记录员工修正和经理反馈。
   364|
   365|建议字段：
   366|
   367|- `id`
   368|- `actor_feishu_user_id`
   369|- `target_type`
   370|- `target_id`
   371|- `feedback_type`
   372|- `comment_text`
   373|- `detail_json`
   374|- `created_at`
   375|
   376|## 7.8 daily_review_report
   377|
   378|用途：记录每日复盘结果。
   379|
   380|建议字段：
   381|
   382|- `id`
   383|- `review_date`
   384|- `total_sessions`
   385|- `completed_sessions`
   386|- `high_friction_sessions`
   387|- `report_text`
   388|- `report_json`
   389|- `created_at`
   390|
   391|## 7.9 optimization_suggestion
   392|
   393|用途：每日复盘提出的优化建议。
   394|
   395|建议字段：
   396|
   397|- `id`
   398|- `daily_review_report_id`
   399|- `suggestion_type`
   400|- `target_scope`
   401|- `title`
   402|- `suggestion_json`
   403|- `status` (`proposed` / `accepted` / `rejected` / `applied`)
   404|- `created_at`
   405|
   406|## 8. 为什么这些表是必须的
   407|
   408|下面这些不是“为了架构好看”，而是飞书和 OpenClaw 无法替你稳定承担。
   409|
   410|### 8.1 checkin_session 必须有
   411|
   412|因为你必须知道：
   413|
   414|- 这个人本周是否已经被发起过
   415|- 现在对话进行到哪一步
   416|- 是否已经确认摘要
   417|- 是否超时
   418|
   419|这些是业务状态，不是飞书会替你维护的。
   420|
   421|### 8.2 message_log 必须有
   422|
   423|因为你要：
   424|
   425|- 复盘对话质量
   426|- 回溯摘要依据
   427|- 找失败模式
   428|- 做后续 prompt 优化
   429|
   430|没有消息日志，就没有自优化闭环。
   431|
   432|### 8.3 risk_item 必须有
   433|
   434|因为“识别出过什么风险、后来是否被验证、是否误报”是产品价值本身。
   435|
   436|这不能只存在临时上下文里。
   437|
   438|### 8.4 daily_review_report 和 optimization_suggestion 必须有
   439|
   440|因为你特别强调“每天自动复盘并提炼改进点”。
   441|
   442|如果不落库，这个机制就只是一次性文本，不构成系统能力。
   443|
   444|## 9. 核心流程极简版
   445|
   446|## 9.1 周报采集
   447|
   448|```text
   449|定时任务发起
   450|  ->
   451|OpenClaw 通过飞书私聊用户
   452|  ->
   453|用户回答
   454|  ->
   455|Orchestrator 调用抽取
   456|  ->
   457|若缺关键信息则追问
   458|  ->
   459|生成摘要
   460|  ->
   461|用户确认/修正
   462|  ->
   463|session 完成
   464|```
   465|
   466|## 9.2 风险追问
   467|
   468|```text
   469|用户提到卡点/延期/依赖
   470|  ->
   471|Risk Analyzer 判断要不要追问
   472|  ->
   473|最多再追 1-2 个问题
   474|  ->
   475|生成 risk_item
   476|```
   477|
   478|## 9.3 经理输出
   479|
   480|```text
   481|读取已确认 session
   482|  ->
   483|生成 team digest 或 1:1 brief
   484|  ->
   485|发给经理
   486|  ->
   487|经理反馈
   488|```
   489|
   490|## 9.4 每日复盘
   491|
   492|```text
   493|每天夜里扫描已结束 session
   494|  ->
   495|打分
   496|  ->
   497|聚类失败模式
   498|  ->
   499|生成优化建议日报
   500|```
   501|
---

## 10. Pilot 阶段监控指标 (Leading Indicators)

### 10.1 核心健康指标

| 指标 | 定义 | 健康阈值 | 告警阈值 |
|------|------|----------|----------|
| 周报完成率 | confirmed sessions / initiated sessions | ≥ 70% | < 50% |
| 平均对话轮数 | messages per session | 3-8 轮 | > 12 轮 |
| 经理采纳率 | briefs viewed / briefs sent | ≥ 60% | < 40% |
| 员工负面反馈率 | negative feedback / total feedback | < 10% | > 20% |

### 10.2 技术健康指标

| 指标 | 定义 | 健康阈值 | 告警阈值 |
|------|------|----------|----------|
| LLM 提取成功率 | successful extractions / total attempts | ≥ 85% | < 70% |
| Feishu 消息送达率 | delivered / sent | ≥ 99% | < 95% |
| Session 超时率 | timed out sessions / total sessions | < 5% | > 15% |
| API 响应延迟 P95 | 95th percentile response time | < 3s | > 10s |

### 10.3 每日监控仪表盘

```python
# src/monitoring/metrics.py

from dataclasses import dataclass
from datetime import date

@dataclass
class DailyMetrics:
    """Daily pilot metrics snapshot."""
    report_date: date
    
    # Core metrics
    sessions_initiated: int
    sessions_completed: int
    sessions_expired: int
    sessions_failed: int
    
    avg_messages_per_session: float
    avg_session_duration_minutes: float
    
    briefs_sent: int
    briefs_viewed: int
    
    # Technical metrics
    llm_calls_total: int
    llm_calls_failed: int
    llm_avg_latency_ms: float
    
    feishu_messages_sent: int
    feishu_messages_failed: int
    
    @property
    def completion_rate(self) -> float:
        if self.sessions_initiated == 0:
            return 0.0
        return self.sessions_completed / self.sessions_initiated
    
    @property
    def manager_adoption_rate(self) -> float:
        if self.briefs_sent == 0:
            return 0.0
        return self.briefs_viewed / self.briefs_sent
    
    @property
    def llm_success_rate(self) -> float:
        if self.llm_calls_total == 0:
            return 1.0
        return 1 - (self.llm_calls_failed / self.llm_calls_total)


async def collect_daily_metrics(report_date: date) -> DailyMetrics:
    """Collect metrics for daily monitoring."""
    return DailyMetrics(
        report_date=report_date,
        sessions_initiated=await count_sessions("initiated", report_date),
        sessions_completed=await count_sessions("confirmed", report_date),
        sessions_expired=await count_sessions("expired", report_date),
        sessions_failed=await count_sessions("failed", report_date),
        avg_messages_per_session=await calc_avg_messages(report_date),
        avg_session_duration_minutes=await calc_avg_duration(report_date),
        briefs_sent=await count_briefs("sent", report_date),
        briefs_viewed=await count_briefs("viewed", report_date),
        llm_calls_total=await count_llm_calls(report_date),
        llm_calls_failed=await count_llm_failures(report_date),
        llm_avg_latency_ms=await calc_llm_latency(report_date),
        feishu_messages_sent=await count_feishu_sent(report_date),
        feishu_messages_failed=await count_feishu_failed(report_date),
    )
```

### 10.4 告警规则

```python
# src/monitoring/alerts.py

ALERT_RULES = [
    {
        "name": "low_completion_rate",
        "metric": "completion_rate",
        "condition": "< 0.5",
        "severity": "warning",
        "message": "周报完成率低于 50%，检查用户引导流程"
    },
    {
        "name": "high_session_timeout",
        "metric": "sessions_expired / sessions_initiated",
        "condition": "> 0.15",
        "severity": "warning",
        "message": "Session 超时率超过 15%，检查超时配置"
    },
    {
        "name": "llm_failure_spike",
        "metric": "llm_calls_failed / llm_calls_total",
        "condition": "> 0.3",
        "severity": "critical",
        "message": "LLM 失败率超过 30%，检查 API 状态"
    },
    {
        "name": "high_message_count",
        "metric": "avg_messages_per_session",
        "condition": "> 12",
        "severity": "info",
        "message": "平均对话轮数过多，优化提取逻辑"
    },
]
```
