SoftRoCE（Software RoCE）是 RoCE（RDMA over Converged Ethernet）的软件实现。它不需要专用 RDMA 网卡，是因为它用软件在普通以太网卡上“模拟”出了 RDMA 功能

具体来说，Linux 内核中的 rdma_rxe 驱动会注册为一个 UDP 封装协议，把 RDMA 操作封装成普通以太网包，通过你现有的任何以太网卡（甚至虚拟网卡）收发。所以它不要求网卡具备 RoCE 引擎，代价是延迟比硬件 RoCE 高，且会消耗更多 CPU 资源