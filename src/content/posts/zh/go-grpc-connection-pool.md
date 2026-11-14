---
title: "Go创建Grpc链接池"
author: "John Yang"
lang: zh
pubDatetime: 2023-03-02T10:39:00Z
tags:
  - Go
  - gRPC
  - Microservices
description: "存在的问题：面临高并发的情况，性能问题很容易就会出现，例如我们在做性能测试的时候，就会发现，打一会性能测试，客户端请求服务端的时候就会报错： rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest..."
---

## 常规用法

`gRPC` 四种基本使用
- 请求响应模式
- 客户端数据流模式
- 服务端数据流模式
- 双向流模式

常见的`gRPC`调用写法

```go
func main(){
	//... some code
	// 链接grpc服务
	conn , err := grpc.Dial(":8000",grpc.WithInsecure)
	if err != nil {
		//...log
	}
	defer conn.Close()
	//...some code
```

存在的问题：面临高并发的情况，性能问题很容易就会出现，例如我们在做性能测试的时候，就会发现，打一会性能测试，客户端请求服务端的时候就会报错：

`
rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: connection error: desc = "transport: Error while dialing dial tcp xxx:xxx: connect: connection refused
`

实际去查看问题的时候，很明显，这是 gRPC 的连接数被打满了，很多连接都还未完全释放。

`gRPC` 的通信本质上也是 `TCP` 的连接，那么一次连接就需要三次握手，和四次挥手，每一次建立连接和释放连接的时候，都需要走这么一个过程，如果我们频繁的建立和释放连接，这对于资源和性能其实都是一个大大的浪费。

在服务端，`gRPC` 服务端的链接管理不用我们操心，但是 `gRPC` 客户端的链接管理非常有必要关心，要实现复用客户端的连接。

***

## 创建链接池

创建链接池需要考虑的问题：

- 连接池是否支持扩缩容
- 空闲的连接是否支持超时自行关闭，是否支持保活
- 池子满的时候，处理的策略是什么样的

### 创建链接池接口
```go
type Pool interface {

	// 获取一个新的连接 ， 当关闭连接的时候，会将该连接放入到池子中
   Get() (Conn, error)

	// 关闭连接池，自然连接池子中的连接也不再可用
   Close() error

	//
   Status() string
}
```
### 实现链接池接口

**创建链接池代码**

```go
func New(address string, option Options) (Pool, error) {
   if address == "" {
      return nil, errors.New("invalid address settings")
   }
   if option.Dial == nil {
      return nil, errors.New("invalid dial settings")
   }
   if option.MaxIdle <= 0 || option.MaxActive <= 0 || option.MaxIdle > option.MaxActive {
      return nil, errors.New("invalid maximum settings")
   }
   if option.MaxConcurrentStreams <= 0 {
      return nil, errors.New("invalid maximun settings")
   }

   p := &pool{
      index:   0,
      current: int32(option.MaxIdle),
      ref:     0,
      opt:     option,
      conns:   make([]*conn, option.MaxActive),
      address: address,
      closed:  0,
   }

   for i := 0; i < p.opt.MaxIdle; i++ {
      c, err := p.opt.Dial(address)
      if err != nil {
         p.Close()
         return nil, fmt.Errorf("dial is not able to fill the pool: %s", err)
      }
      p.conns[i] = p.wrapConn(c, false)
   }
   log.Printf("new pool success: %v\n", p.Status())

   return p, nil
}
```

关于以上的代码，需要特别注意每一个连接的建立也是在 `New` 里面完成的，只要有 `1` 个连接未建立成功，那么咱们的连接池就算是建立失败，咱们会调用 `p.Close()` 将之前建立好的连接全部释放掉。

**关闭链接池代码**

```go
// 关闭连接池
func (p *pool) Close() error {
   atomic.StoreInt32(&p.closed, 1)
   atomic.StoreUint32(&p.index, 0)
   atomic.StoreInt32(&p.current, 0)
   atomic.StoreInt32(&p.ref, 0)
   p.deleteFrom(0)
   log.Printf("close pool success: %v\n", p.Status())
   return nil
}
```

**从具体位置删除链接池代码**

```go
// 清除从 指定位置开始到 MaxActive 之间的连接
func (p *pool) deleteFrom(begin int) {
   for i := begin; i < p.opt.MaxActive; i++ {
      p.reset(i)
   }
}
```

**销毁具体的链接代码**

```go
// 清除具体的连接
func (p *pool) reset(index int) {
   conn := p.conns[index]
   if conn == nil {
      return
   }
   conn.reset()
   p.conns[index] = nil
}
```

### 关闭链接

代码
```go
func (c *conn) reset() error {
   cc := c.cc
   c.cc = nil
   c.once = false

   if cc != nil {
      return cc.Close()
   }
   return nil
}

func (c *conn) Close() error {
   c.pool.decrRef()
   if c.once {
      return c.reset()
   }
   return nil
}
```

在使用连接池通过 `pool.Get()` 拿到具体的连接句柄 `conn` 之后，会使用 `conn.Close()`关闭连接，实际上也是会走到上述的 `Close()` 实现的位置，但是并未指定当然也没有权限显示的指定将 `once` 置位为 `false` ，也就是对于调用者来说，是关闭了连接，对于连接池来说，实际上是将连接归还到连接池中。

***

## 扩缩容

关键代码
```go
func (p *pool) Get() (Conn, error) {
   // the first selected from the created connections
   nextRef := p.incrRef()
   p.RLock()
   current := atomic.LoadInt32(&p.current)
   p.RUnlock()
   if current == 0 {
      return nil, ErrClosed
   }
   if nextRef <= current*int32(p.opt.MaxConcurrentStreams) {
      next := atomic.AddUint32(&p.index, 1) % uint32(current)
      return p.conns[next], nil
   }

   // the number connection of pool is reach to max active
   if current == int32(p.opt.MaxActive) {
      // the second if reuse is true, select from pool's connections
      if p.opt.Reuse {
         next := atomic.AddUint32(&p.index, 1) % uint32(current)
         return p.conns[next], nil
      }
      // the third create one-time connection
      c, err := p.opt.Dial(p.address)
      return p.wrapConn(c, true), err
   }

   // the fourth create new connections given back to pool
   p.Lock()
   current = atomic.LoadInt32(&p.current)
   if current < int32(p.opt.MaxActive) && nextRef > current*int32(p.opt.MaxConcurrentStreams) {
      // 2 times the incremental or the remain incremental  #
      increment := current
      if current+increment > int32(p.opt.MaxActive) {
         increment = int32(p.opt.MaxActive) - current
      }
      var i int32
      var err error
      for i = 0; i < increment; i++ {
         c, er := p.opt.Dial(p.address)
         if er != nil {
            err = er
            break
         }
         p.reset(int(current + i))
         p.conns[current+i] = p.wrapConn(c, false)
      }

      current += i
      log.Printf("#grow pool: %d ---> %d, increment: %d, maxActive: %d\n",
         p.current, current, increment, p.opt.MaxActive)
      atomic.StoreInt32(&p.current, current)
      if err != nil {
         p.Unlock()
         return nil, err
      }
   }
   p.Unlock()
   next := atomic.AddUint32(&p.index, 1) % uint32(current)
   return p.conns[next], nil
}
```

`Get` 代码逻辑

- 先增加连接的引用计数，如果在设定 `current*int32(p.opt.MaxConcurrentStreams)` 范围内，那么直接取连接进行使用即可。
- 若当前的连接数达到了最大活跃的连接数，那么就看我们新建池子的时候传递的 `option` 中的 `reuse` 参数是否是 `true`，若是复用，则随机取出连接池中的任意连接提供使用，如果不复用，则新建一个连接。
- 其余的情况，就需要我们进行 `2` 倍或者 `1` 倍的数量对连接池进行扩容了。

也可以在 `Get` 的实现上进行缩容，具体的缩容策略可以根据实际情况来定，例如当引用计数 `nextRef` 只有当前活跃连接数的 `10%` 的时候（这只是一个例子），就可以考虑缩容了。
