---
title: "User Guide to Generic Call for Streaming [on trial]"
date: 2024-09-19
weight: 6
keywords: ["User Guide to Generic Call for Streaming [on trial]"]
description: ""
---

## Introduction

JSON generic call for streaming interfaces is now supported (client-side only).

**However, due to the ongoing restructuring of the streaming interface to improve the user experience and to avoid making any breaking changes after release, the feature's official release is temporarily on hold. If you have a need for this feature, you can refer to the following to try it out. Please note that once it is officially released, a small adjustment may be required on your end**.

Please try the feature with kitex test branch test/grpc_json_streaming_generic by `go get github.com/cloudwego/kitex@test/grpc_json_streaming_generic`.

## Usage

### Generic Streaming Client Initialization

#### protobuf

Take the following Protobuf IDL as an example:

```protobuf
syntax = "proto3";
package pbapi;
// The greeting service definition.
option go_package = "pbapi";

message Request {
  string message = 1;
}

message Response {
  string message = 1;
}

service Echo {
  rpc StreamRequestEcho (stream Request) returns (Response) {}
  rpc StreamResponseEcho (Request) returns (stream Response) {}
  rpc BidirectionalEcho (stream Request) returns (stream Response) {}
  rpc UnaryEcho (Request) returns (Response) {}
}
```

The four methods included in the example IDL correspond to four scenarios:

1. Client streaming: the client sends multiple messages, the server returns one message, and then closes the stream.
2. Server streaming: The client sends one message, the server returns multiple messages, and then closes the stream. It's suitable for LLM scenarios.
3. Bidirectional streaming: The sending and receiving of client/server are independent, which can be organized in arbitrary order.
4. Unary: non streaming.

First of all, please initialize the streaming client. Here is an example of streaming client initialization.

```go
dOpts := dproto.Options{} // you can specify parsing options as you want
p, err := generic.NewPbFileProviderWithDynamicGo(your_idl, ctx, dOpts)
// create json pb generic
g, err := generic.JSONPbGeneric(p)
// streaming client initialization
cli, err := genericclient.NewStreamingClient("destService", g,
    client.WithTransportProtocol(transport.GRPC),
    client.WithHostPorts(targetIPPort),
)
```

#### thrift

Take the following Thrift IDL as an example:

```thrift
namespace go thrift

struct Request {
    1: required string message,
}

struct Response {
    1: required string message,
}

service TestService {
    Response Echo (1: Request req) (streaming.mode="bidirectional"),
    Response EchoClient (1: Request req) (streaming.mode="client"),
    Response EchoServer (1: Request req) (streaming.mode="server"),
    Response EchoUnary (1: Request req) (streaming.mode="unary"), // not recommended
    Response EchoBizException (1: Request req) (streaming.mode="client"),

    Response EchoPingPong (1: Request req), // KitexThrift, non-streaming
}
```

The four methods included in the example IDL correspond to four scenarios:

1. Client streaming: the client sends multiple messages, the server returns one message, and then closes the stream.
2. Server streaming: The client sends one message, the server returns multiple messages, and then closes the stream. It's suitable for LLM scenarios.
3. Bidirectional streaming: The sending and receiving of client/server are independent, which can be organized in arbitrary order.
4. Unary (gRPC): Non-streaming. With `streaming.mode` annotation. Not recommended due to performance loss.
5. Unary (KitexThrift): Non-streaming. Recommended.

Here is an example of streaming client initialization.

```go
p, err := generic.generic.NewThriftFileProvider(your_idl_path)
/*
// if you use dynamicgo
p, err := generic.NewThriftFileProviderWithDynamicGo(idl)
*/

// create json thrift generic
g, err := generic.JSONThriftGeneric(p)
// streaming client initialization
cli, err := genericclient.NewStreamingClient("destService", g,
    client.WithTransportProtocol(transport.GRPC),
    client.WithHostPorts(targetIPPort),
)
```

### Client Streaming

Example:

```go
// initialize client streaming client using the streaming client you created
streamCli, err := genericclient.NewClientStreaming(ctx, cli, "StreamRequestEcho")
for i := 0; i < 3; i++ {
    req := fmt.Sprintf(`{"message": "grpc client streaming generic %dth request"}`, i)
    // Send
    err = streamCli.Send(req)
    time.Sleep(time.Second)
}
// Recv
resp, err := streamCli.CloseAndRecv()
strResp, ok := resp.(string) // response is json string
```

### Server Streaming

Note: A non-nil error (including `io.EOF`) returned by `Recv` indicates that the server has finished sending (or encountered an error)

Example:

```go
// initialize server streaming client using the streaming client you created, and send a message
streamCli, err := genericclient.NewServerStreaming(ctx, cli, "StreamResponseEcho", `{"message": "grpc server streaming generic request"}`)
for {
    resp, err := streamCli.Recv()
    if err == io.EOF {
       log.CtxInfof(stream.Context(), "serverStreaming message receive done. stream is closed")
       break
    } else if err != nil {
       klog.CtxInfof(stream.Context(), "failed to recv: "+err.Error())
       break
    } else {
       strResp, ok := resp.(string) // response is json string
       klog.CtxInfof(stream.Context(), "serverStreaming message received: %+v", strResp)
    }
}
```

### Bidirectional Streaming

Example:

```go
// initialize bidirectional streaming client using the streaming client you created
streamCli, err := genericclient.NewBidirectionalStreaming(ctx, cli, "BidirectionalEcho")

wg := &sync.WaitGroup{}
wg.Add(2)

// Send
go func() {
    defer func() {
       if p := recover(); p != nil {
          err = fmt.Errorf("panic: %v", p)
       }
       wg.Done()
    }()
    // Tell the server there'll be no more message from client
    defer streamCli.Close()
    for i := 0; i < 3; i++ {
       req := fmt.Sprintf(`{"message": "grpc bidirectional streaming generic %dth request"}`, i)
       if err = streamCli.Send(req); err != nil {
          klog.Warnf("bidirectionalStreaming send: failed, err = " + err.Error())
          break
       }
       klog.Infof("BidirectionalStreamingTest send: req = %+v", req)
    }
}()

// Recv
go func() {
    defer func() {
       if p := recover(); p != nil {
          err = fmt.Errorf("panic: %v", p)
       }
       wg.Done()
    }()
    for {
       resp, err := streamCli.Recv()
       if err == io.EOF {
          log.CtxInfof(stream.Context(), "bidirectionalStreaming message receive done. stream is closed")
          break
       } else if err != nil {
          klog.CtxInfof(stream.Context(), "failed to recv: "+err.Error())
          break
       } else {
          strResp, ok := resp.(string) // response is json string
          klog.CtxInfof(stream.Context(), "bidirectionalStreaming message received: %+v", strResp)
       }
    }
}()
wg.Wait()
```

### Unary

The usage of unary call is similar to normal (non-streaming) generic call.

Example:

```go
resp, err := cli.GenericCall(ctx, "UnaryEcho", `{"message": "unary request"}`)
strResp, ok := resp.(string) // response is json string
```