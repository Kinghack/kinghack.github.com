---
layout: post
title: "Investigate the 3 occurrence of panic among 30m request"
description: ""
category: 
tags: []
---
{% include JB/setup %}
First, let me introduce the *background*. We have one endpoint (http service build with [golang](https://golang.org)) are searching data from the database and return to the client. Recently we would like to switch this part of searching behavior to [elasticsearch](https://www.elastic.co/products/elasticsearch). And after about one weeks' 30% production traffic testing, we switched 100% traffic to elasticsearch. 


At first, everything goes well. But after one hour, it happened 2 **panic** which *never* happen before. After half hour, 1 more panic **happened** again. For stability consideration, we switch back to the database. And then start our investigating. 


The first step is trying to understand what happened exactly. Luckily, we have the full stack. The error message said that :

`fatal error: concurrent map read and map write. 
` 

And it happened at this line of code:
 
`logging.Error(logTag, "context=%v,err=%v", ctx, err)
` 

'logging' is our internal log package.  

So here is *two* question:


One is what happened at that time which caused error make out program go to log error? 

The other one is why the pure log action would cause panic?

With the help of DevOps team, it is clear that one of the nodes in elasticsearch is down at the time of first two panic. And the second time panic happened when there is a network issue on that machine. Both leads that the machine could not access the elasticsearch cluster successfully which lead to log fail action. So the first question is **answered**. The remaining is question two: why log would panic?

After searching the log, we could see that only 3 panics happened. All the others log *successfully*. And with the help of stack, we could know that some concurrent read and map write happened. Because we are trying to log the context so I am thinking maybe I could reproduce it. So I have a try.

```
package main
// go run -race main.go
import (
    "context"
    "fmt"
    "time"
)
func log(input interface{}) {
    fmt.Sprintf("%v", input)
    // fmt.Println(time.Now())
}
func main() {
    a := make(chan bool)
    ctx := context.WithValue(context.Background(), "key", "value")
    mapStruc := map[time.Time]time.Time{
        time.Now(): time.Now(),
    }
    fmt.Println(mapStruc)
    fmt.Println(ctx)
    go func() {
        for {
            // map example
            // log(mapStruc)
            // context example
            log(ctx)
        }
    }()
    go func() {
        for {
            // map example
            // mapStruc[time.Now()] = time.Now()
            // context example
            // ctx = context.WithValue(ctx, "key", time.Now())
        }
    }()
    <-a
    return
}
```

Unfortunately, I could reproduce with the map but not context. Context example would panic but not because of concurrent read but **over stack limit**. At this point, I am thinking maybe some of the middleware in our service do some magic of context. So I add the middleware of test code, still could not reproduce the panic. 

After a second thought, I am thinking since it is so clear that it is due to map structure. I could also set a map in context as value. The test result proves my guess. 

```
 m := map[bool]bool{
        true: false,
    }
 ctx := context.WithValue(context.Background(), "map", m)
```


After reproducing the possibility of panic, the question remaining is which part of data in context is written while logging? 


So I print all the map structure in context.

```
listeners:map[net.Listener]struct {}{(*net.TCPListener)(0xc420406008):struct {}{}}
TLSNextProto:map[string]func(*http.Server, *tls.Conn, http.Handler){“h2”:(func(*http.Server, *tls.Conn, http.Handler))(0x42a0c70)}
“http.url.parameters”:map[string]string{}
activeConn:map[*http.conn]struct {}{(*http.conn)(0xc4207080a0):struct {}{}}
```

Before diving into detail, I would like to know how this structure is inserted in context?  It is clear after checking with the golang [source code](https://github.com/golang/go/blob/master/src/net/http/server.go#L2721).
 

``` 
baseCtx := context.Background() // base is always background, per Issue 16220
 ctx := context.WithValue(baseCtx, ServerContextKey, srv)
```

So let me back to the context detail. My doubt is that it is due to **activeConn**. It will change while connection status change of a service. The remaining thing is to prove it. So I write the code. 

```
package main

import (
 "fmt"
 "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
 fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
 // below would cause panic
 fmt.Println(r.Context())
}

func main() {
 http.HandleFunc("/", handler)
 http.ListenAndServe(":8080", nil)
}
```

It works well when I visit the test endpoint by the web browser. With the help of [ab](https://httpd.apache.org/docs/2.4/programs/ab.html) load testing tool, I could create the heavy load and so the connection status would change which reproduce the panic finally. 
 
The factor is clear now. The **root cause** of panic is due to concurrent map read and map write. It happened when the connection on server open or close while do logging context. 


The lesson learned: Only play with context if you do understand what you are doing.


