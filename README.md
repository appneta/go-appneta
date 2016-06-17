# TraceView for Go

[![GoDoc](https://godoc.org/github.com/appneta/go-appneta/v1/tv?status.svg)](https://godoc.org/github.com/appneta/go-appneta/v1/tv)
[![Build Status](https://travis-ci.org/appneta/go-appneta.svg?branch=master)](https://travis-ci.org/appneta/go-appneta)
[![Coverage Status](https://coveralls.io/repos/github/appneta/go-appneta/badge.svg?branch=master)](https://coveralls.io/github/appneta/go-appneta?branch=master)
[![codecov](https://codecov.io/gh/appneta/go-appneta/branch/master/graph/badge.svg)](https://codecov.io/gh/appneta/go-appneta)

* [Introduction](#introduction)
* [Getting started](#getting-started)
    - [Installing](#installing)
    - [Building your app with tracing](#building-your-app-with-tracing)
* [Instrumenting your application](#instrumenting-your-application)
    - [Usage examples](#usage-examples)
    - [Distributed tracing and context propagation](#distributed-tracing-and-context-propagation)
    - [Configuration](#configuration)
    - [Metrics](#metrics)
* [Help and examples](#help-and-examples)
    - [Support](#support)
    - [Demo app](#demo-app)
* [License](#license)


## Introduction

[AppNeta TraceView](http://appneta.com/products/traceview) provides distributed tracing and
code-level application performance monitoring. TraceView's cross-language, production-ready,
low-overhead distributed tracing system (similar to Dapper, Zipkin, or X-Trace) can follow the path
of an application request as it is processed and forwarded between services written in [Go, Java,
Scala, Node.js, Ruby, Python, PHP, and C#/.NET](https://docs.appneta.com/support-matrix), reporting data to TraceView's cloud platform for
analysis, monitoring, and fine-grained, filterable data visualization. This repository provides
instrumentation for Go, which allows Go-based applications to be monitored using TraceView.

Go support is currently in beta (though we run the instrumentation to process
data in our production environment!) so please share any feedback you have; PRs welcome.
Also, you can read more about how we use Go at AppNeta in our [blog post announcing our Go instrumentation](https://www.appneta.com/blog/go-long-with-golang/)!

## Getting started

### Installing

To get tracing, you'll need a [(free) TraceView account](http://www.appneta.com/products/traceview/).

In the product install flow, **choose to skip the webserver and select any language.**  (You'll notice the absence of Go because we're currently in beta.)

Follow the instructions during signup to install the Host Agent (“tracelyzer”). This will also
install the liboboe and liboboe-dev packages on your platform.

Then, install the following:

* [Go >= 1.5](https://golang.org/dl/)
* This package: go get github.com/appneta/go-appneta/v1/tv

The install flow will wait for 5 traces to come in from your app.  You can
check out the [demo](#demo) app below to get a quick start.  One you have 5,
you can progress to [your account's overview
page](https://login.tv.appneta.com/overview).

### Building your app with tracing

By default, no tracing occurs unless you build your app with the build tag “traceview”. Without it,
calls to this API are no-ops, allowing for precise control over which deployments are traced. To
build and run with tracing enabled, you must do so on a host with TraceView's packages installed.
Once the liboboe-dev (or liboboe-devel) package is installed you can build your service as follows:

```
 $ go build -tags traceview
```


## Instrumenting your application

### Usage examples

The simplest integration option is this package's
[tv.HTTPHandler](https://godoc.org/github.com/appneta/go-appneta/v1/tv#HTTPHandler) wrapper.  This
will monitor the performance of the provided
[http.HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc), visualized with latency
distribution heatmaps filterable by dimensions such as URL host & path, HTTP status & method, server
hostname, etc. [tv.HTTPHandler](https://godoc.org/github.com/appneta/go-appneta/v1/tv#HTTPHandler)
will also continue a distributed trace described in the incoming HTTP request headers (from
TraceView's [automatic](https://docs.appneta.com/support-matrix)
[C#/.NET](https://docs.appneta.com/support-matrix#net),
[Java](https://docs.appneta.com/support-matrix#java-scala),
[Node.js](https://github.com/appneta/node-traceview),
[PHP](https://docs.appneta.com/support-matrix#php),
[Python](https://github.com/appneta/python-traceview),
[Ruby](https://github.com/appneta/ruby-traceview), and
[Scala](https://docs.appneta.com/support-matrix#java-scala) instrumentation) through to the HTTP
response.

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/appneta/go-appneta/v1/tv"
)

var startTime = time.Now()

func normalAround(ts, stdDev time.Duration) time.Duration {
	return time.Duration(rand.NormFloat64()*float64(stdDev)) + ts
}

func slowHandler(w http.ResponseWriter, r *http.Request) {
	time.Sleep(normalAround(2*time.Second, 100*time.Millisecond))
	w.WriteHeader(404)
	fmt.Fprintf(w, "Slow request, path: %s", r.URL.Path)
}

func increasinglySlowHandler(w http.ResponseWriter, r *http.Request) {
	// getting slower at a rate of 1ms per second
	offset := time.Duration(float64(time.Millisecond) * time.Now().Sub(startTime).Seconds())
	time.Sleep(normalAround(time.Second+offset, 100*time.Millisecond))
	fmt.Fprintf(w, "Slowing request, path: %s", r.URL.Path)
}

func main() {
	http.HandleFunc("/slow", tv.HTTPHandler(slowHandler))
	http.HandleFunc("/slowly", tv.HTTPHandler(increasinglySlowHandler))
	http.ListenAndServe(":8899", nil)
}
```
![sample_app screenshot](https://github.com/appneta/go-appneta/raw/master/img/readme-screenshot1.png)

To monitor more than just the overall latency of each request to your Go service, you will need to
break a request's processing time down by placing small benchmarks into your code. To do so, first
start or continue a `Trace` (the root `Layer`), then create a series of `Layer` spans (or `Profile`
timings) to capture the time used by different parts of the app's stack as it is processed.

TraceView provides two ways of measuring time spent by your code: a `Layer` can measure e.g. a
single DB query or cache request, an outgoing HTTP or RPC request, the entire time spent within a
controller method, or the time used by a middleware method between calls to child Layers. A
`Profile` provides a named measurement of time spent inside a `Layer`, and is typically used to
measure a single function call or code block, e.g. to represent the time used by expensive
computation(s) occurring in a `Layer`. Layers can be created as children of other Layers, but a
`Profile` cannot have children.

TraceView identifies a reported Layer's type from its key-value pairs; if keys named "Query" and
"RemoteHost" are used, a Layer is assumed to measure the extent of a DB query. KV pairs can be
appended to Layer and Profile extents as optional extra variadic arguments to methods such as
[BeginLayer()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#BeginLayer) or
[BeginProfile()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#BeginProfile),
[Info()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#Layer), and
[End()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#Layer). We also provide helper methods
such as [BeginQueryLayer()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#BeginQueryLayer),
[BeginCacheLayer()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#BeginCacheLayer),
[BeginRemoteURLLayer()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#BeginRemoteURLLayer),
[BeginRPCLayer()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#BeginRPCLayer), and
[BeginHTTPClientLayer()](https://godoc.org/github.com/appneta/go-appneta/v1/tv#BeginHTTPClientLayer)
that match the spec in our
[custom instrumentation docs](http://docs.appneta.com/traceview-instrumentation#special-interpretation)
to report attributes associated with different types of service calls, used for indexing TraceView's
filterable charts and latency heatmaps.

```go
func slowFunc(ctx context.Context) {
    // profile a slow function call
    defer tv.BeginProfile(ctx, "slowFunc").End()
    // ... do something slow
}

func myHandler(ctx context.Context) {
    // create new tv.Layer and context.Context for this part of the request
    L, ctxL := tv.BeginLayer(ctx, "myHandler")
    defer L.End()

    // profile a slow part of this tv.Layer
    slowFunc(ctxL)

    // Start a new Layer, given a parent layer
    mL, _ := L.BeginLayer("myMiddleware")
    // ... do something slow ...
    mL.End()

    // Start a new query Layer, given a context.Context
    q1L := tv.BeginQueryLayer(ctxL, "myDB", "SELECT * FROM tbl1", "postgresql", "db1.com")
    // perform a query
    q2L.End()
}

func processRequest() {
    // Create tv.Trace and bind to new context.Context
    ctx := tv.NewContext(context.Background(), tv.NewTrace("myApp"))
    myHandler(ctx) // Dispatch handler for request
    tv.EndTrace(ctx)
}
```

### Retrieving the context from an http request

A common pattern when tracing in golang is to call `tv.HTTPHandler(handler)` then retrieve the trace
context inside of the handler.  
```go

func myHandler( w http.ResponseWriter, r *http.Request ) {
    // trace this request, overwriting w with wrapped ResponseWriter
    t, w := tv.TraceFromHTTPRequestResponse("myHandler", w, r)
    ctx := tv.NewContext(context.Background(), t)
    defer t.End()

    //create a query layer
    q1L := tv.BeginQueryLayer(ctx, "myDB", "SELECT * FROM tbl1", "postgresql", "db1.com")
    //run you're query here

    //
    q1L.End()

    fmt.Fprintf(w, "I ran a query")

func main() {
    http.HandleFunc("/endpoint", tv.HTTPHandler(myHandler))
    http.ListenAndServe(":8899", nil)
}
```

### Distributed tracing and context propagation

A TraceView trace is defined by a context (a globally unique ID and metadata) that is persisted
across the different hosts, processes, languages and methods that are used to serve a request. Thus
to start a new Layer you need either a Trace or another Layer to begin from. (The Trace
interface is also the root Layer, typically created when a request is first received.) Each new
Layer is connected to its parent, so you should begin new child Layers from parents in a way that
logically matches your application; for more information [see our custom layer
documentation](http://docs.appneta.com/traceview-instrumentation#custom-layers).

The [Go blog](https://blog.golang.org/context) recommends propagating request context using the
package [golang.org/x/net/context](godoc.org/golang.org/x/net/context): "At Google, we require that
Go programmers pass a [Context](https://godoc.org/golang.org/x/net/context) parameter as the first
argument to every function on the call path between incoming and outgoing requests." Frameworks like
[Gin](https://godoc.org/github.com/gin-gonic/gin#Context) and
[Gizmo](https://godoc.org/github.com/NYTimes/gizmo/server#ContextHandler) use Context
implementations, for example. We provide helper methods that allow a Trace to be associated with a
[context.Context](https://godoc.org/golang.org/x/net/context) interface; for example,
[tv.BeginLayer](https://godoc.org/github.com/appneta/go-appneta/v1/tv#BeginLayer) returns both a
new Layer and an associated context, and
[tv.Info(ctx)](https://godoc.org/github.com/appneta/go-appneta/v1/tv#Info) and
[tv.End(ctx)](https://godoc.org/github.com/appneta/go-appneta/v1/tv#Layer) both use the Layer
defined in the provided context.

It is not required to work with context.Context to trace your app, however. You can also use just
the [Trace](https://godoc.org/github.com/appneta/go-appneta/v1/tv#Trace),
[Layer](https://godoc.org/github.com/appneta/go-appneta/v1/tv#Layer), and
[Profile](https://godoc.org/github.com/appneta/go-appneta/v1/tv#Profile) interfaces directly, if it
better suits your instrumentation use case.

```go
func runQuery(ctx context.Context, query string) {
    l := tv.BeginQueryLayer(ctx, "myDB", query, "postgresql", "db1.com")
    // .. execute query ..
    l.End()
}

func main() {
    // create trace and bind to new context
    ctx := tv.NewContext(context.Background(), tv.NewTrace("myApp"))

    // pass the root layer context to runQuery
    runQuery(ctx)

    // end trace
    tv.EndTrace(ctx)
}
```


### Configuration

The `GO_TRACEVIEW_TRACING_MODE` environment variable may be set to "always", "through", or "never".
- Mode "always" is the default, and will instruct TraceView to consider sampling every inbound request for tracing.
- Mode "through" will only continue traces started upstream by inbound requests, when a trace ID is available in the request metadata (e.g. an "X-Trace" HTTP header).
- Mode "never" will disable tracing, and will neither start nor continue traces.


### Metrics

In addition to distributed tracing and latency measurement, this package also provides Go metrics
monitoring for runtime metrics such as memory and number of goroutines. Below is a screenshot from
our [blog's release announcement](https://www.appneta.com/blog/go-long-with-golang/) of a latency
heatmap underlaid with memory metrics, just after fixing a memory leak and restarting a service
inside a staging environment.

<img width=729 src="https://github.com/appneta/go-appneta/raw/master/img/metrics-screenshot1.png">

## Help and examples

### Support

While we use TraceView to trace our own production Go services, this version of our Go instrumentation is currently in beta
and under active development. We welcome your feedback, issues and feature requests, and please contact us at go@appneta.com!

### Demo web app

If you have installed TraceView and this package, you can run the sample “web app” included with go-appneta:

    $ cd $GOPATH/src/github.com/appneta/go-appneta/examples/sample_app
    $ go run -tags traceview main.go

A web server will run on port 8899. It doesn’t do much, except wait a bit and echo back your URL path:

    $ curl http://localhost:8899/slow
    Slow request... Path: /slow

You should see these requests appear on your TraceView dashboard.

### Distributed app

There is also a demonstration of distributed tracing in examples/distributed_app, a sample system
comprised of two Go services, a Node.js service, and a Python service. It can be built and run from
source in each service's subdirectory, or by using docker-compose:

    $ cd $GOPATH/src/github.com/appneta/go-appneta/examples/distributed_app
    $ APPNETA_KEY="xxx" docker-compose build
    # ... (building)
    $ docker-compose up -d
    Starting distributedapp_alice_1
    Starting distributedapp_bob_1
    Starting distributedapp_caroljs_1
    Starting distributedapp_davepy_1
    Starting distributedapp_redis_1

and substituting "xxx" withyour AppNeta access key. This app currently provides two HTTP handlers:
`aliceHandler`, which randomly forwards requests to either of "bob" (Go), "caroljs", or "davepy",
and `concurrentAliceHandler`, which makes requests to all three in parallel.

    $ curl http://localhost:8890/alice
    {"result":"hello from bob"}
    $ curl http://localhost:8890/alice
    Hello from Flask!
    $ curl http://localhost:8890/alice
    Hello from app.js
    $ curl http://localhost:8890/concurrent
    Hello from Flask!{"result":"hello from bob"}Hello from app.js

You should see traces for these appear on your TraceView dashboard. Here is an example trace of the
concurrent handler:
<img width=729 src="https://github.com/appneta/go-appneta/raw/master/img/concurrent-tracedetails.gif">

## License

Copyright (c) 2016 AppNeta, Inc.

Released under the [AppNeta Open License](http://www.appneta.com/appneta-license), Version 1.0

