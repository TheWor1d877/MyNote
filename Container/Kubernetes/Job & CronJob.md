## Job
Job 创建一个或多个 Pod，并确保指定数量的 Pod 成功终止（exit code 0）。

#### Yaml语法
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: download-dataset
spec:
  template:
    spec:
      containers:
      - name: downloader
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo "Downloading dataset..."
          wget -O /data/train.csv http://example.com/train.csv
          echo "Done."
      restartPolicy: Never  # Job 必须是 Never 或 OnFailure
  backoffLimit: 4  # 最多重试 4 次
```

## CronJob 
基于Cron的表达式，周期性创建Job
```yaml
apiVversion: batch/v1
kind: CronJob
metadata:
  name: daily-model-train
spec:
  schedule: "0 2 * * *"  # 每天凌晨 2 点
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: trainer
            image: my-ai-trainer:v1
            command: ["python", "train.py"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3  # 保留最近 3 个成功 Job
  failedJobsHistoryLimit: 1     # 保留最近 1 个失败 Job
```

- 时间基于 kube-controller-manager 的时区（通常是 UTC！）
- 如果上次 Job 还没结束，下次是否会启动？→ 由 concurrencyPolicy 控制（默认 Allow）
- 不要用 CronJob 做高精度定时（K8s 调度有延迟，适合分钟级以上任务）

