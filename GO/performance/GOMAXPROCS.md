GOMAXPROCS 用来限制 Go 程序同时执行 Go 代码的最大 CPU 并行度，也就是同时运行 Go 代码的 P 的数量。

例如，设置 GOMAXPROCS=4，意味着最多有 4 个 P 可以同时执行 Go 代码。

在 Kubernetes 中，需要特别注意 CPU Limit。容器可能运行在一台拥有 32 核 CPU 的机器上，但自身只被限制使用 2 核。

如果 Go Runtime 只按照宿主机 CPU 核数设置 GOMAXPROCS，就可能创建过多并行执行任务，导致容器频繁触发 CPU Throttling，影响延迟。
CPU Throttling 是指容器使用 CPU 达到配额限制后，Linux CFS 调度器暂时限制该容器继续使用 CPU 的行为。

较新的 Go 版本可以根据容器 CPU 配额自动调整 GOMAXPROCS，使并行度更贴近容器实际可用的 CPU 资源。
