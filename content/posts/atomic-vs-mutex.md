+++
date = '2026-01-09T21:00:00-00:00'
draft = false
title = "Atomic Integer vs Mutex in Go: Why You're Paying for a Lock You Don't Need"
+++

## TL;DR

- **Mutex (Pessimistic)**: Blocks all goroutines—safe but slow under contention
- **Atomic (Optimistic)**: No blocking, retries on collision—fast for simple operations
- **Use Atomic when**: Single operation on a shared int/bool (counters, flags, gauges)
- **Use Mutex when**: Multiple operations must be atomic together, or failure/retry is unacceptable
- **Benchmark result**: Atomic significantly outperforms Mutex as goroutine count increases

---

You're building a high-throughput API server in Go. Requests are flooding in, and you need to count them across multiple goroutines. Simple enough—you reach for `sync.Mutex`, wrap your counter, and move on.

But here's the problem: Mutex is a pessimistic lock. Every goroutine waits in line, even for a single integer increment. Under high concurrency, this creates unnecessary contention and quietly kills your performance.

Is there a better way when your critical section is just one operation on a shared integer?

Yes. Use `atomic.Int64`. It leverages optimistic concurrency control, avoids blocking, and significantly outperforms Mutex for single-operation critical sections.

Let me show you why—and when—to make the switch.

---

## Understanding the Two Approaches: PCC vs OCC

Before diving into code, let's build intuition for *why* these two approaches behave differently.

### Pessimistic Concurrency Control (PCC): The McDonald's Counter

Imagine a McDonald's where you take a token number and wait for your turn. Only one customer is served at a time. You're guaranteed service when your number is called—but everyone waits in line, even if the counter is free.

This is how `sync.Mutex` works. A goroutine acquires the lock, does its work, and releases it. Everyone else blocks until it's their turn.

**Characteristics:**
- Safe: guaranteed exclusive access
- Never fails: you always get your turn
- Slow under load: everyone waits, even for trivial operations

```go
type MutexCounter struct {
    mu    sync.Mutex
    value int64
}

func (c *MutexCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}
```

### Optimistic Concurrency Control (OCC): Grabbing the Last Item on a Shelf

Now imagine a store shelf with one item left. Two shoppers reach for it simultaneously. One grabs it; the other's hand closes on air. The "loser" doesn't wait in line—they check if more stock arrived and try again.

This is how atomic operations work. You attempt the operation optimistically, assuming no conflict. If someone else changed the value, you retry.

**Characteristics:**
- Fast: no waiting, no blocking
- Can fail: collisions happen
- Cheap retries: failures are rare and recovery is instant

### Why Retry Makes Sense

At first, "can fail" sounds worse than "never fails." But consider the math:

- With Mutex, *every* goroutine pays the cost of acquiring and releasing a lock
- With Atomic, *most* operations succeed on the first try; only collisions retry

When your critical section is a single integer increment, collisions are rare and retries are nearly free. You're trading guaranteed-but-slow for almost-always-fast.

### How Atomic Achieves This: Compare-And-Swap (CAS)

Under the hood, atomic operations use a CPU instruction called Compare-And-Swap (CAS). Here's the logic:

```go
func CompareAndSwap(addr *int64, old, new int64) bool {
    // Atomically:
    // if *addr == old {
    //     *addr = new
    //     return true
    // }
    // return false
}
```

A typical atomic increment works like this:

```go
for {
    old := atomic.LoadInt64(&value)
    new := old + 1
    if atomic.CompareAndSwapInt64(&value, old, new) {
        break // Success
    }
    // Someone else changed it—retry
}
```

Go's `atomic.AddInt64` does this for you in a single, optimized call:

```go
type AtomicCounter struct {
    value int64
}

func (c *AtomicCounter) Increment() {
    atomic.AddInt64(&c.value, 1)
}
```

---

## Proof: Benchmark Results

Theory is nice, but let's see the numbers. Here's a complete benchmark you can run yourself:

<script src="https://gist.github.com/arunlakshman/2cedd03cd9513f7c80d986abb4355cb5.js"></script>

The benchmark scales from 1 to 64 goroutines, showing you exactly where Atomic pulls ahead.

---

## When to Use Which

Here's the decision framework:

### Use Atomic When:

- The critical section is a **single operation** on a shared `int`, `int64`, `uint64`, or `bool`
- Retry is cheap and acceptable
- You need maximum throughput for simple counters, flags, or gauges

### Use Mutex When:

**Multiple operations must be atomic together:**
- Rate limiter with time window: check count AND timestamp, conditionally reset, then increment
- Circuit breaker: increment failure count AND check threshold AND flip state
- Connection pool: decrement count AND remove from available list
- User session: update `lastActiveTime` AND increment `requestCount`
- Inventory with reorder: decrement stock AND check threshold AND set `reorderFlag`

**Failure and retry are never acceptable:**
- Financial ledger entries: double-charging or missed credits are catastrophic
- Audit logs with sequence numbers: gaps break compliance
- Distributed lock with lease: partial success corrupts the system

**Operations on complex data structures:**
- Maps, slices, or any composite type that can't be updated atomically

---

## The Bottom Line

If your critical section is a single integer operation, you're paying for a lock you don't need. `sync.Mutex` is the safe default, but safety has a cost. For simple counters, flags, and gauges under high concurrency, `atomic.Int64` gives you the same correctness with significantly better performance.

Run the benchmark yourself. See the numbers. Then make the switch where it makes sense.

---

## Appendix: Other Atomic-Friendly Scenarios

Beyond request counters, here are other cases where Atomic shines:

- **Active connection count**: increment on connect, decrement on disconnect
- **Graceful shutdown flag**: a boolean that signals all goroutines to stop
- **Rate limiter tokens**: decrement available tokens per request
- **Unique ID / sequence generator**: fetch-and-increment for ID generation
- **Circuit breaker failure count**: increment on failure, reset on success
- **Metrics collectors**: counting errors, cache hits, page views

If the pattern is "read-modify-write on a single primitive," Atomic is your friend.
