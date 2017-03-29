---
layout: post
title: Building Microservices from scratch using gRPC on Ruby
---

Today, REST with JSON is the most popular framework amongst web developers for network communication. But, it is not very suitable for a microservice architecture mainly because of latency added by JSON data transmission / serializing / deserializing. 

My quest for finding an optimal network communication framework for microservices brought me to gRPC.  
"*gRPC is a modern, open source remote procedure call (RPC) framework that can run anywhere. It enables client and server applications to communicate transparently, and makes it easier to build connected systems.*"

To read more about benefits of gRPC, visit the official site [here](http://www.grpc.io/faq/).

Serialization in gRPC is based on [Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview), a language and platform independent serialization mechanism for structured data.   
"*Protocol buffers are a flexible, efficient, automated mechanism for serializing structured data – think XML, but smaller, faster, and simpler.*"

In the remaining part of this post, I will be walking you through setting up a simple gRPC server from scratch on Ruby. Let's build *Snip* - a dummy URL shortener!

We will divide our code structure into 3 separate repositories:

1. **`snip`** : contains the proto definitions and converted ruby files for client communication. Basically, this is like an interface between client and server, specifying the RPC methods, and the request an response formats.

2. **`snip-service`** : Service implementation for the RPC methods (This is where the gRPC server sits). 

3. **`X-app`** : This is any application who wishes to call snip-service for shortening URLs. 
 
**`snip`** will be packaged as a gem, and included in both **`snip-service`** and **`X-app`**.

### Step 1: Define proto files

Let's create a new file `snip.proto`

```proto
syntax = "proto3";
package snip;

service UrlSnipService {
    rpc snip_it(SnipRequest) returns (SnipResponse) {}
}

message SnipRequest {
    string url = 1;
}

message SnipResponse {
    string url = 1;
}
```



