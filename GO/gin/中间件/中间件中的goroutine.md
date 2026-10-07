在中间件或处理函数中启动新的 Goroutine 时，不应该在其中使用原始上下文，必须使用只读副本。

Gin 使用 sync.Pool 来复用 gin.Context 对象以提高性能。一旦处理函数返回，gin.Context 就会被返回到池中，可能会被分配给一个完全不同的请求。如果此时一个 goroutine 仍然持有对原始上下文的引用，它将读取或写入现在属于另一个请求的字段。这会导致竞态条件、数据损坏或 panic。

```go
func main() {
	router := gin.Default()

	router.GET("/long_async", func(c *gin.Context) {
		// create copy to be used inside the goroutine
		go func() {
			// simulate a long task with time.Sleep(). 5 seconds
			time.Sleep(5 * time.Second)

			// note that you are using the copied context "cCp", IMPORTANT
			log.Println("2Done! in path " + c.Request.URL.Path)
		}()
		// 此时将c重新放回sync.Pool
	})

	router.GET("/long_sync", func(c *gin.Context) {
		// simulate a long task with time.Sleep(). 5 seconds
		time.Sleep(2 * time.Second)

		// since we are NOT using a goroutine, we do not have to copy the context
		log.Println("1Done! in path " + c.Request.URL.Path)
	})

	// Listen and serve on 0.0.0.0:8080
	router.Run(":8080")
}
```
输出：
2026/06/02 16:56:35 1Done! in path /long_sync
2026/06/02 16:56:37 2Done! in path /long_sync

正确代码:
```go
func main() {
	router := gin.Default()

	router.GET("/long_async", func(c *gin.Context) {
		// create copy to be used inside the goroutine
		cCp := c.Copy()
		go func() {
			// simulate a long task with time.Sleep(). 5 seconds
			time.Sleep(5 * time.Second)

			// note that you are using the copied context "cCp", IMPORTANT
			log.Println("2Done! in path " + cCp.Request.URL.Path)
		}()
		// 此时将c重新放回sync.Pool
	})

	router.GET("/long_sync", func(c *gin.Context) {
		// simulate a long task with time.Sleep(). 5 seconds
		time.Sleep(2 * time.Second)

		// since we are NOT using a goroutine, we do not have to copy the context
		log.Println("1Done! in path " + c.Request.URL.Path)
	})

	// Listen and serve on 0.0.0.0:8080
	router.Run(":8080")
}

```