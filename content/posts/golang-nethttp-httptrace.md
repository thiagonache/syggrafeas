---
title: "Golang net/http httptrace"
date: 2022-05-24T08:24:53-03:00
draft: true
---

Golang has a collection of features for performing HTTP calls within `nethttp` package. Today, I'm going to cover the package [httptrace](https://pkg.go.dev/net/http/httptrace) demonstrating how to collect and measure http internal time, such as DNS resolution or time to open the connection.

## httptrace

The package provides an abstraction layer in order to run code when certain HTTP events happens. It is implemented thru the type `ClientTrace` via hooks that are dispatched on HTTP outgoing requests. The struct [ClientTrace](https://pkg.go.dev/net/http/httptrace#ClientTrace) is very well documented so I won't cover all hooks but here's a list of some interesting ones:

- DNSStart func(DNSStartInfo): DNSStart is called when a DNS lookup begins.
- DNSDone func(DNSDoneInfo): DNSDone is called when a DNS lookup ends.
- ConnectStart func(network, addr string): ConnectStart is called when a new connection's Dial begins.
- ConnectDone func(network, addr string, err error): ConnectDone is called when a new connection's Dial completes. The provided err indicates whether the connection completed successfully.
- TLSHandshakeStart func(): TLSHandshakeStart is called when the TLS handshake is started.
- TLSHandshakeDone func(tls.ConnectionState, error): TLSHandshakeDone is called after the TLS handshake with either the successful handshake's connection state, or a non-nil error on handshake failure.
- GotConn func(GotConnInfo): GotConn is called after a successful connection is obtained.

The `GotConnInfo` type is defined as the following.

```go
type GotConnInfo struct {
	// Conn is the connection that was obtained. It is owned by
	// the http.Transport and should not be read, written or
	// closed by users of ClientTrace.
	Conn net.Conn

	// Reused is whether this connection has been previously
	// used for another HTTP request.
	Reused bool

	// WasIdle is whether this connection was obtained from an
	// idle pool.
	WasIdle bool

	// IdleTime reports how long the connection was previously
	// idle, if WasIdle is true.
	IdleTime time.Duration
}
```

As you can see, we can use it to determine whether a request was reused or not.

## Implementation

In order to trace the call you should add an `*httptrace.ClientTrace` into a request's `context.Context` after writing functions to gather the information you want.

### Storing stats

The first thing we'll need to have is some place to store when the times. For that, let's create a struct called `stats`.

```go
type stats struct {
	ConnectStart time.Time
	ConnectTime  time.Duration
	DNSStart     time.Time
	DNSTime      time.Duration
}
```

For now, let's just track the time to resolve the DNS and the time to connect only, meaning that the time to transfer the body won't be considered.

### Calculating http call statistics

Let's now create our main function.

```go
func main() {
	stats := stats{}
	req, err := http.NewRequest("GET", "https://www.google.com", nil)
	if err != nil {
		log.Fatal("Cannot create a new GET request to https://www.google.com")
	}
	trace := &httptrace.ClientTrace{
		DNSStart:     func(info httptrace.DNSStartInfo) { stats.DNSStart = time.Now() },
		DNSDone:      func(info httptrace.DNSDoneInfo) { stats.DNSTime = time.Since(stats.DNSStart) },
		ConnectStart: func(network, addr string) { stats.ConnectStart = time.Now() },
		ConnectDone: func(network, addr string, err error) {
			if err == nil {
				stats.ConnectTime = time.Since(stats.ConnectStart)
			}
		},
	}
	clientTraceCtx := httptrace.WithClientTrace(req.Context(), trace)
	req = req.WithContext(clientTraceCtx)
	_, err = http.DefaultClient.Do(req)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("HTTP stats:")
	fmt.Printf("DNS %v\n", stats.DNSTime)
	fmt.Printf("Connection %v\n", stats.ConnectTime)
}
```
