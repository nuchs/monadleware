# Monadleware

Monadleware is a helper for managing the middleware in http request pipelines
that is tenuously inspired by monads. 

## Middleware

At it's simplest middleware is a function which is called as part of processing
an http request. It will have a chance to manage the request both before and
after the final handler, e.g.

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("Handled")
}

func middle1(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("Middle 1 standing by")
		next.ServeHTTP(w, r)
		fmt.Println("They came from... behind...")
	})
}

...

// Logs the following for each request
//  Middle 1 standing by
//  Handled
//  They came from... behind...
http.Handle("/", middle1(http.HandlerFunc(myHandler)))
```

And because middleware is just a function which both takes a http.Handler, the
middleware calls can be chained:

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("Handled")
}

func middle1(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("Middle 1 standing by")
		next.ServeHTTP(w, r)
		fmt.Println("They came from... behind...")
	})
}

func middle2(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("Middle 2 standing by")
		next.ServeHTTP(w, r)
		fmt.Println("I can't shake him!, I can't shake him!")
	})
}

...

// Logs the following for each request
//  Middle 1 standing by
//  Middle 2 standing by
//  Handled
//  I can't shake him!, I can't shake him!
//  They came from... behind...
http.Handle("/", middle2(middle1(http.HandlerFunc(myHandler))))
```

More realistically you might have a series of middleware functions that log each
request, authenticate the user, and so on.

```go
http.Handle("/", log(authenticate(checkCache(http.HandlerFunc(myHandler)))))
```

The main annoyance with this approach other then the verbosity is that it isn't
easy to reuse the request pipelines and so you can end up with something like: 

```go
http.Handle("/thisWay", log(authenticate(checkCache(http.HandlerFunc(myHandler1)))))
http.Handle("/thatWay", log(authenticate(checkCache(http.HandlerFunc(myHandler2)))))
http.Handle("/forwards", log(authenticate(checkCache(http.HandlerFunc(myHandler3)))))
http.Handle("/backwards", log(authenticate(checkCache(http.HandlerFunc(myHandler4)))))
```

## Usage

Monadleware provides functions to allow you to chain together middleware
functions in a reusable manner.

```go
// Just call Chain with each of the middleware functions you want to compose in
// the order you want them to be called.
middleware1 := monadleware.Chain(log, authenticate)

// You can add a single middleware to the end of the chain to create a new
// chain, this chain is independent of the first.
middleware2 := middleware1.Bind(checkCache)

// Or you add multiple new middleware functions by calling chain again to
// produce yet another new chain
middleware3 := monadleware.Chain(middleware1, authorise)

// log -> authenticate -> myHandler1
http.Handle("/thisWay", middleware1.Call(myHandlerFunc1))

// log -> authenticate -> checkCache -> myHandler2
http.Handle("/thatWay", middleware2.Call(myHandlerFunc2))

// log -> authenticate -> authorise -> myHandler
http.Handle("/thatWay", middleware3.Call(myHandlerFunc3))
http.Handle("/thatWay", middleware3(myHandler))
```

