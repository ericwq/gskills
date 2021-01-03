# Load Balance - Client
- [Name resolving](#name-resolving)
  - [exampleResolver](#exampleresolver)
  - [dnsResolver](#dnsresolver)
- [Load balancing](#load-balancing)
  - [defaultServiceConfigRawJSON](#defaultserviceconfigrawjson)
  - [service config and dnsResovler](#service-config-and-dnsresovler)
  - [defaultServiceConfig](#defaultserviceconfig)
  - [UpdateClientConnState()](#updateclientconnstate)
  - [UpdateSubConnState()](#updatesubconnstate)
  - [newAttemptLocked()](#newattemptlocked)
  
Let's discuss the name resolving example and load balancing example from [gRPC examples](https://github.com/grpc/grpc-go/tree/master/examples). Through this discussion, we can learn how to perform the name resolving and load balancing tasks. We can also learn load balancing related internal components of gRPC. 

gRPC provides a rich set of load balancing machanism. As Envoy becomes the de facto standard. gRPC adds the support to xDS protocol. We will discuss the xDS protocol in seperated chapter.

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
Here is the code snippet of `dnsBuilder` and `dnsResolver`.  `dnsResolver`'s behavior is different from `exampleResolver`:
- `dnsBuilder.Build()` starts a new goroutine `dnsResolver.watcher()` and calls `dnsResolver.ResolveNow()` to send a signal to `dnsResolver.watcher()`
- Upon receive the signal from `dnsResolver.ResolveNow()`, `dnsResolver.watcher()` goroutine stops waiting and performs `dnsResolver.lookup()` to lookup the corresponding IP address.
- If the `dnsResolver.lookup()` successes, It will call `ccResolverWrapper.UpdateState()` to notify the new `resolver.State` 
- Remember `exampleResolverBuilder.start()` calling `ccResolverWrapper.UpdateState()`? From `ccResolverWrapper.UpdateState()` the [Client Dial](dial.md) process continues.

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

## Load balancing

In this example, two servers will listen on `"localhost:50051"`, `"localhost:50052"` and provide the same echo service. `dial()` will connect to those two servers. RPC method call will use `round_robin` policy to pick up one server to finish the RPC call. Please note, this is client side load balancing. Client side load balancing needs client to know the backend server list in advance. In the large scale production deployment, it prefers to let the server perform the load balancing to avoid the client side load balancing.

Here the `exampleResolver` will resolve the `lb.example.grpc.io` to address `"localhost:50051", "localhost:50052"`. 
- When `exampleResolverBuilder.Build()` is called, the `addrsStore` field of `exampleResolver` is initialized with var `addrs`.
- `exampleResolverBuilder.Build()` will call `exampleResolver.start()`, which converts `addrsStore` to `[]resolver.Address` and sends it to `ccResolverWrapper.UpdateState()`
- In `main()`, the first client uses `exampleResolver` and `pick_first` balancer. This client will always uses the `"localhost:50051"` connection.
- In `main()`, the second client uses `exampleResolver` and `round_robin` balancer. This client will uses the round_robin policy for `"localhost:50051"`, `"localhost:50052"` in turn.
- In this example, the `exampleResolver` is necessary, because it resolves a name to a backend server list. 

The second client uses `grpc.WithDefaultServiceConfig` to set the load balance policy. Although the following statement does work:

```go
grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`)` 
```

It can be replaced by the following statement.

```go
grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin": {} }]}`)
```
Because `loadBalancingPolicy` has been deprecated in gRPC. See [service_config.go](https://github.com/grpc/grpc-go/blob/master/service_config.go) for detail.

Next, let's discuss what happens when `grpc.WithDefaultServiceConfig` is set.
```go
const (
    exampleScheme      = "example"
    exampleServiceName = "lb.example.grpc.io"
)

var addrs = []string{"localhost:50051", "localhost:50052"}

func callUnaryEcho(c ecpb.EchoClient, message string) {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()
    r, err := c.UnaryEcho(ctx, &ecpb.EchoRequest{Message: message})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    fmt.Println(r.Message)
}

func makeRPCs(cc *grpc.ClientConn, n int) {
    hwc := ecpb.NewEchoClient(cc)
    for i := 0; i < n; i++ {
        callUnaryEcho(hwc, "this is examples/load_balancing")
    }
}

func main() {
    // "pick_first" is the default, so there's no need to set the load balancer.
    pickfirstConn, err := grpc.Dial(
        fmt.Sprintf("%s:///%s", exampleScheme, exampleServiceName),
        grpc.WithInsecure(),
        grpc.WithBlock(),
    )
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer pickfirstConn.Close()

    fmt.Println("--- calling helloworld.Greeter/SayHello with pick_first ---")
    makeRPCs(pickfirstConn, 10)

    fmt.Println()

    // Make another ClientConn with round_robin policy.
    roundrobinConn, err := grpc.Dial(
        fmt.Sprintf("%s:///%s", exampleScheme, exampleServiceName),
        //grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`), // This sets the initial balancing policy.
        grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin": {} }]}`), // This sets the initial balancing policy.
        grpc.WithInsecure(),
        grpc.WithBlock(),
    )
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer roundrobinConn.Close()

    fmt.Println("--- calling helloworld.Greeter/SayHello with round_robin ---")
    makeRPCs(roundrobinConn, 10)
}

// Following is an example name resolver implementation. Read the name
// resolution example to learn more about it.

type exampleResolverBuilder struct{}

func (*exampleResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
    r := &exampleResolver{
        target: target,
        cc:     cc,
        addrsStore: map[string][]string{
            exampleServiceName: addrs,
        },
    }
    r.start()
    return r, nil
}
func (*exampleResolverBuilder) Scheme() string { return exampleScheme }

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
    resolver.Register(&exampleResolverBuilder{})
}

```
### defaultServiceConfigRawJSON

`WithDefaultServiceConfig()` uses `DialOption` to set the `defaultServiceConfigRawJSON`. There is a dedicated chapter discuss [DialOption and ServerOption](auth.md#dialoption-and-serveroption). 

`defaultServiceConfigRawJSON` is defined in `dialOptions` and used in `DialContext()`. In `DialContext()`, if  `defaultServiceConfigRawJSON` is not nil, `parseServiceConfig()` will be called to parse the json string. After successfully pared, `ClientConn.dopts.defaultServiceConfig` will be set. `defaultServiceConfig` is the golang data structure to represent the service config.

Using raw json string to set the service config is weird. It's error prone and mysterious. If you read the [Service Config in gRPC](https://github.com/grpc/grpc/blob/master/doc/service_config.md) and [Service Config via DNS](https://github.com/grpc/proposal/blob/master/A2-service-configs-in-dns.md), you will know that service config was originally designed for dns resolver to return both the resolved addresses and the service config. The following quoted statement explains why we get a json string.

" In DNS, the service config data (in the form documented in the previous section) will be encoded in a TXT record via the mechanism described in RFC-1464 using the attribute name grpc_config."

Let's spend some time to discuss the implementation of dns resolver. It's crucial to understand why we get the `defaultServiceConfigRawJSON` and how to use it.

```go
// WithDefaultServiceConfig returns a DialOption that configures the default                                
// service config, which will be used in cases where:                                        
//                                   
// 1. WithDisableServiceConfig is also used.                                
// 2. Resolver does not return a service config or if the resolver returns an                                
//    invalid service config.                                
//                                                                                                              
// Experimental                                                     
//                                                                                                 
// Notice: This API is EXPERIMENTAL and may be changed or removed in a                                
// later release.                                        
func WithDefaultServiceConfig(s string) DialOption {                                
    return newFuncDialOption(func(o *dialOptions) {                                
        o.defaultServiceConfigRawJSON = &s                                    
    })                                
}                                                              

// dialOptions configure a Dial call. dialOptions are set by the DialOption
// values passed to Dial.
type dialOptions struct {
    unaryInt  UnaryClientInterceptor
    streamInt StreamClientInterceptor

    chainUnaryInts  []UnaryClientInterceptor
    chainStreamInts []StreamClientInterceptor

    cp              Compressor
    dc              Decompressor                                                                            
    bs              internalbackoff.Strategy                                                                    
    block           bool                                                    
    returnLastError bool                                                           
    insecure        bool                                             
    timeout         time.Duration     
    scChan          <-chan ServiceConfig                                   
    authority       string                                       
    copts           transport.ConnectOptions                                                                
    callOptions     []CallOption                                                             
    // This is used by WithBalancerName dial option.
    balancerBuilder             balancer.Builder                            
    channelzParentID            int64                                                                        
    disableServiceConfig        bool                         
    disableRetry                bool                                                                            
    disableHealthCheck          bool                                
    healthCheckFunc             internal.HealthChecker                                             
    minConnectTimeout           func() time.Duration                                                  
    defaultServiceConfig        *ServiceConfig // defaultServiceConfig is parsed from defaultServiceConfigRawJSON.
    defaultServiceConfigRawJSON *string                                             
    // This is used by ccResolverWrapper to backoff between successive calls to    
    // resolver.ResolveNow(). The user will have no need to configure this, but
    // we need to be able to configure this in tests.
    resolveNowBackoff func(int) time.Duration                  
    resolvers         []resolver.Builder
}                                                                                                             

func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
    cc := &ClientConn{
        target:            target,
        csMgr:             &connectivityStateManager{},
        conns:             make(map[*addrConn]struct{}),
        dopts:             defaultDialOptions(),
        blockingpicker:    newPickerWrapper(),
        czData:            new(channelzData),
        firstResolveEvent: grpcsync.NewEvent(),
    }
    cc.retryThrottler.Store((*retryThrottler)(nil))
    cc.ctx, cc.cancel = context.WithCancel(context.Background())

+-- 48 lines: for _, opt := range opts {···················································································································

    if cc.dopts.defaultServiceConfigRawJSON != nil {
        scpr := parseServiceConfig(*cc.dopts.defaultServiceConfigRawJSON)
        if scpr.Err != nil {
            return nil, fmt.Errorf("%s: %v", invalidDefaultServiceConfigErrPrefix, scpr.Err)
        }
        cc.dopts.defaultServiceConfig, _ = scpr.Config.(*ServiceConfig)
    }
    cc.mkp = cc.dopts.copts.KeepaliveParams

+--141 lines: if cc.dopts.copts.UserAgent != "" {··········································································································
    return cc, nil
}

func parseServiceConfig(js string) *serviceconfig.ParseResult {
    if len(js) == 0 {
        return &serviceconfig.ParseResult{Err: fmt.Errorf("no JSON service config provided")}
    }
    var rsc jsonSC
    err := json.Unmarshal([]byte(js), &rsc)
    if err != nil {
        logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to %v", js, err)
        return &serviceconfig.ParseResult{Err: err}
    }
    sc := ServiceConfig{
        LB:                rsc.LoadBalancingPolicy,
        Methods:           make(map[string]MethodConfig),
        retryThrottling:   rsc.RetryThrottling,
        healthCheckConfig: rsc.HealthCheckConfig,
        rawJSONString:     js,
    }
    if c := rsc.LoadBalancingConfig; c != nil {
        sc.lbConfig = &lbConfig{
            name: c.Name,
            cfg:  c.Config,
        }
    }

    if rsc.MethodConfig == nil {
        return &serviceconfig.ParseResult{Config: &sc}
    }

    paths := map[string]struct{}{}
    for _, m := range *rsc.MethodConfig {
        if m.Name == nil {
            continue
        }
        d, err := parseDuration(m.Timeout)
        if err != nil {
            logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to %v", js, err)
            return &serviceconfig.ParseResult{Err: err}
        }

        mc := MethodConfig{
            WaitForReady: m.WaitForReady,
            Timeout:      d,
        }
        if mc.RetryPolicy, err = convertRetryPolicy(m.RetryPolicy); err != nil {
            logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to %v", js, err)
            return &serviceconfig.ParseResult{Err: err}
        }
        if m.MaxRequestMessageBytes != nil {
            if *m.MaxRequestMessageBytes > int64(maxInt) {
                mc.MaxReqSize = newInt(maxInt)
            } else {
                mc.MaxReqSize = newInt(int(*m.MaxRequestMessageBytes))
            }
        }
        if m.MaxResponseMessageBytes != nil {
            if *m.MaxResponseMessageBytes > int64(maxInt) {
                mc.MaxRespSize = newInt(maxInt)
            } else {
                mc.MaxRespSize = newInt(int(*m.MaxResponseMessageBytes))
            }
        }
        for i, n := range *m.Name {
            path, err := n.generatePath()
            if err != nil {
                logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to methodConfig[%d]: %v", js, i, err)
                return &serviceconfig.ParseResult{Err: err}
            }

            if _, ok := paths[path]; ok {
                err = errDuplicatedName
                logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to methodConfig[%d]: %v", js, i, err)
                return &serviceconfig.ParseResult{Err: err}
            }
            paths[path] = struct{}{}
            sc.Methods[path] = mc
        }
    }

    if sc.retryThrottling != nil {
        if mt := sc.retryThrottling.MaxTokens; mt <= 0 || mt > 1000 {
            return &serviceconfig.ParseResult{Err: fmt.Errorf("invalid retry throttling config: maxTokens (%v) out of range (0, 1000]", mt)}
        }
        if tr := sc.retryThrottling.TokenRatio; tr <= 0 {
            return &serviceconfig.ParseResult{Err: fmt.Errorf("invalid retry throttling config: tokenRatio (%v) may not be negative", tr)}
        }
    }
    return &serviceconfig.ParseResult{Config: &sc}
}

// ServiceConfig is provided by the service provider and contains parameters for how
// clients that connect to the service should behave.
//
// Deprecated: Users should not use this struct. Service config should be received
// through name resolver, as specified here
// https://github.com/grpc/grpc/blob/master/doc/service_config.md
type ServiceConfig struct {
    serviceconfig.Config

    // LB is the load balancer the service providers recommends. The balancer
    // specified via grpc.WithBalancerName will override this.  This is deprecated;
    // lbConfigs is preferred.  If lbConfig and LB are both present, lbConfig
    // will be used.
    LB *string

    // lbConfig is the service config's load balancing configuration.  If
    // lbConfig and LB are both present, lbConfig will be used.
    lbConfig *lbConfig

    // Methods contains a map for the methods in this service.  If there is an
    // exact match for a method (i.e. /service/method) in the map, use the
    // corresponding MethodConfig.  If there's no exact match, look for the
    // default config for the service (/service/) and use the corresponding
    // MethodConfig if it exists.  Otherwise, the method has no MethodConfig to                                    
    // use.                                                                                                         
    Methods map[string]MethodConfig                                                                                                         
                                                                                                                                
    // If a retryThrottlingPolicy is provided, gRPC will automatically throttle                                                     
    // retry attempts and hedged RPCs when the client’s ratio of failures to                                                                             
    // successes exceeds a threshold.                                                                              
    //                                                                                                                                      
    // For each server name, the gRPC client will maintain a token_count which is                                                             
    // initially set to maxTokens, and can take values between 0 and maxTokens.                                    
    //                                                                                                                            
    // Every outgoing RPC (regardless of service or method invoked) will change                                    
    // token_count as follows:                                                                                      
    //                                                                                                                                          
    //   - Every failed RPC will decrement the token_count by 1.                                                                
    //   - Every successful RPC will increment the token_count by tokenRatio.
    //
    // If token_count is less than or equal to maxTokens / 2, then RPCs will not
    // be retried and hedged RPCs will not be sent.
    retryThrottling *retryThrottlingPolicy
    // healthCheckConfig must be set as one of the requirement to enable LB channel
    // health check.
    healthCheckConfig *healthCheckConfig
    // rawJSONString stores service config json string that get parsed into
    // this service config struct.
    rawJSONString string
}

// healthCheckConfig defines the go-native version of the LB channel health check config.
type healthCheckConfig struct {
    // serviceName is the service name to use in the health-checking request.
    ServiceName string
}

type jsonRetryPolicy struct {
    MaxAttempts          int
    InitialBackoff       string
    MaxBackoff           string
    BackoffMultiplier    float64
    RetryableStatusCodes []codes.Code
}

```
### service config and dnsResovler

Let's briefly introduct how dns resolver use service config:
- `dnsBuilder.Build()` starts a new goroutine `d.watcher()`, which actually starts `dnsResolver.watcher()`.
- upon receive a signal sent by `dnsResolver.ResolveNow()`, `dnsResolver.watcher()` calls `d.lookup()`, which actually calls `dnsResolver.lookup()`
- `dnsResolver.lookup()` calls `d.lookupSRV()`, `d.lookupHost()` to resolve the address and SRV record.
- At the last step, 'dnsResolver.lookup()' calls `d.lookupTXT()` to resolve and parse the TXT record.
  - In `lookupTXT()`, `d.cc.ParseServiceConfig()` will be called to parse the json string, which actually calls 'parseServiceConfig()' function.
- `dnsResolver.lookup()` returns a `*resolver.State` struct, which contains both resolved addresses and service config.
- In `dnsResolver.watcher()`, `d.cc.UpdateState()` will be called to notify the `*resolver.State` gRPC core. Which actually calls `ccResolverWrapper.UpdateState()`.

Now, we understand the purpose of service config. You have two ways to set service config:
- set the service config in dns TXT record and extract that information by dns resolver. Which is stored in `resolver.State.ServiceConfig`
- set the service config through `defaultServiceConfigRawJSON` and extract it via `DialContext()`. Which is stored in `ClientConn.dopts.defaultServiceConfig`

The following is the summary: Before the envoy emerges, gRPC team plans to build a rich set of load balancing machanism, including `grpclb` balancer, `rls` balancer, `round_robin` balancer, `pick_first` balancer. The following statement is quoted from [A27: xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)

"gRPC currently supports its own "grpclb" protocol for look-aside load-balancing.  However, the popular Envoy proxy uses the xDS API for many types of configuration, including load balancing, and that API is evolving into a standard that will be used to configure a variety of data plane software. In order to converge with this industry trend, gRPC will be moving from its original grpclb protocol to the new xDS protocol."

Here is my prediction (2021-Jan-03): `defaultServiceConfigRawJSON` is a short-cuts to let the gRPC core to reuse the existing service config component. This short-cuts will be replaced by a new way in the future. The reason is simple: Using raw json string to set the service config is weird. It's error prone and mysterious. As XDS support becomes mature, we will see the change in 2021.

Next, let's continue to discuss how to use `defaultServiceConfig`.

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

// ResolveNow invoke an immediate resolution of the target that this dnsResolver watches.
func (d *dnsResolver) ResolveNow(resolver.ResolveNowOptions) {
    select {                
    case d.rn <- struct{}{}:
    default:
    }
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

func (d *dnsResolver) lookup() (*resolver.State, error) {
    srv, srvErr := d.lookupSRV()
    addrs, hostErr := d.lookupHost()
    if hostErr != nil && (srvErr != nil || len(srv) == 0) {
        return nil, hostErr
    }
                                                               
    state := resolver.State{Addresses: addrs}
    if len(srv) > 0 {  
        state = grpclbstate.Set(state, &grpclbstate.State{BalancerAddresses: srv})
    }                              
    if !d.disableServiceConfig {                                        
        state.ServiceConfig = d.lookupTXT()
    }                                                     
    return &state, nil 
}    

func (d *dnsResolver) lookupTXT() *serviceconfig.ParseResult {
    ss, err := d.resolver.LookupTXT(d.ctx, txtPrefix+d.host)
    if err != nil {                 
        if envconfig.TXTErrIgnore {                        
            return nil     
        }
        if err = handleDNSError(err, "TXT"); err != nil {      
            return &serviceconfig.ParseResult{Err: err}
        }              
        return nil                                                                
    }                              
    var res string                                                      
    for _, s := range ss {                 
        res += s                                          
    }                  
     
    // TXT record must have "grpc_config=" attribute in order to be used as service config.
    if !strings.HasPrefix(res, txtAttribute) {                                                
        logger.Warningf("dns: TXT record %v missing %v attribute", res, txtAttribute)
        // This is not an error; it is the equivalent of not having a service config.    
        return nil                                   
    }
    sc := canaryingSC(strings.TrimPrefix(res, txtAttribute))
    return d.cc.ParseServiceConfig(sc)
}

func (ccr *ccResolverWrapper) ParseServiceConfig(scJSON string) *serviceconfig.ParseResult {
    return parseServiceConfig(scJSON)                                                           
}                                                                                                   
```
### defaultServiceConfig

Now `defaultServiceConfig` is set. When client dials in [ClientConn.updateResolverState()](dial.md#clientconnupdateresolverstate), `maybeApplyDefaultServiceConfig()` will be called to utilize the `defaultServiceConfig`.
- In `maybeApplyDefaultServiceConfig()`, if `cc.dopts.defaultServiceConfig` is not nil, 'cc.applyServiceConfigAndBalancer()' will be called and `cc.dopts.defaultServiceConfig` will be passed as parameter.
- In `cc.applyServiceConfigAndBalancer()`, `newBalancerName` will get the value from `cc.sc.lbConfig.name` or `*cc.sc.LB` or `PickFirstBalancerName` or `grpclbName`.
- In our case, for the first client, `newBalancerName` will get the value from `PickFirstBalancerName`, which is '"pick_first"'
- In our case, for the second client, `newBalancerName` will get the value from `cc.sc.lbConfig.name`, which is '"round_robin"'
- At the last, `switchBalancer()` will be called to find the balancer builder from balancer map and `newCCBalancerWrapper()` will be called to initialize balancer.

In [ClientConn.updateResolverState()](dial.md#clientconnupdateresolverstate),
- If the parameter `s resolver.State`'s field `s.ServiceConfig.Config` is not nil, which means dns resolver does fill in the `ServiceConfig.Config` field.
- `ClientConn.updateResolverState()` will use the service config to initialize the balancer.
- The service config provided by the resolver takes the higher priority than default service config. 
- The service config provided by the resolver can be disabled by `WithDisableServiceConfig()`

Up to now, we only get a initialized balancer. Next, Let's discuss the behavior of balancer.
```go
func (cc *ClientConn) maybeApplyDefaultServiceConfig(addrs []resolver.Address) {                                                                       
    if cc.sc != nil {                                                                                             
        cc.applyServiceConfigAndBalancer(cc.sc, nil, addrs)                                                       
        return                                                                                                                                    
    }                                                                                                             
    if cc.dopts.defaultServiceConfig != nil {
        cc.applyServiceConfigAndBalancer(cc.dopts.defaultServiceConfig, &defaultConfigSelector{cc.dopts.defaultServiceConfig}, addrs)        
    } else {                                                                                                                                         
        cc.applyServiceConfigAndBalancer(emptyServiceConfig, &defaultConfigSelector{emptyServiceConfig}, addrs)                
    }                                                                                                                                         
}                                                                                                                                                 
                                                                                                                                        
func (cc *ClientConn) applyServiceConfigAndBalancer(sc *ServiceConfig, configSelector iresolver.ConfigSelector, addrs []resolver.Address) {
    if sc == nil {                                                                                                                           
        // should never reach here.                                                                                                                  
        return                                                                                                                 
    }                                                                                                                                         
    cc.sc = sc                                                                                                                                    
    if configSelector != nil {                                                                                                          
        cc.safeConfigSelector.UpdateConfigSelector(configSelector)                                                                                
    }                                                                                                                                       
                                                                                                                                                 
    if cc.sc.retryThrottling != nil {                                                                                                        
        newThrottler := &retryThrottler{                                                                                                              
            tokens: cc.sc.retryThrottling.MaxTokens,                                                                                              
            max:    cc.sc.retryThrottling.MaxTokens,                                                                                       
            thresh: cc.sc.retryThrottling.MaxTokens / 2,                                                                                                  
            ratio:  cc.sc.retryThrottling.TokenRatio,                                                                                                   
        }                                                                                                                                                 
        cc.retryThrottler.Store(newThrottler)                                                                                                    
    } else {       
        cc.retryThrottler.Store((*retryThrottler)(nil))                                                                         
    }                                                                          

    if cc.dopts.balancerBuilder == nil {
        // Only look at balancer types and switch balancer if balancer dial
        // option is not set.
        var newBalancerName string
        if cc.sc != nil && cc.sc.lbConfig != nil {
            newBalancerName = cc.sc.lbConfig.name
        } else {
            var isGRPCLB bool
            for _, a := range addrs {
                if a.Type == resolver.GRPCLB {
                    isGRPCLB = true
                    break
                }
            }
            if isGRPCLB {
                newBalancerName = grpclbName
            } else if cc.sc != nil && cc.sc.LB != nil {
                newBalancerName = *cc.sc.LB
            } else {
                newBalancerName = PickFirstBalancerName
            }
        }
        cc.switchBalancer(newBalancerName)
    } else if cc.balancerWrapper == nil {
        // Balancer dial option was set, and this is the first time handling
        // resolved addresses. Build a balancer with dopts.balancerBuilder.
        cc.curBalancerName = cc.dopts.balancerBuilder.Name()
        cc.balancerWrapper = newCCBalancerWrapper(cc, cc.dopts.balancerBuilder, cc.balancerBuildOpts)
    }
}

// switchBalancer starts the switching from current balancer to the balancer
// with the given name.                           
//                                               
// It will NOT send the current address list to the new balancer. If needed,
// caller of this function should send address list to the new balancer after
// this function returns.                                                         
//                                                                          
// Caller must hold cc.mu.                                     
func (cc *ClientConn) switchBalancer(name string) {    
    if strings.EqualFold(cc.curBalancerName, name) {
        return                                                          
    }                                                         
                                            
    channelz.Infof(logger, cc.channelzID, "ClientConn switching balancer to %q", name)
    if cc.dopts.balancerBuilder != nil {                                     
        channelz.Info(logger, cc.channelzID, "ignoring balancer switching: Balancer DialOption used instead")
        return                                                                                       
    }                                                                     
    if cc.balancerWrapper != nil {                                                                   
        cc.balancerWrapper.close()        
    }                                                           
                                                                            
    builder := balancer.Get(name)                                             
    if builder == nil {                                                    
        channelz.Warningf(logger, cc.channelzID, "Channel switches to new LB policy %q due to fallback from invalid balancer name", PickFirstBalancerName)
        channelz.Infof(logger, cc.channelzID, "failed to get balancer builder for: %v, using pick_first instead", name)
        builder = newPickfirstBuilder()                                         
    } else {                                                                  
        channelz.Infof(logger, cc.channelzID, "Channel switches to new LB policy %q", name)
    }

    cc.curBalancerName = builder.Name()
    cc.balancerWrapper = newCCBalancerWrapper(cc, builder, cc.balancerBuildOpts)
}

```
### UpdateClientConnState()
At Client Dial [ClientConn.updateResolverState()](dial.md#clientconnupdateresolverstate) section, `ccBalancerWrapper.updateClientConnState()` will be called to establish the connection with the resolved target.
- `ccBalancerWrapper.updateClientConnState()` calls `ccb.balancer.UpdateClientConnState()`. The balancer is determined by the run time value.
- For `pick_first` balancer, `pickfirstBalancer.UpdateClientConnState()` will be called. 
  - The behavior is detailed in [Here](dial.md#ccbalancerwrapperupdateclientconnstate).
  - `pick_first` balancer will call `b.cc.NewSubConn()`, `b.sc.Connect()` and `b.cc.UpdateState()` to establish a transport connection
  - `pick_first` balancer calls `b.cc.UpdateState()`, which actually calls `ccBalancerWrapper.UpdateState()`
- For `round_robin` balancer, `baseBalancer.UpdateClientConnState()` will be called
  - For every address, `b.cc.NewSubConn()` and `sc.Connect()` is called to establish a seperated transport connection.
  - `ccBalancerWrapper.UpdateState()` will not be called. 
- `ccBalancerWrapper.UpdateState()`will notify the gRPC the Connectivity state and update the `Picker`. 

```go

func (ccb *ccBalancerWrapper) updateClientConnState(ccs *balancer.ClientConnState) error {
    ccb.balancerMu.Lock()
    defer ccb.balancerMu.Unlock()
    return ccb.balancer.UpdateClientConnState(*ccs)                             
}                                                 
                    

func (b *baseBalancer) UpdateClientConnState(s balancer.ClientConnState) error {
    // TODO: handle s.ResolverState.ServiceConfig?
    if logger.V(2) {                                                   
        logger.Info("base.baseBalancer: got new ClientConn state: ", s)
    }
    // Successful resolution; clear resolver error and ensure we return nil.
    b.resolverErr = nil
    // addrsSet is the set converted from addrs, it's used for quick lookup of an address.
    addrsSet := make(map[resolver.Address]struct{})
    for _, a := range s.ResolverState.Addresses {                           
        // Strip attributes from addresses before using them as map keys. So   
        // that when two addresses only differ in attributes pointers (but with
        // the same attribute content), they are considered the same address.
        //                                                                                                                                     
        // Note that this doesn't handle the case where the attribute content is                                                
        // different. So if users want to set different attributes to create                                                                 
        // duplicate connections to the same backend, it doesn't work. This is                                                            
        // fine for now, because duplicate is done by setting Metadata today.                                                    
        //                                                                                                                                 
        // TODO: read attributes to handle duplicate connections.                                                                           
        aNoAttrs := a                                                                                                          
        aNoAttrs.Attributes = nil                                                                                                          
        addrsSet[aNoAttrs] = struct{}{}                                                                                                      
        if sc, ok := b.subConns[aNoAttrs]; !ok {                                                                                               
            logger.Infof("UpdateClientConnState() prepare to call NewSubConn() %v", a)                                                     
            // a is a new address (not existing in b.subConns).                                                                                
            //                                                                                                                                  
            // When creating SubConn, the original address with attributes is                                                          
            // passed through. So that connection configurations in attributes                                                                  
            // (like creds) will be used.                                                                                          
            sc, err := b.cc.NewSubConn([]resolver.Address{a}, balancer.NewSubConnOptions{HealthCheckEnabled: b.config.HealthCheck})             
            if err != nil {                                                                                                            
                logger.Warningf("base.baseBalancer: failed to create new SubConn: %v", err)                                                     
                continue                                                                                                           
            }                                                                                                                      
            b.subConns[aNoAttrs] = sc
            b.scStates[sc] = connectivity.Idle                                             
            sc.Connect()
        } else {
            // Always update the subconn's address in case the attributes
            // changed.                       
            //          
            // The SubConn does a reflect.DeepEqual of the new and old
            // addresses. So this is a noop if the current address is the same
            // as the old one (including attributes).
            sc.UpdateAddresses([]resolver.Address{a})
        }                                                             
    }                                                                         
    for a, sc := range b.subConns {                  
        // a was removed by resolver.                
        if _, ok := addrsSet[a]; !ok {
            b.cc.RemoveSubConn(sc)
            delete(b.subConns, a)
            // Keep the state of this sc in b.scStates until sc's state becomes Shutdown.
            // The entry will be deleted in UpdateSubConnState.
        }
    }
    // If resolver state contains no addresses, return an error so ClientConn
    // will trigger re-resolve. Also records this as an resolver error, so when
    // the overall state turns transient failure, the error message will have
    // the zero address information.
    if len(s.ResolverState.Addresses) == 0 {
        b.ResolverError(errors.New("produced zero addresses"))
        return balancer.ErrBadResolverState
    }
    return nil
}

func (ccb *ccBalancerWrapper) UpdateState(s balancer.State) {                                                                   
    ccb.mu.Lock()                                                                                                                               
    defer ccb.mu.Unlock()                                                                                                                   
    if ccb.subConns == nil {                                                                                                               
        return                                                                                                                                
    }                                                                                                              
    // Update picker before updating state.  Even though the ordering here does                                                                  
    // not matter, it can lead to multiple calls of Pick in the common start-up                                                          
    // case where we wait for ready and then perform an RPC.  If the picker is                                                                          
    // updated later, we could call the "connecting" picker when the state is                                                    
    // updated, and then call the "ready" picker after the picker gets updated.                                                                     
    ccb.cc.blockingpicker.updatePicker(s.Picker)                                                                                               
    ccb.cc.csMgr.updateState(s.ConnectivityState)                                                                                        
}                                                                                                                                             
                                                                                                                                              
// pickerWrapper is a wrapper of balancer.Picker. It blocks on certain pick                                                              
// actions and unblock when there's a picker update.                                                                   
type pickerWrapper struct {                                                                                                                     
    mu         sync.Mutex                                                                                                       
    done       bool                                                                                                                             
    blockingCh chan struct{}                                                                                                                
    picker     balancer.Picker                                                                                                             
}                                                                                                                                             
                                                                                                                   
func newPickerWrapper() *pickerWrapper {                                                                                                         
    return &pickerWrapper{blockingCh: make(chan struct{})}                                                                               
}                                                                                                                                                       

// updatePicker is called by UpdateBalancerState. It unblocks all blocked pick.                                                                     
func (pw *pickerWrapper) updatePicker(p balancer.Picker) {                                                                                     
    pw.mu.Lock()                                                                                                                         
    if pw.done {                                                                                                                              
        pw.mu.Unlock()                                                                                                                        
        return                                                                                                                                      
    }                                                                                                                                                     
    pw.picker = p                                                                                                                
    // pw.blockingCh should never be nil.                                                                          
    close(pw.blockingCh)                                                                                                                                  
    pw.blockingCh = make(chan struct{})                                                                            
    pw.mu.Unlock()                                                                                                                                         
}                                                                                                                                                    

```
### UpdateSubConnState()

At Client Dial [newCCBalancerWrapper()](dial.md#newccbalancerwrapper) section, goroutine `ccb.watcher()` waiting for the incoming message to monitor the sub connection status. In `ccb.watcher()`, upon receive the `scStateUpdat`:
- `ccb.balancer.UpdateSubConnState()` will be called, The balancer is determined by the run time value.
- For `pick_first` balancer, `pickfirstBalancer.UpdateClientConnState()` will be called. 
  - The behavior is detailed in [Here](dial.md#newccbalancerwrapper).
  - `pickfirstBalancer.UpdateClientConnState()` calls `b.cc.UpdateState`, which actually calls `ccBalancerWrapper.UpdateState()`.
- For `round_robin` balancer, `baseBalancer.UpdateSubConnState()` will be called.
  - In `baseBalancer.UpdateSubConnState()`, 
  - `b.regeneratePicker()` will be called, which actually calls `baseBalancer.regeneratePicker()`
  - `baseBalancer.regeneratePicker()` takes a snapshot of the balancer, and generates a picker from it.
  - `b.cc.UpdateState()` will be called, which actually calls `ccBalancerWrapper.UpdateState()`
- `ccBalancerWrapper.UpdateState()` will notify the gRPC the Connectivity state and update the `Picker`. 

`UpdateSubConnState()` focus on the real connection to the target backend server. `UpdateClientConnState()` focus on the logic connection to the target. 

Now, all the real connection to the target backend server is ready and the `balancer.Picker` is ready. Let's continue the discussion of `balancer.Picker` behavior.

```go
// watcher balancer functions sequentially, so the balancer can be implemented
// lock-free.                                   
func (ccb *ccBalancerWrapper) watcher() {
    for {                                                                                                 
        select {                                                                                      
        case t := <-ccb.scBuffer.Get():                                                                         
            ccb.scBuffer.Load()                                               
            if ccb.done.HasFired() {                                          
                break                                                                                                        
            }                                                                                                                      
            ccb.balancerMu.Lock()                                                                                    
            su := t.(*scStateUpdate)                                                                                                            
            ccb.balancer.UpdateSubConnState(su.sc, balancer.SubConnState{ConnectivityState: su.state, ConnectionError: su.err})
            ccb.balancerMu.Unlock()              
        case <-ccb.done.Done():                                                                                    
        }                                                                                                              
                                                                                                                                                  
        if ccb.done.HasFired() {                                                                                                                      
            ccb.balancer.Close()                                                     
            ccb.mu.Lock()                                                   
            scs := ccb.subConns                                               
            ccb.subConns = nil                                       
            ccb.mu.Unlock()                          
            for acbw := range scs {                                                                   
                ccb.cc.removeAddrConn(acbw.getAddrConn(), errConnDrain)
            }                                                            
            ccb.UpdateState(balancer.State{ConnectivityState: connectivity.Connecting, Picker: nil})
            return                                                            
        }                                                                                    
    }                                                                           
}                                                                      

// regeneratePicker takes a snapshot of the balancer, and generates a picker
// from it. The picker is
//  - errPicker if the balancer is in TransientFailure,
//  - built by the pickerBuilder with all READY SubConns otherwise.
func (b *baseBalancer) regeneratePicker() {
    if b.state == connectivity.TransientFailure {
        b.picker = NewErrPicker(b.mergeErrors())
        return
    }
    readySCs := make(map[balancer.SubConn]SubConnInfo)

    // Filter out all ready SCs from full subConn map.
    for addr, sc := range b.subConns {
        if st, ok := b.scStates[sc]; ok && st == connectivity.Ready {
            readySCs[sc] = SubConnInfo{Address: addr}                    
        }
    }
    b.picker = b.pickerBuilder.Build(PickerBuildInfo{ReadySCs: readySCs})                    
}                                                                     
                                                                                                                   
func (b *baseBalancer) UpdateSubConnState(sc balancer.SubConn, state balancer.SubConnState) {                          
    s := state.ConnectivityState                                                                                                                  
    if logger.V(2) {                                                                                                                                  
        logger.Infof("base.baseBalancer: handle SubConn state change: %p, %v", sc, s)
    }                                                                       
    oldS, ok := b.scStates[sc]                                                                        
    if !ok {                                                                                                    
        if logger.V(2) {                                                   
            logger.Infof("base.baseBalancer: got state changes for an unknown SubConn: %p, %v", sc, s)
        }                                                                                                                    
        return                                                                                                                     
    }                                                                                                                
    if oldS == connectivity.TransientFailure && s == connectivity.Connecting {                                                                  
        // Once a subconn enters TRANSIENT_FAILURE, ignore subsequent                                                    
        // CONNECTING transitions to prevent the aggregated state from being                                       
        // always CONNECTING when many backends exist but are all down.                                                
        return                                                                                                                    
    }
    b.scStates[sc] = s
    switch s {
    case connectivity.Idle:
        sc.Connect()
    case connectivity.Shutdown:
        // When an address was removed by resolver, b called RemoveSubConn but
        // kept the sc's state in scStates. Remove state for this sc here.
        delete(b.scStates, sc)
    case connectivity.TransientFailure:
        // Save error to be reported via picker.
        b.connErr = state.ConnectionError
    }

    b.state = b.csEvltr.RecordTransition(oldS, s)

    // Regenerate picker when one of the following happens:
    //  - this sc entered or left ready
    //  - the aggregated state of balancer is TransientFailure
    //    (may need to update error message)
    if (s == connectivity.Ready) != (oldS == connectivity.Ready) ||
        b.state == connectivity.TransientFailure {
        b.regeneratePicker()
    }

    b.cc.UpdateState(balancer.State{ConnectivityState: b.state, Picker: b.picker})
}

// pickerWrapper is a wrapper of balancer.Picker. It blocks on certain pick                                
// actions and unblock when there's a picker update.                                
type pickerWrapper struct {                                
    mu         sync.Mutex                                
    done       bool                                
    blockingCh chan struct{}                                
    picker     balancer.Picker                                
}                                
```
### newAttemptLocked()
At Send request [Send Request-Headers](request.md#send-request-headers) section, `cs.newAttemptLocked()` will be called. In `cs.newAttemptLocked()`
- `cs.cc.getTransport` will be called, which actually calls `clientStream.newAttemptLocked()`.
- `clientStream.newAttemptLocked()` calls `cs.cc.getTransport`, which actually calls `ClientConn.getTransport()`.
- `ClientConn.getTransport()` calls `cc.blockingpicker.pick()`, which actually calls `pickerWrapper.pick()`.
  - `pickerWrapper.pick()` calls `p.Pick()` to choose the real transport. 
  - `p` is determined by run time value, `p` is set by `pickerWrapper.updatePicker()`.
  - For `pick_first` balancer, `p.Pick()` actually calls `picker.Pick()`.
    - `picker.Pick()` returns the `p.result`, which is updated by `pickfirstBalancer.UpdateSubConnState()`
    - In our case, only '"localhost:50051"' will be returned.
  - For `round_robin` balancer, `p.Pick()` actually calls `rrPicker.Pick()`.
    - `rrPicker.Pick()` uses round robin policy to choose one of the sub connection and returns a new `balancer.PickResult`
    - In our case, either '"localhost:50051"' or `"localhost:50052"` will be returned.
  - The `pickResult` returned by `p.Pick()` is used to convert to `acBalancerWrapper` 
  - Next calls `acw.getAddrConn().getReadyTransport()` to return the real sub connection `transport.ClientTransport`.
- After `ClientConn.getTransport()` returns the `transport.ClientTransport`, the returned `t` is assigned to `newAttempt.t`, then `cs.attempt = newAttempt`.
- The `cs.attempt` is used to calls `newStream()` to continue the [Send request](request.md) process.

```go
func (cc *ClientConn) getTransport(ctx context.Context, failfast bool, method string) (transport.ClientTransport, func(balancer.DoneInfo), error) {
    t, done, err := cc.blockingpicker.pick(ctx, failfast, balancer.PickInfo{
        Ctx:            ctx,               
        FullMethodName: method,                                                    
    })                                                                             
    if err != nil {                                             
        return nil, nil, toRPCErr(err)                                      
    }                                                                       
    return t, done, nil
}                                                                                   
                                                                                    
// pick returns the transport that will be used for the RPC.
// It may block in the following cases:
// - there's no picker
// - the current picker returns ErrNoSubConnAvailable
// - the current picker returns other errors and failfast is false.
// - the subConn returned by the current picker is not READY
// When one of these situations happens, pick blocks until the picker gets updated.
func (pw *pickerWrapper) pick(ctx context.Context, failfast bool, info balancer.PickInfo) (transport.ClientTransport, func(balancer.DoneInfo), error) {
    var ch chan struct{}

    var lastPickErr error
    for {
        pw.mu.Lock()
        if pw.done {      
            pw.mu.Unlock()                       
            return nil, nil, ErrClientConnClosing                                                                                                  
        }
                                                                                                                                                        
        if pw.picker == nil {       
            ch = pw.blockingCh  
        }                                                                                                                                             
        if ch == pw.blockingCh {                                 
            // This could happen when either:          
            // - pw.picker is nil (the previous if condition), or                                                                                    
            // - has called pick on the current picker.                                                                                         
            pw.mu.Unlock()                                                                                                                    
            select {             
            case <-ctx.Done():         
                var errStr string                                           
                if lastPickErr != nil {
                    errStr = "latest balancer error: " + lastPickErr.Error()
                } else {
                    errStr = ctx.Err().Error()
                }
                switch ctx.Err() {
                case context.DeadlineExceeded:
                    return nil, nil, status.Error(codes.DeadlineExceeded, errStr)
                case context.Canceled:
                    return nil, nil, status.Error(codes.Canceled, errStr)
                }
            case <-ch:
            }
            continue
        }                 
                      
        ch = pw.blockingCh
        p := pw.picker
        pw.mu.Unlock()

        pickResult, err := p.Pick(info)

        if err != nil {
            if err == balancer.ErrNoSubConnAvailable {
                continue
            }
            if _, ok := status.FromError(err); ok {
                // Status error: end the RPC unconditionally with this status.
                return nil, nil, err
            }
            // For all other errors, wait for ready RPCs should block and other
            // RPCs should fail with unavailable.
            if !failfast {
                lastPickErr = err
                continue
            }
            return nil, nil, status.Error(codes.Unavailable, err.Error())
        }

        acw, ok := pickResult.SubConn.(*acBalancerWrapper)
        if !ok {
            logger.Error("subconn returned from pick is not *acBalancerWrapper")
            continue
        }
        if t, ok := acw.getAddrConn().getReadyTransport(); ok {
            if channelz.IsOn() {
                return t, doneChannelzWrapper(acw, pickResult.Done), nil
            }
            return t, pickResult.Done, nil
        }
        if pickResult.Done != nil {
            // Calling done with nil error, no bytes sent and no bytes received.
            // DoneInfo with default value works.
            pickResult.Done(balancer.DoneInfo{})
        }
        logger.Infof("blockingPicker: the picked transport is not ready, loop back to repick")
        // If ok == false, ac.state is not READY.
        // A valid picker always returns READY subConn. This means the state of ac
        // just changed, and picker will be updated shortly.
        // continue back to the beginning of the for loop to repick.
    }
}

type picker struct {                                                                                                   
    result balancer.PickResult                                                                                            
    err    error                                                                                                   
}                                                                                                                 
                        
func (p *picker) Pick(info balancer.PickInfo) (balancer.PickResult, error) {                               
    return p.result, p.err
}

type rrPickerBuilder struct{}                                
                                     
func (*rrPickerBuilder) Build(info base.PickerBuildInfo) balancer.Picker {                                
    logger.Infof("roundrobinPicker: newPicker called with info: %v", info)                                
    if len(info.ReadySCs) == 0 {                                      
        return base.NewErrPicker(balancer.ErrNoSubConnAvailable)                                
    }                                                            
    var scs []balancer.SubConn                                                                                           
    for sc := range info.ReadySCs {                                                            
        scs = append(scs, sc)                                
    }                                
    return &rrPicker{                                                                                        
        subConns: scs,                                                                                         
        // Start at a random index, as the same RR balancer rebuilds a new                                   
        // picker when SubConn states change, and we don't want to apply excess                                
        // load to the first server in the list.                                
        next: grpcrand.Intn(len(scs)),                                                        
    }                                                                      
}                                    
                                              
type rrPicker struct {                                
    // subConns is the snapshot of the roundrobin balancer when this picker was                                
    // created. The slice is immutable. Each Get() will do a round robin                                  
    // selection from it and return the selected SubConn.
    subConns []balancer.SubConn

    mu   sync.Mutex
    next int
}

func (p *rrPicker) Pick(balancer.PickInfo) (balancer.PickResult, error) {
    p.mu.Lock()
    sc := p.subConns[p.next]
    p.next = (p.next + 1) % len(p.subConns)
    p.mu.Unlock()
    return balancer.PickResult{SubConn: sc}, nil
}

```
