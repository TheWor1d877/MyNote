## 标签(Labels)
一个指标名 + 一组唯一的标签键值对 = 一条独立的时间序列（Time Series）
举例说明
你定义了一个 CounterVec：
```go
readBytesTotal = NewCounterVec(..., []string{"dataset", "node"})
```

当你在代码中调用：
```go
readBytesTotal.WithLabelValues("imagenet", "node-01").Inc()
readBytesTotal.WithLabelValues("coco", "node-01").Inc()
readBytesTotal.WithLabelValues("imagenet", "node-02").Inc()
```

Prometheus 内部会创建 3 条完全独立的时间序列：

| 时间序列 (Metric + Labels) | 说明 |
| :--- | :--- |
| `ai_storage_read_bytes_total{dataset="imagenet", node="node-01"}` | 序列 1 |
| `ai_storage_read_bytes_total{dataset="coco", node="node-01"}` | 序列 2 |
| `ai_storage_read_bytes_total{dataset="imagenet", node="node-02"}` | 序列 3 |

✅ 每条时间序列都有自己独立的计数器和历史数据点。

## 实例演示
```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	readBytesTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "ai_storage_read_bytes_total",
			Help: "Total bytes read from storage by dataset",
		},
		[]string{"dataset"},
	)
	// NewCounter()     不带标签的Counter
	// NewCounterVec()  带标签的Counter
	// NewCounterFunc() 函数驱动的Counter

	cacheSizeBytes = prometheus.NewGauge(
		prometheus.GaugeOpts{
			Name: "ai_storage_cache_size_bytes",
			Help: "Current size of local cache in bytes",
		},
	)
)

func main() {
	// 2. 注册指标到默认注册表
	prometheus.MustRegister(readBytesTotal)
	prometheus.MustRegister(cacheSizeBytes)

	go simulateStorageActivity()

	http.Handle("/metrics", promhttp.Handler())
	log.Println("start server at : 8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func simulateStorageActivity() {
	cacheSizeBytes.Set(1024 * 1024 * 100)

	datasets := []string{"iamgenet", "coco", "custom_dataset"}
	for {
		// 模拟读取不同数据集
		for _, ds := range datasets {
			// 每次读 10MB
			readBytesTotal.WithLabelValues(ds).Add(10 * 1024 * 1024)
		}
		time.Sleep(2 * time.Second)
	}
}

```