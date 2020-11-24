# Reply with Response

gRPC over HTTP 2 use HTTP 2 frames. But how to do that exactly? let's explain the detail of implementation of server side response. [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) is a good start point to explain the design of gRPC over HTTP 2. In brief, the gRPC call response is transformed into three parts:

```
Response → (Response-Headers *Length-Prefixed-Message Trailers) / Trailers-Only
```

* A header frame (***Response-Headers***),
* Zero or more data frame (***Length-Prefixed-Message***),
* The final part is ***Trailers***. A special kind of header frame which can be sent after the body. These headers allow for metadata that can’t be calculated up front. In special case, Trailers can be send alone. 

Please refer to the [Send Request](request.md) to get more information about the request. After all we need to know the request to give the correct reply.

The following diagram is the invocation sequence. It focus on the reply the response: mainly ***Response-Headers***, ***Length-Prefixed-Message*** and ***Trailers***.

![images/images.004.png][images/images.004.png]

## Application code

Here is the gRPC server side application code snippet. It uses ```net.Listen("tcp", port)``` to create the server side listening port. Then it create a gRPC server with ```grpc.NewServer() ``` and register the implementation of ```"helloworld.GreeterServer"``` service on the gRPC server. At last, it uses ```s.Serve(lis)```to serve the the listening port.

Plase note that the ```server.SayHello()``` will be the destination of gRPC request. Let's continue our analysis from ```pb.RegisterGreeterServer()```.  

It's easy, right? Let's move on to see what happens under the hood.

```go
const (                                
    port = ":50051"                                
)                                
                                
// server is used to implement helloworld.GreeterServer.                                
type server struct {                                
    pb.UnimplementedGreeterServer                                
}                                
                                
// SayHello implements helloworld.GreeterServer                                
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {                                
    log.Printf("Received: %v", in.GetName())                                
    return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil                                
}                                
                                
func main() {                                
    lis, err := net.Listen("tcp", port)                                
    if err != nil {                                
        log.Fatalf("failed to listen: %v", err)                                
    }                                
    s := grpc.NewServer()                                
    pb.RegisterGreeterServer(s, &server{})                                
    if err := s.Serve(lis); err != nil {                                
        log.Fatalf("failed to serve: %v", err)                                
    }                                
}                                
```
## Register Service
```go
func RegisterGreeterServer(s grpc.ServiceRegistrar, srv GreeterServer) {                                
    s.RegisterService(&_Greeter_serviceDesc, srv)                                
}                                
                                
func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error)       {                                                                
    in := new(HelloRequest)                                
    if err := dec(in); err != nil {                                
        return nil, err                                
    }                                
    if interceptor == nil {                                
        return srv.(GreeterServer).SayHello(ctx, in)                                
    }                                
    info := &grpc.UnaryServerInfo{                                
        Server:     srv,                                
        FullMethod: "/helloworld.Greeter/SayHello",                                
    }                                
    handler := func(ctx context.Context, req interface{}) (interface{}, error) {                                
        return srv.(GreeterServer).SayHello(ctx, req.(*HelloRequest))                                
    }                                
    return interceptor(ctx, in, info, handler)                                
}                                
                                
var _Greeter_serviceDesc = grpc.ServiceDesc{                                
    ServiceName: "helloworld.Greeter",                                
    HandlerType: (*GreeterServer)(nil),                                
    Methods: []grpc.MethodDesc{                                
        {                                
            MethodName: "SayHello",                                
            Handler:    _Greeter_SayHello_Handler,                                
        },                                
    },                                
    Streams:  []grpc.StreamDesc{},                                
    Metadata: "examples/helloworld/helloworld/helloworld.proto",                                
}  
```
## Server skeleton

```go
```
