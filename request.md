# Sending Request

gRPC call over HTTP 2 use HTTP 2 frame. But how to do it? let's explain the detail of implementation of gRPC request and gRPC resonse. [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) is a good start point to understand what are going to hanppen. 

Request → Request-Headers *Length-Prefixed-Message EOS. 

Here A Request is composed by a header frame (Request-Headers), zero or more data Frame (Length-Prefixed-Message), the EOS is a flag set in the last data frame.
