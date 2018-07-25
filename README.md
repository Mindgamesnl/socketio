# socketio

[socket.io](https://socket.io/)/[engine.io](https://github.com/socketio/engine.io) implementation in Go

[![GoDoc](https://godoc.org/github.com/zyxar/socketio?status.svg)](https://godoc.org/github.com/zyxar/socketio)
[![Go Report Card](https://goreportcard.com/badge/github.com/zyxar/socketio)](https://goreportcard.com/report/github.com/zyxar/socketio)
[![license](https://img.shields.io/badge/license-New%20BSD-ff69b4.svg)](https://github.com/zyxar/socketio/blob/master/LICENSE)
[![Build Status](https://travis-ci.org/zyxar/socketio.svg?branch=master)](https://travis-ci.org/zyxar/socketio)


## Install

```shell
vgo get -v -u github.com/zyxar/socketio
```

## Features

- compatible with official nodejs implementation (w/o room);
- `socket.io` server;
- `socket.io` client (`websocket` only);
- `engine.io` server;
- `engine.io` client (`websocket` only);
- binary data;
- limited namespace support;
- [socket.io-msgpack-parser](https://github.com/darrachequesne/socket.io-msgpack-parser) support;


## Example

Server:
```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/zyxar/socketio"
)

func main() {
	server, _ := socketio.NewServer(time.Second*25, time.Second*5, socketio.DefaultParser)
	server.OnConnect(func(so socketio.Socket) error {
		so.On("/", "message", func(data string) {
			log.Println(data)
		})
		so.OnError(func(nsp string, err interface{}) {
			log.Printf("socket nsp=%q, error=%v", nsp, err)
		})
		return so.Emit("/", "event", "hello world!")
	})
	http.ListenAndServe(":8081", server)
}
```
Client:
```js
const io = require('socket.io-client');
const socket = io('http://localhost:8081');
var id;

socket.on('connect', function() {
  console.log('connected');
  if (id === undefined) {
    id = setInterval(function() {
      socket.emit('message', 'hello there!')
    }, 2000);
  }
});
socket.on('event', console.log);
socket.on('disconnect', function() {
  console.log('disconnected');
  if (id) {
    clearInterval(id);
    id = undefined;
  }
});
```

### With Acknowledgements

- Server -> Client

Server:
```go
	so.Emit("/", "ack", "foo", func(msg string) {
		log.Println(msg)
	})
```
Client:
```js
  socket.on('ack', function(name, fn) {
    console.log(name);
    fn('bar');
  })
```

- Client -> Server

Server:
```go
	so.On("/", "foobar", func(data string) (string, string) {
		log.Println("foobar:", data)
		return "foo", "bar"
	})
```

Client:
```js
  socket.emit('foobar', '-wow-', function (foo, bar) {
    console.log('foobar:', foo, bar);
  });
```

### With Binary Data

Server:
```go
	so.On("/", "binary", func(data interface{}, b *socketio.Bytes) {
		log.Println(data)
		bb, _ := b.MarshalBinary()
		log.Printf("%x", bb)
	})
	go func() {
		for {
			select {
			case <-time.After(time.Second * 2):
				if err := so.Emit("/", "event", "check it out!", time.Now()); err != nil {
					log.Println(err)
					return
				}
			}
		}
	}()
```

Client:
```js
  var ab = new ArrayBuffer(4);
  var a = new Uint8Array(ab);
  a.set([1,2,3,4]);

  id = setInterval(function() {
    socket.emit('binary', 'buf:', ab);
  }, 2000);

  socket.on('event', console.log);
```

#### Binary Helper for `protobuf`

```go
import (
	"github.com/golang/protobuf/proto"
)

type ProtoMessage struct {
	proto.Message
}

func (p ProtoMessage) MarshalBinary() ([]byte, error) {
	return proto.Marshal(p.Message)
}

func (p *ProtoMessage) UnmarshalBinary(b []byte) error {
	return proto.Unmarshal(b, p.Message)
}
```

#### Binary Helper for `MessagePack`

```go
import (
	"github.com/tinylib/msgp/msgp"
)

type MessagePack struct {
	Message interface {
		msgp.MarshalSizer
		msgp.Unmarshaler
	}
}

func (m MessagePack) MarshalBinary() ([]byte, error) {
	return m.Message.MarshalMsg(nil)
}

func (m *MessagePack) UnmarshalBinary(b []byte) error {
	_, err := m.Message.UnmarshalMsg(b)
	return err
}
```


### Customized Namespace

Server:
```go
	so.On("/ditto", "disguise", func(msg interface{}, b socketio.Bytes) {
		bb, _ := b.MarshalBinary()
		log.Printf("%v: %x", msg, bb)
	})
```

Client:
```js
let ditto = io('http://localhost:8081/ditto');
ditto.emit('disguise', 'pidgey', new ArrayBuffer(8));
```

## Parser

The `encoder` and `decoder` provided by `socketio.DefaultParser` is compatible with [`socket.io-parser`](https://github.com/socketio/socket.io-parser/), complying with revision 4 of [socket.io-protocol](https://github.com/socketio/socket.io-protocol).

An `Event` or `Ack` Packet with any data satisfying `socketio.Binary` interface (e.g. `socketio.Bytes`) would be encoded as `BinaryEvent` or `BinaryAck` Packet respectively.

`socketio.MsgpackParser`, compatible with [socket.io-msgpack-parser](https://github.com/darrachequesne/socket.io-msgpack-parser), is an alternative custom parser.
