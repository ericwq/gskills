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
## client stub
```c.SayHello()``` is the gRPC client stub. The stub provide the ```"/helloworld.Greeter/SayHello"``` parameter and the ```in``` parameter. 

please note the specification for the method string argument:
* Path → ":path" "/" Service-Name "/" {method name} 
* Service-Name → {IDL-specific service name}

```go
func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
    out := new(HelloReply)                 
    err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)      
    if err != nil {                           
        return nil, err
    }
    return out, nil
}
```
```Invoke()``` apply the ```CallOptions``` first. If there is any interceptor, use interceptor to perform the task, otherwise call the ```invoke()``` function.

```go
   25 // Invoke sends the RPC request on the wire and returns after response is                                                                              
   26 // received.  This is typically called by generated code. 
   27 //                                                                                                           
   28 // All errors returned by Invoke are compatible with the status package.                                 
   29 func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
   30     // allow interceptor to see all applicable call options, which means those             
   31     // configured as defaults from dial option as well as per-call options
   32     opts = combine(cc.dopts.callOptions, opts)                                                                         
   33                                                                                 
   34     if cc.dopts.unaryInt != nil {                                                      
   35         return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)             
   36     }                  
   37     return invoke(ctx, method, args, reply, cc, opts...)             
   38 }                                                                              
```
## fork road
```invoke()``` create a client stream. Here is the main fork road:
* besides create the client stream, ```newClientStream``` will also process the Request-Headers
* ```cs.SendMsg(req)``` will process the Length-Prefixed-Message, 
* EOS is just the flag set for the last data frame.

```go
func invoke(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error {
    cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
    if err != nil {      
        return err      
    }                                                                                                                                     
    if err := cs.SendMsg(req); err != nil {      
        return err                       
    }                                              
    return cs.RecvMsg(reply)      
}      
```
### Request-Headers
```newClientStream``` will create the ```clientStream```, retry the ```op``` function several times until success or error. 

please note:
* ```op``` is a anonymous warpper for the ```a.newStream()```, where ```a``` is the ```csAttempt``` we just created with ```cs.newAttemptLocked()```

```go
func newClientStream(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, opts ...CallOption) (_ ClientStream, err error) {
...
    if err := cs.newAttemptLocked(sh, trInfo); err != nil {
        cs.finish(err)
        return nil, err
    }

    op := func(a *csAttempt) error { return a.newStream() }
    if err := cs.withRetry(op, func() { cs.bufferForRetryLocked(0, op) }); err != nil {
        cs.finish(err)
        return nil, err
    }
...
}
```
```newStream()``` is will create the transport stream.

```go
func (a *csAttempt) newStream() error {       
    cs := a.cs       
    cs.callHdr.PreviousAttempts = cs.numRetries       
    s, err := a.t.NewStream(cs.ctx, cs.callHdr)
    if err != nil {       
        if _, ok := err.(transport.PerformedIOError); ok {
            // Return without converting to an RPC error so retry code can
            // inspect.       
            return err            
        }       
        return toRPCErr(err)     
    }                                            
    cs.attempt.s = s       
    cs.attempt.p = &parser{r: s}       
    return nil       
} 
```
