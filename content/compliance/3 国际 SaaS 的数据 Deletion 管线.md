---
title: 3 国际 SaaS 的数据 Deletion 管线
date: 2026-04-24
order: 3
tags:
  - compliance
  - deletion
  - privacy
  - data-pipeline
---

## 背景

Deletion 讨论的是：当客户注销、合同终止、数据主体提出删除请求时，系统如何把相关数据从所有存储介质中删除。

这件事比看起来难很多。一个成熟的 SaaS 系统里，数据通常散落在：

- 主业务数据库；
- 搜索索引；
- Redis / 缓存；
- 消息队列；
- 对象存储；
- 数仓和 BI；
- Feature Store；
- 日志平台；
- 备份系统；
- 第三方子处理方系统。

如果只删主库，合规上是不完整的；如果立刻硬删所有数据，又可能导致客户误操作后无法恢复。因此工程上通常需要设计一条带状态机的 deletion pipeline，同时兼顾删除、审计和恢复窗口。

## 删除场景

可以先把删除请求分成几类：

| 场景 | 触发方 | 典型范围 | 处理特点 |
|------|--------|----------|----------|
| 客户注销 | 客户管理员 | 整个 tenant / shop | 需要宽限期和恢复能力 |
| 合同终止 | CS / 法务 / 系统 | 整个客户账号 | 可能涉及合同约定保留期 |
| 数据主体请求 | 终端用户 / 商家用户 | 某个自然人相关数据 | 需要身份确认和范围识别 |
| 内部误采集修复 | 安全 / 合规团队 | 某类字段或某批数据 | 需要专项清理和证明 |
| 子处理方删除 | 系统触发 | 第三方系统副本 | 需要 API 或工单确认 |

不同场景的删除范围、时限和例外不同，不能只用一个 `delete_shop(shop_id)` 解决。

## 删除权与例外

GDPR Article 17 提供了删除权的基础，但删除权不是绝对的。比如为了遵守法律义务、处理法律主张、公共利益归档或统计研究等目的，某些数据可能可以继续保留。

英国 ICO 的指引也强调：删除请求可以口头或书面提出，通常需要在一个月内响应；组织需要有流程识别请求、判断是否适用、通知接收方，并使用合适的方法删除信息。

对工程团队来说，这意味着 deletion pipeline 要支持：

- 删除范围识别；
- legal hold 或合同保留例外；
- 删除、匿名化、哈希化、归档等不同动作；
- 对外部系统和下游副本的通知；
- 删除证据和审计记录。

## 为什么需要恢复窗口

客户注销经常存在反悔场景：

- 管理员误操作；
- 客户合同续约；
- 客户成功团队仍在挽回；
- 需要短期恢复历史订单、商品、配置；
- 支付或争议处理仍未结束。

所以注销通常不应该立即不可逆硬删，而是分阶段：

1. **Disable**：停止登录、同步、API 调用和后台任务。
2. **Grace Period**：进入宽限期，数据仍可恢复，但对外不可用。
3. **Deletion Scheduled**：宽限期结束，生成删除计划。
4. **Deleting**：按存储介质逐步删除或脱敏。
5. **Deleted**：删除完成，保留删除证明和最小审计记录。
6. **Recoverable / Non-recoverable 分界**：超过某个时间点后不再支持恢复。

这里需要产品、法务、CS 和工程共同确定宽限期长度，以及哪些数据在宽限期内可恢复。

## Deletion Pipeline 状态机

一个可审计的删除管线，不应该是一次性脚本，而应该是状态机。

示例状态：

```text
REQUESTED
  -> VALIDATING
  -> DISABLED
  -> GRACE_PERIOD
  -> SCHEDULED
  -> DELETING_PRIMARY_DB
  -> DELETING_DERIVED_DATA
  -> DELETING_EXTERNAL_PROCESSORS
  -> VERIFYING
  -> COMPLETED
```

异常状态：

```text
BLOCKED_BY_LEGAL_HOLD
BLOCKED_BY_OPEN_INVOICE
PARTIAL_FAILED
MANUAL_REVIEW_REQUIRED
RESTORE_REQUESTED
RESTORED
```

每个状态都应该记录：

- 操作人或触发系统；
- 触发原因；
- 时间戳；
- 当前处理范围；
- 下一个动作；
- 失败原因；
- 审计证据。

## 存储介质清单

删除管线最重要的输入之一，是完整的数据存储清单。

可以按这个维度梳理：

| 存储介质 | 示例 | 删除方式 | 难点 |
|---------|------|----------|------|
| OLTP DB | PostgreSQL / MySQL | 按 tenant key 删除或脱敏 | 外键、事务、性能、分批 |
| NoSQL | DynamoDB / MongoDB | 按 partition key 扫描删除 | 二级索引和历史版本 |
| Cache | Redis | 删除 key prefix | key 命名不统一 |
| Object Storage | S3 / GCS | 删除对象或生命周期规则 | 文件路径是否包含 tenant id |
| Search Index | Elasticsearch / OpenSearch | delete by query / 重建索引 | 索引副本和异步刷新 |
| Data Warehouse | ClickHouse / BigQuery | 分区删除、重写表、脱敏 | 明细表和聚合表传播 |
| Logs | ELK / Loki / Cloud Logging | retention 到期淘汰或定向删除 | 日志里不应写入 PII |
| Backup | Snapshot / PITR | 等备份周期自然淘汰 | 恢复后不能重新污染线上 |
| Third Party | CRM / Support / Analytics | API 删除或工单 | 依赖外部确认 |

后续可以补充公司内部真实的数据地图，以及每类存储的 owner。

## 主库删除策略

主库删除通常要非常谨慎，因为它会影响线上性能和数据一致性。

常见做法：

- 按 tenant / shop 分批删除，避免大事务；
- 先停止写入和后台任务，防止删除过程中产生新数据；
- 删除前生成 dry-run 统计，确认影响行数；
- 按依赖顺序删除子表，再删除主表；
- 每批删除写入进度表，支持断点续跑；
- 删除完成后做抽样校验。

示例骨架：

```sql
-- dry-run：先统计将要删除的数据量
SELECT 'orders' AS table_name, count(*) AS rows
FROM orders
WHERE shop_id = :shop_id;

-- execution：真实删除必须由管线传入参数，不能拼接用户输入
DELETE FROM orders
WHERE shop_id = :shop_id
  AND deletion_job_id = :job_id;
```

真实实现里应使用参数化查询或 ORM 条件构造，不能把外部输入拼接到 SQL。

## 派生数据与数仓删除

最容易漏的是派生数据：

- 订单明细同步到数仓；
- 订单聚合成商家指标；
- 商品行为变成推荐特征；
- 错误日志进入日志平台；
- 客服系统复制了用户信息；
- BI 报表缓存了历史结果。

可以按数据类型决定动作：

- **可识别明细数据**：删除或字段级脱敏；
- **店铺内模型特征**：保留不可恢复哈希或聚合特征；
- **跨店铺模型特征**：需要特别谨慎，默认不应保留可关联到具体店铺或自然人的信息；
- **审计日志**：保留最小必要字段，例如删除请求 ID、时间、执行结果，不保留原始 PII。

这里和 Retention 主题有交集：Deletion 是事件驱动的定向处理，Retention 是时间驱动的周期处理。

## 恢复管线

恢复管线需要和删除管线配套设计，否则宽限期只是口头承诺。

恢复一般只允许发生在不可逆删除之前：

1. 客户或 CS 发起恢复请求；
2. 校验是否仍在恢复窗口内；
3. 校验是否有 legal / billing / abuse 风险；
4. 恢复 tenant 状态、配置、授权和后台任务；
5. 重新建立索引、缓存和调度任务；
6. 执行数据一致性检查；
7. 通知客户恢复完成。

恢复不是简单把 `deleted_at` 置空。很多系统在注销后会停止 webhook、断开 token、清理队列任务、暂停订阅计费，所以恢复也必须是一条 pipeline。

## 删除证明与审计

Deletion pipeline 最后要留下的是删除证明，而不是原始数据。

可以记录：

- deletion request id；
- tenant / shop 的不可逆内部 ID；
- 请求来源；
- 删除原因；
- 处理范围；
- 开始和完成时间；
- 每个存储介质的处理结果；
- 失败和重试记录；
- 审批人或自动化策略版本；
- 不能删除的数据及原因，例如 legal hold。

这部分记录本身也要遵守最小化原则，不应把被删除的 PII 又写进审计表。

## 后续可以补充的真实细节

- 注销状态机设计；
- 删除任务表 schema；
- 每类存储介质的实际处理方式；
- 恢复窗口的产品策略；
- 备份恢复后的再删除机制；
- 第三方子处理方删除确认流程；
- Deletion pipeline 的监控和告警。

## 参考资料

- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)
- [ICO: Right to erasure](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/individual-rights/right-to-erasure/)
- [European Data Protection Board: Implementation of the right to erasure by controllers](https://www.edpb.europa.eu/our-work-tools/our-documents/other/coordinated-enforcement-action-implementation-right-erasure_en)
- [California Privacy Protection Agency: Laws & Regulations](https://cppa.ca.gov/regulations/)
