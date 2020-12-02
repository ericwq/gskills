# gSkills

My original goal is to understand the source code of [gRPC-go](https://github.com/grpc/grpc-go). When researching the source code, I found a lots of coding skills/design which deserved to be noted for later reference. 

Gradually, As more chapter finished, it tbecomes "Understanding gRPC-go".  All content are golang and gRPC-go related. 

Get a quick glimpse from the following diagram. See  [The bigger picture](docs/control.md#the-bigger-picture) for detail.
![images.005](images/images.005.png)

## Content 
### [Send Request](docs/request.md)
* [Application code](docs/request.md#application-code)
* [Client stub](docs/request.md#client-stub)
  * [Fork road](docs/request.md#fork-road)
  * [Send Request-Headers](docs/request.md#send-request-headers)
  * [Send Length-Prefixed-Message and EOS](docs/request.md#send-length-prefixed-message-and-eos)
  
### [Reply with Response](docs/response.md)
* [Application code](docs/response.md#application-code)
* [Register service](docs/response.md#register-service)
* [Serve request](docs/response.md#serve-request)                                
  * [Prepare for stream](docs/response.md#prepare-for-stream)
  * [Serve stream](docs/response.md#serve-stream) 
  * [Handle request](docs/response.md#handle-request)

### [Request parameters](docs/parameters.md)
* [The problem](docs/parameters.md#the-problem)                                
* [The clue](docs/parameters.md#the-clue)                                
* [Trace it](docs/parameters.md#trace-it)                                
* [Message reader](docs/parameters.md#message-reader)                                
* [Message sender](docs/parameters.md#message-sender)

### [controlBuffer and loopy](docs/control.md)
* [The bigger picture](docs/control.md#the-bigger-picture)                                
* [controlBuffer](docs/control.md#controlbuffer)                                
* [loopyWriter](docs/control.md#loopwriter)                                
* [Framer](docs/control.md#framer)                                

### [Interceptor](docs/interceptor.md)
* [Interceptor chain execution flow](docs/interceptor.md#interceptor-chain-execution-flow)   
* [Launch Intercetor](docs/interceptor.md#launch-interceptor)  
* [Use Interceptor](docs/interceptor.md#use-interceptor)

### [Authentication](docs/auth.md)

### [Load Balancing](docs/load.md)

## Reading material
The following books help me building a solid fundation for http2 and gRPC. They answer a lot of questions about Why ,What and some part of How, while gSkills purely focus on How to implement them in gRPC.

* [HTTP/2 in Action](https://www.manning.com/books/http2-in-action?query=http2)
* [gRPC: Up and Running](https://www.oreilly.com/library/view/grpc-up-and/9781492058328/)
* [Learning HTTP/2](https://www.oreilly.com/library/view/learning-http2/9781491962435/)
* [gPRC doc](https://github.com/grpc/grpc/tree/master/doc)

## License
Please read the [LICENSE](LICENSE) for more details.
