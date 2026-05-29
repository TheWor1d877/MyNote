## 创建一个有性能瓶颈的程序
```go
// main.go
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof" // 👈 关键：自动注册 /debug/pprof 路由
	"time"
)

func heavyWork() {
	sum := 0
	for i := 0; i < 1e9; i++ {
		sum += i
	}
	fmt.Println("sum =", sum)
}

func handler(w http.ResponseWriter, r *http.Request) {
	heavyWork()
	fmt.Fprintf(w, "Done!\n")
}

func main() {
	go func() {
		http.ListenAndServe("localhost:6060", nil) // pprof HTTP 服务
	}()

	http.HandleFunc("/", handler)
	fmt.Println("Server running on :8080")
	http.ListenAndServe(":8080", nil)
}
```

#### 触发负载并且profiling
```bash
# 模拟高负载
while true; do curl http://localhost:8080/; done

# 采集 30 秒 CPU 数据
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

#### 生成火焰图
```bash
go tool pprof -http=:8081 /tmp/pprof.samples.cpu.001.pb.gz
```

## pprof ui使用

| 字段     | 全称                 | 含义               | 关键作用           | 示例值         |
| ------ | ------------------ | ---------------- | -------------- | ----------- |
| flat   | Flat time          | 函数自身执行时间（不含子调用）  | 定位纯计算热点        | `28.4s`     |
| flat%  | Flat percent       | `flat / 总采样时间`   | 判断该函数自身开销占比    | `94.7%`     |
| sum%   | Sum percent        | 累计 flat%（从上到下累加） | 快速判断前 N 个函数覆盖度 | `94.7%`     |
| cum    | Cumulative time    | 函数总耗时（含所有子调用）    | 识别调用链总开销       | `28.5s`     |
| cum%   | Cumulative percent | `cum / 总采样时间`    | 判断该调用链整体影响     | `95.0%`     |


## Q & A
Q1：pprof CPU Profiling 的底层原理是什么？
✅ 基于 SIGPROF 信号的定时采样：
Go runtime 每 10ms（100Hz）发送一次 SIGPROF 信号
信号处理函数捕获当前 Goroutine 的调用栈
统计每个函数在栈中出现的频率 → 得出 CPU 占用比例
📌 注意：这是 统计采样（statistical sampling），不是全量追踪，因此开销极低（<1%）

Q2：为什么有些函数没出现在 pprof 结果中？
可能原因：
执行时间太短，未被采样命中（采样间隔 10ms）
被编译器内联（inlined），函数名消失（可用 -gcflags="-l" 禁用内联测试）
采样时间不足，未覆盖该函数执行窗口


