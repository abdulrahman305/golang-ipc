# golang-ipc
 Golang Inter-process communication library for Window, Mac and Linux.


 ### Overview
 
 A simple to use package that uses unix sockets on Macos/Linux and named pipes on Windows to create a communication channel between two go processes.

### Intergration

As well as using this library just for go processes it was also designed to work with other languages, with the go process as the server and the other languages processing being the client.


#### NodeJs

I currently use this library to comunicate between a ElectronJS GUI and a go program.

Below is a link to the nodeJS client library:

https://github.com/james-barrow/node-ipc-client

#### Python 

To do

## Usage

Create a server with the default configuation and start listening for the client:

```go

	s, err := ipc.StartServer("<name of socket or pipe>", nil)
	if err != nil {
		log.Println(err)
		return
	}

```
Create a client and connect to the server:

```go

	c, err := ipc.StartClient("<name of socket or pipe>", nil)
	if err != nil {
		log.Println(err)
		return
	}

```

### Read messages 

Read each message sent:

```go

    for {

        // message, err := s.Read() server 
        message, err := c.Read() // client

        if err == nil {
            // handle error
        }

        // do something with the received messages
    }

```

All received messages are formated into the type Message

```go

type Message struct {
	Err     error  // details of any error
	MsgType int    // 0 = reserved , -1 is an internal message (disconnection or error etc), all messages recieved will be > 0
	Data    []byte // message data received
	Status  string // the status of the connection
}

```

### Write a message


```go

	//err := s.Write(1, []byte("<Message for client"))
    err := c.Write(1, []byte("<Message for server"))

    if err == nil {
        // handle error
    }

```

## Detailed Usage Examples

### Example 1: Simple Echo Server

This example demonstrates a simple echo server that listens for incoming messages from a client and sends the same message back to the client.

```go
package main

import (
	"fmt"
	"log"
	"time"

	"github.com/james-barrow/golang-ipc"
)

func main() {
	// Start the server
	server, err := ipc.StartServer("echo_server", nil)
	if err != nil {
		log.Fatal(err)
	}

	// Start the client
	client, err := ipc.StartClient("echo_server", nil)
	if err != nil {
		log.Fatal(err)
	}

	// Send a message from the client to the server
	err = client.Write(1, []byte("Hello, Server!"))
	if err != nil {
		log.Fatal(err)
	}

	// Read the message on the server side
	go func() {
		for {
			message, err := server.Read()
			if err != nil {
				log.Println(err)
				return
			}
			fmt.Printf("Server received: %s\n", string(message.Data))

			// Echo the message back to the client
			err = server.Write(1, message.Data)
			if err != nil {
				log.Println(err)
				return
			}
		}
	}()

	// Read the echoed message on the client side
	go func() {
		for {
			message, err := client.Read()
			if err != nil {
				log.Println(err)
				return
			}
			fmt.Printf("Client received: %s\n", string(message.Data))
		}
	}()

	// Keep the main function running
	select {}
}
```

### Example 2: Custom Configuration

This example demonstrates how to use custom configurations for the server and client.

```go
package main

import (
	"log"
	"time"

	"github.com/james-barrow/golang-ipc"
)

func main() {
	// Custom server configuration
	serverConfig := &ipc.ServerConfig{
		Encryption:        false,
		MaxMsgSize:        1024,
		UnmaskPermissions: true,
	}

	// Start the server with custom configuration
	server, err := ipc.StartServer("custom_server", serverConfig)
	if err != nil {
		log.Fatal(err)
	}

	// Custom client configuration
	clientConfig := &ipc.ClientConfig{
		Encryption: false,
		Timeout:    5,
		RetryTimer: 2,
	}

	// Start the client with custom configuration
	client, err := ipc.StartClient("custom_server", clientConfig)
	if err != nil {
		log.Fatal(err)
	}

	// Send a message from the client to the server
	err = client.Write(1, []byte("Hello, Custom Server!"))
	if err != nil {
		log.Fatal(err)
	}

	// Read the message on the server side
	go func() {
		for {
			message, err := server.Read()
			if err != nil {
				log.Println(err)
				return
			}
			log.Printf("Server received: %s\n", string(message.Data))
		}
	}()

	// Read the message on the client side
	go func() {
		for {
			message, err := client.Read()
			if err != nil {
				log.Println(err)
				return
			}
			log.Printf("Client received: %s\n", string(message.Data))
		}
	}()

	// Keep the main function running
	select {}
}
```

## Configuration Options and Explanations

### Server Configuration

The `ServerConfig` struct allows you to customize the server's behavior. Here are the available options:

- `Encryption` (bool): Allows encryption to be switched off (default is true).
- `MaxMsgSize` (int): The maximum size in bytes of each message (default is 3145728 / 3MB).
- `UnmaskPermissions` (bool): Makes the socket writable for other users (default is false).

### Client Configuration

The `ClientConfig` struct allows you to customize the client's behavior. Here are the available options:

- `Encryption` (bool): Allows encryption to be switched off (default is true).
- `Timeout` (float64): Number of seconds to wait before timing out trying to connect/reconnect (default is 0, no timeout).
- `RetryTimer` (time.Duration): Number of seconds to wait before connection retry (default is 20).

### Encryption

By default, the connection established will be encrypted. ECDH384 is used for the key exchange, and AES 256 GCM is used for the cipher.

Encryption can be switched off by passing a custom configuration to the server and client start functions:

```go
	Encryption: false
```

### Unix Socket Permissions

Under most configurations, a socket created by a user will by default not be writable by another user, making it impossible for the client and server to communicate if being run by separate users.

The permission mask can be dropped during socket creation by passing a custom configuration to the server start function. **This will make the socket writable for any user.**

```go
	UnmaskPermissions: true	
```
Note: Tested on Linux, not tested on Mac, not implemented on Windows.

## Testing

The package has been tested on Mac, Windows, and Linux and has extensive test coverage.

## License

MIT
