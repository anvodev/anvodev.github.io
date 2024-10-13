---
layout: post
title: 'Understanding And Implementing A Rate Limiter In Go'
date: 2024-04-18
categories: dev
---

Sooner or later, your app will need a rate limiter due to several reasons:

- Prevent DDoS by intended or unintended users
- Reduce cost, increase efficient resource usage by limiting requests related to 3rd parties, prioritize high business value requests over low business value requests
- Prevent the server from being overloaded

In this article, we will explore how rate limiter works, its underlying algorithm, and implement a simple Token Bucket in Go by ourselves

## Where to put the Rate Limiter?

![Where to put the Rate Limiter](/assets/images/2024/04/rate-limiter-options.png)

We have three options to put our rate limiter:

- Inside the client
- Inside the server
- Between the client and the server

Let's analyze these options:

1. Inside the client:
   - Pros: May be very fast and provide a good user experience because it may not need to request to the server and just respond to the user immediately.
   - Cons: Everything on the client side can be manipulated or bypassed. Hence, the rate limiter can be disabled by the user.
2. Inside the server:
   - Pros: Developers have full control over the rate limiter logic. Quick and efficient in small scale.
   - Cons: Some languages don't support building rate limiters well. Does not scale well when multiple independent rate limiters are built in multiple servers.
3. Between the client and the server:
   - Pros: If using a third-party rate limiter, this is the only option. Can be implemented in a language other than the server language. Can centralize cross-cutting concerns into one place (API Gateway).
   - Cons: More complicated to implement and manage.

In this article, we choose option 2: Rate Limiter inside the Server for simplicity. In production, you can use a third-party API Gateway like Kong or Cloudflare.

## Algorithms of the Rate Limiter?

There are many algorithms for implementing a rate limiter, and in this article, we will use Token Bucket.

![Token Bucket Rate Limiter](/assets/images/2024/04/token-bucket-rate-limiter.png)

The Token Bucket algorithm works in this way:

- The rate limiter starts with some tokens.
- Each request will consume 1 token.
- If there are tokens remaining, the request is allowed to pass.
- If no tokens are left, the request is dropped.
- Every second, some tokens will be refilled up to the maximum capacity.
- On average, the rate at which requests pass is the same as the refill rate.
- The maximum number of requests that can pass in one period is equal to the bucket's maximum capacity.

For example, if the bucket has a capacity of 4 tokens and the refill rate is 2 tokens per second, the rate limiter will allow 2 requests to pass every second on average. However, if the bucket is refilled to full capacity without being interrupted by any requests, it will allow a burst of up to 4 tokens at the same time.

The Token Bucket algorithm is simple to implement and memory efficient. It is able to handle burst requests when there is high traffic. However, it is challenging to efficiently configure the right refill rate and maximum capacity to protect the server without being too strict and dropping valid requests.

This algorithm is used by companies such as : [Stripe](https://stripe.com/blog/rate-limiters#:~:text=We%20use%20the%20token%20bucket%20algorithm%20to%20do%20rate%20limiting), [Amazon](https://docs.aws.amazon.com/ec2/latest/devguide/ec2-api-throttling.html#throttling-how).

## Implement the Token Bucket in Go

First, we need a basic HTTP server. Let's create a new folder named `golimiter`. Inside this folder, we will create a new file called `main.go`.

```go
// main.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"

	"github.com/gorilla/mux"
)

func main() {
	mux := mux.NewRouter()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	})
	fmt.Println("Server is running on port :4000")
	http.ListenAndServe(":4000", mux)
}

```

Inside the `golimiter` folder, we can run the server by using the command `go run ./`.

```bash
$ go run ./
---
Server is running on port :4000
```

Then, we can make a request using `curl`:

```bash
$ curl -i http://localhost:4000
---
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 18 Apr 2024 00:10:41 GMT
Content-Length: 16

{"status":"ok"}
```

The server is working fine. Now we will implement a Go rate-limiter middleware to limit the number of requests.

The intuitive way is to have a mechanism to periodically refill the tokens. We can achieve this by using a cron job at the system level or by using the built-in library [Ticker](https://pkg.go.dev/github.com/tjgq/ticker) in Go.

Although the aforementioned methods are viable, they require additional threads or processes, which can complicate our application. Instead, we can use a simpler and more elegant approach inspired by the [rate](https://pkg.go.dev/golang.org/x/time/rate) library, which is a built-in Go rate limiter. We can leverage incoming requests to calculate and refill new tokens.

Let's implement this by creating a new file called `rate-limiter.go`.

```go
// rate-limiter.go
package main

import (
	"encoding/json"
	"net/http"
	"time"
)

type TokenBucket struct {
	rate       float64   // How many tokens are refilled every second
	burst      int       // Maximum capacity of the bucket
	tokens     int       // Current tokens in the bucket
	lastUpdate time.Time // Last time we update the tokens amount
}

func NewTokenBucket(r float64, b int) *TokenBucket {
	return &TokenBucket{
		rate:       r,
		burst:      b,
		tokens:     b, // Initialize the tokens at max capacity of the bucket
		lastUpdate: time.Now(),
	}
}

func (tb *TokenBucket) Allow() bool {
	currentTime := time.Now()
	elapsed := currentTime.Sub(tb.lastUpdate).Seconds() // Calculate the time since the last refill
	tb.tokens += int(tb.rate * elapsed)                 // Refill the tokens base on the rate and the time from the last refill
	tb.lastUpdate = currentTime                         // When the next request comes, we will calculate the refill tokens based on this update
	if tb.tokens > tb.burst {                           // Burst is the maximum capacity of the bucket
		tb.tokens = tb.burst
	}
	if tb.tokens > 0 { // There is some available tokens, so the request is passed and 1 token is consumed
		tb.tokens -= 1
		return true
	} else { // There is no token left, and the request is dropped.
		return false
	}
}

func rateLimiter(next http.Handler) http.Handler {
	// Do something globally, initialization
	tb := NewTokenBucket(0.5, 2) // New tokens are replenished at a rate of 0.5 tokens per second. The maximum burst request amount is 2

	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Do something for each request
		if !tb.Allow() {
			w.Header().Set("Content-Type", "application/json")
			// We can also use some additional headers such as X-Ratelimit-Remaining, X-Ratelimit-Limit, and X-Ratelimit-Retry-After for further clarification
			w.WriteHeader(http.StatusTooManyRequests) // 429 StatusCode
			json.NewEncoder(w).Encode(map[string]string{"error": "rate limited"})
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

Wrap the new middleware to the router in the `main.go` file

```go
// main.go
//...
func main() {
	//...
	http.ListenAndServe(":4000", rateLimiter(mux)) // change mux -> rateLimiter(mux)
}
```

Now, if we run a couple of requests via `curl` again, we will see that our services are rate-limited when we make more than 2 requests per second.

```bash
$ curl -i http://localhost:4000
---
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Date: Thu, 18 Apr 2024 00:19:43 GMT
Content-Length: 25
```

When we exceed the limit, just wait for 0.5 seconds for the token to be refilled, and then we can make the request again.

```bash
$ curl -i http://localhost:4000
---
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 18 Apr 2024 00:21:10 GMT
Content-Length: 16

{"status":"ok"}
```

Congratulations! We have upgraded ourselves to the next level by learning and implementing one essential component in the system design.

See you on the next post!

Full source code can be found [here](https://github.com/anvodev/golimiter)
