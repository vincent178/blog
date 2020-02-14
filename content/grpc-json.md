---
title: "Grpc Json"
date: 2020-02-11T12:56:08+08:00
draft: true
---



### GRPC architecture

IDL (interface definition language)

gRPC is an RPC framework from Google. It uses protocol buffers as the underlying serialization/IDL format. At the transport layer it uses HTTP/2 for request/response multiplexing. Envoy has first class support for gRPC both at the transport layer as well as at the application layer:

### Protobuf

by default, use `lowerCamelCase` as the JSON name, JSON parser should accept both converted `lowerCamelCase` and proto field name.

descriptor contiann information found in `.proto` file, and makes the information accessible.

### Envoy

Network level filter (HTTP connection manager) convert raw bytes to HTTP messages.

Envoy configuration is all proto, so check proto definition is the best way to figure out how to configue anything.

HTTP filters applies on HTTP message (within HTTP connection manager)


```json
{
  "proto_descriptor": "...",
  "proto_descriptor_bin": "...",
  "services": [],
  "print_options": "{...}",
  "match_incoming_request_route": "...",
  "ignored_query_parameters": [],
  "auto_mapping": "...",
  "ignore_unknown_query_parameters": "...",
  "convert_grpc_status": "..."
}
```

`admin` is required to configure the administration server.
`static_resource` specify static configuration when envoy starts.


Envoy has a built in network level filter called HTTP connection manager. This filter translate row bytes into HTTP level messages and events.
It also handles functionality access logging, request ID generation and tracing, request/response header manipulation, route table management and statistics.


*Cluster*: 
Envoy discover members of a cluster via service discovery, use health checking determines the health of cluster members, route a request with load balancing policy.

Envoy recommend a single Envoy per machine

GRPC timestmap <-> RFC 3339 (https://tools.ietf.org/html/rfc3339)

what is in grpc?
Struct
Wrapper types
FieldMask
ListValue
Value

1. https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/grpc_json_transcoder_filter
1. https://developers.google.com/protocol-buffers/docs/proto3#json
1. https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Trailer
