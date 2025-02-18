1. 避免小对象接口、使用批处理接口
```go
// 避免小对象接口
type Reader interface {
    Read([]byte) (int, error)  // 传递切片而不是小对象
}

// 使用批处理接口
type BatchProcessor interface {
    ProcessBatch(items []Item) error  // 批量处理而不是一个个处理
}
```

2. 接口提供默认实现
```go
type Logger interface {
    Log(message string)
}

// 提供默认实现
type DefaultLogger struct{}

func (l *DefaultLogger) Log(message string) {
    fmt.Println(message)
}

// 可以嵌入默认实现
type MyService struct {
    *DefaultLogger  // 嵌入默认实现
}
```

3. 使用泛型优化接口
```go
// 传统方式
type Repository interface {
    Find(id string) (interface{}, error)
    Save(entity interface{}) error
}

// 使用泛型优化
type GenericRepository[T any] interface {
    Find(id string) (T, error)
    Save(entity T) error
}
```

4. 接口隔离原则
```go
// 不好的做法：接口太大
type FileHandler interface {
    Read()
    Write()
    Close()
    Seek()
    Flush()
}

// 好的做法：拆分成小接口
type Reader interface {
    Read() error
}

type Writer interface {
    Write() error
}

type Closer interface {
    Close() error
}

// 组合接口
type ReadWriter interface {
    Reader
    Writer
}
```