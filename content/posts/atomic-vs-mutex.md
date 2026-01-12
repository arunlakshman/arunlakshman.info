+++
date = '2026-01-09T21:00:00-00:00'
draft = false
title = "Atomic Integer vs Mutex in Go: Why You're Paying for a Lock You Don't Need"
+++

## TL;DR

- **Mutex (Pessimistic)**: Blocks all goroutines. It's safe but slow under contention.
- **Atomic (Optimistic)**: No blocking, retries on collision. Fast for simple operations.
- **Use Atomic when**: Single operation on a shared int/bool (counters, flags, gauges)
- **Use Mutex when**: Multiple operations must be atomic together, or failure/retry is unacceptable
- **Benchmark result**: Atomic significantly outperforms Mutex as goroutine count increases


You're building a high-throughput API server in Go. Requests are flooding in, and you need to count them across multiple goroutines. Simple enough, right? You reach for `sync.Mutex`, wrap your counter, and move on.

But here's the problem: Mutex is a pessimistic lock. Every goroutine waits in line, even for a single integer increment. Under high concurrency, this creates unnecessary contention and quietly kills your performance.

Is there a better way when your critical section is just one operation on a shared integer?

Yes. Use `atomic.Int64`. It leverages optimistic concurrency control, avoids blocking, and significantly outperforms Mutex for single-operation critical sections.

Let me show you why and when to make the switch.


## Understanding the Two Approaches: PCC vs OCC

Before diving into code, let's build intuition for *why* these two approaches behave differently.

### Pessimistic Concurrency Control (PCC): The Single-Occupancy Restroom

Picture this: You're at a busy concert venue, and there's only one restroom stall available. The door has a deadbolt lock that can only be locked from the inside. When someone enters, they turn the lock, and the door is sealed shut. Everyone else, whether they need a quick hand wash or a full stop, must wait patiently in a growing line outside, their time ticking away.

The second person in line watches as the first person spends 30 seconds inside. The third person watches both of them. By the time person number 10 gets their turn, they've been standing there for 5 minutes, even though their actual need takes mere seconds.

This is exactly how `sync.Mutex` works. A goroutine acquires the lock, does its work no matter how trivial or quick, and releases it. Every other goroutine blocks, suspended in memory, waiting for their turn in an orderly queue. The operating system wakes them up one by one, but they can't do anything until the lock is released, even for operations as simple as incrementing a counter by 1.

The mutex guarantees fairness: everyone gets their turn eventually. But this fairness comes at a cost. Under high contention, your goroutines spend more time waiting than working. The CPU cores sit idle while the lock holder performs a task that, in isolation, would take nanoseconds.

**Characteristics:**

- Safe: You get guaranteed exclusive access
- Never fails: You always get your turn
- Slow under load: Everyone waits, even for trivial operations

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

Now imagine a different scenario: You're at a bookstore during a Black Friday sale. There's a display table with exactly one copy of a bestselling novel left. You spot it just as another customer does. Both of you rush toward it, hands outstretched.

One of you grabs the book first. Success! The other person's hand closes on empty air. But here's the key difference: the "loser" doesn't stand there waiting for you to finish your purchase. Instead, they immediately check if the staff has restocked the display, and if they see another copy, they grab it right away. If not, they might check another section, try again, or move on. No queue, no waiting, no wasted time.

This is how atomic operations work. You attempt the operation optimistically, assuming no conflict will occur. You read the current value, compute the new value, and attempt to update it atomically. If someone else changed the value in the meantime, like when the book was already taken, your update fails silently. You simply retry with the new current value.

Most of the time, you succeed on the first try because collisions are rare. When collisions do happen, they're detected instantly at the hardware level, and the retry is nearly free. There's no blocking, no queue, no context switching. Your goroutines stay active, making progress on other work while occasionally retrying the operation.

**Characteristics:**

- Fast: No waiting, no blocking
- Can fail: Collisions happen
- Cheap retries: Failures are rare and recovery is instant

### A Third Example: The Concert Ticket Rush

Here's another scenario that highlights the difference: Imagine tickets for a sold-out concert go on sale at exactly 10:00 AM. Thousands of fans are waiting, fingers hovering over their keyboards, ready to claim the last 100 tickets.

With a Mutex approach, it's like having a single ticket window. Everyone lines up in a virtual queue. Person 1 buys their ticket, then person 2, then person 3, each transaction taking precious seconds. By the time person 50 gets to the window, the first 49 people have already purchased tickets, and those behind them are still waiting. The system processes tickets sequentially, one at a time, ensuring fairness but creating a bottleneck.

With an Atomic approach, it's like having a digital ticket board where everyone can see the available tickets simultaneously. When the clock strikes 10:00 AM, thousands of people click "Buy" at the exact same moment. The system processes all these requests in parallel. Some people successfully claim ticket #1, #2, #3... while others find their selected ticket was just claimed by someone else. But here's the crucial part: those who "missed" don't wait in line. They immediately try the next available ticket. Within milliseconds, all 100 tickets are claimed, and the system has handled thousands of simultaneous requests efficiently.

In the Mutex scenario, you might process 100 tickets in several minutes as people wait their turn. In the Atomic scenario, all 100 tickets are gone in seconds because everyone is working in parallel, with only the winners needing to retry when collisions occur.

### Why Retry Makes Sense

At first, "can fail" sounds worse than "never fails." But here's the thing:

- With Mutex, every goroutine pays the cost of acquiring and releasing a lock
- With Atomic, most operations succeed on the first try. Only collisions retry.

When your critical section is a single integer increment, collisions are rare and retries are nearly free. You're trading guaranteed but slow for almost always fast.

### How Atomic Achieves This: Compare-And-Swap (CAS)

Under the hood, atomic operations use a CPU instruction called Compare-And-Swap (CAS). A typical atomic increment works like this:

```go
for {
    old := atomic.LoadInt64(&value)
    new := old + 1
    if atomic.CompareAndSwapInt64(&value, old, new) {
        break // Success
    }
    // Someone else changed it, so we retry
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


## Proof: Benchmark Results

Theory is nice, but let's see the actual numbers. The complete benchmark code is available as a [GitHub Gist](https://gist.github.com/arunlakshman/2cedd03cd9513f7c80d986abb4355cb5). Here are the two counter implementations we're comparing:

### Mutex Counter (Pessimistic)

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

func (c *MutexCounter) Get() int64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

### Atomic Counter (Optimistic)

```go
type AtomicCounter struct {
    value int64
}

func (c *AtomicCounter) Increment() {
    atomic.AddInt64(&c.value, 1)
}

func (c *AtomicCounter) Get() int64 {
    return atomic.LoadInt64(&c.value)
}
```

### Benchmark Results

I ran the benchmark on a machine with 8 CPU cores, scaling from 1 to 64 goroutines. Each goroutine performed 1 million increments. Here are the results:

```text
Mutex vs Atomic Integer Benchmark
==================================
CPU Cores: 8
GOMAXPROCS: 8

Running benchmark with 1 goroutines, 1000000 operations per goroutine...

================================================================================
PERFORMANCE COMPARISON
================================================================================
Atomic is 1.99x faster than Mutex
Mutex Duration:   14.604ms
Atomic Duration:  7.328ms
Time Saved:       7.276ms

Mutex Ops/Second:  68473612
Atomic Ops/Second: 136464130
Difference:        67990518 ops/sec (99.3% improvement)
================================================================================

Running benchmark with 2 goroutines, 1000000 operations per goroutine...

================================================================================
PERFORMANCE COMPARISON
================================================================================
Atomic is 2.55x faster than Mutex
Mutex Duration:   85.616ms
Atomic Duration:  33.594ms
Time Saved:       52.022ms

Mutex Ops/Second:  23359999
Atomic Ops/Second: 59534265
Difference:        36174266 ops/sec (154.9% improvement)
================================================================================

Running benchmark with 4 goroutines, 1000000 operations per goroutine...

================================================================================
PERFORMANCE COMPARISON
================================================================================
Atomic is 2.86x faster than Mutex
Mutex Duration:   261.793ms
Atomic Duration:  91.635ms
Time Saved:       170.158ms

Mutex Ops/Second:  15279242
Atomic Ops/Second: 43651225
Difference:        28371983 ops/sec (185.7% improvement)
================================================================================

Running benchmark with 8 goroutines, 1000000 operations per goroutine...

================================================================================
PERFORMANCE COMPARISON
================================================================================
Atomic is 2.10x faster than Mutex
Mutex Duration:   666.549ms
Atomic Duration:  316.676ms
Time Saved:       349.873ms

Mutex Ops/Second:  12002121
Atomic Ops/Second: 25262398
Difference:        13260277 ops/sec (110.5% improvement)
================================================================================

Running benchmark with 16 goroutines, 1000000 operations per goroutine...

================================================================================
PERFORMANCE COMPARISON
================================================================================
Atomic is 2.26x faster than Mutex
Mutex Duration:   1.46708s
Atomic Duration:  649.424ms
Time Saved:       817.656ms

Mutex Ops/Second:  10906018
Atomic Ops/Second: 24637233
Difference:        13731215 ops/sec (125.9% improvement)
================================================================================

Running benchmark with 32 goroutines, 1000000 operations per goroutine...

================================================================================
PERFORMANCE COMPARISON
================================================================================
Atomic is 2.25x faster than Mutex
Mutex Duration:   2.889466s
Atomic Duration:  1.284447s
Time Saved:       1.605019s

Mutex Ops/Second:  11074710
Atomic Ops/Second: 24913450
Difference:        13838740 ops/sec (125.0% improvement)
================================================================================

Running benchmark with 64 goroutines, 1000000 operations per goroutine...

================================================================================
PERFORMANCE COMPARISON
================================================================================
Atomic is 2.33x faster than Mutex
Mutex Duration:   5.807583s
Atomic Duration:  2.492188s
Time Saved:       3.315395s

Mutex Ops/Second:  11020075
Atomic Ops/Second: 25680243
Difference:        14660168 ops/sec (133.0% improvement)
================================================================================

================================================================================
Method   | Goroutines | Total Ops | Duration   | Ops/Second | Final Value
--------------------------------------------------------------------------------
Mutex    | 1          | 1000000   | 14.604ms   | 68473612   | 1000000
Atomic   | 1          | 1000000   | 7.328ms    | 136464130  | 1000000
Mutex    | 2          | 2000000   | 85.616ms   | 23359999   | 2000000
Atomic   | 2          | 2000000   | 33.594ms   | 59534265   | 2000000
Mutex    | 4          | 4000000   | 261.793ms  | 15279242   | 4000000
Atomic   | 4          | 4000000   | 91.635ms   | 43651225   | 4000000
Mutex    | 8          | 8000000   | 666.549ms  | 12002121   | 8000000
Atomic   | 8          | 8000000   | 316.676ms  | 25262398   | 8000000
Mutex    | 16         | 16000000  | 1.46708s   | 10906018   | 16000000
Atomic   | 16         | 16000000  | 649.424ms  | 24637233   | 16000000
Mutex    | 32         | 32000000  | 2.889466s  | 11074710   | 32000000
Atomic   | 32         | 32000000  | 1.284447s  | 24913450   | 32000000
Mutex    | 64         | 64000000  | 5.807583s  | 11020075   | 64000000
Atomic   | 64         | 64000000  | 2.492188s  | 25680243   | 64000000
================================================================================

Benchmark completed!
```

**Key Observations:**

1. **Atomic maintains consistent performance** (around 25M ops/sec) as goroutine count increases, while Mutex throughput degrades significantly (from 68M to 11M ops/sec).

2. **The performance gap widens with concurrency**: At 1 goroutine, Atomic is about 2x faster. At 64 goroutines, it's still about 2.3x faster, but more importantly, Atomic delivers **2.3x the throughput** under high contention.

3. **Mutex contention costs**: Mutex performance drops dramatically with more goroutines due to lock contention. Goroutines spend time waiting instead of working.

The benchmark clearly shows that for simple counter operations, Atomic provides significant performance advantages, especially under high concurrency.


## When to Use Which

Here's the decision framework:

### Use Atomic When

- The critical section is a **single operation** on a shared `int`, `int64`, `uint64`, or `bool`
- Retry is cheap and acceptable
- You need maximum throughput for simple counters, flags, or gauges

### Use Mutex When

**Multiple operations must be atomic together:**

- Rate limiter with time window: check count and timestamp, conditionally reset, then increment
- Circuit breaker: increment failure count and check threshold and flip state
- Connection pool: decrement count and remove from available list
- User session: update `lastActiveTime` and increment `requestCount`
- Inventory with reorder: decrement stock and check threshold and set `reorderFlag`

**Failure and retry are never acceptable:**

- Financial ledger entries: double-charging or missed credits are catastrophic
- Audit logs with sequence numbers: gaps break compliance
- Distributed lock with lease: partial success corrupts the system

**Operations on complex data structures:**

- Maps, slices, or any composite type that can't be updated atomically


If your critical section is a single integer operation, you're paying for a lock you don't need. `sync.Mutex` is the safe default, but safety has a cost. For simple counters, flags, and gauges under high concurrency, `atomic.Int64` gives you the same correctness with significantly better performance.

Run the benchmark yourself, see the numbers, and then make the switch where it makes sense.

---

## Appendix: Other Atomic-Friendly Scenarios

Beyond request counters, here are other cases where Atomic shines:

- **Active connection count**: increment on connect, decrement on disconnect
- **Graceful shutdown flag**: a boolean that signals all goroutines to stop
- **Rate limiter tokens**: decrement available tokens per request
- **Unique ID or sequence generator**: fetch and increment for ID generation
- **Circuit breaker failure count**: increment on failure, reset on success
- **Metrics collectors**: counting errors, cache hits, page views

If the pattern is "read-modify-write on a single primitive," Atomic is your friend.
