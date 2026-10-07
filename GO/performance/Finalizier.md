Finalizer 是 Go 提供的一种机制，允许你在一个对象被垃圾回收器（GC）回收之前，执行一段自定义的清理代码。
Finalizer (runtime.SetFinalizer) 是一个“最后的、不可靠的兜底”机制，绝不能替代显式的 Close() 或 defer。

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

type MyFile struct {
    name string
}

// 清理函数：必须接受一个与 obj 类型完全相同的参数
func cleanupMyFile(f *MyFile) {
    fmt.Printf("Finalizer: Cleaning up file %s\n", f.name)
    // 这里可以执行一些清理工作，比如记录日志、释放C内存等
}

func main() {
    f := &MyFile{name: "test.txt"}    
    // 设置终结器
    runtime.SetFinalizer(f, cleanupMyFile)
    
    // 让 f 变成不可达 (unreachable)
    f = nil
    
    // 强制触发 GC，希望看到 Finalizer 执行
    runtime.GC()
    time.Sleep(time.Second) // 给 Finalizer 一点时间执行
    
    fmt.Println("Main function ended.")
}
```
当f这个go对象被清理的时候，先回执行cleanupMyFile函数

## Finalizer 的致命缺陷
Finalizer 只会在 GC 扫描到该对象并决定回收它时才可能执行。这个时间可能是几毫秒后，也可能是几分钟后，甚至永远不会（如果程序在 GC 发生前就退出了）

- 循环引用: 如果对象 A 持有对对象 B 的引用，而对象 B 的 Finalizer 又持有对 A 的引用，那么 GC 会认为它们是“可达的”，从而永远不回收它们，Finalizer 也永远不会运行。
- Finalizer 阻塞: 如果 Finalizer 函数内部执行了阻塞操作（如网络 I/O），它会阻塞整个 Finalizer 后台线程，影响其他对象的清理
- 带有 Finalizer 的对象会给 GC 带来额外的负担。GC 需要特殊处理这些对象，将它们放入一个特殊的队列，等待 Finalizer 线程处理。这会增加 GC 的暂停时间。
## 替代：`runtime.AddCleanup`

| 特性 | `runtime.SetFinalizer` | `runtime.AddCleanup` |
| :--- | :--- | :--- |
| 对象复活 (Object Resurrection) | ❌ 会复活对象，导致至少需要两次 GC 周期才能回收内存。 | ✅ 不会复活对象，对象在本次 GC 中即可被回收。 |
| 循环引用 (Cyclic References) | ❌ 会导致内存泄漏。如果带 Finalizer 的对象在循环中，整个循环都无法被回收。 | ✅ 安全处理循环。即使对象在循环中，只要整体不可达，就能被正确回收和清理。 |
| 清理函数数量 | ❌ 只能注册一个。后注册的会覆盖前一个。 | ✅ 可以注册多个。所有清理函数都会被执行。 |
| 参数灵活性 | ❌ 清理函数签名必须严格匹配对象类型 (`func(*T)`)。 | ✅ 清理函数可以接受任意参数，更灵活。 |
| 主要目的 | 通用的“析构”钩子（但用不好）。 | 专门用于资源清理，语义更清晰。 |

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

type MyFile struct {
	name string
}

// 清理函数：必须接受一个与 obj 类型完全相同的参数
func cleanupMyFile(name string) {
	fmt.Printf("Finalizer: Cleaning up file %s\n", name)
	// 这里可以执行一些清理工作，比如记录日志、释放C内存等
}
func main() {
	f := &MyFile{name: "test.txt"}
	runtime.AddCleanup(f, cleanupMyFile, f.name) // 注意：没有括号
	f = nil
	runtime.GC()
	time.Sleep(time.Second)
	fmt.Println("Main function ended.")
}

```