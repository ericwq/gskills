# gSkills

My original goal is to understand the source code of [gRPC-go](https://github.com/grpc/grpc-go). When researching the source code, I found a lot of coding skills/design which deserved to be noted for later reference. 

Gradually, as more chapter finished, it becomes "Understanding gRPC-go". All content are golang and gRPC-go related. 

Get a quick glimpse from the following diagram. See  [The bigger picture](docs/control.md#the-bigger-picture) for detail.
![images.005](images/images.005.png)

## Content 
### [Send Request](docs/request.md)
* [Application code](docs/request.md#application-code)
* [Client stub](docs/request.md#client-stub)
  * [Fork road](docs/request.md#fork-road)
  * [Send Request-Headers](docs/request.md#send-request-headers)
  * [Send Length-Prefixed-Message and EOS](docs/request.md#send-length-prefixed-message-and-eos)
  
### [Send Response](docs/response.md)
* [Application code](docs/response.md#application-code)
* [Register service](docs/response.md#register-service)
* [Serve request](docs/response.md#serve-request)                                
  * [Prepare for stream](docs/response.md#prepare-for-stream)
  * [Serve stream](docs/response.md#serve-stream) 
  * [Handle request](docs/response.md#handle-request)

### [Request parameters](docs/parameters.md)
* [The problem](docs/parameters.md#the-problem)                                
* [The clue](docs/parameters.md#the-clue)                                
* [Lock the method](docs/parameters.md#lock-the-method)
* [Message reader](docs/parameters.md#message-reader)                                
* [Message sender](docs/parameters.md#message-sender)

### [controlBuffer, loopyWriter and framer](docs/control.md)
* [The bigger picture](docs/control.md#the-bigger-picture)                                
* [controlBuffer](docs/control.md#controlbuffer)                                
  * [Component](docs/control.md#component)
  * [Get and Put](docs/control.md#get-and-put)
  * [Threshold](docs/control.md#threshold)
* [loopyWriter](docs/control.md#loopywriter)                                
  * [Component](docs/control.md#component-1)
  * [handle](docs/control.md#handle)
  * [processData](docs/control.md#processdata)
  * [run](docs/control.md#run)
* [framer](docs/control.md#framer)                                

### [Interceptor](docs/interceptor.md)
* [Interceptor chain execution flow](docs/interceptor.md#interceptor-chain-execution-flow)   
* [Launch Interceptor](docs/interceptor.md#launch-interceptor)  
* [Use Interceptor](docs/interceptor.md#use-interceptor)

### [Authentication](docs/auth.md)
* [Client side](docs/auth.md#client-side)
* [Server side](docs/auth.md#server-side)
* [DialOption and ServerOption](docs/auth.md#dialoption-and-serveroption)
* [Server internal](docs/auth.md#server-internal)
* [Client internal](docs/auth.md#client-internal)

### [Load Balancing](docs/load.md)

## Reading material
The following books help me building a solid fundation for http2 and gRPC. They answer a lot of questions about Why ,What and some part of How, while gSkills purely focus on How to implement them in gRPC.

* [HTTP/2 in Action](https://www.manning.com/books/http2-in-action?query=http2)
* [gRPC: Up and Running](https://www.oreilly.com/library/view/grpc-up-and/9781492058328/)
* [Learning HTTP/2](https://www.oreilly.com/library/view/learning-http2/9781491962435/)
* [gPRC doc](https://github.com/grpc/grpc/tree/master/doc)

## License
Please read the [LICENSE](LICENSE) for more details.
