---

title: Go中控制协程数量的几种方法

summary: 展示通过通道和信号量的方式来控制线程数

date: 2022-12-08

draft: false

authors:

    - admin

tags:

    - Golang 

    - 并发

categories:

    - Golang

    - 并发

---

    在Golang中通过```go```关键字可以很方便的创建一个协程， 并且由于协程资源占用很小，可以随意创建成千上万个协程，但是在实际使用中，其数量往往因为各种因素不能无限制的增长，例如网络连接、数据库连接或者下游服务处理能力等，因此需要控制协程数量。下文将分别尝试用通道和信号量的方式来控制协程数。

## 1、Buffer Channel 缓冲通道

通过创建一个buffer channel，size为控制的协程数，每次任务提交时，先```ch <- struct{}{}```，如果channel已满，将阻塞在此，任务执行完后```<-ch```将释放一个空间给下一个任务，以此来控制协程数。

```go
func doThing(d interface{}){ 
    // do your thing
}
func main() { 
    data := []interface{}{} // len == 10
    poolSize := runtime.NumCPU() // 4 线程 CPU 
    ch := make(chan struct{}, poolSize) 
    for _, d := range data { 
        ch <- struct{}{} 
        go func(d interface{}) { 
            doThing(d) 
            <-ch
        }(d) 
    } 
    close(ch)
}
```

## 2、Semaphore信号量

*信号量是一种变量或抽象数据类型，用于控制并发系统（例如多任务操作系统）中的多个进程对公共资源的访问。*

```go
import (
    "context"
    "golang.org/x/sync/semaphore"
)
func doThing(d interface{}){
    // do your thing
}
func main() {
    data := []interface{}{} // len == 10
    poolSize := runtime.NumCPU() // 4 thread CPU
    sem := semaphore.NewWeighted(poolSize)
    for _, d := range data {
        sem.Acquire(context.Background(), 1)
        go func(d interface{}){
            doThing(d)
            sem.Release(1)
        }(d)
    }
}
```

先调用NewWeighted()返回一个sem对象，每次需要创建协程前先尝试Acquire()，超过最大协程数限制时会阻塞在这里，任务执行完毕调用release()方法释放资源。其源码如下：

```go
func NewWeighted(n int64) *Weighted {
    w := &Weighted{size: n}
    return w
}

// Weighted provides a way to bound concurrent access to a resource.
// The callers can request access with a given weight.
type Weighted struct {
    size    int64
    cur     int64
    mu      sync.Mutex
    waiters list.List
}
```

可以看出是通过cur和size的关系来限制，使用锁来确保并发安全，获取资源失败的加入到waiters队列里。

```go
func (s *Weighted) Acquire(ctx context.Context, n int64) error {
    s.mu.Lock()
    if s.size-s.cur >= n && s.waiters.Len() == 0 {
        s.cur += n
        s.mu.Unlock()
        return nil
    }

    if n > s.size {
        // Don't make other Acquire calls block on one that's doomed to fail.
        s.mu.Unlock()
        <-ctx.Done()
        return ctx.Err()
    }

    ready := make(chan struct{})
    w := waiter{n: n, ready: ready}
    elem := s.waiters.PushBack(w)
    s.mu.Unlock()

    select {
    case <-ctx.Done():
        err := ctx.Err()
        s.mu.Lock()
        select {
        case <-ready:
            // Acquired the semaphore after we were canceled.  Rather than trying to
            // fix up the queue, just pretend we didn't notice the cancelation.
            err = nil
        default:
            isFront := s.waiters.Front() == elem
            s.waiters.Remove(elem)
            // If we're at the front and there're extra tokens left, notify other waiters.
            if isFront && s.size > s.cur {
                s.notifyWaiters()
            }
        }
        s.mu.Unlock()
        return err

    case <-ready:
        return nil
    }
}
func (s *Weighted) Release(n int64) {
    s.mu.Lock()
    s.cur -= n
    if s.cur < 0 {
        s.mu.Unlock()
        panic("semaphore: released more than held")
    }
    s.notifyWaiters()
    s.mu.Unlock()
}
```
