## Prometheus与Grafana功能区别
1️⃣ Prometheus：负责 采集、存储、查询 时间序列数据
采集（Scraping）：定期（比如每 15 秒）去你的 :8080/metrics 拉取指标
存储（Storage）：把历史数据存到本地磁盘（默认保留 15 天）
查询（Query Engine）：提供 PromQL 语言，支持 rate(), sum(), topk() 等复杂计算
告警（Alerting）：根据规则触发告警（需配合 Alertmanager）
✅ 没有 Prometheus，你就没有历史数据，也无法计算速率（rate）！

2️⃣ Grafana：负责 展示、交互、告警面板
连接各种数据源（Prometheus、Loki、MySQL、InfluxDB...）
提供拖拽式 Dashboard 编辑器
支持变量、模板、注释、分享
本身不存储数据，也不做复杂计算（依赖后端数据源）
✅ 没有 Grafana，你只能看 Prometheus 自带的简陋图表（或 curl 文本）

## Prometheus Server 核心功能
1. 主动抓取，从各个服务的`/metrics`中拉取数据
2. 存储：把历史数据存下来（比如存 15 天）。
3. 查询：提供 PromQL 语言让你问：“过去 1 小时，imagenet 的读吞吐是多少？”
4. 告警：如果 cache_hit_ratio < 0.5 持续 5 分钟，就发邮件

## Grafana
Prometheus 存的是“数字”，但人更喜欢看图表。

| 工具 | 输出 |
| :--- | :--- |
| Prometheus | `ai_storage_read_bytes_total{dataset="imagenet"} @1716123456 = 1024000` |
| Grafana | 📈 一条漂亮的曲线图：“Imagenet 读吞吐 (MB/s)” |

