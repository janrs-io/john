---
title: "Building a gRPC Connection Pool in Go"
author: "John Yang"
lang: en
pubDatetime: 2023-03-02T10:39:00Z
featured: true
tags:
  - Go
  - gRPC
  - Microservices
description: "Why naive per-request gRPC dialing fails under high concurrency, and how to design a thread-safe, dynamically scaling gRPC client connection pool in Go."
---

## The Problem with Naive gRPC Usage

`gRPC` supports four core communication patterns:
- Unary RPC (request-response)
- Client-side streaming
- Server-side streaming
- Bidirectional streaming

A common way to dial a gRPC service in Go looks like this:

```go
func main() {
	// ... some code
	conn, err := grpc.Dial(":8000", grpc.WithInsecure())
	if err != nil {
		// ... log
	}
	defer conn.Close()
	// ... some code
}
```

Under high concurrency, however, dialing a new connection per request quickly leads to performance degradation and errors during load testing:

```text
rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: connection error: desc = "transport: Error while dialing dial tcp xxx:xxx: connect: connection refused"
```

In practice, this happens because available sockets/connections are exhausted before prior connections are fully released.

Since `gRPC` runs over `TCP`, every new connection incurs a TCP three-way handshake on setup and a four-way handshake on teardown. Repeatedly opening and closing connections wastes CPU, memory, and file descriptors. While the gRPC server handles inbound connection management automatically, **client-side connection reuse** is essential for high-throughput microservices.

---

## Designing the Connection Pool

When building a gRPC client pool, consider three key questions:
- Does the pool support dynamic scaling (growing/shrinking)?
- Are idle connections closed after a timeout or kept alive?
- What happens when the pool reaches `MaxActive` capacity?

### Pool Interface

```go
type Pool interface {
	// Get retrieves a connection from the pool; calling Close() returns it to the pool.
	Get() (Conn, error)

	// Close shuts down the pool and releases all underlying connections.
	Close() error

	// Status returns a human-readable snapshot of pool state.
	Status() string
}
```

### Initializing & Closing the Pool

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
		return nil, errors.New("invalid maximum concurrent streams")
	}

	p := &pool{
		index:   0,
		current: int32(option.MaxIdle),
		ref:     0,
		opt:     option,
		conns:   make([]*conn, option.MaxActive),
		address: address,
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

If any initial dial fails inside `New`, `p.Close()` cleans up all previously opened connections so nothing leaks.

```go
func (p *pool) Close() error {
	atomic.StoreInt32(&p.closed, 1)
	atomic.StoreUint32(&p.index, 0)
	atomic.StoreInt32(&p.current, 0)
	atomic.StoreInt32(&p.ref, 0)
	p.deleteFrom(0)
	log.Printf("close pool success: %v\n", p.Status())
	return nil
}

func (p *pool) deleteFrom(begin int) {
	for i := begin; i < p.opt.MaxActive; i++ {
		p.reset(i)
	}
}

func (p *pool) reset(index int) {
	conn := p.conns[index]
	if conn == nil {
		return
	}
	conn.reset()
	p.conns[index] = nil
}
```

### Returning vs. Closing a Connection

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

When a caller borrows a connection via `pool.Get()` and later calls `conn.Close()`, pooled connections (`once == false`) simply decrement the reference counter and remain open in the pool, whereas overflow one-time connections (`once == true`) are closed immediately.

---

## Dynamic Scaling in `Get()`

```go
func (p *pool) Get() (Conn, error) {
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

	// Pool has reached MaxActive
	if current == int32(p.opt.MaxActive) {
		if p.opt.Reuse {
			next := atomic.AddUint32(&p.index, 1) % uint32(current)
			return p.conns[next], nil
		}
		c, err := p.opt.Dial(p.address)
		return p.wrapConn(c, true), err
	}

	// Grow the pool up to 2x current size (capped at MaxActive)
	p.Lock()
	current = atomic.LoadInt32(&p.current)
	if current < int32(p.opt.MaxActive) && nextRef > current*int32(p.opt.MaxConcurrentStreams) {
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

How `Get()` handles concurrency:
1. Increments the reference counter; if within `current * MaxConcurrentStreams`, round-robins across existing pooled connections lock-free.
2. If the pool is already at `MaxActive`, either reuses an existing connection (`Reuse: true`) or dials a one-off connection (`once: true`) that closes on `Close()`.
3. Otherwise, acquires the write lock and doubles the active pool size (capped at `MaxActive`). Shrinking can similarly be triggered when `nextRef` drops below a threshold (e.g., 10% of active capacity).
