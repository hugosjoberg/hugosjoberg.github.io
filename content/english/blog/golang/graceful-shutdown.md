---
title: "Graceful shutdown of server in Go"
meta_title: ""
description: "Graceful shutdown of server in Go"
date: 2023-09-18T05:00:00Z
categories: ["golang", "server", "shutdown", "concurrency"]
author: "Hugo Sjoberg"
tags: ["golang", "concurrency", "shutdown", "server"]
draft: false
---

# Graceful shutdown

Graceful shutdown refers to shutting down an application or service in a way that allows it to finish any ongoing tasks or transactions, clean up resources, and exit in an orderly and controlled manner. This is important to ensure that the application does not leave any unfinished work, corrupt data, or cause disruptions when it is terminated.

For example, we might be running a service in Kubernetes, and we are experiencing lower traffic so we would naturally scale down the number of pods. The load balancer will not redirect traffic to pods that are in a terminated state.

Now our pods cannot just be terminated, since they might handle a long-running connection with a client(slow database query for example). Let's see how we can gracefully terminate a server in golang.

Golang std-lib provides this neat little package called [errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)

> Package errgroup provides synchronization, error propagation, and Context cancelation for groups of goroutines working on subtasks of a common task.

The context derived from errgroup got this nice property

> The derived Context is canceled the first time a function passed to Go returns a non-nil error or the first time Wait returns, whichever occurs first.

So whenever one of the goroutines encountered an error this context will cancel.

Now that sounds pretty neat, let's take errgroup out on a little spin and create a server that gracefully exists whenever a termination signal(CTRL+C) comes in.

```golang
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"

    "golang.org/x/sync/errgroup"
)

func main() {
    ctxWithCancel, cancel := context.WithCancel(context.Background())
    defer cancel()
    g, ctx := errgroup.WithContext(ctxWithCancel)

    srv := &http.Server{Addr: fmt.Sprintf("0.0.0.0:%d", 8080)}

    g.Go(func() error {
        fmt.Println("starting server")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            return err
        }
        return nil
    })

    g.Go(func() error {
        <-ctx.Done()
        shutDownCTX, cancel := context.WithTimeout(ctx, 1*time.Second)
        if err := srv.Shutdown(context.Background()); err != nil {
            fmt.Println("encountered errors when shutting down")
        }
        fmt.Println("server shut down")
        return nil
    })

    g.Go(func() error {
        signals := make(chan os.Signal, 1)
        signal.Notify(signals, syscall.SIGINT, syscall.SIGTERM)
        <-signals
        fmt.Println("terminating...")
        cancel()
        return nil
    })

    err := g.Wait()
    if err != nil {
        fmt.Println(err)
    }
}
```

Let's run this to see what happens:
```bash
> go main.go
starting server
terminating... <- CTRL+C to terminate
server shut down
```

Using errgroup we create a couple of goroutines:
- goroutine for running the server
- goroutine for shutting down the server, it's waiting for the context `ctx` to be done and then shuts down the server
- goroutine to check for any kind of termination signal from the OS
