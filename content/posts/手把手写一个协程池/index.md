---
title: "手把手写一个协程池"
date: "2024-04-14"
---

## 背景

协程池是啥？顾名思义就是一个有很多可以复用的协程的池子（pool）。那么为什么在 golang 里还需要这么一个协程池呢？

goroutine 不是号称比线程还轻量得多，创建与销毁都非常方便，轻轻松松可以抗成千上万的并发量吗？

没错，这的确是 golang 的显著优点所在，可以说 goroutine 贯穿 go 程序的整个生命周期，包括 main 函数、gc、net 等核心库的实现。但是再小的性能损耗在基数变得很大的时候也会随着一起放大，例如 10w 、100w 乃至 1000w 并发的时候单纯用原生的 goroutine 已经很难扛得住了。

这时候我们可以创建一个 pool 预初始化一些 goroutine 放在里面，当工作过来的时候就取出 goroutine 去执行这些工作，工作结束之后也不销毁 goroutine，而是继续放回去等着下次工作的到来。我们还能通过一些措施改进协程池，例如设定一个容量，如果协程池里的 goroutine 已经全部被占用了，在没有超出容量的情况下就创建新的 goroutine，若超出了容量就得再看整个模型是不是非阻塞式的，如果是非阻塞式的那么立刻返回错误：协程池已经被打满了，如果是阻塞式的那么就可以开启一个自旋锁等待空闲的 goroutine。另外还有 goroutine 的过期时间等等留到下面的具体实现过程再详细叙述。

但是话说回来协程池是某种意义上的屠龙刀，空有刀但世上又有龙吗？至少目前的我是远远接触不到这么高并发量的场景的（希望以后能接触到😭）。换个角度说估计需要单机抗几十上百万的并发的场景可能也比较少吧（应该。

### Golang 网络模型

我们都知道经历了早期的多进程和多线程 TCP 网络 I/O 模型之后现在操作系统用的都是 I/O 多路复用模型，这样就可以在网络层上只使用一个进程来监测多个 Socket 连接。

当然在网络层上是这样，在应用层上我们又会通过非阻塞式异步 I/O serve 调用相应的 handler 来 handle 对应的请求。

```
  for {    rw, err := l.Accept()    if err != nil {      select {      case <-srv.getDoneChan():        return ErrServerClosed      default:      }      if ne, ok := err.(net.Error); ok && ne.Temporary() {        if tempDelay == 0 {          tempDelay = 5 * time.Millisecond        } else {          tempDelay *= 2        }        if max := 1 * time.Second; tempDelay > max {          tempDelay = max        }        srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)        time.Sleep(tempDelay)        continue      }      return err    }    connCtx := ctx    if cc := srv.ConnContext; cc != nil {      connCtx = cc(connCtx, rw)      if connCtx == nil {        panic("ConnContext returned nil")      }    }    tempDelay = 0    c := srv.newConn(rw)    c.setState(c.rwc, StateNew, runHooks) // before Serve can return    go c.serve(connCtx)  }
```

所以说在网络编程中，通常会将非阻塞 IO 和 IO 多路复用结合使用，以实现高效的并发处理。上述讲到的一个在 runtime/netpoll 下，另一个在 net/http 下。

## 实践过程

首先我们来实现整个架子

```
type Pool struct {  workers []*Worker​  l sync.Locker}​type Worker struct {  task chan f​  pool *Pool}​type f func() error​func NewPool(cap int) *Pool {  return &Pool{    workers:  make([]*Worker, 0),    l:        &sync.Mutex{},  }}​func (p *Pool) Submit(task f) error {  return nil}​func (p *Pool) GetWorker() *Worker {  return nil}​func (w *Worker) Do() {​}​func (p *Pool) PutWorker(worker *Worker) {​}
```

### 初始化 Pool

新建了一个 Pool 对象，在这个简陋的版本里只有他掌管的 workers 和一把用来并发控制的锁。

```
func NewPool() *Pool {    return &Pool{       workers: make([]*Worker, 5),       l:       &sync.Mutex{},    }}
```

### 提交任务

首先 pool 会尝试从池里获取空闲的 worker ，若没有空闲的 worker 就会一直堵塞在这里。

```
func (p *Pool) Submit(task f) error {    w := p.GetWorker()    w.task <- task    return nil}
```

### 获取 Worker

这一部分是整个项目最核心的代码所在，首先我们会判断 pool 里是否有空闲的 worker ，在整个过程中是加锁的，同时为了避免死锁会用 一个 waiting 来标识是否有空闲 worker，如果获取了 worker 但是为 nil那我们还需要为其初始化再 run 对应的 task。如果 waiting 为 true 则一直自旋获取是否有新的空闲 worker 出现（这里有较大的性能损耗）。

```
func (p *Pool) GetWorker() *Worker {    var waiting bool    var w *Worker​    p.l.Lock()​    n := len(p.workers)    if n <= 0 {       waiting = true    } else {       w = p.workers[n-1]       p.workers[n-1] = nil       p.workers = p.workers[:n-1]    }​    p.l.Unlock()​    if waiting {       for {          p.l.Lock()          n = len(p.workers)          if n <= 0 {             p.l.Unlock()             continue          }          w = p.workers[n-1]          p.workers[n-1] = nil          p.workers = p.workers[:n-1]          p.l.Unlock()          break       }    } else if w == nil {       w = &Worker{          task: make(chan f, 1),          pool: p,       }       w.Do()    }​    return w}
```

### 执行任务

开启一个 goroutine 执行 worker channel 里的任务，如果堵塞就一直等待。

```
func (w *Worker) Do() {    go func() {​       for f := range w.task {          f()          w.pool.PutWorker(w)       }​    }()}
```

### 放回 Worker

执行任务完毕之后不销毁 goroutine 而是把 worker 放回 pool 等待下一次的调用。

```
func (p *Pool) PutWorker(worker *Worker) {    p.l.Lock()    p.workers = append(p.workers, worker)    p.l.Unlock()}
```

## 性能测试

写了如下的一个测试函数，使用 go test -benchmark 获得测试结果。

```
package test​import (    "pool"    "sync"    "sync/atomic"    "testing")​var wg = sync.WaitGroup{}​var sum int64​func demoTask() {    defer wg.Done()    for i := 0; i < 100; i++ {       atomic.AddInt64(&sum, 1)    }}​var runTimes = 1000000​// 原生 goroutinefunc BenchmarkGoroutineTimeLifeSetTimes(b *testing.B) {​    for i := 0; i < runTimes; i++ {       wg.Add(1)       go demoTask()    }    wg.Wait() // 等待执行完毕}​// 使用协程池func BenchmarkPoolTimeLifeSetTimes(b *testing.B) {    pool := pool.NewPool()​    for i := 0; i < runTimes; i++ {       wg.Add(1)       err := pool.Submit(demoTask)       if err != nil {          panic(err)       }    }​    wg.Wait() // 等待执行完毕}
```

```
goos: darwingoarch: arm64pkg: pool/testBenchmarkGoroutineTimeLifeSetTimesBenchmarkGoroutineTimeLifeSetTimes-8           1  3844667333 ns/opBenchmarkPoolTimeLifeSetTimesBenchmarkPoolTimeLifeSetTimes-8                1  2114626834 ns/opPASS​进程 已完成，退出代码为 0
```

可以看到我们使用了协程池之后性能是有大幅提升的。

## 进一步的优化

- 动态扩容

- worker 过期时间

- 优化锁

参考链接：

[https://taohuawu.club/archives/high-performance-implementation-of-goroutine-pool](https://taohuawu.club/archives/high-performance-implementation-of-goroutine-pool#toc-head-15)

[https://github.com/panjf2000/ants](https://github.com/panjf2000/ants)
