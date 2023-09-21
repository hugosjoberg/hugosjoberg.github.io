---
title: "Singleflight"
meta_title: ""
description: "Singleflight in Golang"
date: 2022-04-04T05:00:00Z
image: "/images/singleflight.png"
categories: ["golang", "concurrency"]
author: "Hugo Sjoberg"
tags: ["golang", "concurrency"]
draft: false
---

# Why Singleflight?

Imagine you are running a service that calls a slow operation, let's say it makes an expensive query to a database. Now this particular service is a popular one, before the first query even returned a result 10 more identical requests were made.

Wouldn't it be nice if we didn't have to call the database 10 times but instead return the data we get back from the first request to all other requests?

Well, this is usually where caching comes in, but let's be a little bit more experimental and give the package `singleflight` a swing.

This is how godoc describes singleflight:

> Package singleflight provides a duplicate function call suppression mechanism.

Alright sounds promising, the `singleflight` package in Go is a utility for managing duplicate function calls concurrently and efficiently. It helps ensure that a given function is executed only once even if multiple goroutines attempt to invoke it simultaneously.

```golang
package main

import (
    "fmt"
    "net/http"
    "time"

    "golang.org/x/sync/singleflight"
)

var (
    counter = 0
    group   singleflight.Group
)

type Response struct {
    Message string
}

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        result, err, _ := group.Do("unique-key", func() (interface{}, error) {
            val := incCounter()
            return &Response{Message: fmt.Sprintf("called %d times", val)}, nil
        })
        if err != nil {
            fmt.Fprintf(w, "error %s", err.Error())
        }
        fmt.Fprintf(w, result.(*Response).Message)
    })

    http.ListenAndServe(":8080", nil)
}

func incCounter() int {
    time.Sleep(10 * time.Second)
    counter = counter + 1
    return counter
}
```

Let's run this to see what happens:
```bash
> go run server.go
> curl localhost:8080/ <- New terminal window
> called 1 times
> curl localhost:8080/ <- New terminal window
> called 1 times
```

As we can see by this example the slow function `incCounter` is only invoked one time :rocket:

`group.Do` accepts a string that serves as an identifier or similar for a call, the third returned value indicated is the returned value shared.

{{< notice "tip" >}}
The shared value is important if the returned value is a pointer then we might have to handle that to ensure we don't run into nasty race conditions
{{< /notice >}}
