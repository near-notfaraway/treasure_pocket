# 【Golang】并发编程

* [【Golang】并发编程](#golang并发编程)
   * [并发语句](#并发语句)
      * [Goroutine](#goroutine)
      * [信道](#信道)
      * [Select](#select)
   * [并发控制](#并发控制)
      * [WaitGroup](#waitgroup)
      * [Context](#context)
   * [协程安全](#协程安全)
      * [锁同步](#锁同步)
      * [状态同步](#状态同步)
      * [安全映射](#安全映射)
      * [安全工具](#安全工具)
      * [原子操作](#原子操作)

## 并发语句
Go 基于 Goroutine 原生支持并发编程，注意 Go 是并发式语言，而不是并行式语言。但通过 `runtime.GOMAXPROCS` 环境变了能够设置 Go Runtime 运行时所使用的 CPU 核数，相当于设置所有的 Goroutine 会被调度和并行于多少个系统线程中

### Goroutine
**Goroutine** 就是 **Go 中的协程 Coroutine**，即一种轻量级线程，是 Go 运行时自动调度的最小执行单元，其无论是内存占用，还是创建、销毁和切换的开销都非常小。通过 `go` 关键字可进行 Goroutine 的创建，其使用方式如下：

``` go
// 创建协程来执行该函数的调用
go funcName(args)

// 创建协程，执行匿名函数的调用
go func() {
    // 函数体
}(args)
```

所传入的 `args` 参数的求值发生在当前协程中，而 `funcName` 函数的执行发生在新的协程中。协程执行的函数无法直接返回结果，需要使用信道来进行结果获取

主协程需要利用信道或其他机制进行阻塞等待，以便它创建的其他协程能够执行完毕

### 信道
**信道（Channel）** 是 Goroutine 之间通信的管道，信道需要关联一个类型，表示该类型的数据可以通过信道传输

信道是一种引用类型，其零值是 `nil` 且没有作用，因此需要用 `make()` 函数来创建信道

``` go
// 声明关联 Type 类型的信道类型
Type TypeCh chan Type

// 创建信道
ch := make(chan Type)
```

信道默认是双向的，即可以通过信道读取和发送数据：

``` go
// 读取信道的数据，并赋值到变量
varName := <- ch

// 将变量的值发送到信道
ch <- varName 
```

信道的操作是协程安全且默认阻塞的，当一个 Goroutine 进行信道操作发生阻塞时，Go 会把控制权切换到其他 Goroutine，直到该信道操作成功后，该 Goroutine 才会解除阻塞并重新获得控制权

信道的阻塞特性能让 Goroutine 之间进行高效的通信，不需要用到显式的锁或变量来判断跨 Goroutine 通信的状态

如果一个 Goroutine 操作一个信道进行发送或读取数据并且进入阻塞时，没有其它 Goroutine 操作该信道读取或发送数据，则在运行时该 Goroutine 会永久阻塞，形成死锁并触发 `panic`

利用信道的协程安全和阻塞特性，实现等待子协程执行完毕：

``` go
func hello(done chan struct{}) {  
    fmt.Println("Hello world goroutine")
    done <- struct{}{}
}

func main() {  
    done := make(chan struct{})
    go hello(done)
    <-done
}
```

用于信号通知的信道，使用空结构体类型信道 `chan struct{}` 是对内存更友好的开发方式，因为在 Go 中对于空结构体类的内存空间申请，返回的是一个固定地址，避免了可能发生的内存滥用

单向信道指只能用于发送或者接收数据的信道，即 **唯送信道（Send Only）** 和 **唯收信道（Receive Only）**，声明时在 `chan` 旁以左箭头表示发送到信道，还是从信道中接收

``` go
// 声明唯送信道类型
Type TypeCh chan<- Type

// 声明唯收信道类型
Type TypeCh <-chan Type
```

信道转化，只能双向信道转为单向信道：

``` go
// 转化 ch 为唯送信道
var sendCh chan<- Type = ch

// 转化 ch 为唯收信道
var receiveCh <-chan Type = ch
```

区分协程对通道的读写，使用实例：

``` go
// 用于发送数据的协程
func sendData(sendCh chan<- int) {  
    sendCh<- 10
}

// 用于接收数据的协程
func receiveData(receiveCh <-chan int) {  
    <-receiveCh
}

// 在主协程中，信道定义为双向信道
// 通过信道可以发送或接收数据
func main() {  
    ch := make(chan int)
    go sendData(ch)
    go receiveData(ch)
    // 定时执行一段时间
    <-time.After(10 * time.Second) 
}
```

关闭信道用于表示该信道不再有数据可被读取，因为向一个已关闭的信道发送数据会触发 `panic`。信道通常情况下无需关闭，只在必须通知接收者信道不再有数据可被读取的情况才关闭

``` go
// 关闭信道
close(ch)

// 接收数据并通过 ok 判断信道是否关闭，若信道已关闭则会读取到信道关联类型的零值
varName, ok := <-ch

// 使用 range 可以迭代读取信道，直到信道关闭
for v := range ch {
    fmt.Println("Received ", v)
}
```

**缓冲信道（Buffered Channel）** 是带有缓冲的信道，用于缓冲数据以在一定条件下实现非阻塞地操作信道。缓冲信道的发送操作只有当缓冲已满时才会阻塞，而接收操作只有当缓冲为空时才会阻塞

``` go
// 创建缓冲信道，capacity 表示缓冲容量，默认为 0，即无缓冲
chanName := make(chan Type, capacity)
```

缓冲信道关闭后，对于接收操作会返回其缓冲的数据，直到缓冲为空后，才返回信道关联类型的零值和信道关闭的标志

由于缓冲信道是非阻塞的，因此在协程中可以读取到自己发送的数据，所以当两个协程之间需要通过非阻塞信道通信时，应该创建两个缓冲信道，一个用于前者写入后者接收，另一个相反

### Select
`select` 语句用于同时监听多个信道操作的状态，完成信道操作并执行对应代码块，以实现信道的多路复用

监听过程协程会阻塞，直到存在至少一个信道操作准备就绪，如果多个信道操作同时准备就绪，则会随机选取其中之一，完成信道操作并执行对应代码块

``` go
select {
    case <-ch1:
        // 接收数据后的操作
    case ch2<- varName:
        // 发送数据后的操作
    default:
        // 操作均未就绪的操作
}
```

没有 `case` 语句的空 `select` 语句会一直阻塞，形成死锁并触发恐慌。可以通过 `default` 语句，避免在没有任一信道操作准备就绪时避免阻塞

利用 `select` 语句实现可以 **工作池（Worker Pool）**，即维护一组用于并发执行任务的 Worker Goroutine，从而避免了反复创建 Goroutine 的开销，每个 Worker Goroutine 一旦完成当前的任务，就可继续等待新的任务

``` go
// 拉起新的 Goroutine 来执行 task 直到个数等于 size
type WorkerPool struct {
	task chan func()
	sem  chan struct{}
}

func NewWorkerPool(size int) *WorkerPool {
	return &WorkerPool{
		task: make(chan func()),
		sem:  make(chan struct{}, size),
	}
}

// 无 worker 接收 task 则新建 worker
func (p *WorkerPool) NewTask(task func()) {
	select {
	case p.task <- task:
	case p.sem <- struct{}{}:
		go p.worker(task)
	}
}

// worker 运行完毕 task 后继续等待其他 task
func (p *WorkerPool) worker(task func()) {
	defer func() { <-p.sem }()
	for {
		task()
		task = <-p.task
	}
}
```

## 并发控制
### WaitGroup
`sync.WaitGroup` 通过计数器的机制来实现对一批 Goroutine 进行等待，一直阻塞到所有的 Goroutine 执行完毕

`sync.WaitGroup` 的使用方法如下：

``` go
// 为 WaitGroup 的计数器增加指定值
func (wg *WaitGroup) Add(delta int) 

// 为 WaitGroup 的计数器减少 1
func (wg *WaitGroup) Done()

// 阻塞直到 WaitGroup 计数器变为 0
func (wg *WaitGroup) Wait()
```

阻塞等待所有 Goroutine 完成的使用实例：

``` go
// 协程需要获取 wg 的指针进行计数器操作
func process(i int, wg *sync.WaitGroup) {  
    fmt.Println("started Goroutine ", i)
    time.Sleep(2 * time.Second)
    fmt.Printf("Goroutine %d ended\n", i)
    wg.Done()
}

// 为计数器增加协程数目的值，并传递 wg 的指针到协程中
func main() {  
    wg := &sync.WaitGroup{}
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go process(i, wg)
    }
    
    wg.Wait()
    fmt.Println("All go routines finished executing")
}
```

### Context
`context` 包提供上下文机制，通过函数调用传播截止时间、取消信号或其他请求范围的值，以实现跨函数以及跨协程的控制

`context.Context` 表示一个上下文，其中的方法都是协程安全的，并且是幂等的，允许反复进行调用，具体方法如下：

``` go
type Context interface {
	// 返回截止时间，截止时间指的是上下文被取消的时间
	// 如果没有截止时间则 ok 为 false
	Deadline() (deadline time.Time, ok bool)

	// 返回上下文被取消时会关闭的信道
	// 如果上下文不能被取消，则返回 nil
	Done() <-chan struct{}截止时间

	// 上下文已取消后，返回非空错误以说明取消的原因
	// 上下文未被取消时，返回 nil
	Err() error

	// 返回上下文中的 key 对应的值，不存在为 nil
	Value(key interface{}) interface{}
}
```

不同类型的上下文具有不同的构造方式，所返回的 `Context` 的底层值都是指针类型，用法如下：

``` go
// 获取一个空上下文，用于发起函数中，作为所有派生上下文的根
// 空上下文指无截止时间，不能被取消且没有值通的上下文
func Background() Context

// 同样获取一个空上下文，用于开发阶段预留的上下文
func TODO() Context

// 返回带有新 Done 信道的派生上下文（内嵌父上下文）和取消函数
// 当取消函数或者父上下文的 Done 信道被关闭时，该派生上下文被取消
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// 返回带有截止时间和新 Done 信道的派生上下文（内嵌父上下文）和取消函数
// 当截止时间到达、取消函数或者父上下文的 Done 信道被关闭时，该派生上下文被取消
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)

// 返回带有根据超时生成的截止时间和新 Done 信道的派生上下文（内嵌父上下文）和取消函数
// 当截止时间到达、取消函数或者父上下文的 Done 信道被关闭时，该派生上下文被取消
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
}

// 返回带有键值的派生上下文（内嵌父上下文）
func WithValue(parent Context, key, val interface{}) Context
```

上下文使用方式需要注意几点：
- 上下文应该以参数方式显示传递，不要放在结构体中，并且应该作为第一个参数
- 传递上下文时不要传递 `nil`，未确定就传递 `context.TODO()`
- 及时取消上下文可以防止协程泄漏

利用上下文进行 Goroutine 控制的实例如下：

``` go
// 生成整数发送到信道中
func gen(ctx context.Context, dst chan<- int) {
    n := 1	
    for {
        select {
        case <-ctx.Done():
            // 关闭信道并返回，避免泄漏协程
            close(dst)
            return
        case dst <- n:
            n++
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    dst := make(chan int)
    go gen(ctx, dst)
    	
    // 当循环获取信道的整数为 5 时，取消上下文
    for n := range dst {
    	if n == 5 {
    	  cancel()
    	}
    }

    // 定时运行 20 秒后返回
    <-time.After(time.Second * 20)
}
```

## 协程安全
当多个 Goroutine 并发运行访问一个共享的资源时，如果这些 Goroutine 对资源被访问的顺序敏感，则多个 Goroutine 中的这个访问操作是不安全的，即 **协程不安全的**。这种情况也可描述为 Goroutine 之间存在 **竞态（Race Condition）**，而导致竞态发生的代码区称作 **临界区（Critical Section）**

修复竞态有两种方法，一是通过容量为 `1` 缓冲信道，二是通过 `sync` 包提供的锁机制。其中利用缓冲信道的协程安全和阻塞特性，实现修复竞态的例子如下：

``` go
// 声明公共变量
var x  = 0

// 创建 wg 和 ch 并用于 Goroutine 中
func main() {  
    wg := &sync.WaitGroup{}
    ch := make(chan struct{}, 1)
    
    for i := 0; i < 1000; i++ {
        w.Add(1)        
        go func() {
            ch <- struct{}{}
            // 临界区，执行协程不安全操作
            x = x + 1
            <- ch
            wg.Done()  
        }()
    }
    
    wg.Wait()
    fmt.Println("final value of x", x)
}
```

### 锁同步
`sync` 包提供了 **互斥锁** `sync.Metux` 和 **读写锁** `sync.RWMetux` 这两种锁同步机制，可针对不同场景用于修复竞态。这两种锁都实现了以下接口：

``` go
type Locker interface {
        // 阻塞直到获取锁
        Lock()
        // 释放锁
        Unlock()
}
```

锁的获取和释放必须成对出现，否则会造成死锁，使得其他 Goroutine 永久阻塞在锁等待的状态，可以在锁获取之后利用 `defer` 语句来保证 Goroutine 结束后必须进行锁释放

`sync.Metux` 用于保证任意时刻只有一个 Goroutine 运行在临界区，即任意时刻只有一个 Goroutine 能获取到锁，从而执行从获取锁和释放锁之间的临界区的代码，其用法如下：

``` go
// 声明公共变量
var x  = 0

// 创建 wg 和 mtx 并用于 Goroutine 中
func main() {
    wg := &sync.sync.WaitGroup{}  
    mtx := &sync.Mutex{}
    
    for i:=0; i<10; i++ {
        wg.Add(1)
        go func() {
            m.Lock()
            // 临界区，执行协程不安全操作
            x += 1
            m.Unlock()
            wg.Done()
        }
    }
    
    wg.Wait()
    fmt.Println("final value of x", x)
}
```

`sync.RWMetux` 用于保证任意时刻有多个进行读操作的 Goroutine 或者一个进行写操作的 Goroutine 运行在临界区，即任意时刻有多个 Goroutine 能获取到读锁，或只有一个 Goroutine 能获取到写锁，适合针对某个资源进行读多写少的场景，其包含另外两个方法：

``` go
// 写锁用法等同于互斥锁
func (rw *RWMutex) Lock()
func (rw *RWMutex) Unlock()

// 读锁的获取和释放
func (rw *RWMutex) RLock()
func (rw *RWMutex) RUnlock()
```

针对多读少写的场景，利用读写锁减少阻塞的实例如下：

``` go
// 声明公共变量
var x = 0

// 获取写锁后来对公共变量进行修改，完成后释放锁
func write(wg *sync.WaitGroup, m *sync.Mutex) {  
    m.Lock()
    defer m.Unlock()
    x = x + 1
    wg.Done()   
}

// 获取读锁后来对公共变量进行打印，完成后释放锁
func read(wg *sync.WaitGroup, m *sync.Mutex) {  
    m.RLock()
    defer m.RUnlock()
    fmt.Println("tmp value of x", x)
    wg.Done()   
}

// 创建 wg 和 m，并将其指针传到协程中
func main() {  
    wg := &sync.WaitGroup{}
    m := &sync.Mutex{}
    
    // 读多写少
    for i := 0; i < 1000; i++ {
        wg.Add(1)        
        if i%5 == 0 {
            go write(wg, m)
        } else {
            go write(wg, m)
        }
    }
    
    w.Wait()
    fmt.Println("final value of x", x)
}
```

### 状态同步
`sync.Cond` 表示需要跨 Goroutine 同步的状态，可用于协调各个需要访问同一共享资源的 Goroutine，当共享资源的状态发生变化时，能够对被互斥锁阻塞的 Goroutine 进行通知

状态同步的优势是在效率方面的提升，当共享资源的状态不满足要求时，不需要负责操作的 Goroutine 不用循环地检查其状态是否发生变化，而是阻塞等待条件变量的通知即可

`sync.Cond` 需要配合 `sync.Locker` 进行使用，用来保证 Goroutine 在等待状态变化前后和通知状态变化时都必须获取到锁，即保证资源访问的过程不会出现竞态，其创建和使用方式如下： 

``` go
// 使用指定的 Locker，创建一个状态
func NewCond(l Locker) *Cond

// 获取 Cond 内部的 Locker
Cond.L

// 阻塞当前 Goroutine，直到收到该状态变化的通知
func (c *Cond) Wait()

// 单播通知，向任一等待该状态变化的 Goroutine 发送通知
func (c *Cond) Signal()

// 广播通知，向所有等待该状态变化的 Goroutine 发送通知
func (c *Cond) Broadcast()
```

`sync.Cond.Wait()` 方法开始时会自动释放 `Cond.L` 的锁，返回时会自动获取 `Cond.L` 的锁，等待侧 Goroutine 的一般写法如下：

``` go
// 声明公共变量
var x = 0

c := sync.NewCond(&sync.Mutex{})
for i := 0; i < 1000; i++ {
    go func shouleWait() {
        // 访问共享资源前获取锁
        c.L.Lock()
        // 共享资源不满足使用条件，则等待状态变化
        if !condition(x) {
            c.Wait()
        }
        // 状态变化，访问共享资源并进行处理
        operation(x)
        c.L.Unlock()
    }
}
```

基于状态同步对队列进行生产和消费的实例如下：

``` go
// 公共队列同一时刻只允许被一个 Goroutine 访问
var list = make([]int)

// 当列表不为空时，消费其所有元素，并发出通知继续生产 
func useList(c *sync.Cond) {
    c.L.Lock()
    if len(list) == 0 {
        c.Wait()
    }
    for _, v := range list {
        fmt.Printf("Use element %d\n", v)
    }
    c.Signal()
    c.L.Unlock()
}

// 当列表为空时，生产一批元素，并发出通知继续消费 
func pushList(c *sync.Cond) {
    c.L.Lock()
    if len(list) > 0 {
        c.Wait()
    }
    for i := 1;i<10;i++ {
        list = append(list, i)
    }
    c.Signal()
    c.L.Unlock()
}

func main {
    c := sync.NewCond(&sync.Mutex{})
    go useList(c)
    go pushList(c)
    // 定时执行一段时间
    <-time.After(10 * time.Second) 
}
```

### 安全映射
内置的映射类型 `map` 在没有锁的情况下是协程不安全的，而 `sync.Map` 则提供了一个开箱即用的协程安全的 `map`，其创建和使用方法如下：

``` go
// 直接通过初始化来创建
m := &sync.Map{}

// 存储一个键值到映射
func (m *Map) Store(key, value interface{})

// 从映射获取指定键的值，若存在则 ok 为 true，否则为 false
func (m *Map) Load(key interface{}) (value interface{}, ok bool)

// 从映射获取指定键的值，若存在则 loaded 为 true，否则为 false，并且会将值定的值存入且返回
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)

// 从映射删除一个键值
func (m *Map) Delete(key interface{})

// 迭代映射，f 是用于接收迭代获取的键值并进行处理的函数
// f 的返回值为 true 时表示继续迭代，false 表示停止
func (m *Map) Range(f func(key, value interface{}) bool)
```

### 安全工具
`sync.Pool` 提供了一个对象池，用于保存和复用临时对象，减少内存分配，降低 GC 压力，同时它是可伸缩的，其容量大小仅受限于内存的大小，并且是协程安全的

声明对象池只需要定义对象创建的 `New()` 函数，当对象池中没有对象时，它会自动调用 `New()` 进行函数创建新的对象

``` go
type obj struct{}

var objPool = sync.Pool{
    New: func() interface{} { 
        return new(obj) 
    },
}
```

可以使用对象池进行对象获取和保存，由于对象获取时返回的是 `interface{}`，因此使用之前需要进行断言处理

``` go
// 对象获取
func (p *Pool) Get() interface{} {}

// 对象保存
func (p *Pool) Put(x interface{}) {}
```

`sync.Once` 提供了使函数只执行一次的实现，并且是协程安全的，只会有一个协程触发函数执行，常应用于单例模式的实现。其使用方式如下：

``` go
// 初始化后执行指定的函数
once := &sync.Once{}
once.Do(func)
```

`sync.Once` 作用与包的 `init` 函数类似，但`init` 函数是包首次被导入时执行，若长时间无使用，可能出现内存浪费和延长程序启动时间的情况；而 `sync.Once` 可以在代码的任意位置执行，即可以延迟到使用前再执行

### 原子操作
`sync/atomic` 包提供了无锁但协程安全的原子操作，它是通过 CPU 指令直接实现的，事实上状态同步、锁同步等协程同步机制，都可以通过信道操作进行替代，因为它们都是基于原子操作实现的，但信道的性能最差，而原子操作的性能最高

对于一个整数类型 `T`，其中 `T` 可以是 `int32`、`int64`、`uint32`、`uint64` 和 `uintptr` 基础类型，都提供了以下基于各个类型指针的原子操作函数：

``` go
// 为指针中的值增加指定值
func AddT(addr *T, delta T)(new T)

// 获取指针中的值
func LoadT(addr *T) (val T)

// 保存指定值到指针
func StoreT(addr *T, val T)

// 交换指定的新值到指针，返回旧值
func SwapT(addr *T, new T) (old T)

// 若指针中的值等于旧值，则交换指定的新值到指针，返回是否完成交换
func CompareAndSwapT(addr *T, old, new T) (swapped bool)
``` 
