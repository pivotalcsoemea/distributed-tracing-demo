# Proof of Concept to demonstrate Distributed Tracing 
This project has been created to demonstrate the following goals using <a href="https://github.com/spring-cloud/spring-cloud-sleuth">Spring Cloud Sleuth</a>: 
 <ul>
 <li>Show we can trace the execution of a request as it traverses multiple applications</li>
 <li>Show various mechanisms used by applications to collaborate between each other: synchronous Restful calls, asynchronous Restful calls, messaging</li>
 <li>Show how we can capture custom request details</li> 
 </ul>   

This project consists of 3 standalone applications: A <b>gateway</b> application which acts as a facade and two internal applications, <b>marketgw</b> and <b>portfoliomgr</b>. The gateway exposes a Restful endpoint "/open" which delegates to another two Restful endpoints exposed by the two internal apps, "/openTrade" and "/openPosition" respectively.

     
<h3>Goal: Demonstrate how we can trace the execution of a request as it traverses several applications using synchronous invocations to external Restful services</h3>
Gateway application receives the request below. It calls synchronously the marketgw Restful application. When it gets the result, it uses to call another Restful application, the portfoliomgr. When the gateway gets the result from the portfoliomgr, it sends it back as a response.
 
Request:<p>
<code> 
   curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X POST -d '{"account":"bob","amount":100}' http://localhost:8080/open
</code>
<p>
Response:<p>
<pre>
<code> 
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
X-Trace-Id: 945a8afc-6cc5-4c3a-abfb-cd77c2c4b079
X-Span-Id: 945a8afc-6cc5-4c3a-abfb-cd77c2c4b079
X-Application-Context: bootstrap
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Tue, 19 Jan 2016 19:24:13 GMT

{"id":"916d50c0-01b3-436a-93aa-7c76a5692d7d","account":"bob","rate":0.5494186987570098,"amount":100.0}
</code>
</pre>

<p>For each incoming request, Sleuth creates a unique identifier and uses it to track the request as it traverses multiple applications. This identifier is called Trace-Id. 

Distributed tracing captured in the standard output in the gateway app:<p>
<pre><code>
[span]{"begin":1453233385739,"end":1453233386017,
<b>"name"</b>:"http/open",
<b>"traceId"</b>:"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","parents":[],
"spanId":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","remote":false,"exportable":false,
"annotations":{"account":"bob"},"processId":null,
"timelineAnnotations":[{"time":1453233385751,"msg":"StartCalculation"},{"time":1453233386005,"msg":"EndCalculation"}],"running":false,"accumulatedMillis":278}
[endspan]
</code></pre>
 
Distributed tracing captured in the standard output in the market-gw app:<p>
<pre><code>
[span]{"begin":1453233385745,"end":1453233385750,
"name":"http/openTrade",
"traceId":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c",
"parents":["8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c"],
"spanId":"2731612c-9b22-4320-8292-a7cbf82f3e1e","remote":false,"exportable":true,
"annotations":{"/http/request/uri":"http://localhost:8081/openTrade","/http/request/endpoint":"/openTrade","/http/request/method":"POST","/http/request/headers/accept":"application/json, application/*+json","/http/request/headers/content-type":"application/json;charset=UTF-8","/http/request/headers/x-trace-id":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","/http/request/headers/x-span-id":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","/http/request/headers/x-span-name":"http/open","/http/request/headers/user-agent":"Java/1.8.0_45","/http/request/headers/host":"localhost:8081","/http/request/headers/connection":"keep-alive","/http/request/headers/content-length":"32","mkt":"6915959203315918557","/http/response/status_code":"200","/http/response/headers/x-trace-id":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","/http/response/headers/x-span-id":"2731612c-9b22-4320-8292-a7cbf82f3e1e","/http/response/headers/x-application-context":"bootstrap:8081","/http/response/headers/content-type":"application/json;charset=UTF-8","/http/response/headers/transfer-encoding":"chunked","/http/response/headers/date":"Tue, 19 Jan 2016 19:56:25 GMT"},"processId":null,
"timelineAnnotations":[],"running":false,"accumulatedMillis":5}
[endspan]
</code></pre>
 

Sleuth is responsible of carrying the trace-Id across remote Restful calls. 

<h3>Goal: Demonstrate how we can trace the execution of a request as it traverses several applications using asynchronous invocations to external Restful services</h3>
The Use case is practically same as the synchronous one with the exception that this time the 2 internal Restful requests are done asynchronously. What has this anything to do with Sleuth? The thread that created the Trace-id is gone! this was the servlet's thread that handled the initial request. The responses from the internal restful request are now handled by a different thread. Sleuth has to be able to propagate the trace-id from the http response to the thread handling the response.

Work in progress

<h3>Use Case - Messaging collaboration</h3>
This time the Gateway application sends a message/request to one of the internal applications. Sleuth has to be able to propagate the trace-id over to the receiver of the message/request. 
 
Pending to do

<h3>Issues to be solved</h3>
<ul>
<li>how do I configure the requests I want Sleuth to track? It seems it is tracking every incoming requests, eg. /somecrazyrequest will produce a 
span event</li>
</ul>


