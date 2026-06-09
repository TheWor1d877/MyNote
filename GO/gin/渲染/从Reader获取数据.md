DataFromReader 允许你将任何 io.Reader 的数据直接流式传输到 HTTP 响应，而无需先将整个内容缓冲到内存中。这对于构建代理端点或高效地从远程源提供大文件至关重要。

常见用例：

- 代理远程资源 — 从外部服务（如云存储 API 或 CDN）获取文件并转发给客户端。数据通过你的服务器流过，而不会完全加载到内存中。
- 提供生成的内容 — 在生产动态生成的数据（如 CSV 导出或报告文件）时进行流式传输。
- 大文件下载 — 提供太大而无法保存在内存中的文件，从磁盘或远程源分块读取。

方法签名为` c.DataFromReader(code, contentLength, contentType, reader, extraHeaders)`。你需要提供 HTTP 状态码、内容长度（让客户端知道总大小）、MIME 类型、要流式传输的 `io.Reader`，以及可选的额外响应头映射（如用于文件下载的` Content-Disposition`）。

#### 代理远程资源（云存储，CDN中转站）
```go
router.GET("/proxy/:bucket/:key", func(c *gin.Context) {
    bucket := c.Param("bucket")
    key := c.Param("key")

    // 1. 调用 S3 API 获取对象（返回 io.ReadCloser）
    resp, err := s3Client.GetObject(context.TODO(), &s3.GetObjectInput{
        Bucket: &bucket,
        Key:    &key,
    })
    if err != nil {
        c.AbortWithStatus(500)
        return
    }
    defer resp.Body.Close()

    // 2. 直接流式转发
    c.DataFromReader(
        200,
        *resp.ContentLength,          // S3 返回的文件大小
        *resp.ContentType,            // S3 返回的 MIME 类型
        resp.Body,                    // S3 的响应体（io.ReadCloser 实现了 io.Reader） 流式转发！
        map[string]string{
            "Content-Disposition": fmt.Sprintf(`attachment; filename="%s"`, key),
        },
    )
})
```

