# Interceptor

gPRC Interceptor is a powerful mechanism to add addtional logic such as logging, authentication, authorization, metrics, tracing, and any other customer requirements.

Here we only talk about server side interceptor. client side interceptor is very similer to the server side.

### Interceptor chain execution flow
There are two function type. ```UnaryHandler``` and ```UnaryServerInterceptor```. Each implementation of ```UnaryServerInterceptor``` is an interceptor. The interceptor chain is just a set of interceptors, e.g. simple slice: ```[]UnaryServerInterceptor```. 

plse note:

* ```UnaryServerInterceptor``` function has an ```UnaryHandler``` argument. with it, interceptor can invoke the target service.
* They both share the same return type: ```(interface{}, error)```. with it, handler can invoke the next interceptor in the chain.

Theis is one of the key pints. please remember it. 

```go
// UnaryHandler defines the handler invoked by UnaryServerInterceptor to complete the normal
// execution of a unary RPC. If a UnaryHandler returns an error, it should be produced by the
// status package, or else gRPC will use codes.Unknown as the status code and err.Error() as
// the status message of the RPC.
type UnaryHandler func(ctx context.Context, req interface{}) (interface{}, error)
                         
// UnaryServerInterceptor provides a hook to intercept the execution of a unary RPC on the server. info
// contains all the information of this RPC the interceptor can operate on. And handler is the wrapper      
// of the service method implementation. It is the responsibility of the interceptor to invoke handler   
// to complete the RPC.
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```
gRPC can prepare the interceptor chain when it's ready to create a gRPC server. in ```NewServer()``` func, ```chainUnaryServerInterceptors(s)``` will chain the interceptors together.
```go
// NewServer creates a gRPC server which has no service registered and has not       
// started to accept requests yet.       
func NewServer(opt ...ServerOption) *Server {       
    opts := defaultServerOptions       
    for _, o := range opt {       
        o.apply(&opts)       
    }       
    s := &Server{       
        lis:      make(map[net.Listener]bool),       
        opts:     opts,       
        conns:    make(map[transport.ServerTransport]bool),       
        services: make(map[string]*serviceInfo),       
        quit:     grpcsync.NewEvent(),       
        done:     grpcsync.NewEvent(),       
        czData:   new(channelzData),       
    }       
    chainUnaryServerInterceptors(s)
    chainStreamServerInterceptors(s)       
    s.cv = sync.NewCond(&s.mu)                                                                                                                    
```
Inside ```chainUnaryServerInterceptors(s)```, it will prepare a slice ```[]UnaryServerInterceptor``` and put all interceptors into it. Then it will build an ***entry Interceptor*** : ```chainedInt```. The ***entry Interceptor*** is an anonymous interceptor, it's an interceptor wrapper. Who's implementation is to invoke the first element of interceptor chain with ```getChainUnaryHandler(interceptors, 0, info, handler)``` as the parameter. 

before ```chainUnaryServerInterceptors(s)``` return, this anonymous interceptor will not be called. because of the wrapper.

```go
// chainUnaryServerInterceptors chains all unary server interceptors into one.
func chainUnaryServerInterceptors(s *Server) {
    // Prepend opts.unaryInt to the chaining interceptors if it exists, since unaryInt will                                                                
    // be executed before any other chained interceptors.
    interceptors := s.opts.chainUnaryInts
    if s.opts.unaryInt != nil {
        interceptors = append([]UnaryServerInterceptor{s.opts.unaryInt}, s.opts.chainUnaryInts...)
    }

    var chainedInt UnaryServerInterceptor
    if len(interceptors) == 0 {
        chainedInt = nil
    } else if len(interceptors) == 1 {
        chainedInt = interceptors[0]
    } else {
        chainedInt = func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (interface{}, error) {
            return interceptors[0](ctx, req, info, getChainUnaryHandler(interceptors, 0, info, handler))
        }
    }

    s.opts.unaryInt = chainedInt
}

// getChainUnaryHandler recursively generate the chained UnaryHandler
func getChainUnaryHandler(interceptors []UnaryServerInterceptor, curr int, info *UnaryServerInfo, finalHandler UnaryHandler) UnaryHandler {
    if curr == len(interceptors)-1 {
        return finalHandler
    }

    return func(ctx context.Context, req interface{}) (interface{}, error) {
        return interceptors[curr+1](ctx, req, info, getChainUnaryHandler(interceptors, curr+1, info, finalHandler))
    }
}
```
please note:

* ```getChainUnaryHandler``` return an anonymous ```UnaryHandler``` or the ```finalHandler```. The anonymous wrapper ensure the func body will not be called at this moment.
* Inside the anonymous ```UnaryHandler```, it will call the next interceptor in the chain. That because the ```UnaryServerInterceptor``` and ```UnaryHandler``` share the same return type, otherwise we can't return the interceptor directly inside the ```UnaryHandler``` *shell*.

Now the interceptor chain is ready . The execution flow is as following.
![source code](images/images.002.png)

1. start with interceptor[0] and getChainUnaryHandler(0), before the preProcessing, getChainUnaryHandler(0) is called and return the handler wrapper.
2. now run the preProcessing part of interceptor[0], 
3. then the  handler wrapper is called. inside the wrapper, call interceptor[1] with getChainUnaryHandler(1) as parameter.
4. continue the recursioin, until getChainUnaryHandler(2) return  finalHandler.
5. after finalHandler is called, the postProcessing part of interceptor[2] start, then interceptor[2] finished
6. control return to the interceptor[1] postProcessing part.
7. continue the recursioin, until we interceptor[0] finished.

now we only have the interceptor chain, we need a entry point to launch this chain.   

### Launch Intercetor
gRPC will generate the following code for you to register the business service. Please note the ServiceName, MethodName and Handler. Your business service will not be used directly, a Handler will be used.
```go
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

func RegisterGreeterServer(s grpc.ServiceRegistrar, srv GreeterServer) {
    s.RegisterService(&_Greeter_serviceDesc, srv)
}      
```
The ```methodHandler``` is also generated by gRPC. It perform two jobs:

* use ```dec``` to decode the incoming message and build the HelloRequest objest.
* if any interceptor (chain) exist, wrap the GreeterServer.SayHello with ```UnaryHandler``` and use it to call the interceptor.

```_Greeter_SayHello_Handler``` is the entry point to launch the interceptor chain, while this entry point need to be called by the gRPC.

please note:
*  ```_Greeter_SayHello_Handler``` is not ```UnaryHandler```, it's a ```methodHandler```.

```go
func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
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

type methodHandler func(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor UnaryServerInterceptor) (interface{}, error)
```
Inside gRPC , when the server receive a RPC call request, base on the information from the header frame, the server can find which ```methodHandler``` to be called. Then the ```_Greeter_SayHello_Handler``` get called.

```go
    d, err := recvAndDecompress(&parser{r: stream}, stream, dc, s.opts.maxReceiveMessageSize, payInfo, decomp)
    if err != nil { 
        if e := t.WriteStatus(stream, status.Convert(err)); e != nil {
            channelz.Warningf(logger, s.channelzID, "grpc: Server.processUnaryRPC failed to write status %v", e)
        }      
        return err
    }                          
    if channelz.IsOn() {      
        t.IncrMsgRecv()
    }                                       
    df := func(v interface{}) error {                                                     
        if err := s.getCodec(stream.ContentSubtype()).Unmarshal(d, v); err != nil {                                                                                         return status.Errorf(codes.Internal, "grpc: error unmarshalling request: %v", err)
        }       
        if sh != nil {       
            sh.HandleRPC(stream.Context(), &stats.InPayload{       
                RecvTime:   time.Now(),       
                Payload:    v,       
                WireLength: payInfo.wireLength + headerLen,       
                Data:       d,       
                Length:     len(d),       
            })       
        }       
        if binlog != nil {       
            binlog.Log(&binarylog.ClientMessage{       
                Message: d,       
            })       
        }       
        if trInfo != nil {       
            trInfo.tr.LazyLog(&payload{sent: false, msg: v}, true)       
        }       
        return nil       
    }       
    ctx := NewContextWithServerTransportStream(stream.Context(), stream)       
    reply, appErr := md.Handler(info.serviceImpl, ctx, df, s.opts.unaryInt)       

```
### Use Interceptor
