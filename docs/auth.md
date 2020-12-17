# Authentication
* [Client side](#client-side)
* [Server side](#server-side)
* [DialOption and ServerOption](#dialoption-and-serveroption)
* [Server internal](#server-internal)
* [Client internal](#client-internal)

Let's discuss the authentication example from the gRPC-go code base. In this way we can learn some internal components of gRPC. It also helps you to write gRPC application.  
<!--
-->
## Client side 

In the following client code snippet, the application performs the one-way(server side) TLS and provides oauth token for gRPC call validation.
* ```perRPC``` is a ```credentials.PerRPCCredentials```, 
  * ```oauth.NewOauthAccess()``` constructs the PerRPCCredentials using a given token. 
  * You can get the token via OAuth2 or other way. Here the token is a fixed oauth2 token. 
  * Furthermore ```perRPC``` requires transport security.
* ```credentials.NewClientTLSFromFile()``` creates a credential object from the server certificate, 
  * ```creds``` is a ```TransportCredentials```. 
  * ```creds``` can be used to establish a secure connection.
* prepare a ```DialOption``` with ```grpc.WithPerRPCCredentials()```, which sets credentials and places auth state on each outbound RPC. 
  * ```grpc.WithPerRPCCredentials()``` just sets the value of ```o.copts.PerRPCCredentials```
* prepare a ```DialOption``` with ```grpc.WithTransportCredentials()```, which configures a connection level security credentials 
  * ```grpc.WithTransportCredentials()``` just sets the value of ```o.copts.TransportCredentials```
* ```o.copts``` is ```transport.ConnectOptions```, which covers all relevant options for communicating with the server.

After finish the above preparing, ```Dial()``` will use ```o.copts.TransportCredentials``` option to establish the TLS connection. gRPC will use ```o.copts.PerRPCCredentials``` option to add the token to the gRPC call context.

```go
// client side
func main() {                                                     
    flag.Parse()                                                            
                                                                             
    // Set up the credentials for the connection.                                
    perRPC := oauth.NewOauthAccess(fetchToken())                                
    creds, err := credentials.NewClientTLSFromFile(data.Path("x509/ca_cert.pem"), "x.test.example.com")                                
    if err != nil {                                                
        log.Fatalf("failed to load credentials: %v", err)                                
    }                                                    
    opts := []grpc.DialOption{                                
        // In addition to the following grpc.DialOption, callers may also use                                
        // the grpc.CallOption grpc.PerRPCCredentials with the RPC invocation                                
        // itself.                                                               
        // See: https://godoc.org/google.golang.org/grpc#PerRPCCredentials                                
        grpc.WithPerRPCCredentials(perRPC),                                
        // oauth.NewOauthAccess requires the configuration of transport                                
        // credentials.                                  
        grpc.WithTransportCredentials(creds),                                
    }                                                        
                                                             
    opts = append(opts, grpc.WithBlock())                                
    conn, err := grpc.Dial(*addr, opts...)                                
    if err != nil {                                
        log.Fatalf("did not connect: %v", err)                                
    }                                                                                     
    defer conn.Close()                                                                 
    rgc := ecpb.NewEchoClient(conn)                                      
                                                                           
    callUnaryEcho(rgc, "hello world")                                     
}                                

var addr = flag.String("addr", "localhost:50051", "the address to connect to")                                
                                
func callUnaryEcho(client ecpb.EchoClient, message string) {                                
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)                                
    defer cancel()                                
    resp, err := client.UnaryEcho(ctx, &ecpb.EchoRequest{Message: message})                                
    if err != nil {                                
        log.Fatalf("client.UnaryEcho(_) = _, %v: ", err)                                
    }                                
    fmt.Println("UnaryEcho: ", resp.Message)                                
}                                

// fetchToken simulates a token lookup and omits the details of proper token
// acquisition. For examples of how to acquire an OAuth2 token, see:
// https://godoc.org/golang.org/x/oauth2
func fetchToken() *oauth2.Token {
    return &oauth2.Token{
        AccessToken: "some-secret-token",
    }
}

// WithPerRPCCredentials returns a DialOption which sets credentials and places
// auth state on each outbound RPC.                                          
func WithPerRPCCredentials(creds credentials.PerRPCCredentials) DialOption {
    return newFuncDialOption(func(o *dialOptions) {
        o.copts.PerRPCCredentials = append(o.copts.PerRPCCredentials, creds)
    })                                                                    
}                                                  

// WithTransportCredentials returns a DialOption which configures a connection                         
// level security credentials (e.g., TLS/SSL). This should not be used together
// with WithCredentialsBundle.                                               
func WithTransportCredentials(creds credentials.TransportCredentials) DialOption {
    return newFuncDialOption(func(o *dialOptions) {                    
        o.copts.TransportCredentials = creds                                
    })                                                                    
}                                                  

// ConnectOptions covers all relevant options for communicating with the server.
type ConnectOptions struct {
    // UserAgent is the application user agent.
    UserAgent string
    // Dialer specifies how to dial a network address.
    Dialer func(context.Context, string) (net.Conn, error)                                                                                                       
    // FailOnNonTempDialError specifies if gRPC fails on non-temporary dial errors.                                                                     
    FailOnNonTempDialError bool                                                                                                                          
    // PerRPCCredentials stores the PerRPCCredentials required to issue RPCs.                                                                                
    PerRPCCredentials []credentials.PerRPCCredentials                                                                                                               
    // TransportCredentials stores the Authenticator required to setup a client                                                                          
    // connection. Only one of TransportCredentials and CredsBundle is non-nil.                                                                                       
    TransportCredentials credentials.TransportCredentials                                                                                                         
    // CredsBundle is the credentials bundle to be used. Only one of                                                                                                   
    // TransportCredentials and CredsBundle is non-nil.                                                                                                                 
    CredsBundle credentials.Bundle
    // KeepaliveParams stores the keepalive parameters.                                                                                                          
    KeepaliveParams keepalive.ClientParameters                                                                                                                  
    // StatsHandler stores the handler for stats.                                                                                                                
    StatsHandler stats.Handler                                                                                                                                            
    // InitialWindowSize sets the initial window size for a stream.                                                                                                       
    InitialWindowSize int32                                                                                                                                             
    // InitialConnWindowSize sets the initial window size for a connection.                                                                                               
    InitialConnWindowSize int32                                                                                                                                    
    // WriteBufferSize sets the size of write buffer which in turn determines how much data can be batched before it's written on the wire.                               
    WriteBufferSize int
    // ReadBufferSize sets the size of read buffer, which in turn determines how much data can be read at most for one read syscall.                                      
    ReadBufferSize int                                                                                                                                                  
    // ChannelzParentID sets the addrConn id which initiate the creation of this client transport.                                                                    
    ChannelzParentID int64
    // MaxHeaderListSize sets the max (uncompressed) size of header list that is prepared to be received.                                                             
    MaxHeaderListSize *uint32
    // UseProxy specifies if a proxy should be used.                                                                                                           
    UseProxy bool                                                                                                                                                      
}

```

## Server side

In the following server code snippet, the application performs
* ```tls.LoadX509KeyPair()``` reads and parses a public/private key pair from a pair of files.
  * ```cert``` is a ```Certificate```
* prepare a ```ServerOption``` with ```grpc.UnaryInterceptor()```, which installs the ```ensureValidToken``` interceptor to gRPC.
  * ```grpc.UnaryInterceptor()``` just sets the value of ```o.unaryInt```
* prepare a ```ServerOption``` with ```grpc.Creds()```, which sets credentials for server connections. 
  * ```grpc.Creds()``` just sets the value of ```o.creds```
  * ```credentials.NewServerTLSFromCert()```  constructs TLS credentials from the input certificate for server.
* ```o``` is ```serverOptions```, which cover all the options for the server.
* ```ensureValidToken()``` is an interceptor, which ensures a valid token exists within a request's metadata.
  * First it uses ```metadata.FromIncomingContext``` to get the metadata from the context.
  * Then it verifies the expected token exist in the metadata. Otherwise the interceptor will stop the execution of handler and retrurn error.
  * There is a dedicated chapter to discuss [Interceptor](interceptor.md) 

```go
func main() {                                                                   
    flag.Parse()                                                                                                                       
    fmt.Printf("server starting on port %d...\n", *port)                                
                                                                                         
    cert, err := tls.LoadX509KeyPair(data.Path("x509/server_cert.pem"), data.Path("x509/server_key.pem"))                                
    if err != nil {                                           
        log.Fatalf("failed to load key pair: %s", err)                                                       
    }                                                                                                        
    opts := []grpc.ServerOption{                                                 
        // The following grpc.ServerOption adds an interceptor for all unary                                
        // RPCs. To configure an interceptor for streaming RPCs, see:                                
        // https://godoc.org/google.golang.org/grpc#StreamInterceptor                                  
        grpc.UnaryInterceptor(ensureValidToken),                                
        // Enable TLS for all incoming connections.                                
        grpc.Creds(credentials.NewServerTLSFromCert(&cert)),                                
    }                                                        
    s := grpc.NewServer(opts...)                                         
    pb.RegisterEchoServer(s, &ecServer{})                                 
    lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))                                
    if err != nil {                                                           
        log.Fatalf("failed to listen: %v", err)                                           
    }                                                                                  
    if err := s.Serve(lis); err != nil {                                 
        log.Fatalf("failed to serve: %v", err)                                
    }                                                                     
}                                

var (
    errMissingMetadata = status.Errorf(codes.InvalidArgument, "missing metadata")
    errInvalidToken    = status.Errorf(codes.Unauthenticated, "invalid token")
)

var port = flag.Int("port", 50051, "the port to serve on")

type ecServer struct {
    pb.UnimplementedEchoServer
}

func (s *ecServer) UnaryEcho(ctx context.Context, req *pb.EchoRequest) (*pb.EchoResponse, error) {
    return &pb.EchoResponse{Message: req.Message}, nil
}

// valid validates the authorization.
func valid(authorization []string) bool {
    if len(authorization) < 1 {
        return false
    }
    token := strings.TrimPrefix(authorization[0], "Bearer ")
    // Perform the token validation here. For the sake of this example, the code
    // here forgoes any of the usual OAuth2 token validation and instead checks
    // for a token matching an arbitrary string.
    return token == "some-secret-token"
}

// ensureValidToken ensures a valid token exists within a request's metadata. If
// the token is missing or invalid, the interceptor blocks execution of the
// handler and returns an error. Otherwise, the interceptor invokes the unary
// handler.
func ensureValidToken(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, errMissingMetadata
    }
    // The keys within metadata.MD are normalized to lowercase.
    // See: https://godoc.org/google.golang.org/grpc/metadata#New
    if !valid(md["authorization"]) {
        return nil, errInvalidToken
    }
    // Continue execution of handler after ensuring a valid token.
    return handler(ctx, req)
}

// Creds returns a ServerOption that sets credentials for server connections.                           
func Creds(c credentials.TransportCredentials) ServerOption {                               
    return newFuncServerOption(func(o *serverOptions) {                                                 
        o.creds = c              
    })                                                                          
}                                                                          
                                                                                                      
// UnaryInterceptor returns a ServerOption that sets the UnaryServerInterceptor for the              
// server. Only one unary interceptor can be installed. The construction of multiple                 
// interceptors (e.g., chaining) can be implemented at the caller.                      
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {                                                                                  
    return newFuncServerOption(func(o *serverOptions) {              
        if o.unaryInt != nil {                                             
            panic("The unary server interceptor was already set and may not be reset.")
        }                                                                                
        o.unaryInt = i                                                                           
    })                                                                            
}                                                                        

```
## DialOption and ServerOption

gRPC client and gRPC server both need options to extend the capability. Let's discuss the design of ```DialOption``` and ```ServerOption```. It's more likely you can also use this design in you application code.

```DialOption``` and ```ServerOption``` has the same design. Lets' discuss the ```ServerOption``` to understand it.

```ServerOption``` is an interface. It has an ```apply(*serverOptions) ``` method.
* one implementaion is ```EmptyServerOption```. It's a empty struct with an empty ```apply()``` method
* one implementaion is ```funcServerOption```. It's a struct with a ```f func(*serverOptions)``` field.
  * ```funcServerOption``` has a ```apply(do *serverOptions)``` method, the only behavior of this method is to call the ```fdo.f()``` 
  * ```newFuncServerOption``` is a function. It accepts the ```f func(*serverOptions)``` parameter and return a ```funcServerOption``` object.
* Both ```grpc.UnaryInterceptor()``` and ```grpc.Creds()``` call ```newFuncServerOption``` to build a ```funcServerOption```.

From the application point of view, ```grpc.UnaryInterceptor()``` and ```grpc.Creds()``` just returns ```ServerOption``` objects. While these objects is a struct  with a ```f func(*serverOptions)``` field and has a ```apply(do *serverOptions)``` method. gRPC still needs some way to use these  ```ServerOption``` objects.

```go
// A ServerOption sets options such as credentials, codec and keepalive parameters, etc.                                                                          
type ServerOption interface {                                                                                                                                  
    apply(*serverOptions)                                                                                                                                                
}                                                                                                                                                                    
                                                                                                                                                                          
// EmptyServerOption does not alter the server configuration. It can be embedded                                                                                          
// in another structure to build custom server options.                                                                                                              
//                                                                                                                                                               
// Experimental                                                                                                                                               
//                                                                                                                                                                         
// Notice: This type is EXPERIMENTAL and may be changed or removed in a                                                                                             
// later release.                                                                                                                                                         
type EmptyServerOption struct{}                                                                                                                     
                                                                                                                                      
func (EmptyServerOption) apply(*serverOptions) {}                                                                                                                  
                                                                                                                                                                  
// funcServerOption wraps a function that modifies serverOptions into an                                                                                        
// implementation of the ServerOption interface.                                                                                                                   
type funcServerOption struct {                                                                                                                              
    f func(*serverOptions)                                                                                                                                                
}                                                                                                                                                                 
                                                                                                                                      
func (fdo *funcServerOption) apply(do *serverOptions) {                                                                                                                   
    fdo.f(do)                                                                                                                                                             
}                                                                                                                                                                         
                                                                                                                                                                
func newFuncServerOption(f func(*serverOptions)) *funcServerOption {                                                                                                 
    return &funcServerOption{                                                                                                                                         
        f: f,                                                                                                                                       
    }                                                                                                                                 
}                                                                                                                                                         

```
We will not discuss the design of ```DialOption``` in detail.

```go
// DialOption configures how we set up the connection.                                                                                                            
type DialOption interface {
    apply(*dialOptions)                                                                                                                               
}                                                                                                                                 
                                                                                                                                  
// EmptyDialOption does not alter the dial configuration. It can be embedded in                                                                                      
// another structure to build custom dial options.                                                                                                           
//             
// Experimental                                                                                                                                                           
//                                                                                                                                                            
// Notice: This type is EXPERIMENTAL and may be changed or removed in a                                                                                                 
// later release.            
type EmptyDialOption struct{}                                                                                                                                            
                                                                                                                                   
func (EmptyDialOption) apply(*dialOptions) {}                                                                                                               
                                                                                                                                               
// funcDialOption wraps a function that modifies dialOptions into an                                                                                         
// implementation of the DialOption interface.                                                                                                  
type funcDialOption struct {                                                                                                                                 
    f func(*dialOptions)                                                                                                                          
}                                                  
                                                   
func (fdo *funcDialOption) apply(do *dialOptions) {
    fdo.f(do)  
}                                                                    
               
func newFuncDialOption(f func(*dialOptions)) *funcDialOption {
    return &funcDialOption{
        f: f,                                                     
    }                                       
}                                                                 
```
Let's continue discuss the use of ```ServerOption``` object. When you call  ```grpc.NewServer()```, ```ServerOption``` objects will be used to init the gRPC server.

In ```grpc.NewServer()```
* all the ```opt ...ServerOption``` parameters will be processed by a for loop.
* before the for loop, ```defaultServerOptions``` will be assigned to ```opts```.
* in the for loop, every ```ServerOption.apply()``` will be called and the ```opts``` will be the parameter of ```ServerOption.apply()```.
* after the for loop, ```opts``` will be assigned to ```Server.opts```. All the options is stored in the gRPC server.

In this way, all the fields of ```serverOptions``` can be processed in a clean and intuitive way. Actually, If more options were added to the ```serverOptions```, you only need to add the corresponding function similar like ```grpc.UnaryInterceptor()```.

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
    if EnableTracing {                                                                                                                                              
        _, file, line, _ := runtime.Caller(1)                                                                                                  
        s.events = trace.NewEventLog("grpc.Server", fmt.Sprintf("%s:%d", file, line))                                            
    }                                                                                                                                                       

    if s.opts.numServerWorkers > 0 {
        s.initServerWorkers()
    }

    if channelz.IsOn() {
        s.channelzID = channelz.RegisterServer(&channelzServer{s}, "")
    }
    return s
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

// ConnectOptions covers all relevant options for communicating with the server.
type ConnectOptions struct {
    // UserAgent is the application user agent.
    UserAgent string
    // Dialer specifies how to dial a network address.
    Dialer func(context.Context, string) (net.Conn, error)
    // FailOnNonTempDialError specifies if gRPC fails on non-temporary dial errors.
    FailOnNonTempDialError bool
    // PerRPCCredentials stores the PerRPCCredentials required to issue RPCs.
    PerRPCCredentials []credentials.PerRPCCredentials
    // TransportCredentials stores the Authenticator required to setup a client
    // connection. Only one of TransportCredentials and CredsBundle is non-nil.
    TransportCredentials credentials.TransportCredentials
    // CredsBundle is the credentials bundle to be used. Only one of
    // TransportCredentials and CredsBundle is non-nil.
    CredsBundle credentials.Bundle
    // KeepaliveParams stores the keepalive parameters.
    KeepaliveParams keepalive.ClientParameters
    // StatsHandler stores the handler for stats.
    StatsHandler stats.Handler
    // InitialWindowSize sets the initial window size for a stream.
    InitialWindowSize int32
    // InitialConnWindowSize sets the initial window size for a connection.
    InitialConnWindowSize int32
    // WriteBufferSize sets the size of write buffer which in turn determines how much data can be batched before it's written on the wire.
    WriteBufferSize int
    // ReadBufferSize sets the size of read buffer, which in turn determines how much data can be read at most for one read syscall.
    ReadBufferSize int
    // ChannelzParentID sets the addrConn id which initiate the creation of this client transport.
    ChannelzParentID int64
    // MaxHeaderListSize sets the max (uncompressed) size of header list that is prepared to be received.
    MaxHeaderListSize *uint32
    // UseProxy specifies if a proxy should be used.
    UseProxy bool
}

type serverOptions struct {                                
    creds                 credentials.TransportCredentials                                
    codec                 baseCodec                                
    cp                    Compressor                                
    dc                    Decompressor                                
    unaryInt              UnaryServerInterceptor                                
    streamInt             StreamServerInterceptor                                
    chainUnaryInts        []UnaryServerInterceptor                                
    chainStreamInts       []StreamServerInterceptor                                
    inTapHandle           tap.ServerInHandle                                
    statsHandler          stats.Handler                                
    maxConcurrentStreams  uint32                                
    maxReceiveMessageSize int                                
    maxSendMessageSize    int                                
    unknownStreamDesc     *StreamDesc                                
    keepaliveParams       keepalive.ServerParameters                                
    keepalivePolicy       keepalive.EnforcementPolicy                                
    initialWindowSize     int32                                
    initialConnWindowSize int32                                
    writeBufferSize       int                                
    readBufferSize        int                                
    connectionTimeout     time.Duration                                
    maxHeaderListSize     *uint32                                
    headerTableSize       *uint32                                
    numServerWorkers      uint32                                
}                                

```
## Server internal
At the [Server side](#server-side), ```grpc.Creds()``` sets the value of ```o.creds```. Let's discuss how the server use ```o.creds``` to achive the server side TLS connection. We will not discuss the ```o.unaryInt``` option which is set by ```grpc.UnaryInterceptor()```. There is a dedicated chapter to discuss [Interceptor](interceptor.md)

From the [Serve Request](response.md#serve-request), in the ```s.handleRawConn()``` method. ```s.useTransportAuthenticator()``` will be called before ```s.newHTTP2Transport()``` and ```s.serveStreams()```. The secret is in ```s.useTransportAuthenticator()```. Let's examine it in detail.

![images.004.png](../images/images.004.png)

```go 
// handleRawConn forks a goroutine to handle a just-accepted connection that
// has not had any I/O performed on it yet.
func (s *Server) handleRawConn(rawConn net.Conn) {
    if s.quit.HasFired() {
        rawConn.Close()
        return
    }
    rawConn.SetDeadline(time.Now().Add(s.opts.connectionTimeout))
    conn, authInfo, err := s.useTransportAuthenticator(rawConn)
    if err != nil {
        // ErrConnDispatched means that the connection was dispatched away from
        // gRPC; those connections should be left open.
        if err != credentials.ErrConnDispatched {
            s.mu.Lock()
            s.errorf("ServerHandshake(%q) failed: %v", rawConn.RemoteAddr(), err)
            s.mu.Unlock()
            channelz.Warningf(logger, s.channelzID, "grpc: Server.Serve failed to complete security handshake from %q: %v", rawConn.RemoteAddr(), err)
            rawConn.Close()
        }
        rawConn.SetDeadline(time.Time{})
        return
    }

    // Finish handshaking (HTTP2)
    st := s.newHTTP2Transport(conn, authInfo)
    if st == nil {
        return
    }

    rawConn.SetDeadline(time.Time{})
    if !s.addConn(st) {
        return
    }
    go func() {
        s.serveStreams(st)
        s.removeConn(st)
    }()
}
```
In ```useTransportAuthenticator()```, if ```s.opts.creds``` is not nil, ```crreds.ServerHandshake()``` method will be called.
* when we prepare the ```s.opts.creds```, we call ```credentials.NewServerTLSFromCert()``` to get the ```TransportCredentials ``` object.
* ```NewServerTLSFromCert()``` calls ```NewTLS()``` to finish the job and provides the certificate as parameter.
* ```NewTLS()``` creates a ```tlsCreds``` object and return ```tlsCreds```.
* ```tlsCreds``` is a struct, its ```ServerHandshake()``` method will perform the TLS handshake to establish the secure connection.
* We will not dive into the ```ServerHandshake()```, it's enough to know the that  ```useTransportAuthenticator()``` will use the ```s.opts.creds``` to finish its job.

After the call of ```useTransportAuthenticator()```, the server gets the secure connection.

```go
func (s *Server) useTransportAuthenticator(rawConn net.Conn) (net.Conn, credentials.AuthInfo, error) {
    if s.opts.creds == nil {                                                                                                                          
        return rawConn, nil, nil
    }    
    return s.opts.creds.ServerHandshake(rawConn)
}             
     
// NewServerTLSFromCert constructs TLS credentials from the input certificate for server.                                
func NewServerTLSFromCert(cert *tls.Certificate) TransportCredentials {                                
    return NewTLS(&tls.Config{Certificates: []tls.Certificate{*cert}})                                                                                 
}                                                                                                                 
                                                                                                                          
// NewTLS uses c to construct a TransportCredentials based on TLS.                                                       
func NewTLS(c *tls.Config) TransportCredentials {                                                      
    tc := &tlsCreds{credinternal.CloneTLSConfig(c)}                                                                                                    
    tc.config.NextProtos = credinternal.AppendH2ToNextProtos(tc.config.NextProtos)                                
    return tc                                                                                                             
}                                                                                                                         

// tlsCreds is the credentials required for authenticating a connection using TLS.                                       
type tlsCreds struct {                                                                                 
    // TLS configuration                                                                                                                               
    config *tls.Config                                                                                            
}                                                                                                                         
                                                                                                                          
func (c *tlsCreds) ServerHandshake(rawConn net.Conn) (net.Conn, AuthInfo, error) {
    conn := tls.Server(rawConn, c.config)
    if err := conn.Handshake(); err != nil {
        conn.Close()
        return nil, nil, err
    }
    tlsInfo := TLSInfo{
        State: conn.ConnectionState(),
        CommonAuthInfo: CommonAuthInfo{
            SecurityLevel: PrivacyAndIntegrity,
        },
    }
    id := credinternal.SPIFFEIDFromState(conn.ConnectionState())
    if id != nil {
        tlsInfo.SPIFFEID = id
    }
    return credinternal.WrapSyscallConn(rawConn, conn), tlsInfo, nil
}

```
## Client internal
