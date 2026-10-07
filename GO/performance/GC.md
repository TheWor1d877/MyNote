## 什么是GC
C++ 方式：手动 new / delete。优点是极致控制，缺点是极易出错（内存泄漏、野指针、重复释放）。
Go 方式：你只管 new（或直接声明），Go 运行时（runtime）在后台自动找出“不再被使用的对象”并回收其内存。
GC 的核心价值：解放开发者，让你专注于业务逻辑，而不是内存簿记。

设计哲学：
低延迟 (Low Latency)：Stop-The-World (STW) 时间极短（微秒级），保证服务响应时间稳定。
高并发 (Concurrency)：GC 工作大部分与你的业务代码并发执行，不阻塞主逻辑。

## GC原理：三色表记法
白色： 可回收的垃圾
黑色： 正在使用的对象
灰色： 待检查的对象

1. 初始标记（STW）：将所有对象标记为白色，从根对象出发找到直接引用的对象，标记为灰色
2. 并发标记： GC线程跟业务goroutine同步运行，GC遍历所有灰色对象,并且将所有灰色对象引用的对像标记为灰色，最后将对象标记为黑色
3. 写屏障：当业务代码修改一个黑色对象，引用一个白色对象的时候，将这个白色对象变成灰色
4. 最终标记（STW）： 检查是否有遗漏的灰色对象
5. 并发清除

整个过程中，只有初始和最终标记需要极短的 STW（通常 < 100 微秒），其余工作都是并发的。
## 参数控制
- `GOGC`
```text
下次 GC 触发阈值 = 上次 GC 结束后存活的堆内存大小 * (1 + GOGC/100)
```
默认100

| 场景 | `GOGC` 值 | 效果 | 代价 |
|------|-----------|------|------|
| 内存敏感型<br>(容器内存小，怕 OOM) | 降低 (e.g., 20, 50) | GC 更频繁，内存占用更低、更平稳 | CPU 使用率升高（更多时间花在 GC 上） |
| CPU 敏感型<br>(追求高吞吐，批处理任务) | 升高 (e.g., 200, off) | GC 更少，CPU 更多用于业务逻辑 | 内存占用峰值更高，可能 OOM |
| 禁用 GC | `GOGC=off` 或 `debug.SetGCPercent(-1)` | 完全关闭 GC | 极度危险！ 内存会无限增长直到 OOM |

- `GOMEMLIMIT`
sets a soft memory limit for the runtime.
GOMEMLIMIT 告诉 Go：“无论如何，别让我的堆内存超过这个值！

Go runtime 会持续估算当前的内存使用量。一旦发现快要达到 GOMEMLIMIT，它会<span style="color:rgb(221, 85, 85)">自动、动态地降低 GOGC 的有效值</span>，从而提前、更激进地触发 GC，把内存压下去

最佳实践：
```yaml
# deployment.yaml
spec:
  containers:
  - name: my-app
    image: my-app:latest
    env:
    - name: GOMEMLIMIT
      value: "800Mi"  # 设置环境变量
    resources:
      limits:
        memory: "1Gi"
```

## 测试小程序
```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	go func() {
		ticker := time.NewTicker(2 * time.Second)
		for range ticker.C {
			printMenStatus()
		}
	}()
	for {
		allocate()
		time.Sleep(2 * time.Second)
	}

}

func printMenStatus() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("Alloc = %d KB", bToKb(m.Alloc))
	fmt.Printf("\tTotalAlloc = %d KB", bToKb(m.TotalAlloc))
	fmt.Printf("\tSys = %d KB", bToKb(m.Sys))
	fmt.Printf("\tNumGC = %d\n", m.NumGC)
}

func bToKb(b uint64) uint64 { return b / 1024 }

func allocate() {
	data := make([]byte, 10*1024*1024) // 10MB
	_ = data
}

```

## 注意：
GC只能管理堆内存，不释放其他系统资源，对于：
- fd，文件描述符
- 网络套接字
- 数据库连接
- 互斥锁
- GPU显存，共享内存等等
都需要手动Close

