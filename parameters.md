# Request parameters

while ```DataFrame``` is a little bit complex. let's make it clear:
* ```HandleStreams()``` run in its goroutine.
* ```HandleStreams()``` does read the data frame which contains the method call parameter. 
* ```handle func(*Stream)``` is a wrapper for ```s.handleStream()```
* ```s.handleStream()``` run in its goroutine.
* ```s.handleStream()``` will block on a channel to wait for the method call parameter.
* in ```HandleStreams()``` goroutine, After ```ReadFrame()``` read the data frame, ```t.handleData()```sends the data frame payload to ```s.handleStream()``` goroutine   through the blocked go channel. 
* now ```s.handleStream()``` get the method call parameter, continue the process.

How the two goroutine communicate with each other deserves another chapter. 

