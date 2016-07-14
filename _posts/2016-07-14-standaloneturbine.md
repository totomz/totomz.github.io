---
layout:     post
title:      Standalone Netflix/Turbine server
date:       2016-07-14
summary:    How to run Netflix/Turbine using an embedded Jetty container
categories: development java
---
Currently I am working on a project based on microservices.
Everone knows about the [CircuitBreaker](http://martinfowler.com/bliki/CircuitBreaker.html) pattern, and why is so important to isolate dependencies *by isolating points of access between the services, stopping cascading failures across them, and providing fallback options, all of which improve your system’s overall resiliency.*

To achieve this goal I'm using [Hystrix](https://github.com/Netflix/Hystrix/wiki), from the Netflix OSS stack (a great stack of tools, by the way). Hystrix comes with a nice and usefull [dashboard](https://github.com/Netflix/Hystrix/wiki/Dashboard). To show the status of multiple services, a "cluster", in a single dashboard, we have to use [Turbine](https://github.com/Netflix/Turbine/wiki/Configuration-(1.x)) to collect and aggregate all the metric streams.

This project has a distributed architecture, with a lot of services spread around several vm, so I had to implement a custom [host discovery plugin](https://github.com/Netflix/Turbine/wiki/Plugging-in-your-own-InstanceDiscovery-(1.x)), based on *Chabotto* (more later).

TL;DL I decided to embed a [jetty servlet container](http://www.eclipse.org/jetty/) in a standalone JavaSE application. Easy job. But when I start the application and I tried to see the live stream connecting to the Turbine servelt, I recived an empty response with this stack in the log:

```java
qtp933699219-25] WARN org.eclipse.jetty.server.HttpChannel - /
java.lang.NoSuchMethodError: javax.servlet.http.HttpServletResponse.getStatus()I
	at org.eclipse.jetty.server.handler.ErrorHandler.handle(ErrorHandler.java:148)
	at org.eclipse.jetty.server.Response.sendError(Response.java:579)
	at org.eclipse.jetty.server.Response.sendError(Response.java:517)
	at org.eclipse.jetty.servlet.ServletHandler$Default404Servlet.doGet(ServletHandler.java:1690)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:707)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:820)
	at org.eclipse.jetty.servlet.ServletHolder.handle(ServletHolder.java:838)
	at org.eclipse.jetty.servlet.ServletHandler.doHandle(ServletHandler.java:552)
	at org.eclipse.jetty.server.handler.ContextHandler.doHandle(ContextHandler.java:1213)
	at org.eclipse.jetty.servlet.ServletHandler.doScope(ServletHandler.java:487)
	at org.eclipse.jetty.server.handler.ContextHandler.doScope(ContextHandler.java:1126)
	at org.eclipse.jetty.server.handler.ScopedHandler.handle(ScopedHandler.java:141)
	at org.eclipse.jetty.server.handler.HandlerWrapper.handle(HandlerWrapper.java:132)
	at org.eclipse.jetty.server.Server.handle(Server.java:550)
	at org.eclipse.jetty.server.HttpChannel.handle(HttpChannel.java:321)
	at org.eclipse.jetty.server.HttpConnection.onFillable(HttpConnection.java:254)
	at org.eclipse.jetty.io.AbstractConnection$ReadCallback.succeeded(AbstractConnection.java:269)
	at org.eclipse.jetty.io.FillInterest.fillable(FillInterest.java:97)
	at org.eclipse.jetty.io.ChannelEndPoint$2.run(ChannelEndPoint.java:124)
	at org.eclipse.jetty.util.thread.QueuedThreadPool.runJob(QueuedThreadPool.java:672)
	at org.eclipse.jetty.util.thread.QueuedThreadPool$2.run(QueuedThreadPool.java:590)
	at java.lang.Thread.run(Thread.java:745)
```
Mh.....the first line of the stacktrace is  `handleErrorPage(request, writer, response.getStatus(), reason);` ; response is an `HttpServletResponse`, and the method `getStatus() ` is there **since Servlet 3.0**. Turbine is quite ~~old~~ stable, maybe is "not compatible" with Servlet 3.0? Lets check its dependencies (of the jar version):
![dsa](/images/turbine/turbine-dependency.png))

Here it is! A dependency conflict with my embedded jetty (I'm building a fat-jar....). So, just remove `servlet-api:2.5` and we are fine

```xml
<dependency>
  <groupId>com.netflix.turbine</groupId>
	<artifactId>turbine-core</artifactId>
	<version>1.0.0</version>
	<exclusions>
		<exclusion>
			<groupId>javax.servlet</groupId>
			<artifactId>servlet-api</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```

Done! Now Turbine aggregates from all the services registered in Chabotto, *my* simple service regstry.
