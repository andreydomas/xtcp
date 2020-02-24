# xtcp

A TCP Server Framework with graceful shutdown,custom protocol.

[![Build Status](https://travis-ci.org/xfxdev/xtcp.svg?branch=master)](https://travis-ci.org/xfxdev/xtcp)
[![Go Report Card](https://goreportcard.com/badge/github.com/xfxdev/xtcp)](https://goreportcard.com/report/github.com/xfxdev/xtcp)
[![GoDoc](https://godoc.org/github.com/xfxdev/xtcp?status.svg)](https://godoc.org/github.com/xfxdev/xtcp)


## Install

```sh
go get github.com/xfxdev/xlog # xtcp use xlog inner.
go get github.com/xfxdev/xtcp
```

## Usage

### define your protocol format:
Before create server and client, you need define the protocol format first.

```go
// Packet is the unit of data.
type Packet interface {
	fmt.Stringer
}

// Protocol use to pack/unpack Packet.
type Protocol interface {
	// return the size need for pack the Packet.
	PackSize(p Packet) int
	// PackTo pack the Packet to w.
	// The return value n is the number of bytes written;
	// Any error encountered during the write is also returned.
	PackTo(p Packet, w io.Writer) (int, error)
	// Pack pack the Packet to new created buf.
	Pack(p Packet) ([]byte, error)
	// try to unpack the buf to Packet. If return len > 0, then buf[:len] will be discard.
	// The following return conditions must be implement:
	// (nil, 0, nil) : buf size not enough for unpack one Packet.
	// (nil, len, err) : buf size enough but error encountered.
	// (p, len, nil) : unpack succeed.
	Unpack(buf []byte) (Packet, int, error)
}
```

### provide event handler:
In xtcp, there are some events to notify the state of net conn, you can handle them according your need:

```go
const (
	// EventAccept mean server accept a new connect.
	EventAccept EventType = iota
	// EventConnected mean client connected to a server.
	EventConnected
	// EventRecv mean conn recv a packet.
	EventRecv
	// EventClosed mean conn is closed.
	EventClosed
)
```

To handle the event, just implement the OnEvent interface.

```go
// Handler is the event callback.
// p will be nil when event is EventAccept/EventConnected/EventClosed
type Handler interface {
	OnEvent(et EventType, c *Conn, p Packet)
}
```

### create server:

```go
// 1. create protocol and handler.
// ...

// 2. create opts.
opts := xtcp.NewOpts(handler, protocol)

// 3. create server.
server := xtcp.NewServer(opts)

// 4. start.
go server.ListenAndServe("addr")
```

### create client:

```go
// 1. create protocol and handler.
// ...

// 2. create opts.
opts := xtcp.NewOpts(handler, protocol)

// 3. create client.
client := NewConn(opts)

// 4. start
go client.DialAndServe("addr")
```

### send and recv packet.
To send data, just call the 'Send' function of Conn. You can safe call it in any goroutines.

```go
func (c *Conn) Send(buf []byte) error
```

To recv a packet, implement your handler function:

```go
func (h *myhandler) OnEvent(et EventType, c *Conn, p Packet) {
	switch et {
		case EventRecv:
			...
	}
}
```

### stop

xtcp have three stop modes, stop gracefully mean conn will stop until all cached data sended.

```go
// StopMode define the stop mode of server and conn.
type StopMode uint8

const (
	// StopImmediately mean stop directly, the cached data maybe will not send.
	StopImmediately StopMode = iota
	// StopGracefullyButNotWait stop and flush cached data.
	StopGracefullyButNotWait
	// StopGracefullyAndWait stop and block until cached data sended.
	StopGracefullyAndWait
)
```

## Example
The example define a protocol format which use protobuf inner.
You can see how to define the protocol and how to create server and client.

[example](https://github.com/xfxdev/xtcp/tree/master/example)
