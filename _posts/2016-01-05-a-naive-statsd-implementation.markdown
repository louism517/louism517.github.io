---
title:  "A (very) Naive Statsd Clone in Go"
date:   2016-01-04 21:36:20 +0000
---
Recently I had cause to write some Go code for the purpose of splurging a bunch of metrics to a third-party collector. The metrics in question would come in at irregular intervals and all that was required was a simple count of each.

Unwilling to fire-hose an indeterminate number of metrics at a pay-per-metric service (_kerching_), I decided some aggregation was in order. Reflexively I reached for [statsd][statsd-github], _the_ metric aggregator of choice for the discerning Devop. But hang on, all I needed to do was keep a count of how many times each metric was fired (a counter, if you will) and flush it every 5 seconds. This seemed like a pretty simple thing to do, and Go has built-in support for concurrency, and I'm already writing Go code, _and_ I want to roll this Go code into a teeny tiny little Docker container that NodeJS would only inflate. So why not? 

The truth is it was refreshingly easy. So much so that I fear I'm in danger of becoming one of those increasingly numerous Golang evangelists you read about. 

The following code outputs to the screen instead of a metric collector but the premise remains the same: have one goroutine deal with generating metrics, and another flushing them every 5 seconds.

```go
package main

import (
  "fmt"
  "time"
  "math/rand"
)

func main() {
  // initialise channels
  ticker := time.Tick(5 * time.Second)
  metrics := make(chan string)
 
  // start metricFlusher in a separate goroutine 
  go metricFlusher(ticker, metrics)

  // pump in some 'metrics'
  for {
    metrics <- "my.useful.count" 
    time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
  }
  
}

func metricFlusher(ticker <- chan time.Time, metrics chan string) {
  metricCount := 0

  for {
    select {
      case <- metrics:
        metricCount++
      case <- ticker:
        fmt.Printf("my.useful.count %d\n", metricCount)
        metricCount = 0
    }
  }
}
```

It is that simple. 

The standard library's `time.Tick(d Duration)` function returns a channel on which the current time will be posted every `Duration` (a Duration is a number of units; seconds, milliseconds etc.). This is called a Ticker, for obvious reasons.  We also initialise a `metrics` channel that will accept our make-believe metrics as strings. 

Then we kick off the `metricFlusher` in a separate goroutine (think _lightweight thread_ if you're not familiar). This function consists of a select statement within a continuous for-loop. Select is used in a similar fashion to the `select(2)` system call but instead of watching file descriptors it 'watches' a number of channels and takes action when one is ready (i.e. there is something to be read). 

If something exists on the `metrics` channel, a counter is incremented. If something exists on the `ticker` channel, then an interval has elapsed, so we flush the count of metrics and reset the counter. 

Back in the main goroutine (yep, `main()` is its own goroutine), we generate some useless metrics in another infinite loop. After sending a metric down the metrics channel we go to sleep for a semi-random number of milliseconds before sending another.

Running this code outputs lines like this at 5 second intervals: 

```
my.useful.count 12
my.useful.count 14
my.useful.count 10
my.useful.count 15
```

You can see that a varying number of metrics have been sent in each 5 second period, and they have all been properly aggregated. Success!

We can make this somewhat more useful by accepting metrics with different names and aggregating a count of each. This can be achieved by replacing the `metricCount`er with a simple map, the keys of which will be the metric name and the values a count of the number of times the metric has been recorded.

```go
func metricFlusher(ticker <- chan time.Time, metrics chan string) {
  metricMap := map[string]int{} 

  for {
    select {
      case <- metrics:
        // if key already exists, increment by 1
        if _, found := metricMap[m]; !found {
          metricMap[m] = 1
        } else {
          metricMap[m] += 1
        }
      case <- ticker:
        for m,count := range metricMap {
          fmt.Printf("%s %d\n", m, count)
        }
        metricMap := map[string]int{} 
    }
  }
}
```

So there we have it, a naive take on the statsd counter. Obviously statsd offers far more, but hopefully this shows how easy it is to utilise Golang's concurrency facilities. 

Incidentally, there does exist a _full_ statsd implementation written in Go, [statsgod][statsgod-github]
 
[statsgod-github]: https://github.com/acquia/statsgod 
[statsd-github]: https://github.com/etsy/statsd
