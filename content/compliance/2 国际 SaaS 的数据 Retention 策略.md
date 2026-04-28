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

Retention 策略的第一步不是写 TTL，而是做数据分类。没有分类，就不知道哪些数据应该被严格控制，哪些数据可以长期保留。这个通常需要和法务来共同讨论来决定

可以先把数据分成几大类：

| Category | 示例                                         | 典型处理方式 |
|---------|--------------------------------------------|-------------|
| 业务交易数据 | order、checkout， product、inventory、shipment | 按合同和业务目的保留，到期删除或脱敏 |
| 认证与安全数据 | login log、audit log、IP、device id           | 为安全和审计保留一段时间，到期清理 |
| 系统运行日志 | request log、job log、error log              | 短周期保留，必要字段脱敏 |
| 分析与模型特征 | 聚合指标、推荐特征、定价特征                             | 优先匿名化、哈希化、聚合化 |
| 备份数据 | DB backup、object snapshot                  | 按备份恢复窗口保留，随周期自然淘汰 |

先定大类的规则，再定小类的规则。保持输入是对齐的，所有业务根据法务的原则来设计 Retention 能力

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

数据库层面的 TTL 适合处理生命周期明确、查询模式简单的数据，通常是清理一行，例如：

- job execution log；
- webhook log；
- API access log；
- 临时导入文件；
- 短期缓存表；
- 低价值明细审计事件。

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
但 TTL 也有边界：

- 不适合直接处理复杂业务对象，因为业务对象通常有复杂外键关系或者只需要处理特定字段（一行的部分列）；
- 不适合处理需要先脱敏再保留的场景；
- 大客户定制化逻辑
- 税务、款项迟滞客户需要单独处理


## 数仓与模型特征的 Retention

数仓是 retention 最容易被低估的地方。很多业务系统删除了数据，但数仓、BI、特征表、离线文件、模型训练集还保留着旧副本。
海外的合规要求和国内不太一样，不能跨店铺来训练模型，这是一个大背景

对电商 SaaS 来说，数仓可能需要保留部分特征用于：

- 店铺级推荐模型；
- 商品定价模型；
- 库存预测；
- 异常检测；
- 其它特定场景

合规上需要注意的是：不能因为模型训练方便，就无限保留可识 PII 数据。可以考虑几种处理方式：

2. **哈希化处理**：对不再需要恢复的标识符做单向哈希，不能从特征反推出原始用户。
3. **聚合化处理**：保留商品类目、时间窗口、销量区间等聚合后的统计特征，而不是保留订单明细。
4. **字段级脱敏**：删除 email、phone、address 等直接识别字段，只保留模型必要的非识别特征。
5. **特征有效期**：模型也需要定期 Rolling Update, 避免有过期的特征影响后续的推理。

这里后续可以补充一个例子：订单明细表到特征表的转换过程，哪些字段删除，哪些字段哈希，哪些字段聚合。

## 不同业务类别的策略差异

不同 category 应该有不同 retention policy，而不是全系统一个数字。

示例骨架：

| 数据类别         | 默认保留          | 到期动作 | 例外             |
|--------------|---------------|---------|----------------|
| API 请求或者普通日志 | 30 天          | 删除或聚合 | 安全事件或可延长       |
| 审计日志         | 1-7 年         | 归档或删除 | 取决于客户合同和审计要求   |
| 订单和相关数据      | Category Wise | 删除或脱敏 |  |
| 临时导入文件       | 7-30 天        | 删除 | 导入失败排查期内保留     |
| 备份快照         | 7-35 天        | 周期淘汰 | 灾备策略决定         |
| 模型特征         | 90-365 天      | 哈希、聚合或删除 | 模型效果和合规共同决定    |

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

## 大客户定制化逻辑
真实的场景中，大客户定制逻辑是系统设计的一个巨大考量因素。

角色层面，需要有一个独立的服务来提供查询配置的能力，如果客户有自定义的阈值，就能检测到。


## 真实落地的技术考虑

TTL or Manual Scan ?

因为 DB 层面只是 TTL 一般是绑定一个静态值，要做 **定制化** (动态 TTL)， 就只能更新这个静态值,对于数据量小的情况比较友好，数据量大的情况下就不适用。
实际上我的系统有更复杂的考虑，所以只能写一个 Worker 去扫表来实现 TTL，这样实现更灵活。


Scan:
如果你的数据库类型可以指定优先级（比如 Spanner 对于 Read 支持 Low Priority），建议以低优先级来扫描。避免影响到实际业务
由于每天扫描的数据在百亿级别，持续数天，需要考虑进度恢复，因为 Worker 可能会因为各种原因重启，所以需要考虑进度恢复。这样机器重启了可以恢复它的进度


### 风险规避
监控日常删除的速率，如果发现异动需要有告警跟进，避免突然清库等突发风险。。



## 持续运营
- 新表上线后要能识别并加入 Retention 逻辑中
- 新的产品需求在定义阶段，需要考虑 Retention 的存在
- 日常故障排查，发现数据缺失了，需要有办法知道是否和 Retention 有关联

业内有某些公司因为做合规而把正常数据误删的情况，会影响线上用户。所以需要特别注意合规服务的新功能上线，每次发布走完整的测试流程，上线 Check。按照最高优先级来考虑


## 参考资料

- [GDPR Article 5: Principles relating to processing of personal data](https://gdpr-info.eu/art-5-gdpr/)
- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)
- [ICO: Right to erasure](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/individual-rights/right-to-erasure/)
- [California Privacy Protection Agency: CCPA FAQ](https://cppa.ca.gov/faq)
