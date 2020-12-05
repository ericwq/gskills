# controlBuffer, loopyWriter and framer
* [The bigger picture](#the-bigger-picture)
* [controlBuffer](#controlbuffer)
  * [Component](#component)
  * [Get and Put](#get-and-put)
  * [Threshold](#threshold)
* [loopyWriter](#loopywriter)
* [framer](#framer)

In [Reply with Response](response.md) and [Send Request](request.md), we mentioned ```t.controlBuffer``` several times. How does it works? It's not easy to answer that question. I found that we need the bigger picture to describe the process of gRPC call reply. Without the bigger picture it's hard to understand ```t.controlBuffer```'s responsibility, the role it plays and the relationship with other parts. Without it it's hard to answer the question correctly. 

It's the typical case: answer one question lead to more questions. If you can answer that question, you will know better about gRPC than before.

And gRPC is a full featured framework. Without the bigger picture, it's easy to get lost in the sea of code.

## The bigger picture
The following is the bigger picture of the server side gRPC call. Compare with client side gRPC call, the server side is more complicated. It's notable that some of the concept are the same for the client side. 

The following diagram discribe the most important goroutines, objects and the interaction between them. In the diagram: 
* yellow box represent goroutine, 
* blue box represent an object, 
* pink box represent a go channel,
* dot line represent relationship between two objects,
* arrow line and text represent the action and data flow.

![images.005](../images/images.005.png)

To better understand the diagram, I added the following explaination: 

* for each client connection, gRPC server create one ```ServerTransport``` object and start one ```st.HandleStreams()``` goroutine. 
  * ```ServerTransport``` is an interface, the implemention is ```*http2Server```,
  * if you have more client connections, there will be more ```ServerTransport``` and ```st.HandleStreams()```,
  * See [Code Snippet 01](#code-snippet-01) for detail.
* for eache stream, ```st.HandleStreams()``` goroutine create one ```Stream``` object and start one ```s.handleStream()``` goroutine. 
  * each ```Stream``` object contains a ```buf``` field,
  * the ```buf``` field is of type ```*recvBuffer```, which contains a ```c chan recvMsg``` field,  
  * ```Stream``` object and ```s.handleStream()``` goroutine are created according to gRPC call reqeust,
  * if one client make several gRPC call reqeusts simultaneously, you will get several ```Stream``` and ```s.handleStream()``` pairs. 
  * See [Code Snippet 01](#code-snippet-01) and [Code Snippet 02](#code-snippet-02) for detail.
* at initilization stage, ```ServerTransport``` create one ```t.controlBuf``` object and one ```t.framer.fr``` object and start one ```t.loopy``` goroutine. 
  * Those objects are created per connection and working together to respond to the request.
  * See [Code Snippet 03](#code-snippet-03) for detail.
* ```st.HandleStreams()``` uses ```t.framer.fr``` to read http 2 frames from the connection.
  * if a header frame is read,  ```st.HandleStreams()``` goroutine create one ```Stream``` object and start ```s.handleStream()``` goroutine.
  * if a data frame is read, ```st.HandleStreams()``` try to write the data frame to ```Stream.buf.c``` channel
  * See [Code Snippet 01](#code-snippet-01) for detail.
  * See [Request Parameters](parameters.md) for detail.
* at the same time, ```s.handleStream()``` is processing the header frame and try to read reqeust parameters from channel ```Stream.buf.c```. 
  * there might be several gRPC request calls simultaneously, in this case, multiple ```s.handleStream()``` goroutines will be started,
  * See [Request Parameters](parameters.md) for detail.
* if ```s.handleStreams()``` goroutine want to send gRPC call response. it need to work with ```t.controlBuf```
  * both ```t.controlBuf.put()``` and ```t.controlBuf.executeAndPut()``` can be called to send response.
  * ```*controlBuffer``` is thread safe.
  * ```*controlBuffer``` is a buffer, it will hold the message temporally until ```t.loopy``` get it.
  * See [controlBuffer](#controlbuffer) for detail.
* ```t.loopy``` goroutine get response from ```t.controlBuf``` and send it back via ```t.framer.fr``` 
  * via ```get()``` provided by ```t.controlBuf```, ```t.loopy``` read respnse from  ```t.controlBuf``` ,
  * via ```WriteXXX()``` provided by ```t.framer.fr```, ```t.loopy``` encode the response and send it back to client ,
  * See [loopWriter](#loopwriter) for detail.
* ```t.framer.fr``` is in charge of read/write http 2 frames from/to the wire.
  * it's get initilized when the connection is created.
  * ```Framer``` is of type ```*http2.Framer```. 
  * See [Frame](#frame) for detail.


### Code snippet 01
Upon receive a connection request, gRPC create a ```ServerTransport``` object ```st``` and start a goroutine ```s.serveStreams(st)``` to serve it.
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
+-- 11 lines: ErrConnDispatched means that the connection was dispatched away from················································································
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

```s.serveStreams()``` calls ```st.HandleStreams()``` to do th job. The for loop of ```st.HandleStreams()``` will last until the connection close or some error happens. 

```go
func (s *Server) serveStreams(st transport.ServerTransport) {
    defer st.Close()
    var wg sync.WaitGroup

    var roundRobinCounter uint32
    st.HandleStreams(func(stream *transport.Stream) {
        wg.Add(1)
        if s.opts.numServerWorkers > 0 {
            data := &serverWorkerData{st: st, wg: &wg, stream: stream}
            select {             
            case s.serverWorkerChannels[atomic.AddUint32(&roundRobinCounter, 1)%s.opts.numServerWorkers] <- data:
            default:    
                // If all stream workers are busy, fallback to the default code path.
                go func() {
                    s.handleStream(st, stream, s.traceInfo(st, stream))
                    wg.Done()                             
                }()                                          
            }                                                                                            
        } else {                      
            go func() {                                    
                defer wg.Done()         
                s.handleStream(st, stream, s.traceInfo(st, stream))
            }()
        }
    }, func(ctx context.Context, method string) context.Context {
        if !EnableTracing {
            return ctx
        }
        tr := trace.New("grpc.Recv."+methodFamily(method), method)
        return trace.NewContext(ctx, tr)
    })
    wg.Wait()
}

// HandleStreams receives incoming streams using the given handler. This is
// typically run in a separate goroutine.
// traceCtx attaches trace to ctx and returns the new context.
func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
    defer close(t.readerDone)                                    
    for {                                                                    
        t.controlBuf.throttle()                         
        frame, err := t.framer.fr.ReadFrame()     
        atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())    
        if err != nil {                         
+-- 28 lines: if se, ok := err.(http2.StreamError); ok {··········································································································
        }                                                        
        switch frame := frame.(type) {                                                                           
        case *http2.MetaHeadersFrame:                                        
            if t.operateHeaders(frame, handle, traceCtx) {
                t.Close()                         
                break                                                 
            }                                                           
        case *http2.DataFrame:                                          
            t.handleData(frame)                  
        case *http2.RSTStreamFrame:                            
            t.handleRSTStream(frame)                           
        case *http2.SettingsFrame:                  
            t.handleSettings(frame)       
        case *http2.PingFrame:              
            t.handlePing(frame)
        case *http2.WindowUpdateFrame:
            t.handleWindowUpdate(frame)
        case *http2.GoAwayFrame:
            // TODO: Handle GoAway from the client appropriately.
        default:         
            if logger.V(logLevel) {
                logger.Errorf("transport: http2Server.HandleStreams found unhandled frame type %v.", frame)
            }                      
        }                                                                                            
    }        
}                    
 
```
### Code snippet 02
Upon receive the gRPC call request header, ```HandleStreams()``` calls ```t.operateHeaders()```, which create the ```Stream``` object ```s``` and start a goroutine ```s.handleStream()``` to process that stream. 

In ```newRecvBuffer()``` , a ```*recvBuffer``` will be created and assigned to ```Stream.buf```. ```*recvBuffer``` has a field ```c chan recvMsg```. 

```go
// operateHeader takes action on the decoded headers.
func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
    streamID := frame.Header().StreamID
    state := &decodeState{
        serverSide: true,
    }
    if h2code, err := state.decodeHeader(frame); err != nil {
+--  9 lines: if _, ok := status.FromError(err); ok {·············································································································
    }

    buf := newRecvBuffer()
    s := &Stream{
        id:             streamID,
        st:             t,
        buf:            buf,
        fc:             &inFlow{limit: uint32(t.initialWindowSize)},
        recvCompress:   state.data.encoding,
        method:         state.data.method,
        contentSubtype: state.data.contentSubtype,
    }
+-- 73 lines: if frame.StreamEnded() {····························································································································
    t.maxStreamID = streamID                
    t.activeStreams[streamID] = s
    if len(t.activeStreams) == 1 {
        t.idle = time.Time{}                         
    }                               
+--  5 lines: t.mu.Unlock()·······································································································································
    s.requestRead = func(n int) {
        t.adjustWindow(s, uint32(n))                
    }                                                                            
+-- 13 lines: s.ctx = traceCtx(s.ctx, s.method)···················································································································
    s.ctxDone = s.ctx.Done()     
    s.wq = newWriteQuota(defaultWriteQuota, s.ctxDone)                            
    s.trReader = &transportReader{  
        reader: &recvBufferReader{   
            ctx:        s.ctx,
            ctxDone:    s.ctxDone,
            recv:       s.buf,
            freeBuffer: t.bufferPool.put,
        },
        windowHandler: func(n int) {
            t.updateWindow(s, uint32(n))
        },
    }
    // Register the stream with loopy.
    t.controlBuf.put(&registerStream{
        streamID: s.id,
        wq:       s.wq,
    })
    handle(s)
    return false
}

// recvBuffer is an unbounded channel of recvMsg structs.
//                                                            
// Note: recvBuffer differs from buffer.Unbounded only in the fact that it
// holds a channel of recvMsg structs instead of objects implementing "item"
// interface. recvBuffer is written to much more often and using strict recvMsg
// structs helps avoid allocation in "recvBuffer.put"
type recvBuffer struct {
    c       chan recvMsg          
    mu      sync.Mutex     
    backlog []recvMsg        
    err     error                                                    
}                                            
                                           
func newRecvBuffer() *recvBuffer {                 
    b := &recvBuffer{
        c: make(chan recvMsg, 1),
    }                                        
    return b                      
}

```
### Code snippet 03
```newHTTP2Transport()``` calls  ```transport.NewServerTransport()```, which calls ```newHTTP2Server()```. 
* In ```newHTTP2Server()```, ```newFramer()``` create a ```framer``` and assign it to ```http2Server.framer```.  
* In ```newHTTP2Server()```, ```newControlBuffer()``` create a ```*controlBuffer``` and assign it to ```http2Server.controlBuf```.  
* In ```newFramer()```, ```t.framer.fr``` is initilized by ```http2.NewFramer()```.
* At the end of ```newHTTP2Transport()```, it creates ```t.loopy``` with ```newLoopyWriter()``` and start a ```t.loopy``` goroutine.

```go
// newHTTP2Transport sets up a http/2 transport (using the                                                                                                   
// gRPC http2 server transport in transport/http2_server.go).                                                                                           
func (s *Server) newHTTP2Transport(c net.Conn, authInfo credentials.AuthInfo) transport.ServerTransport {                                         
    config := &transport.ServerConfig{                                                                                                           
        MaxStreams:            s.opts.maxConcurrentStreams,                                                              
        AuthInfo:              authInfo,                                                                                 
        InTapHandle:           s.opts.inTapHandle,                                                                       
        StatsHandler:          s.opts.statsHandler,                                                                      
        KeepaliveParams:       s.opts.keepaliveParams,                                                                   
        KeepalivePolicy:       s.opts.keepalivePolicy,                                                                   
        InitialWindowSize:     s.opts.initialWindowSize,                                                                 
        InitialConnWindowSize: s.opts.initialConnWindowSize,                                                             
        WriteBufferSize:       s.opts.writeBufferSize,                                                                   
        ReadBufferSize:        s.opts.readBufferSize,                                                                                            
        ChannelzParentID:      s.channelzID,                                                                             
        MaxHeaderListSize:     s.opts.maxHeaderListSize,                                                                 
        HeaderTableSize:       s.opts.headerTableSize,                                                                                                  
    }                                                                                                                    
    st, err := transport.NewServerTransport("http2", c, config)                                                                                             
    if err != nil {                                                                                                                                         
        s.mu.Lock()                                                                                                                                
        s.errorf("NewServerTransport(%q) failed: %v", c.RemoteAddr(), err)                                                                             
        s.mu.Unlock()                                                                                                                               
        c.Close()                                                                                                                                   
        channelz.Warning(logger, s.channelzID, "grpc: Server.Serve failed to create ServerTransport: ", err)                                      
        return nil                                                                                                                                  
                                                                                                                                                               
    return st                                                                                                                         
}                                                                                                                                              

// NewServerTransport creates a ServerTransport with conn or non-nil error                                                
// if it fails.                                                                                                           
func NewServerTransport(protocol string, conn net.Conn, config *ServerConfig) (ServerTransport, error) {                                          
    return newHTTP2Server(conn, config)                                                                                   
}                                                                                                                         

// newHTTP2Server constructs a ServerTransport based on HTTP2. ConnectionError is
// returned if something goes wrong.
func newHTTP2Server(conn net.Conn, config *ServerConfig) (_ ServerTransport, err error) {
    writeBufSize := config.WriteBufferSize
    readBufSize := config.ReadBufferSize
    maxHeaderListSize := defaultServerMaxHeaderListSize
    if config.MaxHeaderListSize != nil {
        maxHeaderListSize = *config.MaxHeaderListSize
    }
    framer := newFramer(conn, writeBufSize, readBufSize, maxHeaderListSize)
    // Send initial settings as connection preface to client.
    isettings := []http2.Setting{{
        ID:  http2.SettingMaxFrameSize,
        Val: http2MaxFrameLen,
    }}
    // TODO(zhaoq): Have a better way to signal "no limit" because 0 is
    // permitted in the HTTP2 spec.
+-- 37 lines: maxStreams := config.MaxStreams·····················································································································
    if err := framer.fr.WriteSettings(isettings...); err != nil {
        return nil, connectionErrorf(false, err, "transport: %v", err)
    }
+-- 28 lines: Adjust the connection flow control window if needed.································································································
    done := make(chan struct{})
    t := &http2Server{
        ctx:               context.Background(),
        done:              done,
        conn:              conn,
        remoteAddr:        conn.RemoteAddr(),
        localAddr:         conn.LocalAddr(),
        authInfo:          config.AuthInfo,
        framer:            framer,
        readerDone:        make(chan struct{}),
        writerDone:        make(chan struct{}),
        maxStreams:        maxStreams,
        inTapHandle:       config.InTapHandle,
        fc:                &trInFlow{limit: uint32(icwz)},
        state:             reachable,
        activeStreams:     make(map[uint32]*Stream),
        stats:             config.StatsHandler,
        kp:                kp,
        idle:              time.Now(),
        kep:               kep,
         initialWindowSize: iwz,
        czData:            new(channelzData),
        bufferPool:        newBufferPool(),
    }
    t.controlBuf = newControlBuffer(t.done)
+-- 27 lines: if dynamicWindow {··································································································································

    // Check the validity of client preface.
    preface := make([]byte, len(clientPreface))
+-- 19 lines: if _, err := io.ReadFull(t.conn, preface); err != nil {·····························································································
    t.handleSettings(sf)

    go func() {
        t.loopy = newLoopyWriter(serverSide, t.framer, t.controlBuf, t.bdpEst)
        t.loopy.ssGoAwayHandler = t.outgoingGoAwayHandler
        if err := t.loopy.run(); err != nil {
            if logger.V(logLevel) {
            }                                                                      
        }
        t.conn.Close()
        close(t.writerDone)
    }()
    go t.keepalive()
    return t, nil
}

func newFramer(conn net.Conn, writeBufferSize, readBufferSize int, maxHeaderListSize uint32) *framer {
    if writeBufferSize < 0 {                                                                                                      
        writeBufferSize = 0      
    }                            
    var r io.Reader = conn                    
    if readBufferSize > 0 {                  
        r = bufio.NewReaderSize(r, readBufferSize)
    }                              
    w := newBufWriter(conn, writeBufferSize)    
    f := &framer{                               
        writer: w,                     
        fr:     http2.NewFramer(w, r),         
    }                                                      
    f.fr.SetMaxReadFrameSize(http2MaxFrameLen)
    // Opt-in to Frame reuse API on framer to reduce garbage.
    // Frames aren't safe to read from after a subsequent call to ReadFrame.
    f.fr.SetReuseFrames()      
    f.fr.MaxHeaderListSize = maxHeaderListSize
    f.fr.ReadMetaHeaders = hpack.NewDecoder(http2InitHeaderTableSize, nil)
    return f                    
}                                             
```
## controlBuffer
From the bigger picture, the ```controlBuffer``` is the buffer when you wan to send message to ```loopy```. Specifically speaking the buffer is the ```list *itemList```. ```controlBuffer``` can be initilized by ```newControlBuffer()```. 

### Component
* ```mu sync.Mutex``` is used to protect the ```list *itemList```
* ```list *itemList``` is a normal linked list. It's the buffer to temporally store the message.
* ```ch chan struct{}``` and ```consumerWaiting bool``` is used for the bloking read mode. see [Get and Put](#get-and-put) for detail.
* ```done <-chan struct{}``` is used for the stop operation.
* ```transportResponseFrames int``` and ```trfChan atomic.Value``` is used for threshold. see [Threshold](#threshold) for detail.
* ```atomic.Value``` provides an atomic load and store of a consistently typed value. see [official doc](https://golang.org/pkg/sync/atomic/#Value) for detail.

```go
// controlBuffer is a way to pass information to loopy.                                                                    
// Information is passed as specific struct types called control frames.                                                                           
// A control frame not only represents data, messages or headers to be sent out                                                        
// but can also be used to instruct loopy to update its internal state.                                                                  
// It shouldn't be confused with an HTTP2 frame, although some of the control frames                                                                 
// like dataFrame and headerFrame do go out on wire as HTTP2 frames.                                                                       
type controlBuffer struct {                                                                                                               
    ch              chan struct{}                                                                                                              
    done            <-chan struct{}                                                                                                           
    mu              sync.Mutex                                                                                                                              
    consumerWaiting bool                                                                                                                             
    list            *itemList                                                                                                           
    err             error                                                                                                 
                                                                                                                          
    // transportResponseFrames counts the number of queued items that represent                                                        
    // the response of an action initiated by the peer.  trfChan is created                                               
    // when transportResponseFrames >= maxQueuedTransportResponseFrames and is                                                                       
    // closed and nilled when transportResponseFrames drops below the                                                                    
    // threshold.  Both fields are protected by mu.                                                                                       
    transportResponseFrames int                                                                                           
    trfChan                 atomic.Value // *chan struct{}                                                                 
}                                                                                                                                              
                                                                                                                                       
func newControlBuffer(done <-chan struct{}) *controlBuffer {                                                                             
    return &controlBuffer{                                                                                                                     
        ch:   make(chan struct{}, 1),                                                                                                    
        list: &itemList{},                                                                                                                         
        done: done,                                                                                                                             
    }
}

type itemNode struct {
    it   interface{}
    next *itemNode
}                                                                                                                         
                                                                                                                          
type itemList struct {                                                                                                                 
    head *itemNode                                                                                                        
    tail *itemNode                                                                                                                                   
}                                                                                                                                        
                                                                                                                                          
func (il *itemList) enqueue(i interface{}) {                                                                              
    n := &itemNode{it: i}                                                                                                  
    if il.tail == nil {                                                                                                                        
        il.head, il.tail = n, n                                                                                                        
        return                                                                                                                           
    }                                                                                                                                          
    il.tail.next = n                                                                                                                     
    il.tail = n                                                                                                                                    
}                                                                                                                                               
     
// peek returns the first item in the list without removing it from the
// list.
func (il *itemList) peek() interface{} {                                       
    return il.head.it
}                                   
                                              
func (il *itemList) dequeue() interface{} {
    if il.head == nil {
        return nil
    }
    i := il.head.it
    il.head = il.head.next
    if il.head == nil {
        il.tail = nil
    }
    return i
}
```

### Get and Put
Under the protection of ```mu```, ```put()``` and ```executeAndPut()``` store the item in buffer ```c.list```.
* counts ```c.transportResponseFrames```, if the buffer size exceed ```maxQueuedTransportResponseFrames```, create a throttling channel, 
* compare with ```put()```, ```executeAndPut()``` add a extra step: try to execute the ```f func(it interface{}) bool``` before the put action,
* if ```c.consumerWaiting``` is true,  send the the signal (```struct{}{}```) to ```c.ch```, to wake up the get operation. After that the blocked read operation can continue now.

Also under the protection of ```mu```, ```get()``` fetch the item from buffer ```c.list```.
* count down ```c.transportResponseFrames```, if the buffer size less than ```maxQueuedTransportResponseFrames```, close the throttling channel,
* if run in blocking mode, ```block``` parameter is true, if the buffer is empty, ```get()``` will wait signal from ```c.ch```, until ```put()``` send the signal.

In general, ```*controlBuffer``` is thread safe. 

```go
// maxQueuedTransportResponseFrames is the most queued "transport response"
// frames we will buffer before preventing new reads from occurring on the
// transport.  These are control frames sent in response to client requests,
// such as RST_STREAM due to bad headers or settings acks.
const maxQueuedTransportResponseFrames = 50
 
type cbItem interface {
    isTransportResponseFrame() bool                                                          
}                  

func (c *controlBuffer) put(it cbItem) error {
    _, err := c.executeAndPut(nil, it)
    return err
}

func (c *controlBuffer) executeAndPut(f func(it interface{}) bool, it cbItem) (bool, error) {
    var wakeUp bool
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return false, c.err
    }
    if f != nil {
        if !f(it) { // f wasn't successful
            c.mu.Unlock()
            return false, nil
        }
    }
    if c.consumerWaiting {
        wakeUp = true
        c.consumerWaiting = false
    }
    c.list.enqueue(it)
    if it.isTransportResponseFrame() {
        c.transportResponseFrames++
        if c.transportResponseFrames == maxQueuedTransportResponseFrames {
            // We are adding the frame that puts us over the threshold; create
            // a throttling channel.
            ch := make(chan struct{})
            c.trfChan.Store(&ch)
        }
    }
    c.mu.Unlock()
    if wakeUp {
        select {
        case c.ch <- struct{}{}:
        default:
        }
    }
    return true, nil
}
 
func (c *controlBuffer) get(block bool) (interface{}, error) {                                                                                                  
    for {                                                                                                                                            
        c.mu.Lock()                                                                                                                                            
        if c.err != nil {                                                                                                                  
            c.mu.Unlock()                                                                                                                        
            return nil, c.err                                                                                                                      
        }                                                                                                                                                   
        if !c.list.isEmpty() {                                                                                                                                  
            h := c.list.dequeue().(cbItem)                                                                                                        
            if h.isTransportResponseFrame() {                                                                             
                if c.transportResponseFrames == maxQueuedTransportResponseFrames {                                        
                    // We are removing the frame that put us over the                                                                          
                    // threshold; close and clear the throttling channel.                                                 
                    ch := c.trfChan.Load().(*chan struct{})                                                                                          
                    close(*ch)                                                                                                                        
                    c.trfChan.Store((*chan struct{})(nil))                                                                                             
                }                                                                                                         
                c.transportResponseFrames--                                                                                                                     
            }                                                                                                                                      
            c.mu.Unlock()                                                                                                                          
            return h, nil                                                                                                                     
        }                                                                                                                                      
        if !block {                                                                                                                                  
            c.mu.Unlock()                                                                                                                          
            return nil, nil                                                                                                                     
        }                                                                                                                                        
        c.consumerWaiting = true                                                                                                                                
        c.mu.Unlock()                                                                                                                                        
        select {                                                                                                                                     
        case <-c.ch:                                                                                                                                
        case <-c.done:                                                                                                                         
            c.finish()                                                                                                                                            
            return nil, ErrConnClosing                                                                                                           
        }                                                                                                                                          
    }                                                                                                                                                
}

// Note argument f should never be nil.
func (c *controlBuffer) execute(f func(it interface{}) bool, it interface{}) (bool, error) {
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return false, c.err
    }
    if !f(it) { // f wasn't successful
        c.mu.Unlock()
        return false, nil
    }
    c.mu.Unlock()
    return true, nil
}
``` 

### Threshold

User of ```controlBuffer``` need to explicitly call ```throttle()``` to make the threshold control work. if ```c.trfChan``` is not nil, ```throttle()``` will wait, until the threshhold released. 

Here we show some code snippet from ```HandleStreams()```. It's the typical use case for ```t.controlBuf.throttle() ```.

```go
// throttle blocks if there are too many incomingSettings/cleanupStreams in the
// controlbuf.
func (c *controlBuffer) throttle() {
    ch, _ := c.trfChan.Load().(*chan struct{})
    if ch != nil {
        select {
        case <-*ch:
        case <-c.done:
        }
    }
}

// HandleStreams receives incoming streams using the given handler. This is                                                                    
// typically run in a separate goroutine.                                                                                                   
// traceCtx attaches trace to ctx and returns the new context.                                                                                
func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {                           
    defer close(t.readerDone)                                                                                                                                 
    for {                                                                                                                                              
        t.controlBuf.throttle()                                                                                                                                
        frame, err := t.framer.fr.ReadFrame()                                                                                                                
        atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())                                                                                       
    ...
    }
    ...
}

```

## loopyWriter

still need some time.

## framer
```framer``` is a struct. ```newFramer``` builds a ```*framer``` object. I think it decorates the ```http2.NewFramer()``` by add read/write buffer capability. 

From the function body of ```newFramer``` you can understand what it is:
* use ```bufio.NewReaderSize()``` and ```conn``` to build a ```r``` object, thus add the buffer capability to ```conn```,
* use ```newBufWriter()``` and ```conn``` to build a ```w``` object, thus add the buffer capability to ```conn```,
* ```*bufWriter``` is a simple buffer writer, it implements ```io.Writer``` interface. with addtional ```Flush()``` method,
* ```fr``` is initilized by ```http2.NewFramer()```, using the just created ```w``` and ```r```, which means ```fr``` is using the bufferd ```Reader``` and ```Writer```
* actually, the most functionality of ```framer``` is provided by ```http2.Framer```. It provides a lot of method to read/write frame data. Such as ```WriteHeaders()```, ```WriteData()```, ```ReadFrame()``` etc.
* we will not discuss the implementation of ```http2.NewFramer()```, it's out of our scope. see [offical document](https://pkg.go.dev/golang.org/x/net/http2) for more detail.

The only limitation of ```framer``` is the user need to call ```framer.writer.Flush()``` to clean the writer buffer. 

There are two questions confuse me:
* ```http2.NewFramer()``` already has the read buffer ```readBuf  []byte``` , why use ```bufio.Reader```? 
* ```http2.NewFramer()``` also has the write buffer ```wbuf []byte```, why use ```bufWriter```?

Anyway, the buffered ```w``` and ```r``` are still working. That's the most important.

```go
type framer struct {                                                                                                                                  
    writer *bufWriter                                                                                                                                 
    fr     *http2.Framer                                                                                                                        
}                                                                                                                                             
                                                                                                                                              
func newFramer(conn net.Conn, writeBufferSize, readBufferSize int, maxHeaderListSize uint32) *framer {                                          
    if writeBufferSize < 0 {                                                                                                                
        writeBufferSize = 0                                                                                                                               
    }                                                                                                                                         
    var r io.Reader = conn                                                                                                                         
    if readBufferSize > 0 {                                                                                                                         
        r = bufio.NewReaderSize(r, readBufferSize)                                                                                               
    }                                                                                                                                             
    w := newBufWriter(conn, writeBufferSize)                                                                                                             
    f := &framer{                                                                                                                                     
        writer: w,                                                                                                                              
        fr:     http2.NewFramer(w, r),                                                                                     
    }                                                                                                                                           
    f.fr.SetMaxReadFrameSize(http2MaxFrameLen)                                                                             
    // Opt-in to Frame reuse API on framer to reduce garbage.                                                                         
    // Frames aren't safe to read from after a subsequent call to ReadFrame.                                              
    f.fr.SetReuseFrames()                                                                                                 
    f.fr.MaxHeaderListSize = maxHeaderListSize                                                                            
    f.fr.ReadMetaHeaders = hpack.NewDecoder(http2InitHeaderTableSize, nil)                                                                                         
    return f                                                                                                              
}                                                                                                                                                                

type bufWriter struct {
    buf       []byte
    offset    int     
    batchSize int  
    conn      net.Conn
    err       error
                                                            
    onFlush func()                           
}                                                           
                             
func newBufWriter(conn net.Conn, batchSize int) *bufWriter {
    return &bufWriter{       
        buf:       make([]byte, batchSize*2),
        batchSize: batchSize,
        conn:      conn,                                
    }                  
}                                                       
                     
func (w *bufWriter) Write(b []byte) (n int, err error) {
    if w.err != nil {         
        return 0, w.err                               
    }                         
    if w.batchSize == 0 { // Buffer has been disabled.
        return w.conn.Write(b)
    }                                  
    for len(b) > 0 {
        nn := copy(w.buf[w.offset:], b)
        b = b[nn:]         
        w.offset += nn              
        n += nn            
        if w.offset >= w.batchSize {
            err = w.Flush()        
        }            
    }                              
    return n, err
}

func (w *bufWriter) Flush() error {
    if w.err != nil {
        return w.err
    }
    if w.offset == 0 {
        return nil
    }
    if w.onFlush != nil {
        w.onFlush()
    }
    _, w.err = w.conn.Write(w.buf[:w.offset])
    w.offset = 0
    return w.err
}
```
