---
title: 2 国际 SaaS 的数据 Retention 策略
date: 2026-04-24
order: 2
tags:
  - compliance
  - retention
  - privacy
  - data-governance
---

## 背景

Retention 讨论的是：数据应该保留多久，到期后如何处理。

对国际 SaaS 来说，Retention 不是简单地给数据库表加一个 TTL。真正复杂的地方在于：不同数据类别、不同业务场景、不同合规要求、不同客户合同，可能对应完全不同的保留策略。

GDPR Article 5 里有一个重要原则：个人数据保存时间不应超过处理目的所必要的时间。这个原则落到工程上，就是要回答几个问题：

- 哪些字段是 PII？
- 哪些数据是业务运行必须保留的？
- 哪些数据可以聚合、匿名化或哈希化后继续使用？
- 哪些数据必须按客户或地区配置不同 retention？
- 到期处理是物理删除、逻辑删除、脱敏，还是归档？

## 从数据分类开始

Retention 策略的第一步不是写 TTL，而是做数据分类。没有分类，就不知道哪些数据应该被严格控制，哪些数据可以长期保留。

可以先把数据分成几类：

| Category | 示例 | 典型处理方式 |
|---------|------|-------------|
| 账号与身份数据 | email、name、phone、address | 明确 PII 标注，受删除和导出请求约束 |
| 业务交易数据 | order、product、inventory、shipment | 按合同和业务目的保留，到期删除或脱敏 |
| 认证与安全数据 | login log、audit log、IP、device id | 为安全和审计保留一段时间，到期清理 |
| 系统运行日志 | request log、job log、error log | 短周期保留，必要字段脱敏 |
| 分析与模型特征 | 聚合指标、推荐特征、定价特征 | 优先匿名化、哈希化、聚合化 |
| 备份数据 | DB backup、object snapshot | 按备份恢复窗口保留，随周期自然淘汰 |

后续可以补充一张公司内部的数据分类标准，把字段级别的 PII 标注、数据 owner、存储位置和 retention policy 关联起来。

## PII 标注与数据目录

PII 标注最好不要只存在文档里，而应该进入数据目录和 schema 管理流程。

可以考虑为每个字段维护这些元数据：

- `data_category`：identity、transaction、security_log、analytics_feature 等；
- `pii_level`：none、low、high、sensitive；
- `retention_policy`：保留策略 ID；
- `delete_strategy`：hard_delete、soft_delete、anonymize、hash、aggregate；
- `owner_team`：负责解释和处理该字段的团队；
- `legal_basis`：合同履约、合法利益、同意、安全审计等。

这样做的好处是：当客户发起删除请求，或者某个 retention policy 发生变化时，系统可以知道哪些表、哪些字段、哪些下游数据需要处理。

## DB TTL 策略

数据库层面的 TTL 适合处理生命周期明确、查询模式简单的数据，例如：

- job execution log；
- webhook delivery log；
- API request log；
- 临时导入文件；
- 短期缓存表；
- 低价值明细审计事件。

但 TTL 也有边界：

- 不适合直接处理复杂业务对象，因为业务对象通常有外键关系和下游副本；
- 不适合处理需要先脱敏再保留的场景；
- 不适合替代客户注销删除，因为注销通常要求跨库、跨存储介质处理；
- 不适合处理有 legal hold 的数据。

可以补充的工程实践：

```sql
-- 示例：ClickHouse 日志表保留 90 天
CREATE TABLE api_request_log
(
    event_time DateTime,
    shop_id UInt64,
    endpoint LowCardinality(String),
    status_code UInt16,
    latency_ms UInt32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(event_time)
ORDER BY (shop_id, event_time)
TTL event_time + INTERVAL 90 DAY DELETE;
```

真实生产环境里，TTL 还需要配合监控：过期分区是否被清理、磁盘是否下降、是否存在长尾表没有配置策略。

## 数仓与模型特征的 Retention

数仓是 retention 最容易被低估的地方。很多业务系统删除了数据，但数仓、BI、特征表、离线文件、模型训练集还保留着旧副本。

对电商 SaaS 来说，数仓可能需要保留部分特征用于：

- 店铺级推荐模型；
- 商品定价模型；
- 库存预测；
- 异常检测；
- 成本和容量规划。

合规上需要注意的是：不能因为模型训练方便，就无限保留可识别个人或跨店铺可关联的数据。可以考虑几种处理方式：

1. **店铺内可用，跨店铺隔离**：推荐模型、定价模型只使用本店铺历史特征，不把 A 店铺的可识别数据用于 B 店铺。
2. **哈希化处理**：对不再需要恢复的标识符做单向哈希，不能从特征反推出原始用户。
3. **聚合化处理**：保留商品类目、时间窗口、销量区间等聚合特征，而不是保留订单明细。
4. **字段级脱敏**：删除 email、phone、address 等直接识别字段，只保留模型必要的非识别特征。
5. **特征有效期**：模型特征也要有 retention，不是进了 feature store 就永久保存。

这里后续可以补充一个例子：订单明细表到特征表的转换过程，哪些字段删除，哪些字段哈希，哪些字段聚合。

## 不同业务类别的策略差异

不同 category 应该有不同 retention policy，而不是全系统一个数字。

示例骨架：

| 数据类别 | 默认保留 | 到期动作 | 例外 |
|---------|---------|---------|------|
| API 请求日志 | 30-90 天 | 删除或聚合 | 安全事件可延长 |
| 审计日志 | 1-7 年 | 归档或删除 | 取决于客户合同和审计要求 |
| 订单同步数据 | 合同期间 + 宽限期 | 删除或脱敏 | 税务、争议处理可能例外 |
| 临时导入文件 | 7-30 天 | 删除 | 导入失败排查期内保留 |
| 备份快照 | 7-35 天 | 周期淘汰 | 灾备策略决定 |
| 模型特征 | 90-365 天 | 哈希、聚合或删除 | 模型效果和合规共同决定 |

这些数字不是建议值，只是骨架。真正上线前需要结合法务、合同、业务恢复需求和基础设施成本来定。

## Retention 的执行链路

一个可落地的 retention 系统，通常需要几层能力：

1. 数据目录记录每张表、每个字段的分类和策略。
2. 定时任务扫描到期数据，生成处理计划。
3. 执行器按策略删除、脱敏、哈希或归档。
4. 处理结果写入审计日志。
5. 监控看板展示待处理、处理中、失败、已完成的数据量。
6. 异常失败进入工单系统，由 owner team 处理。

关键点是可证明：不是说"我们会删除"，而是能证明某类数据在某个时间被什么任务按什么策略处理了。

## 后续可以补充的真实细节

- 公司内部 PII 分类标准；
- 数据目录字段设计；
- 不同数据库的 TTL 实现方式；
- 数仓分层中的 retention 处理；
- 模型特征哈希和聚合策略；
- Retention 任务失败后的补偿机制；
- Legal hold 如何暂停 retention。

## 参考资料

- [GDPR Article 5: Principles relating to processing of personal data](https://gdpr-info.eu/art-5-gdpr/)
- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)
- [ICO: Right to erasure](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/individual-rights/right-to-erasure/)
- [California Privacy Protection Agency: CCPA FAQ](https://cppa.ca.gov/faq)
