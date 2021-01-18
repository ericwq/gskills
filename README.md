# gSkills

My original goal is to understand the source code of [gRPC-go](https://github.com/grpc/grpc-go). When researching the source code, I found a lot of coding skills/design which deserved to be noted for later reference. 

Gradually, as more chapter finished, it becomes "Understanding gRPC-go": the code analysis book. All content are golang and gRPC-go related. 

Get a quick glimpse from the following diagram. See  [Balancer and Resolver API](docs/dial.md#balancer-and-resolver-api) for detail.
![images.008](images/images.008.png)

## Status
- currently reading the Envoy document, preparing for the xDS chapter.
- `Send Request` needs review because I learn more from `Client Dial` and `Load Balancing` chapters
- `Send Response` also need review to update the diagram

## Content 

### [Client Dial](docs/dial.md)
- [Balancer and Resolver API](docs/dial.md#balancer-and-resolver-api)
- [Dial process part I](docs/dial.md#dial-process-part-i)
  - [newCCResolverWrapper()](docs/dial.md#newccresolverwrapper)
  - [ccResolverWrapper.UpdateState()](docs/dial.md#ccresolverwrapperupdatestate)
  - [ClientConn.updateResolverState()](docs/dial.md#clientconnupdateresolverstate)
  - [newCCBalancerWrapper()](docs/dial.md#newccbalancerwrapper)
  - [ccBalancerWrapper.updateClientConnState()](docs/dial.md#ccbalancerwrapperupdateclientconnstate)
- [Dial process part II](docs/dial.md#dial-process-part-ii)
  - [addrConn.connect()](docs/dial.md#addrconnconnect)
  - [addrConn.resetTransport()](docs/dial.md#addrconnresettransport)

### [Send Request](docs/request.md)
- [Application code](docs/request.md#application-code)
- [Client stub](docs/request.md#client-stub)
  - [Fork road](docs/request.md#fork-road)
  - [Send Request-Headers](docs/request.md#send-request-headers)
  - [Send Length-Prefixed-Message and EOS](docs/request.md#send-length-prefixed-message-and-eos)
  
### [Send Response](docs/response.md)
- [Application code](docs/response.md#application-code)
- [Register service](docs/response.md#register-service)
- [Serve request](docs/response.md#serve-request)                                
  - [Prepare for stream](docs/response.md#prepare-for-stream)
  - [Serve stream](docs/response.md#serve-stream) 
  - [Handle request](docs/response.md#handle-request)

### [Request parameters](docs/parameters.md)
- [The problem](docs/parameters.md#the-problem)                                
- [The clue](docs/parameters.md#the-clue)                                
- [Lock the method](docs/parameters.md#lock-the-method)
- [Message reader](docs/parameters.md#message-reader)                                
- [Message sender](docs/parameters.md#message-sender)

### [controlBuffer, loopyWriter and framer](docs/control.md)
- [The bigger picture](docs/control.md#the-bigger-picture)                                
- [controlBuffer](docs/control.md#controlbuffer)                                
  - [Component](docs/control.md#component)
  - [Get and Put](docs/control.md#get-and-put)
  - [Threshold](docs/control.md#threshold)
- [loopyWriter](docs/control.md#loopywriter)                                
  - [Component](docs/control.md#component-1)
  - [handle](docs/control.md#handle)
  - [processData](docs/control.md#processdata)
  - [run](docs/control.md#run)
- [framer](docs/control.md#framer)                                

### [Interceptor](docs/interceptor.md)
- [Interceptor chain execution flow](docs/interceptor.md#interceptor-chain-execution-flow)   
- [Launch Interceptor](docs/interceptor.md#launch-interceptor)  
- [Use Interceptor](docs/interceptor.md#use-interceptor)

### [Authentication](docs/auth.md)
- [Client side](docs/auth.md#client-side)
- [Server side](docs/auth.md#server-side)
- [DialOption and ServerOption](docs/auth.md#dialoption-and-serveroption)
- [Server internal](docs/auth.md#server-internal)
- [Client internal](docs/auth.md#client-internal)
- [Server internal 2](docs/auth.md#server-internal-2)

### [Load Balancing - Client](docs/load.md)
- [Name resolving](docs/load.md#name-resolving)
  - [exampleResolver](docs/load.md#exampleresolver)
  - [dnsResolver](docs/load.md#dnsresolver)
- [Load balancing](docs/load.md#load-balancing)
  - [defaultServiceConfigRawJSON](docs/load.md#defaultserviceconfigrawjson)
  - [service config and dnsResovler](docs/load.md#service-config-and-dnsresovler)
  - [defaultServiceConfig](docs/load.md#defaultserviceconfig)
  - [UpdateClientConnState()](docs/load.md#updateclientconnstate)
  - [UpdateSubConnState()](docs/load.md#updatesubconnstate)
  - [newAttemptLocked()](docs/load.md#newattemptlocked)

### [xDS protocol support](docs/xds.md)

## Reading material
The following books help me to build a solid foundation for [HTTP2](https://tools.ietf.org/html/rfc7540) and [gRPC](https://www.grpc.io/docs/). They answer a lot of questions about Why, What and some part of How, while `gSkills` purely focus on How to implement them in gRPC.

- HTTP2
  - [Learning HTTP/2](https://www.oreilly.com/library/view/learning-http2/9781491962435/)
  - [HTTP/2 in Action](https://www.manning.com/books/http2-in-action?query=http2)
- gRPC
  - [gRPC: Up and Running](https://www.oreilly.com/library/view/grpc-up-and/9781492058328/)
  - [gRPC doc](https://github.com/grpc/grpc/tree/master/doc)

The following articles are very helpful to understand [envoy](http://envoyproxy.io). The official [Introduction](https://www.envoyproxy.io/docs/envoy/v1.16.0/intro/intro) is not easy to understand, full of terminology, too much details to easily get distracted.
- Envoy introduction
  - [Introduction to modern network load balancing and proxying](https://blog.envoyproxy.io/introduction-to-modern-network-load-balancing-and-proxying-a57f6ff80236)
  - [Announcing Envoy: C++ L7 proxy and communication bus](https://eng.lyft.com/announcing-envoy-c-l7-proxy-and-communication-bus-92520b6c8191)
  - [Anatomy of envoy proxy: the architecture of envoy and how it works](https://songrgg.github.io/architecture/deeper-understanding-to-envoy/)
- Envoy data plane and control plane
  - [The universal data plane API](https://blog.envoyproxy.io/the-universal-data-plane-api-d15cec7a)
  - [Service mesh data plane vs. control plane](https://blog.envoyproxy.io/service-mesh-data-plane-vs-control-plane-2774e720f7fc)
- Envoy internal
  - [Envoy hot restart](https://blog.envoyproxy.io/envoy-hot-restart-1d16b14555b5)
  - [Envoy threading model](https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310)
- Envoy proto3 <> JSON <> YAML
  - [Dynamic extensibility and Protocol Buffers](https://blog.envoyproxy.io/dynamic-extensibility-and-protocol-buffers-dcd0bf0b8801)
  - [Evolving a Protocol Buffer canonical API](https://blog.envoyproxy.io/evolving-a-protocol-buffer-canonical-api-e1b2c2ca0dec)
- Envoy and gRPC-Web
  - [Envoy and gRPC-Web: a fresh new alternative to REST](https://blog.envoyproxy.io/envoy-and-grpc-web-a-fresh-new-alternative-to-rest-6504ce7eb880)

## License
Please read the [LICENSE](LICENSE) for more details.
