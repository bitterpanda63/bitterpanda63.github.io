---
layout: post
title: Changing the output of a Java function using Byte Buddy
order: 1
---

Hiya! I'm a developer for Aikido working on [Zen by Aikido](https://aikido.dev/zen). It's a project that requires a lot of instrumentation/monkey-patching. So for our Java version we've decided to use [ByteBuddy](https://bytebuddy.net),
which is also used by (iirc) Opentelemetry and Datadog.

One of the things I sometimes have to do is to change the output of a given function. This is how you can do that with Byte Buddy :

## Skipping function execution
The first step in changing the return value is most likely skipping the execution of the function. You *can* change the return value of a function without skipping execution however we won't do that in this blogpost.

In Byte Buddy you have a handy [`skipOn`](https://javadoc.io/static/net.bytebuddy/byte-buddy/1.10.4/net/bytebuddy/asm/Advice.OnMethodEnter.html#skipOn--) attribute which you can set in your `OnMethodEnter` annotation, to quote from Byte Buddy docs : 
> A value defining what return values of the advice method indicate that the instrumented method should be skipped or void if the instrumented method should never be skipped.

**TL;DR** you set this to the class that represents the return value of your OnMethodEnter function, and if that value is "positive" (*meaning true, 1, not null, ...*) the function execution will be skipped.
```java
@Advice.OnMethodEnter(skipOn = boolean.class)
public static boolean onEnter() {
    if (wantToSkipExecution) {
        return true;
    }
    return false;
}
```

## Skipping function execution and rewriting output
Now that we know how to skip the execution, we also want to start rewriting the output, you can do this using an 
[`@Advice.Return`](https://javadoc.io/static/net.bytebuddy/byte-buddy/1.10.4/net/bytebuddy/asm/Advice.Return.html).
What this does is basically give you the return value of the function, with the ability to overwrite it, example : 

```java
@Advice.OnMethodExit
public static void onExit(
    @Advice.Return(readOnly = false) String returnValue
) {
    returnValue = "My new return string";
}
```

Et voila! As easy as pie.

## Skipping function execution and writing from onEnter
If you want to skip or not depending on something in your OnMethodEnter function but you also generate the return value in your OnMethodEnter function 
you can use a wrapper class like this : 
```java
public record SkipOnWrapper(String newReturnValue) {}
```

And making your OnMethodEnter function return a `SkipOnWrapper`, if you want to skip the function you return this object with your new return value, if you do not want to skip you return `null` :

```java
@Advice.OnMethodEnter(skipOn = SkipOnWrapper.class)
public static SkipOnWrapper onEnter() {
    if (wantToSkipExecution) {
        return new SkipOnWrapper("My new return value");
    }
    return null;
}

@Advice.OnMethodExit
public static void onExit(
    @Advice.Enter SkipOnWrapper enterResult,
    @Advice.Return(readOnly = false) String returnValue
) {
    // Overwrite return value based on code in onEnter : 
    if (enterResult != null) {
        returnValue = enterResult.newReturnValue();
    }
}
```

And that's it, thanks for reading this blogpost. See you soon with more Java madness :0