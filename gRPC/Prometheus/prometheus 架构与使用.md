![[Attachments/Pasted image 20260520114220.png]]


## 定期拉取多个运行在不同端口的go应用
第一步：确保你的 Go 应用暴露了 /metrics 接口
Prometheus 默认会请求 `http://<target>/metrics`。

✅ 第二步：修改 prometheus.yml，添加你的 Go 应用作为 target
编辑你的 Prometheus 配置文件（通常是 prometheus.yml），在 scrape_configs 下新增一个 job：

1. 都在本机运行
```yaml
scrape_configs:
  - job_name: "my-go-app"
    scrape_interval: 15s  # 每15秒拉一次（可选，默认用 global 的）
    static_configs:
      - targets: ["localhost:8080"]
        labels:
          app: "my-go-service"
```

2. prometheus在docker中，但是go在宿主机器
```yaml
scrape_configs:
  - job_name: "my-go-app"
    static_configs:
      - targets: ["host.docker.internal:8080"]  # 关键！不能用 localhost
        labels:
          app: "my-go-service"
```

3. prometheus在宿主机器，但是go应用在docker中
Prometheus（宿主机）必须能通过某个 IP + 端口访问到容器内 Go 应用的 /metrics 接口。关键在于网络可达性
启动 Go 应用容器时，将内部端口映射到宿主机
```bash
# 假设你的 Go 应用在容器内监听 :8080
docker run -d \
  --name my-go-app \
  -p 8080:8080 \          # 👈 关键！把容器 8080 映射到宿主机 8080
  your-go-app-image
```

