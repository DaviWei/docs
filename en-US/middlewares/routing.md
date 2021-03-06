---
root: false
name: Routing
sort: 2
---

# Routing

In Macaron, a route is an HTTP method paired with a URL-matching pattern.
Each route can take one or more handler methods:

```go
m.Get("/", func() {
	// show something
})

m.Patch("/", func() {
	// update something
})

m.Post("/", func() {
	// create something
})

m.Put("/", func() {
	// replace something
})

m.Delete("/", func() {
	// destroy something
})

m.Options("/", func() {
	// http options
})

m.Any("/", func() {
	// do anything
})

m.Route("/", "GET,POST", func() {
	// combine something
})

m.Combo("/").
	Get(func() string { return "GET" }).
	Patch(func() string { return "PATCH" }).
	Post(func() string { return "POST" }).
	Put(func() string { return "PUT" }).
	Delete(func() string { return "DELETE" }).
	Options(func() string { return "OPTIONS" }).
	Head(func() string { return "HEAD" })

m.NotFound(func() {
	// Custom handle for 404
})
```

Notes:

- Routes are matched in the order they are defined,
- ...but, narrow range routes have higher priority than wider range routes(e.g.: fixed URLs > regex URLs)
- The first route that matches the request is invoked.

If you want to use suburl without having a huge group indent, use `m.SetURLPrefix(suburl)`.

Route patterns may include named parameters, accessible via the method [`*Context.Params`](https://gowalker.org/github.com/Unknwon/macaron#Context_Params):

```go
m.Get("/hello/:name", func(ctx *macaron.Context) string {
	return "Hello " + ctx.Params(":name")
})

m.Get("/date/:year/:month/:day", func(ctx *macaron.Context) string {
	return fmt.Sprintf("Date: %s/%s/%s", ctx.Params(":year"), ctx.Params(":month"), ctx.Params(":day"))
})
```

Routes can be matched with globs:

```go
m.Get("/hello/*", func(ctx *macaron.Context) string {
	return "Hello " + ctx.Params("*")
})
```

Regular expressions can be used as well:

- Regular match:

	```go
	m.Get("/user/:username([\\w]+)", func(ctx *macaron.Context) string {
		return fmt.Sprintf("Hello %s", ctx.Params(":username"))
	})

	m.Get("/user/:id([0-9]+)", func(ctx *macaron.Context) string {
		return fmt.Sprintf("User ID: %s", ctx.Params(":id"))
	})

	m.Get("/user/*.*", func(ctx *macaron.Context) string {
		return fmt.Sprintf("Last part is: %s", ctx.Params(":path"), ctx.Params(":ext"))
	})
	```

- Mixed match:

	```go
	m.Get("/cms_:id([0-9]+).html", func(ctx *macaron.Context) string {
		return fmt.Sprintf("The ID is %s", ctx.Params(":id"))
	})
	```

- Optional match:
	- `/user/?:id`, matches both `/user/` and `/user/123`.
- Shortcuts:
	- `/user/:id:int`, `:int` is shortcut for `([0-9]+)`.
	- `/user/:name:string`, `:string` is shortcut for `([\w]+)`.

## Advanced Routing

Route handlers can be stacked on top of each other, which is useful for things like authentication and authorization:

```go
m.Get("/secret", authorize, func() {
	// this will execute as long as authorize doesn't write a response
})
```

Let's see an extreme example:

```go
package main

import (
	"fmt"

	"github.com/Unknwon/macaron"
)

func main() {
	m := macaron.Classic()
	m.Get("/",
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = 1
		},
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = ctx.Data["Count"].(int) + 1
		},
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = ctx.Data["Count"].(int) + 1
		},
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = ctx.Data["Count"].(int) + 1
		},
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = ctx.Data["Count"].(int) + 1
		},
		func(ctx *macaron.Context) string {
			return fmt.Sprintf("There are %d handlers before this", ctx.Data["Count"])
		},
	)
	m.Run()
}
```

Guess what's output will be? Yes, `There are 5 handlers before this`. There are no hard limitation of how many handlers you can have for a route, but you may wonder how does Macaron know when to stop calling next handler?

To answer this question, please consider the following example:

```go
package main

import (
	"fmt"

	"github.com/Unknwon/macaron"
)

func main() {
	m := macaron.Classic()
	m.Get("/",
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = 1
		},
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = ctx.Data["Count"].(int) + 1
		},
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = ctx.Data["Count"].(int) + 1
		},
		func(ctx *macaron.Context) {
			ctx.Data["Count"] = ctx.Data["Count"].(int) + 1
		},
		func(ctx *macaron.Context) string {
			return fmt.Sprintf("There are %d handlers before this", ctx.Data["Count"])
		},
		func(ctx *macaron.Context) string {
			return fmt.Sprintf("There are %d handlers before this", ctx.Data["Count"])
		},
	)
	m.Run()
}
```

In this case, the output will always be `There are 4 handlers before this`, and the last handler never gets chance to call. Why? Because we write response in 5th handler. Thus, once any handler writes anything to the response stream, Macaron will stop calling next handler.

### Group Routing

Route groups can be added too using the [`macaron.Group`](https://gowalker.org/github.com/Unknwon/macaron#Router_Group) method:

```go
m.Group("/books", func() {
    m.Get("/:id", GetBooks)
    m.Post("/new", NewBook)
    m.Put("/update/:id", UpdateBook)
    m.Delete("/delete/:id", DeleteBook)

    m.Group("/chapters", func() {
	    m.Get("/:id", GetBooks)
	    m.Post("/new", NewBook)
	    m.Put("/update/:id", UpdateBook)
	    m.Delete("/delete/:id", DeleteBook)
	})
})
```

Just like you can pass middlewares to a handler you can pass middlewares to groups:

```go
m.Group("/books", func() {
    m.Get("/:id", GetBooks)
    m.Post("/new", NewBook)
    m.Put("/update/:id", UpdateBook)
    m.Delete("/delete/:id", DeleteBook)

    m.Group("/chapters", func() {
	    m.Get("/:id", GetBooks)
	    m.Post("/new", NewBook)
	    m.Put("/update/:id", UpdateBook)
	    m.Delete("/delete/:id", DeleteBook)
	}, MyMiddleware3, MyMiddleware4)
}, MyMiddleware1, MyMiddleware2)
```

Still, no hard limitation of how many nested group routes and group level handlers(middlewares) you can have.
