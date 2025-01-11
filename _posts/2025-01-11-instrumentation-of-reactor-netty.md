---
layout: post
title: Instrumentation of Reactor Netty
order: 1
---
Instrumentation of Reactor Netty, used by Webflux, to get an object that is similar to Jakarta's `HttpServletRequest`. I thought this might be a nice example to walk people through how I usually figure out my instrumentation.
Be sure to checkout my [ByteBuddy tips](https://bitterpanda63.github.io/bytebuddy-debugging/) which I'll be using here.


## Exploration phase
So we want to intercept requests going to **Spring Webflux** before they reach a filter (*AKA Middleware*). 

Spring Webflux primarily uses the Reactor Project which uses [Netty](https://netty.io/) as it's server. Scouring through the Netty docs for the current stable version (*4.1* at time of writing) we quickly find that most servers extend [`ChannelInboundHandlerAdapter`](https://netty.io/4.1/api/io/netty/channel/ChannelInboundHandlerAdapter.html). Due to it's async nature and piping it looks fairly complex to instrument this, we'll now take a look at Reactor.

[Project Reactor](https://projectreactor.io/) has a couple of parts, but the one we care about is **Reactor Netty**. We can easily verify that this is used by webflux using a [quick github search](https://github.com/search?q=repo%3Aspring-projects%2Fspring-framework+reactor.netty&type=code&p=3). Looking through the docs it becomes clear we would want to instrument [`HttpServer`](https://projectreactor.io/docs/netty/release/api/reactor/netty/http/server/HttpServer.html)
## Running phase
Since I have a sample app with Webflux ready I'll now use that to look at all transformations and function calls for the HttpServer. I start this process by creating a Wrapper class in our project that looks like this : 
```java
public class NettyWrapper implements Wrapper {  
    public String getName() {  
        return MyGenericAdvice.class.getName();  
    }  
    public ElementMatcher<? super MethodDescription> getMatcher() {  
        return ElementMatchers.isDeclaredBy(getTypeMatcher());  
    }  
    @Override  
    public ElementMatcher<? super TypeDescription> getTypeMatcher() {  
        return ElementMatchers.nameContainsIgnoreCase("reactor.netty.http.server.HttpServer");  
    }  
      
    public class MyGenericAdvice {  
        @Advice.OnMethodEnter  
        public static void before(  
                @Advice.This(typing = DYNAMIC, optional = true) Object target,  
                @Advice.Origin Executable method,  
                @Advice.AllArguments(readOnly = false, typing = DYNAMIC) Object[] args  
        ) {  
            System.out.println("[C] >> " + method);  
        }  
  
        @Advice.OnMethodExit  
        public static void after(  
                @Advice.This(typing = DYNAMIC, optional = true) Object target,  
                @Advice.Origin Executable method,  
                @Advice.AllArguments(readOnly = false, typing = DYNAMIC) Object[] args,  
                @Advice.Return(readOnly = false, typing = DYNAMIC) Object returnValue  
        ) {  
            System.out.println("[C] << " + method);  
        }  
    }  
}
```

### Fixing an error
Now when we look at our logs first we got an error :
```
[Byte Buddy] ERROR reactor.netty.http.server.HttpServer [org.springframework.boot.loader.launch.LaunchedClassLoader@2c58dcb1, unnamed module @4b3a01d8, Thread[#1,main,5,main], loaded=false]
java.lang.IllegalStateException: An illegal stack manipulation must not be applied
```
A first great debugging step is disabling the advice class :
```java
public ElementMatcher<? super MethodDescription> getMatcher() {  
    return ElementMatchers.none();  
    //return ElementMatchers.isDeclaredBy(getTypeMatcher());  
}
```
Great that fixed our issue, now we know that something is going wrong for a certain function of the class we are wrapping, we can also see that by using `nameEndsWith` instead of a `contains` in our type matcher we only get one transformation : 
```
[Byte Buddy] TRANSFORM reactor.netty.http.server.HttpServer [org.springframework.boot.loader.launch.LaunchedClassLoader@2c58dcb1, unnamed module @4b3a01d8, Thread[#1,main,5,main], loaded=false]
```
We'll make the choice of wrapping the `handle` function, and let's look if it's called. And we get the call, very cool!

### handle() function, how to get our request?
Okay we have successfully verified the handle() function is wrappable, now we want to figure out how to extract our `HttpServerRequest` Looking at docs :
 > public final [HttpServer](https://projectreactor.io/docs/netty/release/api/reactor/netty/http/server/HttpServer.html "class in reactor.netty.http.server") handle([BiFunction](https://docs.oracle.com/javase/8/docs/api/java/util/function/BiFunction.html?is-external=true "class or interface in java.util.function")\<? super [HttpServerRequest](https://projectreactor.io/docs/netty/release/api/reactor/netty/http/server/HttpServerRequest.html "interface in reactor.netty.http.server"),? super [HttpServerResponse](https://projectreactor.io/docs/netty/release/api/reactor/netty/http/server/HttpServerResponse.html "interface in reactor.netty.http.server"),? extends [Publisher](https://www.reactive-streams.org/reactive-streams-1.0.3-javadoc/org/reactivestreams/Publisher.html?is-external=true "class or interface in org.reactivestreams")<[Void](https://docs.oracle.com/javase/8/docs/api/java/lang/Void.html?is-external=true "class or interface in java.lang")>> handler)
 
 Since I'm not a Java developer I'll first have to figure out what a **BiFunction** is. Looking into the docs we can see this is a function that takes the HttpServerRequest as an argument, in essence we would want to wrap this function and insert our own functionality. Since this is a bit complex we will first also discuss the opportunity of wrapping HttpServerRequest.

### Wrapping HttpServerRequest, an interface.
Wrapping an interface is actually very possible, you can check if a given class is an implemenation of that interface using the following type matchers :
```java
hasSuperType(nameContains("reactor.netty.http.server.HttpServerRequest").and(isInterface()));
```
Using this I came across an internal class called [`HttpServerOperations`](https://github.com/reactor/reactor-netty/blob/7dcfe8585bba9903be570adc02ebbfd202c1c4b9/reactor-netty-http/src/main/java/reactor/netty/http/server/HttpServerOperations.java#L178), which has a constructor method who gets the HttpRequest from Netty. however this interface is very primitive, it provides support for method, uri & headers. The best course of action is to parse data on exit of this function, and then read from HttpServerOperations object.

### Ok, but how do we do extraction?
Luckily Byte Buddy has got us covered, it merges the different ClassLoaders so that we can just use the interface. We do need to add a `compileOnly` entry so the compiler knows what we are referencing : 
```java
compileOnly 'io.projectreactor.netty:reactor-netty-core:1.2.1' // For Spring Webflux
```
After we do that it's smooth sailing, we can use `@This` in the exit advice to get the object after the constructor has finished it's job. So that results in : 
```java
@Advice.OnMethodExit(suppress = Throwable.class)
public static void after(
	@Advice.This(typing = DYNAMIC, optional = true) HttpServerRequest target
) {
	// ...
}
```

Happy coding! 💜