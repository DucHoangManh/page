---
title: "Create a web framework in Go part 1: static routing"
date: 2021-12-31T23:24:26+07:00
tags: ["go", "framework", "english"]
categories: ["technical"]
---
This post contains my notes while implement `caneweb` -  a gin-like minimal web framework/router after reading the first section of 7 days golang by geektutu (https://github.com/geektutu/7days-golang).
  
## How standard net/http package handle request?

First, let's look at a sample written with net/http package:
```go
func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/post", getAllPost)
    log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```
This piece of code binds two endpoints with the corresponding handler function, and starts a web server at port 8000, terminates the server if some errors are returned.  
All http handlers must implement the handler interface, and we can use multiple handlers to handle a single request by passing the parameters around:
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```
The standard library package does provide basic functions to create a web server including monitor port, static routing, request parsing... Some other functions need to be implemented when necessary:
- Dynamic routing: routing with rules like `post/:id`, `post/*`,... or using regular expression.
- Middleware: By passing around the request, and response between multiple functions to change the result.
- Group requests: Group endpoints into a cluster that share some commons.
- Validate requests. 
- ...

## Create the static-routing version of the framework

First, we need to design a struct that represents the state of the current request data and have methods to easily work with requests.

### Why encapsulate response and request into a single struct?
- Reduce the complexity of the handler, a user no longer need to care about which data resides in request, or responseWriter
- Easier to create a response and less error-prone, reduce repetitive

For example, in order to write the response for a request, instead of 
```go
obj = map[string]interface{}{
    "title": "my first blog post",
    "read_time": "5",
}
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusOK)
encoder := json.NewEncoder(w)
if err := encoder.Encode(obj); err != nil {
    http.Error(w, err.Error(), 500)
}
```
We can achive the same result in a simpler and clearer way:
```go
c.JSON(http.StatusOK, cane.Map{
    "title": "my first blog post",
    "read_time": 5,
})
```

`ctx.go`
```go
package cane

import "net/http"

type Ctx struct {
    // origin objects
    Writer http.ResponseWriter
    Req    *http.Request
    // request data
    Path   string
    Method string
    // response data
    StatusCode int
}

// newCtx create new Ctx with original data
func newCtx(w http.ResponseWriter, r *http.Request) *Ctx {
    return &Ctx{
        Writer: w,
        Req:    r,
        Path:   r.URL.Path,
        Method: r.Method,
    }
}
```
Add some functions to work with request and reponse morre efficiently:
```go
// responseWriter 
func (c *Ctx) SetHeader(key, value string) {
    c.Writer.Header().Set(key, value)
}

func (c *Ctx) Status(code int) {
    c.StatusCode = code
    c.Writer.WriteHeader(code)
}

func (c *Ctx) String(code int, formatString string, values ...interface{}) {
    c.SetHeader("Content-Type", "text/plain")
    c.Status(code)
    _, err := fmt.Fprintf(c.Writer, formatString, values...)
    if err != nil {
        http.Error(c.Writer, err.Error(), http.StatusInternalServerError)
    }
}

func (c *Ctx) JSON(code int, data interface{}) {
    c.SetHeader("Content-Type", "application/json")
    c.Status(code)
    encoder := json.NewEncoder(c.Writer)
    err := encoder.Encode(data)
    if err != nil {
        http.Error(c.Writer, err.Error(), http.StatusInternalServerError)
    }
}

func (c *Ctx) HTML(code int, html string) {
    c.SetHeader("Content-Type", "text/html")
    c.Status(code)
    _, err := c.Writer.Write([]byte(html))
    if err != nil {
        http.Error(c.Writer, err.Error(), http.StatusInternalServerError)
    }
}

// request
func (c *Ctx) FormValue(key string) string {
    return c.Req.FormValue(key)
}

func (c *Ctx) Query(key string) string {
    return c.Req.URL.Query().Get(key)
}
```

Our handler interface should look like this:
```go
type Handler interface {
    Serve(c *Ctx)
}
```
Instead of creating a struct for each handler, we can use an adapter to use a function for a simple task, like in the standard net/http package:
```go
type HandleFunc func(c *Ctx)

func (f HandleFunc) Serve(c *Ctx) {
    f(c)
}
``` 

### The static-routing router
Now, we need an object to keep all our static routing configurations and decide which handler to use with each URL and method a.k.a router
In this first version of the web framework, I simply use a map to represent the router with static routing:  
`router.go`
```go
type router struct {
    handlers map[string]Handler
}

func newRouter() *router {
    return &router{
        handlers: make(map[string]Handler),
    }
}

// create route from method and path
func getRoute(method, path string) (route string) {
    var builder strings.Builder
    fmt.Fprintf(&builder, "%s%s%s", method, "-", path)
    route = builder.String()
    return
}

// add a new route to the router
func (r *router) addRoute(method, path string, handler Handler) {
    log.Printf("add route %s %s", method, path)
    route := getRoute(method, path)
    r.handlers[route] = handler
}

//route request to appropriate handler, error code if no handler found
func (r *router) handle(c *Ctx) {
    route := getRoute(c.Method, c.Path)
    if handler, ok := r.handlers[route]; ok {
        handler.Serve(c)
    } else {
        c.Writer.WriteHeader(http.StatusNotFound)
        c.String(http.StatusNotFound, "No route found")
    }
}
```

### Create the interface of caneweb

Lastly, we need a layer for the user to work with the web framework, by hiding the internal implementation of the router and such.
```go
type Engine struct {
    router *router
}

// constructor of cane web framework
func New() *Engine {
    return &Engine{
        router: newRouter(),
    }
}

func (e *Engine) addRoute(method, pattern string, handler Handler) {
    e.router.addRoute(method, pattern, handler)
}

// define some simple operations
func (e *Engine) GET(pattern string, handler Handler) {
    e.addRoute(http.MethodGet, pattern, handler)
}

func (e *Engine) POST(pattern string, handler Handler) {
    e.addRoute(http.MethodPost, pattern, handler)
}

// implement the standard package Handler interface  
// and transform incoming request to our handler
func (e *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ctx := newCtx(w, r)
    e.router.handle(ctx)
}

func (e *Engine) Run(addr string) error {
    return http.ListenAndServe(addr, e)
}

```

## Create a simple server with the framework
`main.go`
```go
package main

import (
    "caneweb/cane"
    "log"
    "net/http"
)

func main() {
    server := cane.New()
    server.GET("/hello", cane.HandleFunc(hello))
    server.POST("/post", cane.HandleFunc(createPost))
    log.Fatal(server.Run(":5445"))
}

func hello(c *cane.Ctx) {
    c.String(http.StatusOK, "hello %s", c.Query("name"))
}

func createPost(c *cane.Ctx) {
    title := c.FormValue("title")
    desc := c.FormValue("desc")
    c.JSON(http.StatusOK, cane.Map{
        "post_title":  title,
        "description": desc,
    })
}
```
Send requests to the web server with curl.
```bash
curl "127.0.0.1:5445/hello?name=duchm"                                     
hello duchm                                                             

curl "http://localhost:5445/post" -X POST -d 'title="first post"&desc="rainny day"'
{"description":"\"rainny day\"","post_title":"\"first post\""}

curl "127.0.0.1:5445/any"                                                  
No route found
```
That's it, after this, we have implemented the prototype version of caneweb framework in go.
Currently, the framework does nothing more than the standard net/http package but worry not, some other nice features will be added in the next versions. 

See the source code at https://github.com/duchng/caneweb





 

