---
title: Memberlist - Gossip Protocol in Go
date: 2023-02-18T14:40:00+01:00
tags: 
    - snippets
    - go
help: |
    voglio scrivere il post che avrei voluto trovare. 
    Quindi quello dove faccio set/get di un valore e questo valore si condivide tra diverse istanzes

---

This post is to show a simple, "hello world" level of gossip protocol communication using [hashicorp/memberlist](https://github.com/hashicorp/memberlist), a Go library that manages cluster membership and member failure detection using a gossip based protocol.

We're going to build a proof-of-concept for this library, with a simple service that get and set a variable whose value is shared across several instances of the services.

Gossip protocol is a *peer-to-peer* communication protocol where nodes in a cluster can broadcast messages to all other nodes, without the need of maintaining a list of the cluster members.
The final code is available [here](https://github.com/totomz/blog-test-memberlist) 

## The basics of hashicorp/memberlist

### Messages
Our nodes will send a receive messages; a message **must** implement the [Broadcast](https://pkg.go.dev/github.com/hashicorp/memberlist#Broadcast) interface. This is a simple implementation to broadcast a `string`:
```go
package main

import "github.com/hashicorp/memberlist"

type Message struct {
	Value string
}

// Message Returns a byte form of the message
func (m *Message) Message() []byte {
	return []byte(m.Value)
}

// Invalidates checks if enqueuing the current broadcast
// invalidates a previous broadcast
func (m *Message) Invalidates(b memberlist.Broadcast) bool {
	return false
}

// Finished is invoked when the message will no longer
// be broadcast, either due to invalidation or to the
// transmit limit being reached
func (m *Message) Finished() {

}
```

### Delegates
Delegate and Events are delegates for receiving and providing data to memberlist via callback mechanisms.
You **must** provide an implementation of the [Delegate](https://pkg.go.dev/github.com/hashicorp/memberlist#Delegate) to hook into the gossip layer of Memberlist. All the methods must be thread-safe, as they can and generally will be called concurrently. 


The [EventDelegate](https://pkg.go.dev/github.com/hashicorp/memberlist#EventDelegate) is an **optional** interface, that defines the callback to get notified by the events in the cluster, as when a node join or leave the pool. 

### Receiving a broadcast messages
To receive a broadcasted message, you must implement the [Delegate](https://pkg.go.dev/github.com/hashicorp/memberlist#Delegate), and provide the implementation to the library (see #wiring all together).
To send/broadcast a message, we need an instance of `memberlist.TransmitLimitedQueue`(more in the #wiring all together)
```go
package main

import "github.com/hashicorp/memberlist"

// SimpleDelegate handles incoming messages, see NotifyMsg
type SimpleDelegate struct {
	// SharedVariableValue keeps avalue shared accross all members of the cluster
	SharedVariableValue atomic.Value
	
	// Broadcasts is the object we use to broadcast a message
	Broadcasts          *memberlist.TransmitLimitedQueue
}

func (delegate *SimpleDelegate) NotifyMsg(message []byte) {
	value := string(message)
	stdout.Printf("GOT MESSAGE: %s", value)
	delegate.SharedVariableValue.Store(value)
}

func (delegate *SimpleDelegate) NodeMeta(limit int) []byte {
	return []byte("")
}


func (delegate *SimpleDelegate) GetBroadcasts(overhead, limit int) [][]byte {
	return delegate.Broadcasts.GetBroadcasts(overhead, limit)
}

func (delegate *SimpleDelegate) LocalState(join bool) []byte {
	// see https://github.com/hashicorp/memberlist/issues/184
	return []byte("")
}

func (delegate *SimpleDelegate) MergeRemoteState(buf []byte, join bool) {
	stdout.Printf("MergeRemoteState?")
}
```

## The full example
This `main.go` showcase how to intercat with the library. The `SimpleDelegate` handles track the state of the cluster: `SharedVariableValue` is a variable whose value is shared accross all cluster members, `Broadcasts` is the service used to broadcast a message, and `func NotifyMsg` handles incoming messages.

```go
package main

import (
	"fmt"
	"github.com/hashicorp/memberlist"
	"log"
	"net/http"
	"os"
	"sync/atomic"
	"time"
)

var stdout = log.New(os.Stdout, "", 0)
var delegate SimpleDelegate

// region: Step1: Implement thebasic interfaces

// SimpleDelegate handles incoming messages, see NotifyMsg
type SimpleDelegate struct {
	// SharedVariableValue keeps avalue shared accross all members of the cluster
	SharedVariableValue atomic.Value

	// Broadcasts is the object we use to broadcast a message
	Broadcasts *memberlist.TransmitLimitedQueue
}

func (delegate *SimpleDelegate) NotifyMsg(message []byte) {
	value := string(message)
	stdout.Printf("GOT MESSAGE: %s", value)
	delegate.SharedVariableValue.Store(value)
}

func (delegate *SimpleDelegate) NodeMeta(limit int) []byte {
	return []byte("")
}

func (delegate *SimpleDelegate) GetBroadcasts(overhead, limit int) [][]byte {
	return delegate.Broadcasts.GetBroadcasts(overhead, limit)
}

func (delegate *SimpleDelegate) LocalState(join bool) []byte {
	// see https://github.com/hashicorp/memberlist/issues/184
	return []byte("")
}

func (delegate *SimpleDelegate) MergeRemoteState(buf []byte, join bool) {
	stdout.Printf("MergeRemoteState?")
}

// Message implements the memberlist.Broadcast interface, to serialize and enqueue a message
type Message struct {
	Value string
}

// Message Returns a byte form of the message
func (m *Message) Message() []byte {
	return []byte(m.Value)
}

// Invalidates checks if enqueuing the current broadcast
// invalidates a previous broadcast
func (m *Message) Invalidates(b memberlist.Broadcast) bool {
	return false
}

// Finished is invoked when the message will no longer
// be broadcast, either due to invalidation or to the
// transmit limit being reached
func (m *Message) Finished() {

}

// endregion

func main() {

	// this is the list of members in the cluster
	var list *memberlist.Memberlist

	// Setup the broadcast queue
	brodcastQueue := new(memberlist.TransmitLimitedQueue)
	// This numeber is the multiplier for each retransmission: if you send a mesage to a node, this node will send themessage to 5 other node.
	// Higher the number, higher the TCP traffic, lower the time to converge to a common state
	brodcastQueue.RetransmitMult = 5
	brodcastQueue.NumNodes = func() int { return len(list.Members()) }

	// Configure the delegate
	delegate = SimpleDelegate{
		SharedVariableValue: atomic.Value{},
		Broadcasts:          brodcastQueue,
	}

	// Setup the library
	// DefaultLANConfig is a sane fedault configuration for a nodes in a LAN
	config := memberlist.DefaultLANConfig()
	config.Delegate = &delegate

	list, err := memberlist.Create(config)
	PanicIfErr(err)

	// Join another node of the cluster.
	// If this is the first node it can't join a cluster - there is no cluster
	joinServer := os.Getenv("NODE")
	if joinServer != "" {
		stdout.Printf("joining node %s", joinServer)
		n, err := list.Join([]string{joinServer})
		if err != nil {
			stdout.Panicf("error joining node: %v", err)
		}
		stdout.Printf("found %v nodes", n)
	}

	// Asynchronously poll the member of the cluster.
	// No need to have this routine, it's just for debug
	go func() {
		for {
			for _, member := range list.Members() {
				stdout.Printf("Members: %s %s\n", member.Name, member.Addr)
			}
			time.Sleep(1 * time.Minute)
		}
	}()

	// Finally, start the web server
	http.HandleFunc("/", requestHandler)
	err = http.ListenAndServe(":3333", nil)
	PanicIfErr(err)

}

func requestHandler(writer http.ResponseWriter, request *http.Request) {
	query := request.URL.Query()
	valueToSet := query.Get("set")

	// if there is a query param, set thevalue; otherwise, read the value
	if valueToSet == "" {
		lastValue := GetValue()
		_, _ = writer.Write([]byte(fmt.Sprintf("last value: %s", lastValue)))
		return
	}

	stdout.Printf("setting value to %s", valueToSet)
	SetValue(valueToSet)
	_, _ = writer.Write([]byte(fmt.Sprintf("SET last value: %s", valueToSet)))

}

func SetValue(valueToSet string) {
	stdout.Printf("setting value %s", valueToSet)
	delegate.Broadcasts.QueueBroadcast(&Message{Value: valueToSet})
}

func GetValue() string {
	return delegate.SharedVariableValue.Load().(string)
}

func PanicIfErr(err error) {
	if err != nil {
		panic(err)
	}
}
```

## Bonus: Local test with docker
You can checkout the full code from this [github.com/totomz/blog-test-memberlist](https://github.com/totomz/blog-test-memberlist). If you want to try [shMake](github.com/totomz/shmake), you can easily start a cluster by opening 3 shell and run this commands:
```shell
# on the first shell, create the docker network and spawn the first container 
# that will listen on port 3031
docker network create gossip
shmake run --port=3031 --node=localhost

# on a separate shall, start 2 more node
# The ip for the first node can be found by inspecting docker:
#  docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container id>
shmake run --port=3032 --node=172.20.0.2
shmake run --port=3033 --node=172.20.0.2


```