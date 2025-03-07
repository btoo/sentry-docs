---
title: Martini
---

## Martini Handler for Sentry-go SDK

**Godoc:** https://godoc.org/github.com/getsentry/sentry-go/martini
**Example:** https://github.com/getsentry/sentry-go/tree/master/example/martini

### Installation

```sh
go get github.com/getsentry/sentry-go/martini
```

```go
import (
    "fmt"

    "github.com/getsentry/sentry-go"
    sentrymartini "github.com/getsentry/sentry-go/martini"
    "github.com/go-martini/martini"
)

// To initialize Sentry's handler, you need to initialize Sentry itself beforehand
if err := sentry.Init(sentry.ClientOptions{
    Dsn: "your-public-dsn",
}); err != nil {
    fmt.Printf("Sentry initialization failed: %v\n", err)
}

// Then create your app
app := martini.Classic()

// Once it's done, you can attach the handler as one of your middleware
app.Use(sentrymartini.New(sentrymartini.Options{}))

// Set up routes
app.Get("/", func() string {
    return "Hello world!"
})

// And run it
app.Run()
```

### Configuration

`sentrymartini` accepts a struct of `Options` that allows you to configure how the handler will behave.

Currently it respects 3 options:

```go
// Whether Sentry should repanic after recovery, in most cases it should be set to true,
// as martini.Classic includes its own Recovery middleware that handles http responses.
Repanic         bool
// Whether you want to block the request before moving forward with the response.
// Because Martini's default `Recovery` handler doesn't restart the application,
// it's safe to either skip this option or set it to `false`.
WaitForDelivery bool
// Timeout for the event delivery requests.
Timeout         time.Duration
```

### Usage

`sentrymartini` maps an instance of `*sentry.Hub` (https://godoc.org/github.com/getsentry/sentry-go#Hub) as one of the services available throughout the rest of the request's lifetime.
You can access it by providing a `hub *sentry.Hub` parameter in any of your proceeding middleware and routes.
And it should be used instead of the global `sentry.CaptureMessage`, `sentry.CaptureException`, or any other calls, as it keeps the separation of data between the requests.

**Keep in mind that `*sentry.Hub` won't be available in middleware attached before `sentrymartini`!**

```go
app := martini.Classic()

app.Use(sentrymartini.New(sentrymartini.Options{
    Repanic: true,
}))

app.Use(func(rw http.ResponseWriter, r *http.Request, c martini.Context, hub *sentry.Hub) {
    hub.Scope().SetTag("someRandomTag", "maybeYouNeedIt")
})

app.Get("/", func(rw http.ResponseWriter, r *http.Request, hub *sentry.Hub) {
    if someCondition {
        hub.WithScope(func (scope *sentry.Scope) {
            scope.SetExtra("unwantedQuery", rw.URL.RawQuery)
            hub.CaptureMessage("User provided unwanted query string, but we recovered just fine")
        })
    }
    rw.WriteHeader(http.StatusOK)
})

app.Get("/foo", func() string {
    // sentrymartini handler will catch it just fine. Also, because we attached "someRandomTag"
    // in the middleware before, it will be sent through as well
    panic("y tho")
})

app.Run()
```

#### Accessing Request in `BeforeSend` callback

```go
sentry.Init(sentry.ClientOptions{
    Dsn: "your-public-dsn",
    BeforeSend: func(event *sentry.Event, hint *sentry.EventHint) *sentry.Event {
        if hint.Context != nil {
            if req, ok := hint.Context.Value(sentry.RequestContextKey).(*http.Request); ok {
                // You have access to the original Request here
            }
        }

        return event
    },
})
```
