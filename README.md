# gSkills

My original goal is to understand the source code of [gRPC-go](https://github.com/grpc/grpc-go). When researching the source code, I found a lots of coding skills which deserved to be noted for later reference. All content are golang and gRPC-go related. 

## Content 

### [Send Request](request.md)
* [Application code](request.md#application-code)
* [Client stub](request.md#client-stub)
  * [Fork road](request.md#fork-road)
  * [Send Request-Headers](request.md#send-request-headers)
  * [Send Length-Prefixed-Message and EOS](request.md#send-length-prefixed-message-and-eos)
### [Reply with Response](response.md)
* [Application code](response.md#application-code)
* [Register service](response.md#register-service)
* [Serve request](response.md#serve-request)                                
  * [Prepare for stream](response.md#prepare-for-stream)
  * [Serve stream](response.md#serve-stream) 
  * [Handle request](response.md#handle-request)
### [Request parameters](parameters.md)
* [The problem](parameters.md#the-problem)                                
* [The clue](parameters.md#the-clue)                                
* [Trace it](parameters.md#trace-it)                                
* [Message reader](parameters.md#message-reader)                                
* [Message sender](parameters.md#message-sender)
### [controlBuffer and loopy](control.md)
### [Interceptor](interceptor.md)
* [Interceptor chain execution flow](interceptor.md#interceptor-chain-execution-flow)   
* [Launch Intercetor](interceptor.md#launch-interceptor)  
* [Use Interceptor](interceptor.md#use-interceptor)
### [Authentication](auth.md)
### [Load Balancing](load.md)

## Reading material
The following books help me building a solid fundation for http2 and gRPC. They answer a lot of questions about Why ,What and some part of How, while gSkills purely focus on How to implement them in gRPC.

* [HTTP/2 in Action](https://www.manning.com/books/http2-in-action?query=http2)
* [gRPC: Up and Running](https://www.oreilly.com/library/view/grpc-up-and/9781492058328/)
* [Learning HTTP/2](https://www.oreilly.com/library/view/learning-http2/9781491962435/)
* [gPRC doc](https://github.com/grpc/grpc/tree/master/doc)

## License
Please read the [LICENSE](LICENSE) for more details.
