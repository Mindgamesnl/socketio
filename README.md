# socketio

[socket.io](https://socket.io/)/[engine.io](https://github.com/socketio/engine.io) implementation in Go

# Fork
Maintained fork of https://github.com/zyxar/socketio with bug fixes.
Changes compared to the original:
 - Fixed client error handling
 - Fixed client disconnect event
 - Fixed client disconnect trigger (auto reconnect loop now doesn't get stuck)
 - Minor improvements
 - Merged room implementation
 - Added support for multiple handlers per event
 - TLS support

## Install

```shell
go get -v -u github.com/Mindgamesnl/socketio
```

## Features

- compatible with official nodejs implementation;
- `socket.io` server;
- `socket.io` client (`websocket` only);
- `engine.io` server;
- `engine.io` client (`websocket` only);
- binary data;
- namespace support;
- [socket.io-msgpack-parser](https://github.com/darrachequesne/socket.io-msgpack-parser) support;


## Example

Server:
```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/Mindgamesnl/socketio"
)

func main() {
	server, _ := socketio.NewServer(time.Second*25, time.Second*5, socketio.DefaultParser)
	server.Namespace("/").
		OnConnect(func(so socketio.Socket) {
			log.Println("connected:", so.RemoteAddr(), so.Sid(), so.Namespace())
		}).
		OnDisconnect(func(so socketio.Socket) {
			log.Printf("%v %v %q disconnected", so.Sid(), so.RemoteAddr(), so.Namespace())
		}).
		OnError(func(so socketio.Socket, err ...interface{}) {
			log.Println("socket", so.Sid(), so.RemoteAddr(), so.Namespace(), "error:", err)
		}).
		OnEvent("message", func(so socketio.Socket, data string) {
			log.Println(data)
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
	so.Emit("ack", "foo", func(msg string) {
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
	server.Namespace("/").OnEvent("foobar", func(data string) (string, string) {
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
	server.Namespace("/").
		OnEvent("binary", func(data interface{}, b *socketio.Bytes) {
			log.Println(data)
			bb, _ := b.MarshalBinary()
			log.Printf("%x", bb)
		}).
		OnConnect(func(so socketio.Socket) {
			go func() {
				for {
					select {
					case <-time.After(time.Second * 2):
						if err := so.Emit("event", "check it out!", time.Now()); err != nil {
							log.Println(err)
							return
						}
					}
				}
			}()
		})
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
	server.Namespace("/ditto").OnEvent("disguise", func(msg interface{}, b socketio.Bytes) {
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


## TLS Terminator example using `golang.org/x/crypto/acme/autocert`

```go
type WrappedServer struct {
	OriginalHandler http.Handler
}

func (wrapper WrappedServer) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	res.Header().Set("Access-Control-Allow-Origin", "*")
	res.Header().Set("Access-Control-Allow-Methods", "GET, HEAD, POST, OPTIONS")
	res.Header().Set("Access-Control-Allow-Headers", "Content-Type")
	wrapper.OriginalHandler.ServeHTTP(res, req)
}

func StartSocketServer() {
	server, _ := socketio.NewServer(time.Second*25, time.Second*5, socketio.DefaultParser)
	
	certManager := autocert.Manager{
		Prompt:     autocert.AcceptTOS,
		HostPolicy: autocert.HostWhitelist("socket.example.com"), // your domain
		Cache:      autocert.DirCache("certs"),                   // cache folder
		Email:      "example@example.com",                        // owner email
	}

	actualServer := &http.Server{
		Addr: ":https",
		TLSConfig: &tls.Config{
			GetCertificate: certManager.GetCertificate,
			ServerName:     "socket.example.com",
		},
	}

	wrapper := WrappedServer{
		OriginalHandler: server,
	}
	actualServer.Handler = wrapper
	
	go http.ListenAndServe(":http", certManager.HTTPHandler(nil))
	log.Fatal(actualServer.ListenAndServeTLS("", ""))
}
```
