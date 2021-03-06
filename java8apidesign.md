#Java 8 API Design#

Source: blog.jooq.org/2014/05/16/java-8-friday-api-designers-be-careful/ 

##Lean Functional API Design##

With Java 8, API design has gotten a whole lot more interesting, but also a bit harder. 
As a successful API designer, it will no longer suffice to think about all sorts of 
object-oriented aspects of your API, you will now also need to consider functional 
aspects of it. In other words, instead of simply providing methods like:

```java
void performAction(Parameter parameter);
```

```java 
// Call the above:
object.performAction(new Parameter(...));
. . .

```

you should now think about whether your method arguments are better modelled as 
functions for lazy evaluation:

```java
// Keep the existing method for convenience
// and for backwards compatibility
void performAction(Parameter parameter);
 
// Overload the existing method with the new
// functional one:
void performAction(Supplier<Parameter> parameter);
 
// Call the above:
object.performAction(() -> new Parameter(...));
```

This is great. Your API can be Java-8 ready even before you’re actually targeting 
Java 8. But if you’re going this way, there are a couple of things to consider.

##JDK dependency##

The above example makes use of the JDK 8 Supplier type. 
This type is not available before the JDK 8, so if you’re using it, you’re 
going to limit your APIs use to the JDK 8. If you want to continue supporting 
older Java versions, you’ll have to roll your own supplier, or maybe use 
Callable, which has been available since Java 5:

```java
// Overload the existing method with the new
// functional one:
void performAction(Callable<Parameter> parameter);
 
// Call the above:
object.performAction(() -> new Parameter(...));
```

One advantage of using Callable is the fact that your lambda expressions 
(or “classic” Callable implementations, or nested / inner classes) are 
allowed to throw checked exceptions. Following is another 
possibility to circumvent this limitation.

##Taming Checked Exception##

Yes. Unfortunately, those beasts from the past still haunt us, 
more than ever when we’re using Java 8′s lambda expressions. 
Already before Java 8′s release, there are a couple of Stack Overflow questions related to the subject.

Mandatory checked exceptions handling in lambda expressions for standard Java 8 functional interfaces
Lambda function that throws exception?
Lambda-Streams, Filter by Method with Exception

Let’s remember how the IOExceptions caused issues when traversing 
the file system. Unless you write your own utility, 
you’ll have to resort to this beauty:

```java
Arrays.stream(dir.listFiles()).forEach(file -> {
    try {
        System.out.println(file.getCanonicalPath());
    }
    catch (IOException e) {
        throw new RuntimeException(e);
    }
 
    // Ouch, my fingers hurt! All this typing!
});
```

We think it is safe to say:

> Java 8 and checked exceptions don’t match.

A workaround is to write your own CheckedConsumer that wraps 
the checked exception. Such a consumer will be highly reusable, 
but… Did you think of all the other FunctionalInterfaces? 
There are quite a few of them in the java.util.function package.

To solve this issue, we have created jOOλ where we have duplicated 
pretty much every FunctionalInterface that is available from the JDK 
to support checked exceptions. 
Here’s how you would use jOOλ in the above example:

```java
Arrays.stream(dir.listFiles()).forEach(
    Unchecked.consumer(file -> {
        // Throw all sorts of checked exceptions
        // here, we don't care...
        System.out.println(file.getCanonicalPath());
    })
);
```

The above example shows how you can simply ignore and pass checked exceptions 
as RuntimeExceptions. If you actually want to handle them, you can pass an 
exception handler lambda:

```java
Arrays.stream(dir.listFiles())
      .forEach(Unchecked.consumer(file -> {
                                              System.out.println(file.getCanonicalPath());
                                          },
                                  e ->    {
                                              log.info("Log stuff here", e);
                                              throw new MyRuntimeException(e);
                                          }
);
```

The second example now seems equally verbose, but don’t worry. 
You will probably reuse that exception handler and fall back to this:

```java
Arrays.stream(dir.listFiles())
      .forEach(Unchecked.consumer(file -> {
                                              System.out.println(file.getCanonicalPath());
                                          },
                                  myExceptionHandler         
);
```


##Overloading##

While it is (probably) perfectly fine to overload these two methods

```java
void performAction(Parameter parameter);
void performAction(Supplier<Parameter> parameter);
. . .
```

you should stay wary when overloading “more similar” methods, like these ones:

```java
void performAction(Supplier<Parameter> parameter);
void performAction(Callable<Parameter> parameter);
```

If you produce the above API, your API’s client code will not be able to 
make use of lambda expressions, as there is no way of disambiguating a 
lambda that is a Supplier from a lambda that is a Callable. 
We’ve also mentioned this in the following.

## Overloading gets worse in Java 8##

Overloading, generics, and varargs aren’t friends. 
We’ve explained this in a previous article, and also in this Stack Overflow question. 
These might not be every day problems in your odd application, 
but they’re very important problems for API designers and maintainers.

With lambda expressions, things get “worse”. So you think you can provide 
some convenience API, overloading your existing run() method that accepts 
a Callable to also accept the new Supplier type:

```java
static <T> T run(Callable<T> c) throws Exception {
    return c.call();
}
 
static <T> T run(Supplier<T> s) throws Exception {
    return s.get();
}
```

What looks like perfectly useful Java 7 code is a major pain in Java 8, now. 
Because you cannot just simply call these methods with a lambda argument:

```java
public static void main(String[] args)
throws Exception {
    run(() -> null);
    //  ^^^^^^^^^^ ambiguous method call
}
```

Tough luck. You’ll have to resort to either of these “classic” solutions:

```java
run((Callable<Object>) (() -> null));

run(new Callable<Object>() {
    @Override
    public Object call() throws Exception {
        return null;
    }
});
```

So, while there’s always a workaround, these workarounds always “suck”. 
That’s quite a bummer, even if things don’t break from a 
backwards-compatibility perspective.



## *void-compatible* vs *value-compatible* ##

I’ve recently (re-)discovered this interesting early JDK 8 compiler bug, 
where the compiler wasn’t able to disambiguate the following:

```java
void run(Consumer<Integer> consumer);
void run(Function<Integer, Integer> function);
 
// Remember, the above types are roughly:
interface Consumer<T> {
    void accept(T t);
//  ^^^^ void-compatible
}
 
interface Function<T, R> {
    R apply(T t);
//  ^ value-compatible
}
```

The terms **void-compatible** and **value-compatible** are defined in the 
JLS §15.27.2 for lambda expressions. According to the JLS, the following 
two calls are not ambiguous:

```java
// Only run(Consumer) is applicable
run(i -> {});
 
// Only run(Function) is applicable
run(i -> 1);
```

In other words, it is safe to overload a method to take two 
“similar” argument types, such as Consumer and Function, 
as lambda expressions used to express method arguments 
will not be ambiguous.

This is quite useful, because having an optional return value 
is very elegant when you’re using lambda expressions. 
Consider the upcoming jOOQ 3.4 transaction API, which is 
roughly summarised as such:

```java
// This uses a "void-compatible" lambda
ctx.transaction(c -> {
    DSL.using(c).insertInto(...).execute();
    DSL.using(c).update(...).execute();
});
 
// This uses a "value-compatible" lambda
Integer result =
ctx.transaction(c -> {
    DSL.using(c).update(...).execute();
    DSL.using(c).delete(...).execute();
 
    return 42;
});
```

In the above example, the first call resolves to TransactionalRunnable whereas 
the second call resolves to TransactionalCallable whose API are like these:

```java
interface TransactionalRunnable {
    void run(Configuration c) throws Exception;
}
 
interface TransactionalCallable<T> {
    T run(Configuration c) throws Exception;
}
```

##Generic methods are not SAMs##

Do note that “SAM” interfaces that contain a single abstract generic method 
are NOT SAMs in the sense for them to be eligible as lambda expression targets. 
The following type will never form any lambda expression:

```java
interface NotASAM {
    <T> void run(T t);
}
```

This is specified in the JLS §15.27.3

> A lambda expression is congruent with a function type 
> if all of the following are true:

> * The function type has no type parameters.
> * [ ... ]

##What do you have to do now?##

If you’re an API designer, you should now start writing 
unit tests / integration tests also in Java 8. 
Why? For the simple reason that if you don’t you’ll get 
your API wrong in subtle ways for those users that are 
actually using it with Java 8. These things are extremely subtle. 
Getting them right takes a bit of practice and a lot of regression tests. 
Do you think you’d like to overload a method? Be sure you don’t break 
client API that is calling the original method with a lambda.

