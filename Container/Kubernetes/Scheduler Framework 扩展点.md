k8s的调度器本质上就是一个“匹配引擎”，将Pod的需求跟node的能力做匹配

他是由一系列拓展点串联起来的流水线，每个拓展点都是一个hook，可以加载自己的逻辑进入
## 扩展点
经历的完整过程：
```text
Pod 创建
  ↓
PreEnqueue          ← 能否进入活动队列？
  ↓
QueueSort           ← 队列里谁先被调度？
  ↓
PreFilter           ← Pod 级前置检查（跟节点无关）
  ↓
Filter              ← 逐个节点过滤
  ↓
PostFilter          ← 全挂了才触发（抢占入口）
  ↓
PreScore            ← 为打分做公共预处理
  ↓
Score               ← 给可行节点打分
  ↓
NormalizeScore      ← 修正分数分布
  ↓
Reserve             ← 乐观预留资源
  ↓
Permit              ← 批准 / 拒绝 / 等待（调度周期末尾）
  ↓
═══════════════════════ 调度周期结束，进入绑定周期
  ↓
PreBind             ← 绑定前准备
  ↓
Bind                ← 写入 pod.spec.nodeName
  ↓
PostBind            ← 绑定后清理
```
详细解释:

PreEnqueue 检查这个Pod能不能进入活动队列:  外部依赖没就绪时，先不让它排队
QueueSort 队列中谁先调度: 训练任务优先于推理任务
PreFilter Filter的前置检查点,检查这个Pod本身有没有前置条件不满足，需要的资源是否都能提供
Filter 这个节点能不能跑这个 Pod？	GPU 型号、显存、NVLink 拓扑、存储带宽检查
PostFilter 所有能调度的节点都挂了怎么办 欻发抢占,驱逐低优先级别的Pod
PreScore	打分前有没有公共数据要算一次？	提前算好节点拓扑信息，避免每个节点重复算
Score	多个可用节点里，哪个更合适？	按 GPU 拓扑、镜像局部性、资源碎片度打分
NormalizeScore	某个插件分数会不会压倒其他插件？	把分数缩放到合理范围
Reserve	选完节点到绑定前，资源会不会被别人抢走？	预占 GPU 显存，防止并发调度冲突
Permit	现在能不能真正绑定？	Gang Scheduling：等 8 个 worker 都就绪再放行
PreBind	绑定前要不要做准备？	挂载分布式存储卷
Bind	把 Pod 和节点绑在一起	一般用默认实现，写入 pod.spec.nodeName
PostBind	绑定成功后要不要清理？	上报指标、清理临时状态
#### 关键细节
Permit 属于调度周期，不属于绑定周期。 它在调度周期末尾调用，返回 Success 后才进入 PreBind → Bind → PostBind。
Reserve 和 Unreserve 是成对的。 Reserve 成功但后续 Permit/PreBind/Bind 失败时，调度器调用 Unreserve 回滚。Unreserve 必须幂等，不能自己再失败。
Bind 是短路的。 多个 Bind 插件按顺序调用，第一个真正处理了绑定的插件执行完后，后面的全部跳过。
PostFilter 只在 Filter 全挂时触发。 它是抢占的入口，不是每个 Pod 都会走。
NormalizeScore 不是必写的。 只有当你的 Score 插件原始分数范围可能压倒其他插件时才需要它。

## 接口定义
代码在 `k8s.io/kubernetes/pkg/scheduler/framework `包下。 核心接口是 Plugin
```go
type Plugin interface {
    Name() string
}

type FilterPlugin interface {
    Plugin
    Filter(ctx context.Context, state *CycleState, pod *PodInfo, nodeInfo *NodeInfo) *Status
}

type ScorePlugin interface {
    Plugin
    Score(ctx context.Context, state *CycleState, pod *PodInfo, nodeName string) (int64, *Status)
    ScoreExtensions() ScoreExtensions  // 可选，返回 NormalizeScore
}
```
调度器启动时，会读取配置文件里的 plugins，把插件注册到对应扩展点上。
调度运行时，当流程走到 Filter 阶段，调度器就会遍历所有注册在 Filter 上的插件，逐个调用它们的 Filter() 方法；走到 Score 阶段，就调用所有注册在 Score 上的插件的 Score() 方法。

举例说明:
```go
type MyPlugin struct{}

func (p *MyPlugin) Name() string { return "MyPlugin" }

func (p *MyPlugin) Filter(ctx context.Context, state *CycleState, pod *PodInfo, nodeInfo *NodeInfo) *Status {
	// 逻辑省略
	return nil
}

func (p *MyPlugin) Score(ctx context.Context, state *CycleState, pod *PodInfo, nodeName string) (int64, *Status) {
	// 逻辑省略
	return 0, nil
}

func (p *MyPlugin) ScoreExtensions() ScoreExtensions { return p }

func (p *MyPlugin) NormalizeScore(ctx context.Context, state *CycleState, pod *PodInfo, scores NodeScoreList) *Status {
	// 逻辑省略
	return nil
}
```

## Framework 拓展点全表
| 扩展点            | 阶段                  | 触发条件 / 输入         | 返回值                            | 典型用途                                             |
| -------------- | ------------------- | ----------------- | ------------------------------ | ------------------------------------------------ |
| PreEnqueue     | 入队前                 | Pod 准备进入活动队列      | Success / 拒绝                   | 控制 Pod 是否进入调度队列，例如 Scheduling Gates、外部依赖未就绪时先不排队 |
| QueueSort      | 队列排序                | 两个 Pod            | `Less(p1, p2)`                 | 决定谁先被调度；只能启用一个 QueueSort 插件                      |
| PreFilter      | Scheduling Cycle    | Pod + 共享状态        | Success / Unschedulable / 错误   | Pod 级前置检查；失败则整个调度周期终止                            |
| Filter         | Scheduling Cycle    | Pod + 每个 Node     | Pass / Fail                    | 节点级过滤；任一插件 Fail 就淘汰该节点，通常并行执行                    |
| PostFilter     | Scheduling Cycle    | Filter 后无可用节点     | Success / Unschedulable        | 抢占入口；默认实现是 DefaultPreemption                     |
| PreScore       | Scheduling Cycle    | Pod + 可行节点集合      | Success / 错误                   | 为 Score 做公共数据预处理，失败则终止调度周期                       |
| Score          | Scheduling Cycle    | Pod + 每个可行 Node   | 0–100 分                        | 给节点打分，按插件权重合并                                    |
| NormalizeScore | Scheduling Cycle    | 某 Score 插件的全部打分结果 | 归一化后的分数                        | 修正某个 Score 插件的分数分布；每个调度周期调用一次                    |
| Reserve        | Scheduling Cycle    | 已选中的 Pod + Node   | Success / 错误                   | 乐观预留资源；失败或后续阶段出错时触发 Unreserve                    |
| Permit         | Scheduling Cycle 末尾 | Pod + 选中 Node     | Success / Wait / Unschedulable | Gang Scheduling 的核心位置；可阻塞绑定                      |
| PreBind        | Binding Cycle       | Pod + 选中 Node     | Success / 错误                   | 绑定前准备，如 Volume 挂载；失败会触发 Unreserve                |
| Bind           | Binding Cycle       | Pod + 选中 Node     | Success / 跳过                   | 真正写入 `pod.spec.nodeName`；第一个处理成功的插件会跳过后续 Bind 插件 |
| PostBind       | Binding Cycle       | Pod + 已绑定 Node    | 无                              | 绑定成功后的清理、通知、指标上报                                 |
