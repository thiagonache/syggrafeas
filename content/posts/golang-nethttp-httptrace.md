---
title: "Golang net/http httptrace"
date: 2022-05-24T08:24:53-03:00
draft: true
---

## The problem

The HTTP protocol is a pretty reliable and fast but on the other hand complex. It works on top of others protocols, such as TCP, and also relies on another application protocols, such as DNS. In scenarios where the requests starts to take more time than usual, we need to be able to inspect all the steps that occours within the protocol and not only the total time that the request took.

The steps involved in a simple HTTP request are:

1. DNS Lookup: The client tries to resolve the domain name for the request.

   a) Client sends DNS Query to local ISP DNS server.

   b) DNS server responds with the IP address for hostname.com

1. Connect: Client establishes TCP connection with the IP address of hostname.com

   a) Client sends SYN packet.

   b) Web server sends SYN-ACK packet.

   c) Client answers with ACK packet, concluding the three-way TCP connection establishment.

1. Send: Client sends the HTTP request to the web server.

1. Client waits for the server to respond to the request.

   a) Web server processes the request, finds the resource, and sends the response to the Client. Client receives the first byte of the first packet from the web server, which contains the HTTP Response headers and content.

1. Load: Client loads the content of the response.

   a) Web server sends second TCP segment with the PSH flag set.

   b) Client sends ACK. (Client sends ACK every two segments it receives. from the host)

   c) Web server sends third TCP segment with HTTP_Continue.

1. Close: Client sends a a FIN packet to close the TCP connection.

## The solution

Golang has a collection of features for performing HTTP calls within `nethttp` package. Today, I'm going to cover the package [httptrace](https://pkg.go.dev/net/http/httptrace) demonstrating how to collect and measure http internal time, such as DNS resolution or time to open the connection.

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

## Conclusion

HTTP tracing is a valuable addition to Go for those who are interested in debugging HTTP request latency and writing tools for network debugging for outbound traffic.
