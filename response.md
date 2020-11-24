#Reply with Response

gRPC over HTTP 2 use HTTP 2 frames. But how to do that exactly? let's explain the detail of implementation of server side response. [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) is a good start point to explain the design of gRPC over HTTP 2. In brief, the gRPC call response is transformed into three parts:

```
Response → (Response-Headers *Length-Prefixed-Message Trailers) / Trailers-Only
```

* A header frame (***Response-Headers***),
* Zero or more data frame (***Length-Prefixed-Message***),
* The final part is ***Trailers***. A special kind of header frame which can be sent after the body. These headers allow for metadata that can’t be calculated up front. In special case, Trailers can be send alone. 

Please refer to the [Send Request](request.md) to get more information about the request. After all we need to know the request to give the correct reply.

## Application code

Here is the gRPC server side application code snippet. It uses ```net.Listen("tcp", port)``` to create the server side listening port. Then it create a server with ```grpc.NewServer() ``` and register the implementation of ```"helloworld.GreeterServer"``` gRPC service. At last, it ```s.Serve(lis)``` the listening port.

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
## Server skeleton
