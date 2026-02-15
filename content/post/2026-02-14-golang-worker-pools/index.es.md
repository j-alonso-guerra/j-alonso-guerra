---
title: "Dominando Worker Pools en Go: Guía Práctica"
date: 2026-02-14 10:00:00 -0500
description: >-
  Aprende a construir worker pools eficientes en Go usando goroutines y channels.
  Desde lo básico hasta patrones listos para producción con graceful shutdown, manejo de errores y escalado dinámico.
categories:
  - Golang
  - Concurrencia
tags:
  - golang
  - worker-pools
  - concurrencia
  - channels
  - goroutines
---

Si has trabajado con Go por algún tiempo, probablemente llegaste al punto donde lanzar una goroutine por tarea empieza a sentirse mal. Quizás estás golpeando rate limits de APIs, agotando conexiones de base de datos, o simplemente viendo cómo tu uso de memoria escala sin control. Ahí es donde entran los **worker pools**.

Un worker pool es un patrón de concurrencia donde un número fijo de goroutines (workers) procesan tareas desde una cola compartida. Te da **paralelismo controlado** — la capacidad de hacer muchas cosas a la vez sin hacer *todo* a la vez.

## ¿Por qué Worker Pools?

Considera este enfoque ingenuo:

```go
for _, url := range urls {
    go fetch(url) // 10,000 URLs = 10,000 goroutines
}
```

Esto funciona hasta que deja de hacerlo. Con 10,000 URLs abrirás 10,000 conexiones simultáneas, probablemente recibiendo rate-limit, quedándote sin file descriptors, o tumbando el servidor destino.

Un worker pool resuelve esto limitando la concurrencia:

```text
           ┌──────────┐
  Tareas ─>│  Channel  │──> Worker 1 ──> Resultados
           │  (Cola)   │──> Worker 2 ──> Resultados
           │           │──> Worker 3 ──> Resultados
           └──────────┘
```

Workers fijos, throughput controlado, uso de recursos predecible.

## El Patrón Básico

El worker pool más simple en Go:

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        fmt.Printf("Worker %d procesando job %d\n", id, job)
        results <- job * 2
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10

    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)

    var wg sync.WaitGroup

    // Iniciar workers
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go worker(i, jobs, results, &wg)
    }

    // Enviar jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs)

    // Esperar y cerrar resultados
    go func() {
        wg.Wait()
        close(results)
    }()

    // Recolectar resultados
    for result := range results {
        fmt.Println("Resultado:", result)
    }
}
```

Desglosemos las piezas clave:

1. **Canal `jobs`** — la cola. Los workers leen de él, el productor escribe en él.
2. **Canal `results`** — donde los workers envían su output.
3. **`sync.WaitGroup`** — rastrea cuándo todos los workers terminaron.
4. **`close(jobs)`** — señala a los workers que no hay más trabajo. El loop `range` termina.

Esta es la base. Todo lo demás se construye sobre esto.

## Agregando Manejo de Errores

Las tareas del mundo real fallan. Necesitas una forma de propagar errores de vuelta sin perderlos:

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

Ahora el consumidor puede verificar cada resultado:

```go
for r := range results {
    if r.Err != nil {
        log.Printf("job falló: %v", r.Err)
        continue
    }
    fmt.Println("Éxito:", r.Value)
}
```

## Graceful Shutdown con Context

En producción, necesitas detener workers limpiamente — ante SIGTERM, timeout, o cuando ocurre un error crítico. `context.Context` es la forma estándar:

```go
func worker(ctx context.Context, id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d: apagando\n", id)
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

El caller controla el ciclo de vida:

```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

// Ante señal de interrupción
go func() {
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh
    cancel()
}()
```

Cuando se llama `cancel()`, el `ctx.Done()` de cada worker se dispara y salen limpiamente. Sin goroutines fugadas, sin trabajo a medio procesar.

## Un Pool Listo para Producción

Juntemos todo en una estructura reutilizable:

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

El uso es limpio:

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

## Cuándo Usar Worker Pools

| Escenario | Tamaño sugerido del Pool |
|---|---|
| Llamadas HTTP con rate limits | Igualar el rate limit |
| Operaciones batch de base de datos | Igualar el connection pool |
| Procesamiento CPU-bound (resize de imágenes, hashing) | `runtime.NumCPU()` |
| Operaciones de File I/O | 2x-4x conteo de CPUs |
| Cargas mixtas | Perfilar y ajustar |

## Conclusiones Clave

- **No lances goroutines sin límite** — usa un pool para controlar la concurrencia.
- **Siempre usa `context.Context`** — es el mecanismo estándar de cancelación en Go.
- **Channels con buffer** para la cola de jobs evitan que el productor se bloquee.
- **`sync.WaitGroup` + `close()`** es la forma más limpia de señalar completitud.
- **Generics** (Go 1.18+) hacen que los pools reutilizables sean prácticos sin casting de `interface{}`.

Los worker pools son uno de esos patrones que parecen simples pero desbloquean poder serio cuando construyes sistemas a escala. Empieza con el patrón básico, agrega manejo de errores, conecta context para graceful shutdown, y tienes un building block listo para producción.

---

*¿Te fue útil? Revisa mis otros posts sobre patrones de concurrencia en Go.*
