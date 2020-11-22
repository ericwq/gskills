# Interceptor

gPRC Interceptor is a powerful mechanism to add addtional logic such as logging, authentication, authorization, metrics, tracing, and any other customer requirements.

let's talk about server side interceptor first. client side interceptor is rarely used in practice.
### Interceptor chain execution flow
There are two functions type. ```UnaryHandler``` and ```UnaryServerInterceptor```. each implementation of type ```UnaryServerInterceptor``` is an interceptor function. while interceptor chain is just a simple slice: ```[]UnaryServerInterceptor```. 

plse note:

* ```UnaryServerInterceptor``` function has an ```UnaryHandler``` argument. 
* They both share the same return type: ```(interface{}, error)```. 

The above two points are one of the key design. 

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
gRPC need to prepare the interceptor chain when it's ready to create a gRPC server. in ```NewServer()``` func, ```chainUnaryServerInterceptors(s)``` will chain the Interceptors together.
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
in ```chainUnaryServerInterceptors(s)```, it will prepare a ```[]UnaryServerInterceptor``` and put all interceptors into it. Then it will build an entry Interceptor: ```chainedInt```. who's implementation is to invoke the first element of interceptor chain with ```getChainUnaryHandler(interceptors, 0, info, handler)``` as the last parameter.

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

* ```getChainUnaryHandler``` return an anonymous ```UnaryHandler``` func. the wrapper ensure the func body will not be invoked before the return.
* ```chainedInt``` return an anonymous ```UnaryServerInterceptor``` func. this is also a wrapper for it.

The above points are one of the key design. While the interceptor chain is ready . The execution flow is as follwing.
![source code](images/images.002.png)

### Launch Intercetor
![source code](images/images.004.png)
