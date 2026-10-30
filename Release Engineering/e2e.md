大型项目里 e2e 放哪：同 module 是主流，单独拆是例外

- 什么时候才值得单独开 module（例外情况）：
e2e 依赖和生产依赖版本冲突（e2e 需要锁死某个 k8s/test-framework 版本，不想污染生产 go.mod）；
想让 e2e 从生产 module 的 go test ./... 里彻底隔离，避免 CI 每次跑单测都编译重型 e2e 依赖；
e2e 要被别的仓库复用、或独立发版。
不满足这三条，单开 module 就是纯维护成本：多一份 go.mod、多一个 CI 矩阵条目、依赖升级两处同步。这个仓库目前都不满足（e2e 的 Go 代码只有一个 40 行的 mkbucket）。

- 具体放哪个 module：判断依据是“组合根”，而非“逻辑起始的包”。
单组件测试 → 该组件自己的 module（本仓库 agent/e2e 测 agentd，在 agent module）。

整系统测试 → 拥有部署清单和系统组装的那个 module。本仓库四件套的 kustomize 在 
controller/deploy，e2e 部署的就是它，所以归 controller；Kubernetes 的主 module 本身就是“整个系统”，所以 test/e2e 在主 module。这跟“WarmupTask 在哪个包被 reconcile”只是恰好重合，不是判据——如果明天把部署清单挪到顶层 deploy/，e2e 跟着部署清单走，而不是跟着 reconcile 代码走。


如果组合根不属于任何 module（比如顶层 deploy/ + 纯 shell 的 e2e），那 e2e 可以放顶层 test/e2e 并单独建一个小 module，或者干脆只留 shell 脚本、不涉及 module 归属。这个仓库的 e2e 恰好 99% 是 shell，唯一受 module 约束的就是 mkbucket 这个 helper。
一句话：e2e 放在“它要部署的那套东西的归属模块”里；只有当依赖冲突或构建隔离成为真问题时，才升级成独立 module。