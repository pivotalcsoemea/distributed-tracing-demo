# Proof of Concept to demonstrate Distributed Tracing using Spring Cloud Sleuth
This project has been created to demonstrate the following goals using <a href="https://github.com/spring-cloud/spring-cloud-sleuth">Spring Cloud Sleuth</a>: 
 <ul>
 <li>Trace the execution of a request as it traverses multiple applications</li>
 <li>Demonstrate tracing regardless of the remoting technology used: synchronous Restful calls, asynchronous Restful calls, messaging</li>
 <li>Trace custom request details</li> 
 </ul>   
<b>Note</b>: Tracing will be collected in the standard output using a json format. There are other means to collect tracing. For instance, we can use Zipkin or send the tracing to a messaging endpoint. To know how to configure zipkin or messaging check out the <a href="https://github.com/spring-cloud/spring-cloud-sleuth">official documentation</a>. 

This project consists of 3 standalone applications: A <b>gateway</b> application which acts as a facade and two internal applications, <b>marketgw</b> and <b>portfoliomgr</b>. The gateway exposes a Restful endpoint "/open" which delegates to another two Restful endpoints exposed by the two internal apps, "/openTrade" and "/openPosition" respectively.

     
<h3>Goal: Demonstrate how we can trace the execution of a request as it traverses several applications using synchronous invocations to external Restful services</h3>
 
 <pre>
----http://localhost:8008/open---> [ Gateway app : SynchronousController class ] 
                                     -------http://localhost:8081/openTrade----> [market-gw app : MarketController class]
                                     <-----Trade---------------------------
                                     ...
                                     apply spread to Trade and produce a DealDone
                                     ... 
                                     ...  // send DealDone to portfoliomgr           										
                                     -------http://localhost:8002/openPosition----> [portfoliomgr app : PortfolioController class]
                                     <-----Position---------------------------
<-----Trade---------------------------										  
  </pre>
   
Request:<p>
<code> 
   curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X POST -d '{"account":"bob","amount":100,"symbol":"EUR/USD"}' http://localhost:8080/open
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

{"id":1250656390455939089,"account":"bob","symbol":"EUR/USD","rate":0.01058461760416928,"amount":100.0}
</code>
</pre>

<p>Let's see the tracing generated by Sleuth. For each incoming request, Sleuth creates a unique identifier and uses it to track the request as it traverses multiple applications. This identifier is called Trace-Id. Additionally, Sleuth allows us to attach request's details to the tracing. These details can be:
<ul>
<li>Tag or annotations, e.g. the account who submitted the request: "account":"bob"
<pre><code>
	span.addAnnotation("account", request.account);		
</code></pre>
We can see this annotation in the generated tracing under the attribute "annotations". See below.
<p>  
</li>
<li>Timestamps, e.g. Say we want to capture the time when we calculated the spread and before we send the position to the PositionManager.
<pre><code>
	public DealDone applySpread(TradeRequest request, Trade trade) throws InterruptedException {
	
		Span span = traceManager.getCurrentSpan();
		
		// do something
		try {
			Thread.sleep(250);
			double price = trade.rate + 0.01; // very simplistic spread. 
			return new DealDone(trade.id, trade.symbol, request.account, price, trade.amount);
		} finally {
			span.addTimelineAnnotation("AppliedSpread");
		}
		
	}
</code></pre>
We can see these timestamps in the generated tracing under the attribute "timelineAnnotations". See below.
</li>   
</ul>   

Distributed tracing captured in the standard output in the gateway app:<p>
<pre><code>
[span]{"begin":1453233385739,"end":1453233386017,
<b>"name"</b>:"http/open",
<b>"traceId"</b>:"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","parents":[],
"spanId":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","remote":false,"exportable":false,
<b>"annotations":{"account":"bob"}</b>,"processId":null,
<b>"timelineAnnotations":[{"time":1453233385751,"msg":"AppliedSpread"}]</b>,"running":false,"accumulatedMillis":278}
[endspan]
</code></pre>
 
Distributed tracing captured in the standard output in the market-gw app:<p>
<pre><code>
[span]{"begin":1453233385745,"end":1453233385750,
"name":"http/openTrade",
"traceId":<b>"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c"</b>,
<b>"parents":["8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c"],</b>
"spanId":"2731612c-9b22-4320-8292-a7cbf82f3e1e","remote":false,"exportable":true,
"annotations":{"/http/request/uri":"http://localhost:8081/openTrade","/http/request/endpoint":"/openTrade","/http/request/method":"POST","/http/request/headers/accept":"application/json, application/*+json","/http/request/headers/content-type":"application/json;charset=UTF-8","/http/request/headers/x-trace-id":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","/http/request/headers/x-span-id":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","/http/request/headers/x-span-name":"http/open","/http/request/headers/user-agent":"Java/1.8.0_45","/http/request/headers/host":"localhost:8081","/http/request/headers/connection":"keep-alive","/http/request/headers/content-length":"32","mkt":"6915959203315918557","/http/response/status_code":"200","/http/response/headers/x-trace-id":"8cde3cb5-937a-4ee7-bc70-f8dc3b14ab8c","/http/response/headers/x-span-id":"2731612c-9b22-4320-8292-a7cbf82f3e1e","/http/response/headers/x-application-context":"bootstrap:8081","/http/response/headers/content-type":"application/json;charset=UTF-8","/http/response/headers/transfer-encoding":"chunked","/http/response/headers/date":"Tue, 19 Jan 2016 19:56:25 GMT"},"processId":null,
"timelineAnnotations":[],"running":false,"accumulatedMillis":5}
[endspan]
</code></pre>
 

Sleuth is responsible of carrying the trace-Id across remote Restful calls. 

<h3>Goal: Demonstrate how we can trace the execution of a request as it traverses several applications using asynchronous invocations to external Restful services</h3>
The Use case is practically same as the synchronous one with the exception that this time the 2 internal Restful requests are done asynchronously. Furthermore, the @RequestMapping method that attends the request in the Gateway is also asynchronous, i.e. it does not return a result but a <code>ListenableFuture<Position></code>. 

What has this anything to do with Sleuth? Sleuth has to keep sending the trace-id to downstream services. And in addition to that, the thread that created the Trace-id is gone! this was the servlet's thread that handled the initial request. The responses from the internal restful request are now handled by a different thread. Sleuth has to be able to propagate the trace-id from the http response to the thread handling the response.

<b>This is a piece of functionality not <a href="https://github.com/spring-cloud/spring-cloud-sleuth/issues/124">supported by Spring yet</a> </b>
 
 <pre>
----http://localhost:8008/open---> [ Gateway app : AsyncController class ] 
                                     -------http://localhost:8081/openTrade----> [market-gw app : MarketController class]
                                     {Nio thread}<-----Trade---------------------------
                                            ...
                                            apply spread to Trade and produce a DealDone
                                            ... 
                                            ...  // send DealDone to portfoliomgr           										
                                            -------http://localhost:8002/openPosition----> [portfoliomgr app : PortfolioController class]
                                            <-----Position---------------------------
<-----Trade----------------------------------										  
  </pre>


<h3>Use Case - Messaging collaboration</h3>
This time the Gateway application sends a message/request to one of the internal applications. Sleuth has to be able to propagate the trace-id over to the receiver of the message/request. 
 
Pending to do

<h3>Issues to be solved</h3>
<ul>
<li>how do I configure Sleuth not to generate a span event for unmapped routes? It seems it is tracking every incoming requests, eg. /somecrazyrequest will produce a 
span event. There is a TraceFilter used to filter some mappings like /info and others. But this is not enough. </li>
</ul>


