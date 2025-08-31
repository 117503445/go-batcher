# Go Batcher

Go Batcher 是一个通用且多功能的批处理算法实现，专为 Go 语言设计，无第三方依赖。该算法可以在空间（操作数量）和时间（超时）上进行限制，具有简单而强大的 API，使开发人员能够轻松地将批处理功能集成到他们的实时服务中。

## 特性

- **简单易用**：最小化的 API 设计，易于集成
- **资源高效**：有效利用内存和 goroutine
- **灵活配置**：支持按数量或时间触发批处理
- **并发安全**：支持多 goroutine 安全访问
- **上下文支持**：完整支持 Go 的 context 包进行取消和超时控制
- **泛型支持**：使用 Go 1.18+ 泛型，类型安全
- **零依赖**：无第三方依赖，轻量级实现

## 安装

```bash
go get github.com/117503445/go-batcher
```

## 快速开始

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/117503445/go-batcher"
)

func main() {
	// 定义提交函数，处理一批操作
	commitFn := func(ctx context.Context, ops batcher.Operations[string, string]) {
		// 检查上下文是否已取消
		select {
		case <-ctx.Done():
			// 如果批处理过程被中断，为所有操作设置错误
			ops.SetError(ctx.Err())
			return
		default:
		}

		// 处理批次中的所有操作
		for _, op := range ops {
			// 为每个操作设置结果
			op.SetResult(fmt.Sprintf("processed: %s", op.Value))
		}
	}

	// 创建一个批处理器，每 5 个操作或每 1 秒提交一次
	b := batcher.New(
		commitFn,
		batcher.WithMaxSize[string, string](5),
		batcher.WithTimeout[string, string](time.Second),
	)

	// 在后台运行批处理器
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	go func() {
		if err := b.Batch(ctx); err != nil {
			log.Printf("Batcher error: %v", err)
		}
	}()

	// 发送操作到批处理器
	op, err := b.Send(ctx, "hello world")
	if err != nil {
		log.Fatal(err)
	}

	// 等待操作结果
	result, err := op.Wait(ctx)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(result) // 输出: processed: hello world
}
```

## 核心概念

### Batcher

[Batcher](file:///workspace/project/go-batcher/batcher.go#L29-L34) 是核心类型，负责接收操作并按配置的策略进行批处理。

创建 Batcher 需要提供一个 [CommitFunc](file:///workspace/project/go-batcher/commit.go#L5-L5) 和可选的配置选项：

```go
b := batcher.New(commitFn, batcher.WithMaxSize[string, string](10))
```

### Operation

[Operation](file:///workspace/project/go-batcher/operation.go#L6-L11) 代表单个操作，包含输入值和结果。

使用 [Send](file:///workspace/project/go-batcher/batcher.go#L91-L99) 方法向 Batcher 发送操作：

```go
op, err := b.Send(ctx, "value")
```

使用 [Wait](file:///workspace/project/go-batcher/operation.go#L35-L46) 方法等待操作结果：

```go
result, err := op.Wait(ctx)
```

### Operations

[Operations](file:///workspace/project/go-batcher/operations.go#L2-L2) 是 [Operation](file:///workspace/project/go-batcher/operation.go#L6-L11) 的切片，代表一批操作。

在 [CommitFunc](file:///workspace/project/go-batcher/commit.go#L5-L5) 中，您可以遍历所有操作并为它们设置结果或错误：

```go
commitFn := func(ctx context.Context, ops batcher.Operations[string, string]) {
    for _, op := range ops {
        op.SetResult(process(op.Value))
    }
}
```

## 配置选项

### WithMaxSize

设置批处理的最大操作数：

```go
b := batcher.New(commitFn, batcher.WithMaxSize[string, string](10))
```

### WithTimeout

设置批处理的超时时间：

```go
b := batcher.New(commitFn, batcher.WithTimeout[string, string](time.Second))
```

## 使用场景

1. **批量数据库操作**：将多个数据库操作合并为单个批量操作，提高性能
2. **API 请求合并**：合并多个小的 API 请求为单个批量请求
3. **日志处理**：将多条日志记录批量写入存储系统
4. **消息队列**：批量处理消息以提高吞吐量
