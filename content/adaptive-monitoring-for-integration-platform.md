---
title: 电商数据集成平台的自适应监控：从技术指标到业务指标
date: 2025-12-20
tags:
  - monitoring
  - clickhouse
  - e-commerce
  - observability
---

## 背景

我负责的业务是电商数据集成平台，对接了数十个电商平台（Shopify、Amazon、WooCommerce）和 SaaS 工具（ERP、物流系统等），将订单、库存、商品等数据在各系统之间同步。

技术指标：
这里主要关注几方面
- 延迟：延迟过高会影响业务数据的时效性，比如库存更新，严重的会导致超卖等问题
- 错误：可能网络层面就有问题，或者 Cloudflare AWS 等云基础设置故障
- 错误码：可能返回了错误码，但是没返回错误内容。

每个上游平台的稳定性差异极大
大型平台的稳定性稍微好一些，比如 Amazon, Shopify，WooCommerce 这类自建站参差不齐
- 有的平台 API 99.99% 可用，有的隔三差五抽风
- 有的延迟稳定在 100ms，有的在高峰期飙到 10s。
当你只对接 3-5 个平台时，为每个平台手动设阈值还能接受。但当平台数量膨胀到几十个，传统的静态阈值监控就变成了一场噩梦。

业务指标：
业务指标相对于技术指标来说，会更复杂
- 周期性（周级别属性比较明显，黑五等大节假日暂不讨论，大促的流量会更复杂）
  - 一天内的流量是不均匀的，夜间少，白天多
  - 一周内的流量是不均匀的，周末少，工作日多


# 从静态阈值到动态基线

## 静态阈值
最基本的做法是为每个平台设置告警规则，

```yaml
# 简单粗暴的告警规则
- alert: HighErrorRate
  expr: rate(api_requests_total{status=~"5.."}[5m]) / rate(api_requests_total[5m]) > 0.05
  for: 5m
  labels:
    severity: critical
```

这在只有少数几个平台的时候能工作，但问题很快就暴露了：

1. **阈值难以统一**：平台 A 的正常错误率是 0.1%，平台 B 由于 API 设计的原因，正常错误率在 2% 左右（比如频繁返回 429 限流）。一个统一的 5% 阈值对 A 太宽松，对 B 又太敏感。
2. **时间模式各异**：有的平台在北美工作时间延迟升高是正常的，有的平台每周四凌晨会有维护窗口。静态阈值无法理解这些周期性波动。
3. **告警疲劳**：当你有 40 个平台 × 3 种指标（错误率、延迟、吞吐量），维护 120+ 条告警规则是不现实的。最终的结果是大家开始忽略告警

对于静态阈值，我的实践是，阈值要足够宽松，满足大部分平台。这个时候就把它作为兜底措施，一旦超过这个宽松的阈值，比如错误率 20%，就触发高优先级告警

## 动态基线：让监控更聪明

核心思路是：不要告诉系统什么是"异常"，而是让系统自己学习什么是"正常"，偏离正常的才是异常。



### 基于统计的方法

这里先讨论统计学的检测方法，当然业界也有使用机器学习模型来实现异常检测的办法，但是后者要求的配套基础设置要更复杂，暂不讨论

最实用的方法是利用 Prometheus 的 `avg_over_time`、`stddev_over_time` 等函数，为每个平台自动计算动态基线：

```promql
# 计算过去 7 天同一时段的平均错误率作为基线
avg_over_time(
  platform_error_rate{platform="shopify"}[7d:1h]
)
```

然后用 Z-Score 的思路来判断当前值是否异常：

```
z = (当前值 - 历史均值) / 历史标准差
```

当 |z| > 3 时（当前值偏离均值超过 3 个标准差），我们有较高的信心认为这是一个真正的异常，而不是正常波动。

用 PromQL 表达：

```promql
# 动态异常检测：当前错误率偏离基线超过 3 个标准差
(
  rate(api_errors_total{platform="shopify"}[5m])
  - avg_over_time(rate(api_errors_total{platform="shopify"}[5m])[7d:1h])
)
/ stddev_over_time(rate(api_errors_total{platform="shopify"}[5m])[7d:1h])
> 3
```

这个方法的好处是**全自动适配**：平台 A 的正常错误率是 0.1% 且波动很小，那它偏离到 0.5% 就会触发；平台 B 的正常错误率在 1%-3% 之间波动，那它要到 5%+ 才会报警。

### 延迟监控的分位数方法

错误率用 Z-Score 效果不错，但延迟监控需要更细腻的处理。延迟分布通常是长尾的，我更倾向于用分位数：

```promql
# P99 延迟偏离历史 P99 基线的程度
histogram_quantile(0.99, rate(api_request_duration_seconds_bucket[5m]))
/ avg_over_time(
    histogram_quantile(0.99, rate(api_request_duration_seconds_bucket[5m]))[7d:1h]
  )
> 3
```

当 P99 延迟是历史基线的 3 倍以上时触发告警。这比硬编码"延迟 > 2s"灵活得多——对于一个正常 P99 就在 5s 的慢平台，2s 的阈值毫无意义。

## 告警降噪：从指标到可执行的通知
真实环境的大规模数据集成，大部分新配置的监控都会伴随着监控噪声，也就是很多小平台或者小卖家可能很容易出问题，如果 RD 都要投人去做，会非常消耗人力，尤其是夜间的值班。
我的实践是，告警和人力匹配原则

对于基础的技术指标，阈值尽可能宽松，一旦有问题就要人力响应并干预
对于业务指标对用户进行分级，根据分级来决定响应优先级，大卖家优先服务，一旦有问题就人力响应，小卖家则等到上班慢慢排期处理



# 告警上下文与高频运维手册

在告警消息中附带足够的上下文，让值班的人能快速判断严重程度：
- 标题最好可以反映优先级
- 打开面板之后，最好附带运维手册的地址

大部分服务在上线的时候都会制定运维手册，但是当服务多的时候，重大事故告警的时候，依靠人工取翻技术手册大海捞针，并不是一种高效的办法

所以在告警的聊天群中，我们会把致命告警对应的高频处理手册置顶，方便在紧急事故中快速响应






---

*参考资料：*
- [Smart Alerting: Dynamic Thresholds - Rendiment](https://rendiment.io/monitoring/prometheus/2024/09/10/smart-alerting-dynamic-thresholds.html)
- [AI Anomaly Detection: Complete Guide for DevOps & SRE 2026](https://openobserve.ai/blog/ai-anomaly-detection-guide/)
- [Prometheus Alertmanager Noise-Reduction - Netdata](https://www.netdata.cloud/academy/prometheus-alert-manager/)
- [Prometheus Alerting Best Practices](https://prometheus.io/docs/practices/alerting/)
- [What if you never had to tweak your alerting thresholds?](https://medium.com/doubleverify-engineering/what-if-you-never-had-to-tweak-your-alerting-thresholds-for-your-metrics-14a08da8d3d4)
