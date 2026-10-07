Zap 性能好，原生支持结构化数据，字段类型安全有保障String，Int，Any等等
## 创建一个生产级别的Logger
```go
package main

import (
	"os"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

func newLogger() *zap.Logger {
	//定义编码器
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.TimeKey = "timestamp"
	encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder

	encoder := zapcore.NewJSONEncoder(encoderConfig)

	// 定义输出位置
	file, err := os.OpenFile("storage.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
	if err != nil {
		panic(err)
	}
	cores := []zapcore.Core{
		zapcore.NewCore(encoder, zapcore.AddSync(file), zapcore.WarnLevel),
		zapcore.NewCore(encoder, zapcore.AddSync(os.Stdout), zapcore.InfoLevel),
	}

	//合并多个输出
	core := zapcore.NewTee(cores...)

	//创建 logger
	return zap.New(core, zap.AddCaller(), zap.Development())
}

func main() {
	logger := newLogger()
	defer logger.Sync()

	logger.Info("hello world",
		zap.String("nodeId", "node-01"),
		zap.Int("port", 8080),
		zap.String("ip", "127.0.0.1"),
	)

	logger.Warn("hello world",
		zap.String("nodeId", "node-02"),
		zap.Int("port", 8080),
	)

}

```

## Zap快的原因
关键在于避免内存分配与反射

zap.S().Info("hello") 预分配缓冲区加上对象池，类型字段安全
1. 预分配Buffer+对象池复用：zap直接将日志字段序列化到一个`[]byte`缓冲区中(来自sync.Pool)
2. 避免反射，类型字段安全：自己实现高效的json编码，比如直接写kv字节，而不是使用json.Marshal
如zap.Int("size",4096)： 直接将4096转为字节写入缓冲区，没有interface{}封装
3. 可以内联优化

而fmt.Println()是interface{}封装，运行时期反射开销大

## zap.Sugar() 不推荐使用：
虽然支持：
```go
sugar.Infof("Hello %s", name)
```
但是底层就要反射