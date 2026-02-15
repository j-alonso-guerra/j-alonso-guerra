---
title: "Mastering Worker Pools in Go: A Practical Guide"
date: 2026-02-14 10:00:00 -0500
description: >-
  Learn how to build efficient worker pools in Go using goroutines and channels.
  From the basics to production-ready patterns with graceful shutdown, error handling, and dynamic scaling.
categories:
  - Golang
  - Concurrency
tags:
  - golang
  - worker-pools
  - concurrency
  - channels
  - goroutines
---

If you've worked with Go for any amount of time, you've probably reached a point where spinning up a goroutine per task starts to feel wrong. Maybe you're hitting API rate limits, exhausting database connections, or just watching your memory usage climb. That's where **worker pools** come in.

A worker pool is a concurrency pattern where a fixed number of goroutines (workers) process tasks from a shared queue. It gives you **controlled parallelism** — the ability to do many things at once without doing *everything* at once.

## Why Worker Pools?

Consider this naive approach:

```go
for _, url := range urls {
    go fetch(url) // 10,000 URLs = 10,000 goroutines
}
```

This works until it doesn't. With 10,000 URLs you'll open 10,000 connections simultaneously, likely getting rate-limited, running out of file descriptors, or crashing the target server.

A worker pool solves this by limiting concurrency:

```text
           ┌──────────┐
  Tasks ──>│  Channel  │──> Worker 1 ──> Results
           │  (Queue)  │──> Worker 2 ──> Results
           │           │──> Worker 3 ──> Results
           └──────────┘
```

Fixed workers, controlled throughput, predictable resource usage.

## The Basic Pattern

Here's the simplest worker pool in Go:

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        results <- job * 2
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10

    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)

    var wg sync.WaitGroup

    // Start workers
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go worker(i, jobs, results, &wg)
    }

    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs)

    // Wait and close results
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    for result := range results {
        fmt.Println("Result:", result)
    }
}
```

Let's break down the key pieces:

1. **`jobs` channel** — the queue. Workers read from it, the producer writes to it.
2. **`results` channel** — where workers send their output.
3. **`sync.WaitGroup`** — tracks when all workers are done.
4. **`close(jobs)`** — signals workers there's no more work. The `range` loop exits.

This is the foundation. Everything else builds on top of it.

## Adding Error Handling

Real-world tasks fail. You need a way to propagate errors back without losing them:

```go
type Result struct {
    Value int
    Err   error
}

func worker(id int, jobs <-chan int, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        val, err := process(job)
        results <- Result{Value: val, Err: err}
    }
}
```

Now the consumer can check each result:

```go
for r := range results {
    if r.Err != nil {
        log.Printf("job failed: %v", r.Err)
        continue
    }
    fmt.Println("Success:", r.Value)
}
```

## Graceful Shutdown with Context

In production, you need to stop workers cleanly — on SIGTERM, on timeout, or when a critical error occurs. `context.Context` is the standard way:

```go
func worker(ctx context.Context, id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d: shutting down\n", id)
            return
        case job, ok := <-jobs:
            if !ok {
                return
            }
            results <- process(job)
        }
    }
}
```

The caller controls the lifecycle:

```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

// On interrupt signal
go func() {
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh
    cancel()
}()
```

When `cancel()` is called, every worker's `ctx.Done()` fires and they exit gracefully. No leaked goroutines, no half-processed work.

## A Production-Ready Pool

Let's put it all together into a reusable structure:

```go
package pool

import (
    "context"
    "sync"
)

type Task[T any, R any] struct {
    Payload T
}

type Result[R any] struct {
    Value R
    Err   error
}

type Pool[T any, R any] struct {
    workers int
    handler func(context.Context, T) (R, error)
    jobs    chan Task[T, R]
    results chan Result[R]
}

func New[T any, R any](workers, queueSize int, handler func(context.Context, T) (R, error)) *Pool[T, R] {
    return &Pool[T, R]{
        workers: workers,
        handler: handler,
        jobs:    make(chan Task[T, R], queueSize),
        results: make(chan Result[R], queueSize),
    }
}

func (p *Pool[T, R]) Start(ctx context.Context) {
    var wg sync.WaitGroup
    for i := 0; i < p.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case task, ok := <-p.jobs:
                    if !ok {
                        return
                    }
                    val, err := p.handler(ctx, task.Payload)
                    p.results <- Result[R]{Value: val, Err: err}
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(p.results)
    }()
}

func (p *Pool[T, R]) Submit(task Task[T, R]) {
    p.jobs <- task
}

func (p *Pool[T, R]) Close() {
    close(p.jobs)
}

func (p *Pool[T, R]) Results() <-chan Result[R] {
    return p.results
}
```

Usage is clean:

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    p := pool.New[string, int](5, 100, func(ctx context.Context, url string) (int, error) {
        resp, err := http.Get(url)
        if err != nil {
            return 0, err
        }
        defer resp.Body.Close()
        return resp.StatusCode, nil
    })

    p.Start(ctx)

    urls := []string{"https://go.dev", "https://github.com", "https://google.com"}
    for _, u := range urls {
        p.Submit(pool.Task[string, int]{Payload: u})
    }
    p.Close()

    for r := range p.Results() {
        if r.Err != nil {
            log.Printf("Error: %v", r.Err)
            continue
        }
        fmt.Printf("Status: %d\n", r.Value)
    }
}
```

## When to Use Worker Pools

| Scenario | Pool Size Hint |
|---|---|
| HTTP API calls with rate limits | Match the rate limit |
| Database batch operations | Match connection pool size |
| CPU-bound processing (image resize, hashing) | `runtime.NumCPU()` |
| File I/O operations | 2x-4x CPU count |
| Mixed workloads | Profile and tune |

## Key Takeaways

- **Don't spawn unbounded goroutines** — use a pool to control concurrency.
- **Always use `context.Context`** — it's the standard cancellation mechanism in Go.
- **Buffered channels** for the job queue prevent the producer from blocking.
- **`sync.WaitGroup` + `close()`** is the cleanest way to signal completion.
- **Generics** (Go 1.18+) make reusable pools practical without `interface{}` casting.

Worker pools are one of those patterns that look simple but unlock serious power when building systems at scale. Start with the basic pattern, add error handling, wire in context for graceful shutdown, and you've got a production-ready building block.

---

*Found this useful? Check out my other posts on Go concurrency patterns.*
