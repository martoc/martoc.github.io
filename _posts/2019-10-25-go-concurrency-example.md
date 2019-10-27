---
title: A simple Go program that uses concurrency
subtitle: Go Routines and Channels
layout: post
author: martoc
image: https://martoc.london/blog/images/go-concurrency-example-v1.0.png
---

Go supports multiple concurrency models, this service shows how to implement the
actor model using go functions and channels, the implementation parallelises 2
long running operations, these calls to external services are mocked with
different random delays (5-10 ms).

![Service](/blog/images/go-concurrency-example-v1.0.png)
