# Team Pulse MVP 极简架构方案

版本：v1.0  
状态：MVP 极简方案  
日期：2026-04-17  
目标：最快完成真实团队验证

## 1. 先回答你的问题

是，会太复杂。

如果当前目标是：

- 尽快上线
- 在真实团队里验证
- 用最少工程量跑通闭环

那就不应该先按“完整 SaaS 平台”来设计。

对 Team Pulse MVP 来说，正确思路应该是：

**能复用飞书的，复用飞书。能复用 OpenClaw 的，复用 OpenClaw。只有飞书和 OpenClaw 都不能可靠承担的状态，才自己存。**

所以第一版不要做：

- 自建完整组织架构系统
- 自建复杂权限系统
- 自建大而全的团队管理后台
- 自建过多中间服务
- 自建大量通用化数据模型

第一版必须做的，只有 4 个核心机制：

1. `异步引导式对话`
2. `结构化摘要与风险提取`
3. `经理可用输出`
4. `每日自动复盘与优化`

## 2. 极简架构原则

### 2.1 飞书是组织与身份源

直接复用飞书里的：

- 用户身份
- 部门/团队组织结构
- 上下级关系
- 群和私聊入口

MVP 不单独维护一套完整 `organization/team/user/membership` 主数据。

本地只缓存最少映射：

- 飞书用户 ID
- 飞书 open_id
- 用户显示名
- 直属经理 ID
- 所属团队标识

这些数据只作为运行时缓存，不作为“主系统”。

### 2.2 OpenClaw 是渠道与 Agent 编排底座

直接复用 OpenClaw：

- Feishu Channel 接入
- 消息路由
- Agent 执行入口
- 基础上下文管理

Team Pulse 自己只补业务逻辑，不重复造一层聊天平台。

### 2.3 本地数据库只存“不可替代”的业务状态

第一版本地数据库只存 8 类数据：

- check-in 会话
- 对话消息
- 抽取结果
- 风险项
- 已确认摘要
- 1:1 brief
- 反馈
- 每日复盘结果

也就是说：

**不存组织全量主数据，只存业务闭环需要的最小状态。**

### 2.4 不做后台，先做飞书内闭环

MVP 的用户界面只有：

- 飞书私聊机器人
- 飞书消息卡片
- 飞书群消息
- 可选飞书文档输出

先不做 Web 管理后台。

### 2.5 不做自动策略自修改，只做“自动发现问题 + 生成优化建议”

为了降低风险，第一版的“自优化”不要自动改 prompt。

第一版只做：

- 自动复盘
- 自动打分
- 自动找失败模式
- 自动生成优化建议

然后由你人工决定是否更新 prompt。

这已经足够验证“系统能不能越来越好”。

## 3. 极简系统架构

```text
飞书用户
  ->
OpenClaw Feishu Channel
  ->
Team Pulse Agent / App
  ->
PostgreSQL
  ->
每日复盘任务
```

就这么简单。

如果再展开一层：

```text
Feishu
  ->
OpenClaw
  ->
1. Check-in Orchestrator
2. Extract & Risk Analyzer
3. Summary/Brief Generator
4. Daily Review Job
  ->
PostgreSQL
```

第一版不要拆微服务，不要引入复杂事件总线。

推荐部署形态：

- 一个 API 服务
- 一个 Worker/定时任务进程
- 一个 PostgreSQL

可选：

- Redis，用于队列和幂等

## 4. 哪些能力直接复用飞书

飞书直接承担：

- 用户登录和身份识别
- 部门和团队关系
- 私聊触达
- 群通知
- 交互卡片确认
- 文档输出载体
- 可能的日历 1:1 提醒

所以第一版不要做：

- 用户注册登录
- 组织架构后台
- 团队管理页面
- 单独通知系统

## 5. 哪些能力直接复用 OpenClaw

OpenClaw 直接承担：

- 飞书渠道接入
- Agent 路由
- 消息处理入口
- 基础对话上下文拼接

所以 Team Pulse 不再单独实现：

- 渠道适配框架
- 通用消息总线层
- 通用 Agent 平台壳

## 6. 第一版真正要自己做的模块

只做 4 个模块。

## 6.1 Check-in Orchestrator

职责：

- 发起周报/日报对话
- 维护当前对话阶段
- 决定继续追问还是结束
- 触发摘要生成

这是核心中的核心。

## 6.2 Extract & Risk Analyzer

职责：

- 从自然语言中抽取进展、阻塞、计划、支持诉求
- 识别风险信号
- 生成结构化风险项

第一版建议规则 + LLM 混合，不要纯模型黑盒。

## 6.3 Summary / Brief Generator

职责：

- 生成员工摘要
- 生成团队摘要
- 生成 1:1 brief

注意：

第一版团队摘要和 1:1 brief 都可以直接基于已确认摘要生成，不需要再额外做复杂知识图谱。

## 6.4 Daily Review Job

职责：

- 每晚扫描所有已结束对话
- 对每条对话打分
- 汇总失败模式
- 生成一份优化建议日报

第一版最关键的是“看见问题”，不是“自动修自己”。

## 7. 极简数据模型

第一版建议把表数直接砍到最少，够用就行。

## 7.1 feishu_user_cache

用途：缓存飞书用户最少信息，不做主数据中心。

建议字段：

- `id`
- `feishu_user_id`
- `open_id`
- `name`
- `manager_feishu_user_id`
- `team_key`
- `raw_profile_json`
- `updated_at`

说明：

- `team_key` 可以先直接用飞书部门 ID 或业务自定义团队标识
- 不单独拆 `organization/team/membership`

## 7.2 checkin_session

用途：记录一次员工 check-in 主流程。

建议字段：

- `id`
- `feishu_user_id`
- `manager_feishu_user_id`
- `team_key`
- `session_type` (`daily_pulse` / `weekly_checkin`)
- `period_key`
- `status` (`initiated` / `collecting` / `risk_followup` / `summary_confirmation` / `confirmed` / `expired`)
- `started_at`
- `last_message_at`
- `confirmed_at`
- `summary_text`
- `summary_json`
- `prompt_version`

说明：

- 先把 summary 直接挂在 session 上，减少单独 summary 主表复杂度
- 后期再拆表

## 7.3 message_log

用途：记录对话消息。

建议字段：

- `id`
- `session_id`
- `sender_type`
- `channel_message_id`
- `message_text`
- `message_json`
- `sequence_no`
- `created_at`

## 7.4 extraction_snapshot

用途：记录某次抽取结果。

建议字段：

- `id`
- `session_id`
- `source_message_id`
- `structured_json`
- `missing_fields_json`
- `confidence_json`
- `created_at`

## 7.5 risk_item

用途：记录识别出的风险。

建议字段：

- `id`
- `session_id`
- `feishu_user_id`
- `manager_feishu_user_id`
- `team_key`
- `title`
- `description`
- `severity`
- `needs_manager_attention`
- `status`
- `evidence_json`
- `created_at`
- `updated_at`

## 7.6 manager_brief

用途：记录团队摘要或 1:1 brief。

建议字段：

- `id`
- `brief_type` (`team_digest` / `one_on_one`)
- `manager_feishu_user_id`
- `target_feishu_user_id` null
- `team_key`
- `period_key`
- `brief_text`
- `brief_json`
- `source_session_ids_json`
- `created_at`

说明：

- 团队摘要和 1:1 brief 先共用一张表
- MVP 不必拆成两张

## 7.7 feedback_log

用途：记录员工修正和经理反馈。

建议字段：

- `id`
- `actor_feishu_user_id`
- `target_type`
- `target_id`
- `feedback_type`
- `comment_text`
- `detail_json`
- `created_at`

## 7.8 daily_review_report

用途：记录每日复盘结果。

建议字段：

- `id`
- `review_date`
- `total_sessions`
- `completed_sessions`
- `high_friction_sessions`
- `report_text`
- `report_json`
- `created_at`

## 7.9 optimization_suggestion

用途：每日复盘提出的优化建议。

建议字段：

- `id`
- `daily_review_report_id`
- `suggestion_type`
- `target_scope`
- `title`
- `suggestion_json`
- `status` (`proposed` / `accepted` / `rejected` / `applied`)
- `created_at`

## 8. 为什么这些表是必须的

下面这些不是“为了架构好看”，而是飞书和 OpenClaw 无法替你稳定承担。

### 8.1 checkin_session 必须有

因为你必须知道：

- 这个人本周是否已经被发起过
- 现在对话进行到哪一步
- 是否已经确认摘要
- 是否超时

这些是业务状态，不是飞书会替你维护的。

### 8.2 message_log 必须有

因为你要：

- 复盘对话质量
- 回溯摘要依据
- 找失败模式
- 做后续 prompt 优化

没有消息日志，就没有自优化闭环。

### 8.3 risk_item 必须有

因为“识别出过什么风险、后来是否被验证、是否误报”是产品价值本身。

这不能只存在临时上下文里。

### 8.4 daily_review_report 和 optimization_suggestion 必须有

因为你特别强调“每天自动复盘并提炼改进点”。

如果不落库，这个机制就只是一次性文本，不构成系统能力。

## 9. 核心流程极简版

## 9.1 周报采集

```text
定时任务发起
  ->
OpenClaw 通过飞书私聊用户
  ->
用户回答
  ->
Orchestrator 调用抽取
  ->
若缺关键信息则追问
  ->
生成摘要
  ->
用户确认/修正
  ->
session 完成
```

## 9.2 风险追问

```text
用户提到卡点/延期/依赖
  ->
Risk Analyzer 判断要不要追问
  ->
最多再追 1-2 个问题
  ->
生成 risk_item
```

## 9.3 经理输出

```text
读取已确认 session
  ->
生成 team digest 或 1:1 brief
  ->
发给经理
  ->
经理反馈
```

## 9.4 每日复盘

```text
每天夜里扫描已结束 session
  ->
打分
  ->
聚类失败模式
  ->
生成优化建议日报
```

## 10. MVP 实施顺序

如果按最快验证来做，我建议只分 4 步。

### 第 1 步：跑通周报对话主链路

只做：

- 飞书消息接入
- check-in session
- message log
- 抽取
- 动态追问
- 摘要确认

先不要：

- 团队摘要
- 1:1 brief
- 自动优化

目标：

先证明员工愿意用，并且摘要有用。

### 第 2 步：补风险项

只增加：

- risk_item
- 风险追问规则
- 风险摘要输出

目标：

先证明能更早暴露风险。

### 第 3 步：补经理输出

只增加：

- manager_brief
- 团队 digest
- 1:1 brief
- 经理反馈

目标：

先证明经理真的节省时间。

### 第 4 步：补每日复盘

只增加：

- daily_review_report
- optimization_suggestion
- 基础评分规则

目标：

先证明系统能发现自己的问题，并指导下一轮 prompt 优化。

## 11. 现在最该砍掉的内容

如果要极简，我建议把上一版方案里的这些先全部砍掉：

- `organization`
- `team`
- `team_membership`
- `manager_relation`
- `prompt_template`
- `prompt_version` 独立复杂发布体系
- `experiment_assignment`
- 复杂内部 API 分层
- 微服务化思路
- 自动应用优化建议

这些都不是第一批真实验证的前置条件。

## 12. 但有 3 件事不能砍

### 12.1 会话状态

没有状态机，就跑不稳多轮引导。

### 12.2 原始消息与摘要确认

没有原始消息和确认机制，就无法保证摘要可信。

### 12.3 每日复盘落库

没有复盘数据，你就没法持续优化，也没法证明系统在变好。

## 13. 我对第一版的明确建议

把第一版产品重新定义成：

**“一个飞书里的异步周报与风险追踪助手。”**

先不要把 `1:1`、`组织管理`、`平台化配置` 放得太重。

第一版的验收只看 3 个问题：

1. 员工是否愿意完成这种引导式汇报？
2. 经理是否觉得总结和风险提示有用？
3. 每日复盘是否能稳定找出可优化的问题？

如果这 3 个问题成立，再扩 1:1 brief 和更复杂的架构。

## 14. 下一步最合理的动作

如果你认可这个“极简方案”，下一步不应该继续写更复杂的架构文档，而应该直接做二选一：

1. 输出一版 `极简 SQL 表结构草案`
2. 输出一版 `FastAPI/OpenClaw 项目目录与模块骨架`

如果你要最快开工，我建议先做 `1`。
