## workpool
C++ 需手动管理线程池（如 std::thread + 任务队列 + 条件变量）
Go 用 goroutine + channel 几行代码实现
固定数量的 worker goroutine
从共享 job channel 消费任务
结果写入 result channel
#### 使用场景
HTTP 请求批量处理
图片/视频转码
数据库批量写入
#### 代码示例
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Job struct {
	ID int
}

type Result struct {
	JobID int
	Data  string
}

func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
	defer wg.Done()
	for job := range jobs {
		time.Sleep(1 * time.Second)
		results <- Result{JobID: job.ID, Data: "this is job"}
	}
}

func main() {
	const numWorker = 4
	const numJobs = 10

	jobs := make(chan Job, numJobs)
	results := make(chan Result, numJobs)
	var wg sync.WaitGroup

	for w := 1; w <= numWorker; w++ {
		wg.Add(1)
		go worker(w, jobs, results, &wg)
	}

	for j := 1; j < numJobs; j++ {
		jobs <- Job{ID: j}
	}

	time.Sleep(1 * time.Second)
	close(jobs)

	go func() {
		wg.Wait()
		close(results)
	}()

	for res := range results {
		fmt.Println(res)
	}
}
```
#### 何时使用？
每个请求 CPU 密集（如加密/解压）
需要严格限制并发处理数（而非连接数）
所以适用情况较少，最佳实践还是goroutine per connection

## Pipeline
将任务拆分为多个 stage
每个 stage 是一个 goroutine
stage 之间用 channel 连接：gen → sq → sink
#### 使用场景
日志处理（解析 → 过滤 → 写入）
数据 ETL（抽取 → 转换 → 加载）
流式计算
#### 代码示例
```go
package main

import "fmt"

func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
		fmt.Println("Done generating numbers")
	}()
	return out
}

func sq(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n
		}
	}()
	return out
}

func main() {
	nums := gen(1, 2, 3, 4)
	fmt.Println("hello")
	squares := sq(nums)

	for n := range squares {
		fmt.Println(n)
	}
}

```