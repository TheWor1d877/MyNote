CRD实例都以CRD对象的形式存在于k8s的etcd里面，但是控制器对他们的处理方式可能不同

例：
```text
etcd (控制平面)
├── Dataset/jfsdemo          ← 名词，声明状态
├── JuiceFSRuntime/jfsdemo   ← 名词，声明运行时
└── DataLoad/jfs-load        ← 动词，触发一次动作
```

根据行为模式的不同可以分成“声明式”与 “命令式”

## 声明式API（名词）
核心语义是“是什么“
通过提交一份”期望状态“的声明
之后控制器持续工作，让现实无限逼近你的声明
典型代表: Dataset、JuiceFSRuntime、Deployment

Kubernetes 核心的设计模式就是控制器模式（Controller Pattern），它是声明式 API 的引擎

## 命令式API
核心语义是“做什么“
通过下达一条“立即执行”的指令
之后系统执行一次，然后返回结果
典型代表:DataLoad、Job、kubectl run

命令式 API 更像一个按钮。你按下去 (提交 DataLoad 或 Job)，它就执行一个预定义的动作



