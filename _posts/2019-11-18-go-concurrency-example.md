---
title: A simple Go program that uses concurrency
subtitle: Go Routines and Channels
layout: post
author: martoc
image: https://martoc.github.io/blog/images/golang.jpg
---

Go supports multiple concurrency models, this diagram shows what actually want
to implement, it parallelises 3 long running operations, generally these are
calls to external services, for testing purposes these external call are mocked
with different random delays (1-50 ms) and there is a restriction the whole
service execution time cannot last more than 20ms.

![Service](/blog/images/go-concurrency-example-v1.0.png)

In online advertising speed is everything and having all the information is
secondary, this proof of concept tries to check if concurrent operations can be
forked and joined and if any of these operations excess more than a cap value
the execution carries on and omits these slow operations. I've created 3
datasets to tests this, each data set contains 1000 requests and the latency
for the different operations, the `requests-lowlatency.csv` the operations
last between 1-3 ms, the next file is `requests-highlatency.csv` the latency in
this file is set between 1-10 ms and finally the `requests-veryhighlatency.csv`
in this file the latency is set between 1-50 ms (most of the operations time
out).

The `main`, `LoadRequests` functions as well as `Request` and the
`avgExecutionRequest` array are only required for the execution of this proof of
concept

```go
package main

import (
	"encoding/csv"
	"fmt"
	"os"
	"strconv"
	"time"
)

type Request struct {
	Id       string
	LatencyC int
	LatencyA int
	LatencyB int
	ResultA string
	ResultB string
	ResultC string
}

var avgExecutionRequest [1000]int64

func LoadRequests(filename string) {
	f, _ := os.Open(filename)
	defer f.Close()
	reader := csv.NewReader(f)
	for row, err := reader.Read(); err == nil; row, err = reader.Read() {
		latencyA, _ := strconv.Atoi(row[1])
		latencyB, _ := strconv.Atoi(row[2])
		latencyC, _ := strconv.Atoi(row[3])
		request := &Request{
			Id:       row[0],
			LatencyC: latencyC,
			LatencyA: latencyA,
			LatencyB: latencyB,
		}
		go Execute(request)
	}
}

func IsClosed(ch <-chan string) bool {
	select {
	case <-ch:
		return true
	default:
	}

	return false
}

func Execute(request *Request) {
	t1 := time.Now()
	channelA := make(chan string)
	channelB := make(chan string)
	channelC := make(chan string)
	defer close(channelA)
	defer close(channelB)
	defer close(channelC)
	go func(req Request) {
		time.Sleep(time.Duration(req.LatencyA) * time.Millisecond)
		if !IsClosed(channelA) {
			channelA <- "Result Operation A"
		}
	}(*request)
	go func(req Request) {
		time.Sleep(time.Duration(req.LatencyB) * time.Millisecond)
		if !IsClosed(channelB) {
			channelB <- "Result Operation B"
		}
	}(*request)
	go func(req Request) {
		time.Sleep(time.Duration(req.LatencyC) * time.Millisecond)
		if !IsClosed(channelC) {
			channelB <- "Result Operation C"
		}
	}(*request)
	for i := 0; i < 3; i++ {
		select {
		case resultA := <-channelA:
			request.ResultA = resultA
		case resultB := <-channelB:
			request.ResultB = resultB
		case resultC := <-channelC:
			request.ResultC = resultC
		case <-time.After(20 * time.Millisecond):
			break
		}
	}
	t2 := time.Now()
	diff := t2.Sub(t1)
	idx, _ := strconv.Atoi(request.Id)
	avgExecutionRequest[idx - 1] = diff.Milliseconds()
}


func main() {
	LoadRequests("requests-lowlatency.csv")
	fmt.Print("Press Enter to finish...")
	fmt.Scanln()
	total := int64(0)
	for _, v := range avgExecutionRequest {
		total += v
	}
	fmt.Printf("Average latency: %2.2fms\n", float64(total/1000))
}
```

These are the observations of my tests

* When the low latency file is used, the average execution time is ~2ms.
* When the high latency file is used, the average execution time is ~7ms.
* When the very high latency file is used, the average execution time is ~20ms.

With this simple model, it would be possible to build services that can fork,
join and omit slow operations. My next step will be to integrate this with the
http package to create another proof of concept for a low latency web service or
API. This project will evolve <https://github.com/martoc/go-concurrency-example/blob/master/main.go>
