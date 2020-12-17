# Authentication
* [Client side](#client-side)
* [Server side](#server-side)
* [DialOption and ServerOption](#dialoption-and-serveroption)

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
  * ``cert``` is a ```Certificate```
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

``` go
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

