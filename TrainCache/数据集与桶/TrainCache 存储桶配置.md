一个集群一个桶是 TrainCache 的默认配置。所有数据集的对象放在同一个桶里，靠键前缀区分。`DataSet.spec.bucket` 与 Agent 的 `objstoreBucket` 必须逐字相等，这条约束本期不改。

「每个数据集一个独立桶」的一对一映射本期不做。真要拆，落点在 Controller 层与 Agent 的存储选路，不在 `Store` 接口上。

## 当前实现

| 位置 | 现状 | 代码位置 |
|---|---|---|
| 存储抽象 | `Store` 接口的方法里没有桶参数，桶在实现里 | `core/objectstore/store.go:26`；`core/objectstore/s3.go:30` 的 `NewS3Store(endpoint, accessKey, secretKey, bucket, secure)` |
| Agent | 一个进程一个桶一个 Store，桶来自 ConfigMap `objstoreBucket` | `agent/svc/assemble.go:99` |
| Controller | 每个数据集一个 Store，按 endpoint 与 bucket 建，按 Secret 版本缓存 | `controller/internal/storage/factory.go:64` |
| 键布局 | 数据块 `<dataset>/blocks/<path>#<chunk>`；索引 `dataset/<name>/...` | `core/objectstore/blockkey.go:11`；`core/index/format.go` 的 `CurrentKey` 与 `ManifestKey` |
| 契约 | Agent 的桶必须等于 `DataSet.spec.bucket` | `docs/PARAMETER-CONFIGURATION.md` 的 `TRAINCACHE_OBJSTORE_BUCKET` 一行 |

Controller 侧本来就支持一个数据集指向一个任意桶。Agent 侧不支持。缺口只在 Agent。

## 每个数据集一个桶需要改动的地方

Agent 的存储选路。一个进程内持多个 Store，按 dataset 选。选路的键已经在手上：`core/fetch` 的块键第一段是 dataset，`core/index` 的 `Catalog` 在构造时绑定了 dataset，`agent/rescan` 的块键同样带 dataset 前缀。`Store` 接口签名不用改，改的是「谁持有哪个 Store」。现在 `agent/svc/assemble.go` 建一个 Store 给全进程用，改完是每个 dataset 一个。

凭证。一个桶一套 AK/SK。多桶意味着每个数据集一套凭证，ConfigMap `objstoreBucket` 这一项的形态要从单值变成数据集到桶与凭证的映射。这一项按仓库 `docs/PARAMETER-CONFIGURATION.md` 的「落位判据」属「换集群要改」，留在清单里，不写进代码。

枚举。`agent/svc/wire.go:382` 的 `List(ctx, "")` 是全桶枚举，桶一多就是 N 倍的键列表。单桶下这一个调用覆盖全部数据集。

网关侧路由。合并回源网关按 dataset 分流的前提是同一个后端。多桶之后网关要按 dataset 选后端，这是 9.x 的改动量。

## 单桶的代价

隔离靠约定。桶里放的是别人的数据，删除要靠键前缀精确算。`core/index.Collector.Sweep` 的活块集合只按当前 dataset 计算，删的却是节点本地 CAS 里的块。本地 CAS 是节点级全局，不按数据集分区。多数据集共节点时，为数据集 A 跑一次 Sweep 会删掉只被数据集 B 引用的本地块。这是单桶加全局 CAS 叠出来的缺陷，与桶的数量无关，但多数据集场景把它暴露出来。

列举成本与数据集数成正比。全桶 LIST 的响应随桶里对象总数增长，Sweep 与重扫描都触发它。

凭证只有一套。数据集之间没有凭证级隔离。

## 重新考虑的条件

接入方要求租户隔离，或不同数据集必须落在不同账号的桶里。

单桶里的对象量让全桶枚举不可接受。

本地 CAS 按数据集分区。这是另一条线的需求，与桶无关，但会一起改到删块逻辑。

重开时的第一步是把 Agent 的 Store 选路抽出来。现在它是一个进程一个 Store，选路一旦存在，上面凭证、枚举、网关路由三条各自就能单独做。

## 当前限制

`DataSet.spec.bucket` 字段仍在，值对所有数据集都是同一个桶名。它现在是一个「必须与 Agent 一致」的断言，不是「可以各不相同」的开关。