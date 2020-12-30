# Load Balancing - gRPC part
- [Name resolving](#name-resolving)
  - [exampleResolver](#exampleresolver)
  - [dnsResolver](#dnsresolver)

Let's discuss the name resolving example and load balancing example from [gRPC examples](https://github.com/grpc/grpc-go/tree/master/examples). Through this discussion, we can learn how to perform the name resolving and load balancing tasks. We can also learn load balancing related internal components of gRPC. 

gRPC provides a rich set of load balancing machanism. As Envoy becomes the de facto standard. gRPC add the support to xDS protocol. We will discuss the xDS protocol in next chapter.

## Name resolving

Here we will discuss `name_resolving` example. This examples shows how application can pick different name resolvers. Please read the example READEME.md before continue.

Please refer to the the following diagram (left side) and Resolver API to understand the whole design, see [Balancer and Resolver API](dial.md#balancer-and-resolver-api) for general summary.

![image.008.png](../images/images.008.png)

```go
// Builder creates a resolver that will be used to watch name resolution updates.                                                                               
type Builder interface {                                                                                                   
    // Build creates a new resolver for the given target.                                                                  
    //                                                                                                                                                     
    // gRPC dial calls Build synchronously, and fails if the returned error is                                              
    // not nil.                                                                                                                                         
    Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)                                                
    // Scheme returns the scheme supported by this resolver.                                                                                
    // Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.                                                     
    Scheme() string                                                                                                                                  
}                                                                                                                                                                  
                                                                                                                           
// ResolveNowOptions includes additional information for ResolveNow.                                                        
type ResolveNowOptions struct{}                                                                                                              
                                                                                                                                        
// Resolver watches for the updates on the specified target.                                                                                      
// Updates include address updates and service config updates.                                                                                   
type Resolver interface {                                                                                                                      
    // ResolveNow will be called by gRPC to try to resolve the target name                                                  
    // again. It's just a hint, resolver can ignore this if it's not necessary.                                                                  
    //                                                                                                                                   
    // It could be called multiple times concurrently.                                                                     
    ResolveNow(ResolveNowOptions)                                                                                                                
    // Close closes the resolver.                                                                                           
    Close()                                                                                                                                         
}                                                                                                                                        

// ClientConn contains the callbacks for resolver to notify any updates                                                     
// to the gRPC ClientConn.                                                                                                                  
//                                                                                                                                      
// This interface is to be implemented by gRPC. Users should not need a                                                                              
// brand new implementation of this interface. For the situations like                                                                                             
// testing, the new implementation should embed this interface. This allows                                                
// gRPC to add new methods to this interface.                                                                               
type ClientConn interface {                                                                                                                  
    // UpdateState updates the state of the ClientConn appropriately.                                                                   
    UpdateState(State)                                                                                                                            
    // ReportError notifies the ClientConn that the Resolver encountered an                                                                      
    // error.  The ClientConn will notify the load balancer and begin calling                                                                  
    // ResolveNow on the Resolver with exponential backoff.                                                                 
    ReportError(error)                                                                                                                           
    // NewAddress is called by resolver to notify ClientConn a new list                                                                  
    // of resolved addresses.                                                                                              
    // The address list should be the complete list of resolved addresses.                                                                       
    //                                                                                                                      
    // Deprecated: Use UpdateState instead.                                                                                                         
    NewAddress(addresses []Address)                                                                                                      
    // NewServiceConfig is called by resolver to notify ClientConn a new                                                                                       
    // service config. The service config should be provided as a json string.                                             
    //                                                                                                                     
    // Deprecated: Use UpdateState instead.                                                                                                       
    NewServiceConfig(serviceConfig string)                                                                                                        
    // ParseServiceConfig parses the provided service config and returns an                                                 
    // object that provides the parsed config.                                                                                                    
    ParseServiceConfig(serviceConfigJSON string) *serviceconfig.ParseResult                                                              
}                                                                                                                                      

// State contains the current Resolver state relevant to the ClientConn.
type State struct {
    // Addresses is the latest set of resolved addresses for the target.
    Addresses []Address

    // ServiceConfig contains the result from parsing the latest service
    // config.  If it is nil, it indicates no service config is present or the
    // resolver does not provide service configs.
    ServiceConfig *serviceconfig.ParseResult

    // Attributes contains arbitrary data about the resolver intended for
    // consumption by the load balancing policy.
    Attributes *attributes.Attributes                                                                                                                      
}                                                                                                                           
                                                                                                                                                        

```
### exampleResolver

exampleResolver is created to resolve `resolver.example.grpc.io` to `localhost:50051`. 
- The start point is the `init()` function. In `init()`,  `exampleResolverBuilder` is registerd in resover map `map[string]Builder`.
- In [newCCResolverWrapper()](dial.md#newccresolverwrapper), `DialContext()` parses the `target` string and calls `getResolver()` to find the registerd `resolver.Builder`.
- Next, `DialContext()` calls `newCCResolverWrapper()`, in `newCCResolverWrapper()`, `rb.Build()` will be called to build the resolver.
- In our case, `rb.Build()` is actually `exampleResolverBuilder.Build()`.  
- In `exampleResolverBuilder.Build()`, `r.start()` will be called.
- In `exampleResolverBuilder.Build()`, `addrsStore` will be initialized by 
```
        map[string][]string{                                                                                   
            exampleServiceName: {backendAddr},                                                                             
        }
``` 
- where `exampleServiceName="resolver.example.grpc.io"` and `backendAddr="localhost:50051"`
- `exampleResolverBuilder.start()` will calls [ccResolverWrapper.UpdateState()](dial.md#ccresolverwrapperupdatestate), a.k.a `r.cc.UpdateState()` to update the the `resolver.State` to `ccResolverWrapper`, 
- the new `resolver.State` whose `Addresses` field has value: `["localhost:50051"]`. 
- In [ccResolverWrapper.UpdateState()](dial.md#ccresolverwrapperupdatestate), `ccr.poll()` goroutine will call `ccr.resolveNow()`. 
- `ccr.resolveNow()` calls `ccr.resolver.ResolveNow()`, which actually calls`exampleResolver.ResolveNow()`.
- In our case, `exampleResolver.ResolveNow()` do nothing. It's the designed behavior of the `exampleResolver`.
- Next, following the [Client Dial](dial.md) process, a new connection with the `localhost:50051` will be established.
- `exampleResolver` is very similar to `passthroughResolver`: they both has `start()` method, in `start()` they both call `r.cc.UpdateState()`

```go
func (cc *ClientConn) getResolver(scheme string) resolver.Builder {
    for _, rb := range cc.dopts.resolvers {
        if scheme == rb.Scheme() {              
            return rb
        }
    }                                
    return resolver.Get(scheme)                                                    
}                                                                                     

// Following is an example name resolver. It includes a                                                                    
// ResolverBuilder(https://godoc.org/google.golang.org/grpc/resolver#Builder)                                              
// and a Resolver(https://godoc.org/google.golang.org/grpc/resolver#Resolver).                                             
//                                                                                                                         
// A ResolverBuilder is registered for a scheme (in this example, "example" is                                             
// the scheme). When a ClientConn is created for this scheme, the                                                          
// ResolverBuilder will be picked to build a Resolver. Note that a new Resolver                                            
// is built for each ClientConn. The Resolver will watch the updates for the                                               
// target, and send updates to the ClientConn.                                                                             
                                                                                                                           
// exampleResolverBuilder is a                                                                                             
// ResolverBuilder(https://godoc.org/google.golang.org/grpc/resolver#Builder).                                             
type exampleResolverBuilder struct{}                                                                                       
                                                                                                                           
const (                                                                                      
    exampleScheme      = "example"                                      
    exampleServiceName = "resolver.example.grpc.io"                                
                                          
    backendAddr = "localhost:50051"                                
)                                                                     

func (*exampleResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
    r := &exampleResolver{                                                                                                 
        target: target,                                                                                                    
        cc:     cc,                                                                                                        
        addrsStore: map[string][]string{                                                                                   
            exampleServiceName: {backendAddr},                                                                             
        },                                                                                                                 
    }                                                                                                                      
    r.start()                                                                                                              
    return r, nil                                                                                                          
}                                                
func (*exampleResolverBuilder) Scheme() string { return exampleScheme }

// exampleResolver is a
// Resolver(https://godoc.org/google.golang.org/grpc/resolver#Resolver).
type exampleResolver struct {
    target     resolver.Target
    cc         resolver.ClientConn
    addrsStore map[string][]string
}

func (r *exampleResolver) start() {
    addrStrs := r.addrsStore[r.target.Endpoint]
    addrs := make([]resolver.Address, len(addrStrs))
    for i, s := range addrStrs {
        addrs[i] = resolver.Address{Addr: s}
    }
    r.cc.UpdateState(resolver.State{Addresses: addrs})
}
func (*exampleResolver) ResolveNow(o resolver.ResolveNowOptions) {}
func (*exampleResolver) Close()                                  {}

func init() {
    // Register the example ResolverBuilder. This is usually done in a package's
    // init() function.
    resolver.Register(&exampleResolverBuilder{})
}

```
### dnsResolver
Here is the code snippet of `dnsBuilder` and `dnsResolver`.  `dnsResolver`'s behavior is different:
- `dnsBuilder.Build()` starts a new goroutine `dnsResolver.watcher()` and calls `dnsResolver.ResolveNow()` to send a signal to `dnsResolver.watcher()`
- Upon receive the signal from `dnsResolver.ResolveNow()`, `dnsResolver.watcher()` goroutine stops waiting and performs `dnsResolver.lookup()` to lookup the corresponding IP address.
- If the `dnsResolver.lookup()` successes, It will call `ccResolverWrapper.UpdateState()` to notify the new `resolver.State` 
- Remember `exampleResolverBuilder.start()` calling `ccResolverWrapper.UpdateState()`? From there the dialing process continues.

```go
// Build creates and starts a DNS resolver that watches the name resolution of the target.
func (b *dnsBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
    host, port, err := parseTarget(target.Endpoint, defaultPort)
    if err != nil {
        return nil, err
    }

    // IP address.
    if ipAddr, ok := formatIP(host); ok {
        addr := []resolver.Address{{Addr: ipAddr + ":" + port}}
        cc.UpdateState(resolver.State{Addresses: addr})
        return deadResolver{}, nil
    }

    // DNS address (non-IP).
    ctx, cancel := context.WithCancel(context.Background())
    d := &dnsResolver{
        host:                 host,
        port:                 port,
        ctx:                  ctx,
        cancel:               cancel,
        cc:                   cc,
        rn:                   make(chan struct{}, 1),
        disableServiceConfig: opts.DisableServiceConfig,
    }

    if target.Authority == "" {
        d.resolver = defaultResolver
    } else {                                                       
        d.resolver, err = customAuthorityResolver(target.Authority)
        if err != nil {
            return nil, err
        }
    }

    d.wg.Add(1)
    go d.watcher()
    d.ResolveNow(resolver.ResolveNowOptions{})
    return d, nil
}
// Scheme returns the naming scheme of this resolver builder, which is "dns".
func (b *dnsBuilder) Scheme() string {
    return "dns"
}
type dnsResolver struct {
    host     string
    port     string
    resolver netResolver
    ctx      context.Context
    cancel   context.CancelFunc
    cc       resolver.ClientConn
    // rn channel is used by ResolveNow() to force an immediate resolution of the target.
    rn chan struct{}
    // wg is used to enforce Close() to return after the watcher() goroutine has finished.
    // Otherwise, data race will be possible. [Race Example] in dns_resolver_test we
    // replace the real lookup functions with mocked ones to facilitate testing.
    // If Close() doesn't wait for watcher() goroutine finishes, race detector sometimes
    // will warns lookup (READ the lookup function pointers) inside watcher() goroutine
    // has data race with replaceNetFunc (WRITE the lookup function pointers).
    wg                   sync.WaitGroup
    disableServiceConfig bool
}

// ResolveNow invoke an immediate resolution of the target that this dnsResolver watches.
func (d *dnsResolver) ResolveNow(resolver.ResolveNowOptions) {
    select {
    case d.rn <- struct{}{}:
    default:
    }
}

// Close closes the dnsResolver.
func (d *dnsResolver) Close() {
    d.cancel()
    d.wg.Wait()
}

func (d *dnsResolver) watcher() {
    defer d.wg.Done()
    for {
        select {
        case <-d.ctx.Done():
            return
        case <-d.rn:
        }

        state, err := d.lookup()
        if err != nil {
            d.cc.ReportError(err)
        } else {
            d.cc.UpdateState(*state)
        }

        // Sleep to prevent excessive re-resolutions. Incoming resolution requests
        // will be queued in d.rn.
        t := time.NewTimer(minDNSResRate)
        select {
        case <-t.C:
        case <-d.ctx.Done():
            t.Stop()
            return
        }
    }
}

```
