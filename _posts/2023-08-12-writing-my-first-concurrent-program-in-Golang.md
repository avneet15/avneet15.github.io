---
layout: post
title:  "Writing my first concurrent program in Golang"
author: Avneet
categories: [ Golang ]
image: assets/images/go_img.png
---

I’m a recent beginner with the Go programming language and was trying to write my first concurrent program in it. I started my search and a number of keywords came up - coroutines, channels, wait groups. It took me a bit to wade through a number of resources to put together a working program for what I was trying to do.

This post tries to summarize my understanding of these concepts and to put it all together to produce working code.

### What I was trying to accomplish

So I hear concurrency is pretty easy with Go…right ?


### Goroutines

Goroutines are essentially lightweight threads which is why you can run a LOT more goroutines than threads. It’s a function that can execute concurrently with the rest of the program.

To make a function a Goroutine, the function needs to be prefixed with the “go” keyword. Sounds easy enough, let’s give it a try.

On trying a simple example like this:

```

package main

import (
   "fmt"
)

func worker() {
	// Perform a network call
  fmt.Println("Task Completed")
}

func main() {
   fmt.Println("Starting main function")

   go worker()

   fmt.Println("Exiting main function")
}
```

Expected output:

```go
Starting main function
Task Completed
Exiting main function
```

Actual output:

```go
Starting main function
Exiting main function
```

So the **worker** Goroutine wasn’t invoked..or was it ? In the example here, the **worker** Gouroutine is kicked off but was unable to be executed before the termination of the main function, which itself is a Goroutine.

This can be fixed with providing the Goroutine enough time to execute as seen below.

```go
package main

import (
	"fmt"
	"time"
)

func worker() {
	// Perform a network call
  fmt.Println("Task Completed")
}

func main() {
	fmt.Println("Starting main function")
	
	time.Sleep(1 * time.Second)
	go worker()

	fmt.Println("Exiting main function")
}
```

We’re now waiting long enough for the **worker** Goroutine to execute before the main one terminates.

Go provides a way to help synchronize execution by means of WaitGroups, let’s explore that below.

### WaitGroups

WaitGroups in Go provide a mechanism to wait for the desired number of Goroutines to finish.

I can now introduce spawning multiple workers and ensuring that all of them complete before the main goroutine finishes.

```
package main

import (
   "fmt"
   "sync"
)

var wg sync.WaitGroup

func worker(workerId int, wg sync.WaitGroup) {
   defer wg.Done()
   // Perform a network call
   fmt.Println("Task Completed by worker", workerId)
}

func main() {
   fmt.Println("Starting main function")
   wg := sync.WaitGroup{}
   wg.Add(3)

   workers := []int{1, 2, 3}
   for _, workerId := range workers {
      go worker(workerId, wg)
   }

   wg.Wait()

   fmt.Println("Exiting main function")
}
```

I create 3 workers and initialize a WaitGroup in the main Goroutine to indicate that it needs to wait for all 3 to finish.

The WaitGroup is then passed to the worker which will signal when done, using the waitgroup.

The `wg.Wait` will block until it receives the expected number of `Done` invocations.

Expected output:

```go
Starting main function
Task Completed by worker 3
Task Completed by worker 2
Task Completed by worker 1
Exiting main function
```

Actual output

```go
Starting main function
Task Completed by worker 3
Task Completed by worker 2
Task Completed by worker 1
fatal error: all goroutines are asleep - deadlock!
```

```go
goroutine 1 [semacquire]:
sync.runtime_Semacquire(0x140000021a0?)
/opt/homebrew/Cellar/go/1.20.7/libexec/src/runtime/sema.go:62 +0x2c
sync.(*WaitGroup).Wait(0x140000ae020)
/opt/homebrew/Cellar/go/1.20.7/libexec/src/sync/waitgroup.go:116 +0x78
main.main()
/Users/aoberoi/projects/pocket_integrations/eg2_goroutine_wg.go:25 +0x110
```

Hm, interesting…

A few more Google searches later, I learn that the WaitGroup functions need to be called on a pointer per their documentation. On modifying the program per this:

```
package main

import (
   "fmt"
   "sync"
)

func worker(workerId int, wg *sync.WaitGroup) {
   defer wg.Done()
   // Perform a network call
   fmt.Println("Task Completed by worker", workerId)
}

func main() {
   fmt.Println("Starting main function")
   wg := sync.WaitGroup{}
   wg.Add(3)

   workers := []int{1, 2, 3}
   for _, workerId := range workers {
      go worker(workerId, &wg)
   }

   wg.Wait()

   fmt.Println("Exiting main function")
}
```

The expected output is now seen!

While this is great as a synchronization mechanism, my use case is to actually pass a result/error back to the main function. This leads me to channels!

### Channels

Channels are a mechanism to send and receive data to and from goroutines. Channels can be buffered or unbuffered, which is the default. Below is a simple example of an unbuffered channel:

```
package main

import (
   "fmt"
   "time"
)

func worker(ch chan<- int) {
   // Perform a network call and return it's result, eg 1
   ch <- int(1)
   
}

func main() {
   fmt.Println("Starting main function")

   ch := make(chan int)

   go worker(ch)
   // Receive the result from the channel
   result := <-ch

   fmt.Println("Exiting main function with result:", result)
}
```

The program here creates an unbuffered channel to pass an integer. The channel is only meant to pass a single value as no size has been specified. 

The worker goroutine in it’s function signature specifies that it expects a “send only channel” i.e channel that it sends data to, by means of the direction of the arrow here: `chan<-` . 

In a similar manner, “receive only” channel can be defined as `<-chan` to indicate they should be used for receiving only. 

Lastly a channel can be defined without any directionality, meaning it can both send as well as receive data.

The worker goroutine populates the result of it’s computation on the channel, which the main goroutine is blocked on by creating a receiver to wait for a value on the channel. 

Note than any processing done in the worker goroutine after passing the value to the channel would not be guaranteed since the main goroutine may receive, be unblocked and exit before then.

Coming back to the task at hand, my use case was to make parallel network calls and gather results back.

I thus make use of a buffered channel with a custom struct type which will return result or error depending on the result of the network call made by each worker. Putting this all together below:

```
package main

import (
   "fmt"
)

type ResultError struct {
   result int
   err    error
}

func networkCall() (int, error) {
   return 1, nil
}

func worker(workerId int, resultChannel chan<- ResultError) {
   fmt.Println("Task started by worker", workerId)

   // Perform a network call
   resp, err := networkCall()
   resultChannel <- ResultError{
      result: resp,
      err:    err,
   }
}

func main() {
   fmt.Println("Starting main function")
   numWorkers := 3

   resultChannel := make(chan ResultError, numWorkers)

   workers := []int{1, 2, 3}
   for _, workerId := range workers {
      go worker(workerId, resultChannel)
   }

   resultMap := make(map[int]int, numWorkers)
   for _, workerID := range workers {
      result := <-resultChannel
      if result.err != nil {
         fmt.Println("Error:", result.err)
      } else {
         resultMap[workerID] = result.result
      }
   }

   fmt.Println("Exiting main function with result:", resultMap)
}
```

Output:

```go
Starting main function
Task started by worker 3
Task started by worker 1
Task started by worker 2
Exiting main function with result: map[1:1 2:1 3:1]
```

The program above:

- Creates a struct `ResultError` to hold the response of the worker goroutine.
- Creates a dummy implementation of the `networkCall` function to simulate a network call.
- Uses a buffered channel `resultChannel` passed to the workers to send their result when it is ready.
- Uses a receiver in the main goroutine to block until the expected number of values are read from the buffered channel  `resultChannel` to collect results into the `resultMap` .

An alternative to the above approach would be to have each workers update the result of their computation into the same map directly along with signalling completion back to the main goroutine. 

The in-built map provided by Go isn’t safe for concurrent usage i.e by multiple goroutines though. Golang also provides a concurrent map implementation in the form of the *[syncMap](https://pkg.go.dev/golang.org/x/sync/syncmap)* which comes with a few issues, as explained quite well [here](https://medium.com/@deckarep/the-new-kid-in-town-gos-sync-map-de24a6bf7c2c).

Let's see how mutexes can help us with this instead.
### Mutexes

A mutex prevents multiple goroutines from accessing a shared resource simultaneously. 

We can use a *sync.Mutex* in order to lock the critical section of code when we need to conservatively block all other goroutines from concurrently reading or writing the same resource.

If concurrent reading of the shared resource is permissible, we can use *sync.RWMutex* which would allow concurrent reading instead of blocking on single read operations.

The updated program now looks like this:

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
)

func worker(workerId int, resultMap map[int]int, wg *sync.WaitGroup, mu *sync.Mutex) {
	defer wg.Done()

	mu.Lock()
	defer mu.Unlock()
	resultMap[workerId] = rand.Intn(100)
	fmt.Println("Task Completed by worker", workerId)
}

func main() {
	fmt.Println("Starting main function")
	wg := sync.WaitGroup{}

	mu := sync.Mutex{}
	resultMap := make(map[int]int)

	workers := []int{1, 2, 3}
	for _, workerId := range workers {
		wg.Add(1)
		go worker(workerId, resultMap, &wg, &mu)
	}

	wg.Wait()

	fmt.Println("Exiting main function with result", resultMap)
}
```

Output:

```go
Starting main function
Task Completed by worker 3
Task Completed by worker 2
Task Completed by worker 1
Exiting main function with result map[1:89 2:11 3:57]
```

And there we have it! The access to the resultant map is now protected with a mutex and we’re able to use a waitgroup to signal completion of all the workers.

The above is an obviously simplified example. A typical pattern for such functionality is to embed the lock and waitgroup in the core struct itself, to make it easily usable by methods defined on the struct directly.

And that’s it for this post, see you in the next one!