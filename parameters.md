# Request parameters

At [Serve stream](response.md#serve-stream), there is a problem we didn't tell the detail. In one word, How does the server read the request parameter?

According to the [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md). The request is composed by the following parts.
```
Request → Request-Headers *Length-Prefixed-Message EOS. 
```
Let's describe the probelm in detail. In [Serve stream](response.md#serve-stream), 
* ```s.handleStream()``` and ```st.HandleStreams``` are called to handle the stream and run in its goroutine.
* Meanwhile, ```s.handleStream()``` is called to handle the gRPC method call and run in its goroutine.
* ```st.HandleStreams``` will read the frame from the wire continuesly
* ```s.handleStream()``` also need to read the request parameter.

Now you has the same problem as I had: How the two goroutine communicate with each other?

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
The following diagram is the serve request sequence.
![images.004.png](images/images.004.png)

After check the serve request sequence, The ```recvAndDecompress()``` show his face. Before the invocation of ```recvAndDecompress()``` there is no sign of read the reqeust data frame. After ```recvAndDecompress()``` gRPC is ready to call ```md.Handler().
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
In ```recvAndDecompress()```, ```p.recvMsg()``` read the messsage. Follow it.

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
In ```recvMsg()```, It looks like ```p.r.Read()``` read the data frame. From  [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) we know this:

```
The repeated sequence of Length-Prefixed-Message items is delivered in DATA frames

* Length-Prefixed-Message → Compressed-Flag Message-Length Message
* Compressed-Flag → 0 / 1 # encoded as 1 byte unsigned integer
* Message-Length → {length of Message} # encoded as 4 byte unsigned integer (big endian)
* Message → *{binary octet}
```
It's clear that ```p.r.Read()``` is used to read the data frame. From the parser's definition, it's just a normal struct with a recvMsg method. While nothing is special, except the ```r io.Reader``` field.

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
