## 无缓冲channel
```go
// 创建无缓冲通道
ch := make(chan int)  // 或 make(chan int, 0)
```

特点：发送和接收必须同时准备好（握手同步）
发送方阻塞直到接收方准备好
接收方阻塞直到发送方准备好

## 有缓冲channel
```go
// 创建有缓冲通道，容量为 N
ch := make(chan int, 3)  // 可以缓冲 3 个元素
```
特点：缓冲区未满时发送不阻塞，缓冲区不空时接收不阻塞
用作 任务队列、限流器、信号池 的理想工具。
#### 工作池workpool
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	jobs := make(chan int, 100)
	results := make(chan int, 100)

	var wg sync.WaitGroup
	for w := 0; w < 3; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for job := range jobs {
				results <- job * job
			}
		}()
	}

	for i := 1; i <= 5; i++ {
		jobs <- i
	}
	close(jobs)

	go func() {
		wg.Wait()
		close(results)
	}()

	for result := range results {
		fmt.Println(result)
	}
}

```