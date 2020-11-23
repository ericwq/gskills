# Sending Request

gRPC call over HTTP 2 use HTTP 2 frame. But how to do it? let's explain the detail of implementation of gRPC request. [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) is a good start point to explain the design of gRPC over HTTP 2. In brief, the gRPC call request is transformed into three parts: 
```
Request → Request-Headers *Length-Prefixed-Message EOS. 
```
* a header frame (Request-Headers), 
* zero or more data Frame (Length-Prefixed-Message), 
* the final part is EOS(end of stream) is a flag, set in the last data frame.
