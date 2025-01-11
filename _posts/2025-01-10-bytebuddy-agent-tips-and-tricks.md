---
layout: post
title: Tips & Tricks for your Byte Buddy Agent
order: 1
---

Hiya! I've been working on Zen by Aikido for a couple of months now, It's a project that requires a lot of instrumentation/monkey-patching. So for our Java version we've decided to use [ByteBuddy](https://bytebuddy.net),
which is also used by (iirc) Opentelemetry and Datadog.

Now the way you select which classes should be transformed is using `ElementMatchers`, but you can quickly get lost when stuff starts to break.
So here are a couple of tips for dealing with ByteBuddy I've learned :

## Enable debug logging to see what gets transformed
This is a no-brainer when you get an issue : 
```java
agentBuilder = agentBuilder
    .with(AgentBuilder.Listener.StreamWriting.toSystemError().withTransformationsOnly())
    .with(AgentBuilder.InstallationListener.StreamWriting.toSystemError());
```
This will also print out all transformations happening, so you know exactly which classes gets transformed. This helps you narrow down your definition to exclude unnecessary transformations.

## Use a generic Advice class to catch everything when you are beginning your wrapping
If your Advice class is not generic enough, ByteBuddy might not call it and then you will not know if the issue is your ElementMatchers or your Advice class.
An Advice class that I always use when I start my wrapping journey came from [an issue](https://github.com/raphw/byte-buddy/issues/857) on the ByteBuddy github repo. Here is the Advice class :
```java
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
```

## Use suppress to ensure safety of the application
If you are planning on instrumenting somebody else's app or even your own app it's smart to use the suppress feature of Byte Buddy ([doc](https://javadoc.io/static/net.bytebuddy/byte-buddy/1.10.4/net/bytebuddy/asm/Advice.OnMethodEnter.html#suppress--)). 
```diff
- @Advice.OnMethodEnter
+ @Advice.OnMethodEnter(suppress = Throwable.class)
```
This will catch all errors you made in your snippet of advice code, even runtime errors.


That's it for now with my ByteBuddy tips, I'll link to new blog post with new tips when I remember them!

## Make use of Byte Buddy merging the user's classloaders
If you're instrumenting somebody elses application, you want to limit your dependencies as much as possible. But what if you want to make use of the result of a certain function which itself is a class that's not native to Java? Do you then import that package yourself? 
Well not quite. 

You only need to import it for compilation, so for gradle a `compileOnly` would do. That's because Byte Buddy merges the user's classloaders for your **Advice** classes. This is very handy in curbing your *dependency hell*.
