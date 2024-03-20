# golang 运行时死锁排查和检测

> 当运行的系统发生goroutine等待获取锁时间超过预期时，判定为发生了死锁。因目前代码中使用了一些公开的锁实例，调用链也比较长，对问题排查带来了很大困扰。为了便于问题排查，需要借助工具来实现。

## 1. 发生死锁的判定依据和原因

### 1.1 判定依据

如下为使用Mutex锁产生的锁等待，并持续了222分钟。这类不合理现象，就判定为发生了死锁。（以下为cpu-pprof导出的stack信息）

```plain
goroutine 61508 [semacquire, 222 minutes]:
sync.runtime_SemacquireMutex(0xc00048f564, 0x4f4300, 0x1)
    /opt/go/src/runtime/sema.go:71 +0x47
sync.(*Mutex).lockSlow(0xc00048f560)
    /opt/go/src/sync/mutex.go:138 +0x105
sync.(*Mutex).Lock(...)
    /opt/go/src/sync/mutex.go:81
......
```

### 1.2 产生死锁的原因

- 锁重入

- 多把锁加锁，并发访问时加锁顺序不一致

- 锁未释放

## 2. 死锁问题诊断

常用的标准库锁： `sync.Mutex`, `sync.RWMutex`，目标暂定为以检测这两类锁的死锁问题。

检测死锁的方法：分为静态代码分析、运行期分析。

### 2.1 静态代码分析

- 优势：

  可以做到场景全覆盖。

- 劣势：

  好像没有劣势，但开发难度较高，目前发现的一个开源工具 [go-tools](https://github.com/dominikh/go-tools)，验证下来无法检测互锁问题，但可以检测锁未释放的问题。

### 2.2 运行期分析

- 优势：

  可以快速定位问题，检测相对容易。

- 劣势：

  对代码有侵入性，需要封装标准库的锁，对性能会有一定影响，应采取条件编译方式，生产环境勿启用。

## 3. 运行期分析工具开发

### 3.1 目标

- 支持 Mutex 和 RWMutex

- 支持锁重入、互锁问题的检测

### 3.2 用法

```go
import (
    "github.com/berkaroad/detectlock-go"
)

// 应用启动时，设置启用调试
detectlock.EnableDebug()

// 声明 sync.Mutex、sync.RWMutex 替换为 detectlock.Mutex、detectlock.RWMutex
var locker1 *detectlock.Mutex = &detectlock.Mutex{}
var locker2 *detectlock.RWMutex = &detectlock.RWMutex{}

// 异步检测死锁
items := detectlock.Items()
fmt.Println(detectlock.DetectAcquired(items)) // 检测获得锁的goroutine列表
fmt.Println(detectlock.DetectReentry(items)) // 检测锁重入的goroutine列表
fmt.Println(detectlock.DetectLockedEachOther(items)) // 检测互锁的goroutine列表

// 关闭调试，并清理锁使用信息
detectlock.DisableDebug()
```

### 3.3 死锁示例数据

- 检测 sync.Mutex 多把锁顺序不一致导致的互锁

```plain
--- DetectAcquired ---
goroutine 29: [(0xc0000b4008, acquired, main.B (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex01/main.go:30)), (0xc0000b4000, wait, main.B (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex01/main.go:33))]
goroutine 30: [(0xc0000b4000, acquired, main.A (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex01/main.go:20)), (0xc0000b4008, wait, main.A (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex01/main.go:23))]

--- DetectLockedEachOther ---
goroutine 29: [(0xc0000b4008, acquired, main.B (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex01/main.go:30)), (0xc0000b4000, wait, main.B (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex01/main.go:33))]
goroutine 30: [(0xc0000b4000, acquired, main.A (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex01/main.go:20)), (0xc0000b4008, wait, main.A (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex01/main.go:23))]

```

- 检测 sync.Mutex 锁重入

```plain
--- DetectAcquired ---
goroutine 53: [(0xc0000160b8, acquired, main.C (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex02/main.go:19)), (0xc0000160b8, wait, main.C (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex02/main.go:22))]

--- DetectReentry ---
goroutine 53: [(0xc0000160b8, acquired, main.C (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex02/main.go:19)), (0xc0000160b8, wait, main.C (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex02/main.go:22))]

```

- 检测 sync.Mutex、sync.RWMutex 多把锁顺序不一致导致的互锁

```plain
--- DetectAcquired ---
goroutine 8: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 10: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 13: [(0xc0000160b8, acquired, main.E (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:30)), (0xc0000180c0, wait, main.E (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:33))]
goroutine 14: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 16: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 50: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 52: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 54: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 56: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 58: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]

--- DetectLockedEachOther ---
goroutine 8: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 10: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 13: [(0xc0000160b8, acquired, main.E (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:30)), (0xc0000180c0, wait, main.E (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:33))]
goroutine 14: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 16: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 50: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 52: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 54: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 56: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]
goroutine 58: [(0xc0000180c0, r-acquired, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:21)), (0xc0000160b8, wait, main.D (file: /home/berkaroad/github/berkaroad/detectlock-go/examples/mutex03/main.go:24))]

```

---------------------------------分割线-------------------------------------------------------------

我的[detectlock-go项目: https://github.com/berkaroad/detectlock-go](https://github.com/Berkaroad/detectlock-go)，已经挂在github上，有兴趣的可以去了解下。

使用中有何问题，欢迎在github上给我提issue，谢谢！
