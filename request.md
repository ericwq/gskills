# Sending Request

gRPC over HTTP 2 use HTTP 2 frames. But how to do that exactly? let's explain the detail of implementation of gRPC request. [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) is a good start point to explain the design of gRPC over HTTP 2. In brief, the gRPC call request is transformed into three parts: 
```
Request → Request-Headers *Length-Prefixed-Message EOS. 
```
* a header frame (Request-Headers), 
* zero or more data Frame (Length-Prefixed-Message), 
* the final part is EOS(end of stream) is a flag, set in the last data frame.

## application code
Here is the gRPC client application code snippet. we use ```c := pb.NewGreeterClient(conn)``` to create the connection. and call ```r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})``` to send the request over HTTP 2.

```go
    // Set up a connection to the server.                                       
    conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
    if err != nil {                                      
        log.Fatalf("did not connect: %v", err)
    }                                             
    defer conn.Close()                                                                                                                 
    c := pb.NewGreeterClient(conn)
                                         
    // Contact the server and print out its response.      
    name := defaultName              
    if len(os.Args) > 1 {                                                                                                                                    
        name = os.Args[1]      
    }      
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)      
    defer cancel()      
    r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})      
    if err != nil {                                  
        log.Fatalf("could not greet: %v", err)            
    }                               
    log.Printf("Greeting: %s", r.GetMessage())     
```
