# Request parameters
* [The problem](#the-problem)
* [The clue](#the-clue)
* [Lock the method](#lock-the-method)
* [Message reader](#message-reader)
* [Message sender](#message-sender)

At [Serve stream](response.md#serve-stream), there is a problem we didn't discuss the detail. In one word, How does the server read the request parameter?

## The problem
According to the [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md). The request is composed by the following parts.
```
Request → Request-Headers *Length-Prefixed-Message EOS. 
```
Let's describe the probelm in detail. In [Serve stream](response.md#serve-stream), 
* ```st.HandleStreams()``` are called to handle the stream and run in its goroutine.
* Meanwhile, ```s.handleStream()``` is called to handle the gRPC method call and run in its goroutine.
* ```st.HandleStreams()``` will read the frame from the wire continuesly
* ```s.handleStream()``` also need to read the request parameter.

Now you has the same problem as I had: How does the two goroutine communicate with each other?

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
            if se, ok := err.(http2.StreamError); ok {                                                                                              
                if logger.V(logLevel) {                                                                                                            
                    logger.Warningf("transport: http2Server.HandleStreams encountered http2.StreamError: %v", se)                              
                }                                                                                                                                  
                t.mu.Lock()                                                                                                                       
                s := t.activeStreams[se.StreamID]                                                                                                    
                t.mu.Unlock()                                                                                                                            
                if s != nil {                                                                                                                         
                    t.closeStream(s, true, se.Code, false)                                                                                           
                } else {                                                                                                                          
                    t.controlBuf.put(&cleanupStream{                                                                                     
                        streamID: se.StreamID,                                                                             
                        rst:      true,                                                                                                        
                        rstCode:  se.Code,                                                                                             
                        onWrite:  func() {},                                                                               
                    })                                                                                                                       
                }                                                                                                                            
                continue                                                                                                                               
            }                                                                                                              
            if err == io.EOF || err == io.ErrUnexpectedEOF {                                                               
                t.Close()                                                                                                  
                return                                                                                                                                           
            }                                                                                                              
            if logger.V(logLevel) {                                                                                        
                logger.Warningf("transport: http2Server.HandleStreams failed to read frame: %v", err)                      
            }                                                                                                              
            t.Close()                                                                                                      
            return                                                                                                                                        
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

func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
    sm := stream.Method()
    if sm != "" && sm[0] == '/' {
        sm = sm[1:]
    }
    pos := strings.LastIndex(sm, "/")
    if pos == -1 {
        if trInfo != nil {
            trInfo.tr.LazyLog(&fmtStringer{"Malformed method name %q", []interface{}{sm}}, true)
            trInfo.tr.SetError()
        }
+-- 12 lines: errDesc := fmt.Sprintf("malformed method name: %q", stream.Method())··················································································
    }                                                                                              
    service := sm[:pos]       
    method := sm[pos+1:]                                                       
                                    
    srv, knownService := s.services[service]
    if knownService {                                                                                           
        if md, ok := srv.methods[method]; ok {
            s.processUnaryRPC(t, stream, srv, md, trInfo)
            return            
        }
        if sd, ok := srv.streams[method]; ok {
            s.processStreamingRPC(t, stream, srv, sd, trInfo)
            return     
        }               
    }
    // Unknown service, or known server unknown method.
    if unknownDesc := s.opts.unknownStreamDesc; unknownDesc != nil {
        s.processStreamingRPC(t, stream, nil, unknownDesc, trInfo)
        return                                           
    }             
+-- 20 lines: var errDesc string····································································································································
}                                             
```
## The clue

I know the clue part is too simple to believe it. Yes, sometimes the answer is simple, while it took me some time to find it. No magic, pity.

In ```s.handleStream()``` 
* before the invocation of ```recvAndDecompress()``` there is no sign of reading the reqeust parameter.
* after ```recvAndDecompress()``` the gRPC is ready to call ```md.Handler()```. 

So the ```recvAndDecompress()``` must did something.
```go
func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, info *serviceInfo, md *MethodDesc, trInfo *traceInfo) (err error) {
+-- 80 lines: sh := s.opts.statsHandler·····························································································································
    // comp and cp are used for compression.  decomp and dc are used for
    // decompression.  If comp and decomp are both set, they are the same;
    // however they are kept separate to ensure that at most one of the
    // compressor/decompressor variable pairs are set for use later.
    var comp, decomp encoding.Compressor
    var cp Compressor
    var dc Decompressor

+-- 27 lines: If dc is set and matches the stream's compression, use it.  Otherwise, try············································································

    var payInfo *payloadInfo
    if sh != nil || binlog != nil {
        payInfo = &payloadInfo{}
    }
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
        if err := s.getCodec(stream.ContentSubtype()).Unmarshal(d, v); err != nil {
            return status.Errorf(codes.Internal, "grpc: error unmarshalling request: %v", err)
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
    if appErr != nil {
+-- 27 lines: appStatus, ok := status.FromError(appErr)·············································································································
    }
    if trInfo != nil {
        trInfo.tr.LazyLog(stringer("OK"), false)
    }
    opts := &transport.Options{Last: true}

    if err := s.sendResponse(t, stream, reply, cp, opts, comp); err != nil {
+-- 27 lines: if err == io.EOF {····································································································································
    }
+-- 15 lines: if binlog != nil {····································································································································
    // TODO: Should we be logging if writing status failed here, like above?
    // Should the logging be in WriteStatus?  Should we ignore the WriteStatus
    // error or allow the stats handler to see it?
    err = t.WriteStatus(stream, statusOK)
    if binlog != nil {
        binlog.Log(&binarylog.ServerTrailer{
            Trailer: stream.Trailer(),
            Err:     appErr,
        })
    }
    return err
}
```
In ```recvAndDecompress()```, ```p.recvMsg()``` is called to read the messsage. Follow it.

```go
func recvAndDecompress(p *parser, s *transport.Stream, dc Decompressor, maxReceiveMessageSize int, payInfo *payloadInfo, compressor encoding.Compressor) ([]byte, er  ror) {                             
    pf, d, err := p.recvMsg(maxReceiveMessageSize)
    if err != nil {
        return nil, err                                                                                       
    }                              
    if payInfo != nil {                                               
        payInfo.wireLength = len(d)                                                                             
    }                                                                                           
                            
    if st := checkRecvPayload(pf, s.RecvCompress(), compressor != nil || dc != nil); st != nil {
        return nil, st.Err()
    }                  
                              
    var size int                                                                                       
    if pf == compressionMade {                                                     
        // To match legacy behavior, if the decompressor is set by WithDecompressor or RPCDecompressor,
        // use this decompressor as the default.
        if dc != nil {   
            d, err = dc.Do(bytes.NewReader(d))                                                                                                                 
            size = len(d)                                                  
        } else {
            d, size, err = decompress(compressor, d, maxReceiveMessageSize)
        }                                                                                                       
        if err != nil {
            return nil, status.Errorf(codes.Internal, "grpc: failed to decompress the received message %v", err)
        }            
    } else {
        size = len(d)                
    }                                                                          
    if size > maxReceiveMessageSize {
        // TODO: Revisit the error code. Currently keep it consistent with java                                                              
        // implementation.
        return nil, status.Errorf(codes.ResourceExhausted, "grpc: received message larger than max (%d vs. %d)", size, maxReceiveMessageSize)
    }
    return d, nil
}
```
## Lock the method
In ```recvMsg()```, It looks like ```p.r.Read()``` read the data frame. From  [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) we know the following:
```
The repeated sequence of Length-Prefixed-Message items is delivered in DATA frames

* Length-Prefixed-Message → Compressed-Flag Message-Length Message
* Compressed-Flag → 0 / 1 # encoded as 1 byte unsigned integer
* Message-Length → {length of Message} # encoded as 4 byte unsigned integer (big endian)
* Message → *{binary octet}
```

After carefully check the source code of ```recvMsg()```. 
* ```pf``` is the *Compressed-Flag*. 
* ```length``` is the *Message-Length*.
* ```msg``` is the *Message*

It's clear that ```p.r.Read()``` is used to read the data frame. From the ```parser``` definition, it's just a normal struct with a recvMsg method. Its ```header [5]byte``` field is normal, while the ```r io.Reader``` field is the suspicious. Let's check the ```p.r.Read()``` method.

```go
// parser reads complete gRPC messages from the underlying reader.
type parser struct {
    // r is the underlying reader.
    // See the comment on recvMsg for the permissible
    // error types.
    r io.Reader                                                            
                                                                         
    // The header of a gRPC message. Find more detail at
    // https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md
    header [5]byte                                                      
}

// The format of the payload: compressed or not?
type payloadFormat uint8

// recvMsg reads a complete gRPC message from the stream.
//
// It returns the message and its payload (compression/encoding)
// format. The caller owns the returned msg memory.
//                          
// If there is an error, possible values are:                                                   
//   * io.EOF, when no messages remain
//   * io.ErrUnexpectedEOF
//   * of type transport.ConnectionError
//   * an error from the status package                                                                
// No other error values or types must be returned, which also means               
// that the underlying io.Reader must not return an incompatible                                       
// error.                                       
func (p *parser) recvMsg(maxReceiveMessageSize int) (pf payloadFormat, msg []byte, err error) {
    if _, err := p.r.Read(p.header[:]); err != nil {                                                                                                           
        return 0, nil, err                                                 
    }           
                                                                           
    pf = payloadFormat(p.header[0])                                                                             
    length := binary.BigEndian.Uint32(p.header[1:])
                                                                                                                
    if length == 0 { 
        return pf, nil, nil
    }                                
    if int64(length) > int64(maxInt) {                                         
        return 0, nil, status.Errorf(codes.ResourceExhausted, "grpc: received message larger than max length allowed on current machine (%d vs. %d)", length, maxInt  )                                                                                                                                            
    }                     
    if int(length) > maxReceiveMessageSize {                                                                                                 
        return 0, nil, status.Errorf(codes.ResourceExhausted, "grpc: received message larger than max (%d vs. %d)", length, maxReceiveMessageSize)
    }            
    // TODO(bradfitz,zhaoq): garbage. reuse buffer after proto decoding instead
    // of making it for each message:
    msg = make([]byte, int(length))                        
    if _, err := p.r.Read(msg); err != nil {
        if err == io.EOF {
            err = io.ErrUnexpectedEOF
        }
        return 0, nil, err
    }
    return pf, msg, nil
}
```
The value of ```p``` is assigned through the following statement:

```go
d, err := recvAndDecompress(&parser{r: stream}, stream, dc, s.opts.maxReceiveMessageSize, payInfo, decomp)
```
The following call stack will happens: 
* ```p.r.Read()``` calls ```Stream.Read()```,
* ```Stream.Read()``` calls ```s.requestRead()```, which calls ```t.adjustWindow()```, no data read, ignore it.
* ```Stream.Read()``` calls ```io.ReadFull(s.trReader, p)```, which calls ```s.trReader.Read()```
* ```s.trReader.Read()``` calls ``` t.reader.Read(p)```, 
* ```t.reader.Read(p)``` calls ```recvBufferReader.Read()```,
* ```recvBufferReader.Read()``` calls ```recvBufferReader.read()```,
* ```recvBufferReader.read()``` read the ```recvMsg``` from channel ```m := <-r.recv.get()``` and calls ```recvBufferReader.readAdditional()``` to finish the read action.

Let's check the ```r.recv.get()```.

For convenience, All realated code is showned in one place.
```go
// Read reads all p bytes from the wire for this stream.                                                                                          
func (s *Stream) Read(p []byte) (n int, err error) {                                                                                               
    // Don't request a read if there was an error earlier                                                                                               
    if er := s.trReader.(*transportReader).er; er != nil {                                                                                                
        return 0, er                                                                                                                                    
    }                                                                                                                                                  
    s.requestRead(len(p))                                                                                                                             
    return io.ReadFull(s.trReader, p)                                                                                                                     
}                                                                                                                                                      

// operateHeader takes action on the decoded headers.                                                                                                
func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
   ...
   s.requestRead = func(n int) {                                                                                                                           
       t.adjustWindow(s, uint32(n))                                                                                                                     
   }                                                                                                                                                       
   ...
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
   ...
}

// adjustWindow sends out extra window update over the initial window size                                                                                          
// of stream if the application is requesting data larger in size than                                                        
// the window.                                                                                                                
func (t *http2Server) adjustWindow(s *Stream, n uint32) {
    if w := s.fc.maybeAdjust(n); w > 0 {                                                                                      
        t.controlBuf.put(&outgoingWindowUpdate{streamID: s.id, increment: w})                                                 
    }                                                                                                                                                        
                                                                                                                              
}                                                                                                                                                                
                                                                                                                                                                 
// updateWindow adjusts the inbound quota for the stream and the transport.                                                   
// Window updates will deliver to the controller for sending when                                                             
// the cumulative quota exceeds the corresponding threshold.                                                                  
func (t *http2Server) updateWindow(s *Stream, n uint32) {                                                                                                       
    if w := s.fc.onRead(n); w > 0 {                                                                                           
        t.controlBuf.put(&outgoingWindowUpdate{streamID: s.id,                                                                
            increment: w,                                                                                                                                 
        })                                                                                                                                                           
    }                                                                                                                                                               
}                                                                                                                             

// tranportReader reads all the data available for this Stream from the transport and                                                             
// passes them into the decoder, which converts them into a gRPC message stream.                                                          
// The error is io.EOF when the stream is done or another non-nil error if
// the stream broke.                                                                                                                            
type transportReader struct {                                                                                                                   
    reader io.Reader                                                                                                                                      
    // The handler to control the window update procedure for both this                                                       
    // particular stream and the associated transport.
    windowHandler func(int)                                                                                                   
    er            error                                                                                                                                             
}                                                                                                                             
                                                                                                                              
func (t *transportReader) Read(p []byte) (n int, err error) {                                                                 
    n, err = t.reader.Read(p)                                                                                                 
    if err != nil {                                                                                                           
        t.er = err                                                                                                                                           
        return                           
    }                                                                                                                                                            
    t.windowHandler(n)                                                                                                                                           
    return                              
}                                                                                                                             

// recvBufferReader implements io.Reader interface to read the data from
// recvBuffer.
type recvBufferReader struct {
    closeStream func(error) // Closes the client transport stream with the given error and nil trailer metadata.
    ctx         context.Context                                                                                               
    ctxDone     <-chan struct{} // cache of ctx.Done() (for performance).
    recv        *recvBuffer                                                                                                   
    last        *bytes.Buffer // Stores the remaining data in the previous calls.                                                                                   
    err         error                                                                                                         
    freeBuffer  func(*bytes.Buffer)                                                                                           
}                                                                                                                             
                                                                                                                              
// Read reads the next len(p) bytes from last. If last is drained, it tries to                                                
// read additional data from recv. It blocks if there no additional data available                                                                           
// in recv. If Read returns any non-nil error, it will continue to return that error.
func (r *recvBufferReader) Read(p []byte) (n int, err error) {                                                                                                   
    if r.err != nil {                                                                                                                                            
        return 0, r.err                 
    }                                                                                                                         
    if r.last != nil {                                                                                                        
        // Read remaining data left in last call.                                                                                             
        copied, _ := r.last.Read(p)    
        if r.last.Len() == 0 {                                             
            r.freeBuffer(r.last)         
            r.last = nil
        }
        return copied, nil
    }
    if r.closeStream != nil {
        n, r.err = r.readClient(p)
    } else {
        n, r.err = r.read(p)
    }
    return n, r.err
}

func (r *recvBufferReader) read(p []byte) (n int, err error) {                                                                                    
    select {                                                                                                                                                     
    case <-r.ctxDone:                                                                                                                                              
        return 0, ContextErr(r.ctx.Err())                                                                                                              
    case m := <-r.recv.get():                                                                                                                             
        return r.readAdditional(m, p)                                                                                                       
    }                                                                                                                                                           
}                                                                                                                                                             

func (r *recvBufferReader) readAdditional(m recvMsg, p []byte) (n int, err error) {
    r.recv.load()
    if m.err != nil {
        return 0, m.err
    }
    copied, _ := m.buffer.Read(p)
    if m.buffer.Len() == 0 {
        r.freeBuffer(m.buffer)
        r.last = nil
    } else {
        r.last = m.buffer
    }
    return copied, nil
}
```
## Message reader
* ```r.recv``` is ```*recvBuffer```. It's ```get()``` method just return the ```<-chan recvMsg```. 
* ```r.recv``` is assigned by ```s.buf``` from the above code. 
* ```s.buf``` is assigned by ```buf```, which is the return value of ```newRecvBuffer()```
* ```newRecvBuffer()``` simple create and return a ```*recvBuffer```, the ```recvBuffer.c``` field is a buffered channel:```chan recvMsg``` 

In Conclusion: ```s.handleStream()``` try to read the reqeust parameter from a channel ```chan recvMsg```. Which is a buffered channel belong to ```Stream.buf.c``` 

Then the next question is: who send the request parameter to that channel?

For convenience, All realated code is showned in one place.

```go
// recvMsg represents the received msg from the transport. All transport
// protocol specific info has been removed.
type recvMsg struct {                                                                                           
    buffer *bytes.Buffer          
    // nil: received some data                                           
    // io.EOF: stream is completed. data is nil.                    
    // other non-nil error: transport failure. data is nil.                      
    err error                                                        
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

// get returns the channel that receives a recvMsg in the buffer.                                                                                       
//                                                                                                                                                
// Upon receipt of a recvMsg, the caller should call load to send another                                                                     
// recvMsg onto the channel if there is any.                                                                                                                       
func (b *recvBuffer) get() <-chan recvMsg {                                                                                                            
    return b.c                               
}                                                                                                                 

func newRecvBuffer() *recvBuffer {                                                                                                                     
    b := &recvBuffer{                                                                                                                    
        c: make(chan recvMsg, 1),                                                                                                                
    }                                                                                                                                         
    return b                                                                                                                                                    
}                                                                                                                              

// operateHeader takes action on the decoded headers.                                                                          
func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
    ...
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
    ...
}
```
## Message sender

Let's back to the start point. In ```HandleStreams()```, ```t.framer.fr.ReadFrame()``` has already been checked. There is no sign of sending message to channel. The next reasonable method is ```t.handleData(frame)```.

```go
func (t *http2Server) handleData(f *http2.DataFrame) {
    size := f.Header().Length
    var sendBDPPing bool
    if t.bdpEst != nil {
        sendBDPPing = t.bdpEst.add(size)
    }
    // Decouple connection's flow control from application's read.
    // An update on connection's flow control should not depend on
    // whether user application has read the data or not. Such a
    // restriction is already imposed on the stream's flow control,
    // and therefore the sender will be blocked anyways.
    // Decoupling the connection flow control will prevent other
    // active(fast) streams from starving in presence of slow or
    // inactive streams.
    if w := t.fc.onData(size); w > 0 {
        t.controlBuf.put(&outgoingWindowUpdate{
            streamID:  0,
            increment: w,
        })
    }
    if sendBDPPing {
        // Avoid excessive ping detection (e.g. in an L7 proxy)
        // by sending a window update prior to the BDP ping.
        if w := t.fc.reset(); w > 0 {
            t.controlBuf.put(&outgoingWindowUpdate{
                streamID:  0,
                increment: w,
            })
        }
        t.controlBuf.put(bdpPing)
    }
    // Select the right stream to dispatch.
    s, ok := t.getStream(f)
    if !ok {  
        return
    }
    if s.getState() == streamReadDone {
        t.closeStream(s, true, http2.ErrCodeStreamClosed, false)
        return
    }
    if size > 0 {
        if err := s.fc.onData(size); err != nil {
            t.closeStream(s, true, http2.ErrCodeFlowControl, false)
            return
        }
        if f.Header().Flags.Has(http2.FlagDataPadded) {
            if w := s.fc.onRead(size - uint32(len(f.Data()))); w > 0 {
                t.controlBuf.put(&outgoingWindowUpdate{s.id, w})
            }
        }
        // TODO(bradfitz, zhaoq): A copy is required here because there is no
        // guarantee f.Data() is consumed before the arrival of next frame.
        // Can this copy be eliminated?
        if len(f.Data()) > 0 {
            buffer := t.bufferPool.get()
            buffer.Reset()
            buffer.Write(f.Data())
            s.write(recvMsg{buffer: buffer})
        }
    }
    if f.Header().Flags.Has(http2.FlagDataEndStream) {
        // Received the end of stream from the client.
        s.compareAndSwapState(streamActive, streamReadDone)
        s.write(recvMsg{err: io.EOF})
    }
}
```
```handleData()``` perform the following work:
* connection flow control
* forward the data frame to the selected ```Stream```

The fowarding work started by:
* copy the payload to ```buffer```,
* build a ```recvMsg``` with the payload ```buffer```,
* call ```s.write()```, which will call ```s.buf.put()```,
* in ```*recvBuffer.put()```, the payload ```recvMsg``` will be sent to the ```recvBuffer.c```.

In conclusion: ```st.HandleStreams()``` will send the request parameter to the same channel ```chan recvMsg```. Which is a buffered channel belong to ```Stream.buf.c``` 
```go
func (t *http2Server) handleData(f *http2.DataFrame) {
    size := f.Header().Length
    var sendBDPPing bool
    if t.bdpEst != nil {
        sendBDPPing = t.bdpEst.add(size)
    }
    // Decouple connection's flow control from application's read.
    // An update on connection's flow control should not depend on
    // whether user application has read the data or not. Such a
    // restriction is already imposed on the stream's flow control,
    // and therefore the sender will be blocked anyways.
    // Decoupling the connection flow control will prevent other
    // active(fast) streams from starving in presence of slow or
    // inactive streams.
    if w := t.fc.onData(size); w > 0 {
        t.controlBuf.put(&outgoingWindowUpdate{
            streamID:  0,
            increment: w,
        })
    }
    if sendBDPPing {
        // Avoid excessive ping detection (e.g. in an L7 proxy)
        // by sending a window update prior to the BDP ping.
        if w := t.fc.reset(); w > 0 {
            t.controlBuf.put(&outgoingWindowUpdate{
                streamID:  0,
                increment: w,                                                                                                                      
            })                                                                                                                                                     
        }                                                                                                                                                   
        t.controlBuf.put(bdpPing)                                                                                                                                   
    }                                                                                                                                                             
    // Select the right stream to dispatch.                                                                                                              
    s, ok := t.getStream(f)                                                                                                                        
    if !ok {                                                                                                                                           
        return                                                                                                                                         
    }
    if s.getState() == streamReadDone {
        t.closeStream(s, true, http2.ErrCodeStreamClosed, false)
        return
    }
    if size > 0 {
        if err := s.fc.onData(size); err != nil {
            t.closeStream(s, true, http2.ErrCodeFlowControl, false)
            return
        }
        if f.Header().Flags.Has(http2.FlagDataPadded) {
            if w := s.fc.onRead(size - uint32(len(f.Data()))); w > 0 {
                t.controlBuf.put(&outgoingWindowUpdate{s.id, w})
            }
        }
        // TODO(bradfitz, zhaoq): A copy is required here because there is no
        // guarantee f.Data() is consumed before the arrival of next frame.
        // Can this copy be eliminated?
        if len(f.Data()) > 0 {
            buffer := t.bufferPool.get()
            buffer.Reset()
            buffer.Write(f.Data())
            s.write(recvMsg{buffer: buffer})
        }
    }
    if f.Header().Flags.Has(http2.FlagDataEndStream) {
        // Received the end of stream from the client.
        s.compareAndSwapState(streamActive, streamReadDone)
        s.write(recvMsg{err: io.EOF})
    }
}

func (s *Stream) write(m recvMsg) {                                          
    s.buf.put(m)                                                           
}                                      

func (b *recvBuffer) put(r recvMsg) {                                        
    b.mu.Lock()                                                            
    if b.err != nil {                  
        b.mu.Unlock()         
        // An error had occurred earlier, don't accept more
        // data or errors.                          
        return                                           
    }                                                     
    b.err = r.err   
    if len(b.backlog) == 0 {
        select {                                      
        case b.c <- r:                                
            b.mu.Unlock()                                  
            return                   
        default:                                                                     
        }                                                                       
    }                                                                     
    b.backlog = append(b.backlog, r)                            
    b.mu.Unlock()                                                                                           
}                                   
```
