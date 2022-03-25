# Extent-local variables

## Summary

Define a standard, lightweight way to pass contextual information from
a caller to callees in the currently executing thread, and in some
cases to a group of threads. In particular, extent locals are intended
to replace `ThreadLocal`s for sharing information with a very large
number of threads. This usage especially applies to Project Loom's
virtual threads.

## Goals

## Non-Goals

It is not a goal to change the Java Programming Language.

This JEP is only concerned with associating local names and values. It
doesn't attempt to replace, for example, try-with-resources. It
doesn't perform cleanup operations when an extent ends, and isn't
related to a C++-style destructor.

It is not a goal to force virtual threads to use extent locals instead
of thread local variables. While we expect extent locals to be a better
fit in many or most cases, thread-local variables may still be useful
in some contexts,

## Motivation

In Java programs, a method may receive its input from several
sources. The primary source is the method's arguments, but there are
others, such as static fields.

### Contexts

Sometimes a method needs information that has not been passed to it
via its arguments. For example, a security-sensitive method in a
library might need to check that a caller has appropriate permissions
to perform an action.

In other words, we have a caller and a callback, with third-party code in between. The initiator of the call chain needs a channel through which to pass some private data to the callback. In such cases, there is a notion of _context_ that is set at the original point of call. This could even apply to the Java system itself, which starts a user's program by calling `main`. The Java program then calls back into the runtime to perform I/O. 

[ It is reasonable to ask why all such information should not be
passed explicitly as arguments. However, this can be very
inconvenient. For example, it might be necessary to add logging to a
library, and adding a current logger to every API entry point would be
onerous. But it would not just be inconvenient. Having to explicitly
add loggers everywhere would constrain the evolution of that library's
API. ]

### Not static fields!

The Java language has never had a really good way to do this. Static
fields might appear at first glance to be a solution. However, in a
multi-threaded application, which includes the vast majority of server
applications, it is useful to associate context with a thread, rather
than globally. Static fields are shared by all threads, so this won't
work.

### Thread local variables

Since Java 1.2, `ThreadLocal`s have been the standard way to associate
context with a thread.

```
class Example {

    // Define a thread local variable that may be bound to Integer values:
    static final ThreadLocal<Integer> X = new ThreadLocal();

    void printIt() {
        // Set X to the value 5, then run anotherMethod()
        X.set(5);
        anotherMethod();
    }

    void anotherMethod() {
        for (int i = 0; i < 10; i++) {
            System.out.println(X.get() * i);
        }
    }
}
```

A `ThreadLocal` instance such as `X` above is a key that is used to
look up a value in the current thread. Despite `X` being declared
static in class `Example`, there is _not_ exactly one incarnation of
the field shared across all instances of Foo; instead, there are
_multiple_ incarnations of the field, one per thread, and the
incarnation of `X` that is used when code performs a field access
(`X.get()`) depends on the thread which is executing the code.

The `Thread` object returned by `Thread.currentThread()` and
`ThreadLocal` have a relationship between the scenes.
`ThreadLocal.get()` is caller sensitive: there is a hidden parameter,
the current thread. (A `ThreadLocal` works behind the scenes with the
current thread to do the magic.)

In other words, the Java runtime has long supported low-ceremony
contextual per-thread information.

### Thread local variables are not ideal for contexts

Unfortunately, ThreadLocals have some disadvantages when used for the
purpose of carrying contexts.

In essence, a `ThreadLocal` is a key that may be used to set and get
the current thread's copy of a thread-local variable. Once set in a
thread, its value my be returned by `get()`. Once set, a ThreadLocal
is _persistent_: that is to say, it is retained for the lifetime of
the thread, or until a method running on that thread calls `remove()`

Also, every thread-local variable is _mutable_: that is to say,
everyone who can access a `ThreadLocal`'s `get()` can alse `set()` it
to a different value, or `remove()` is altogether. Frameworks can wrap
`ThreadLocal`s to make sure they cannot be set inappropriately, but
you're still carrying the cost of mutability.


### The problem with unconstrained mutability

Mutability by default is a bad decision.

[ TLs being mutable fies in the face of the repeated advice in
_Effective Java_ to minimize mutability.

_Effective Java_, Item 8: Avoid finalizers and cleaners, by using
try-with-resources and try...finally. Item 17: minimize mutability:
don't provide methods that modify the object's state. Yet
`ThreadLocal` depends on these things we're advised to minimize and
avoid. ]

Thread local variables are prone to abuse. Fundamentally, programming
with thread local variables can lead to spaghetti-like coding, for
example when used to return a hidden value from a method to some
distant caller, far away in a deep call stack. This leads to code
whose structure is hard to discern, let alone maintain. 

While using a thread local variable to store context seems reasonable at first, it suffers from unconstrained mutability. Any callee with access to ThreadLocal.get() also can call set() or even remove(). This results in a kind of "action at a distance" where the relationship between a caller which sets the context is impossible to determine from the code alone. The problem is not distance as such, but the fact that the direction of communication is two way. When a method sets a thread local variable, it doen't just change the situation for all its callees, but also for its callers. There is no way to tell from looking at the code which is intended.  

It is far better, then, to have the structure (of what?) exposed in
the code, so that it is possible to write maintainable
programs. Maintainability is more important than programming
tricks. Reading a program is more important than writing it.

### Thread locals and manual cleanup

It is unfortunately common for developers to forget to remove a
`ThreadLocal`, which can lead to a long-term memory leak. Even though
the program has long since moved on from having any use for an object,
it will not be garbage collected if no method has called
`remove()`. 

It would be better if the context associated with a thread were to be
cleaned up automatically. Also, having to call `remove()` on a
`ThreadLocal` to clean it up when it's no longer in use is somewhat
antithetical to the way that Java usually works.

In practice, thread locals are managed by the `Thread` itself.  Every
thread must maintain a thread local map. This is an object that maps
from `ThreadLocal` instances to each thread's copy of that
thread-local variable. Just as a program associates data with a
thread-local variable, the Java runtime associates a `ThreadLocal`
instances with data via a thread local map.

### Inheritance versus mutability

Unfortunately, thread locals present another problem, because thread locals may be inheritable.
This feature, _inheritability_, is predicated on there being a
relatively small set of threads sharing domain objects. When a new
`Thread` instance is created, its parent's set of inheritable
thread-local variables is deeply copied. This is necessary because a
thread's set of thread locals is, by design, mutable, so it cannot be
shared between threads. Every child thread ends up carrying a local
copy of its parent's entire set of `InheritableThreadLocal`s.

For such uses we want sharing, but we do not want mutability. It
should be possible for a child thread to share its parent's context,
but it's not necessary for a child to mutate it. In contrast, thread
local variables assume mutability.

### Virtual threads versus thread local variables

The need for something like extent local variables arose in the
context of Project Loom's virtual threads. These are cheap and
plentiful, unlike today's platform threads which are expensive and
scarce.

Platform Threads are:

* Long-running
* Heavyweight
* Pooled

Virtual Threads are:

* Short-running
* Lightweight
* Singular (?)

It would certainly be useful for these numerous cheap and plentiful
threads to be able to access some context from their parent. For
example, they may share a logger on an output stream. Perhaps they may
share some kind of security policy too.

Because virtual threads are still threads, it is legitimate to for a
virtual thread to carry thread-local variables. The short lifetime of
virtual threads means that the programming model of thread locals
doesn't matter, because it doesn't matter when they die at a prodigous
rate - `remove()` isn't necessary when a thread, virtual or not,
terminates. However, if you have a million threads and every one has
its own inevitably mutable set of thread local variables, the memory
footprint may become significant.

[ Stuff about virtual inheritance here. ]  While it makes sese for a parent to
share context with a million children, it makes no sense at all for
them to maintain mutable copies of that context.

Context is a fine thing to be propagated from caller to callee, where
it should be immutable, but is is a terrible thing when a caller's
context is mutable by callees.

It would be ideal if the Java Platform provided a way to have per
thread context for millions of virtual threads that is immutable and,
given the low cost of forking virtual threads, inheritable. Because
these ideal per thread variables are immutable, thir data can be
easily shared by child threads, rather than copied to child
threads. Thread local variables were the 90s realization of per thread
variabels; we need a better realization of per thread vairables for
the modern era.

Because these ideal per thread variables are immutable and
lightweight, they align well with the thread-per-request model given
new life by virtual threads

## Description

An extent local variable is a per thread variable that allows context
to be set in a caller and read by callees. Unlike a thread local
variable, an extent local variable is immutable: there is no `set()`
method. Context can be anything from a business object to an instance
of a system-wide logger.

The term extent local derives from the idea of an extent in the Java
Virtual Machine. The JVM specification describes an extent as follows:

"It is often useful to describe the situation where, in a given
thread, a given method m1 invokes a method m2, which invokes a method
m3, and so on until the method invocation chain includes the current
method mn. None of m1..mn have yet completed; all of their frames are
still stored on the Java Virtual Machine stack (2.5.2). Collectively,
their frames are called an _extent_. The frame for the current method
mn is called the _top most frame_ of the extent. The frame for the
given method m1 is called the _bottom most frame_ of the extent."

(That is to say, m1's extent is the set of methods m1 invokes, and any
methods invoked transitively by them.)

The value associated with an extent local variable is defined in the
bottom most frame of some extent, and is accessible in every frame of
that extent. The extent local is bound to the value.



### For example

The following example uses an extent local variable to make
credentials available to callees.

The ultimate caller is the server framework. It is resoponsible for
binding an extent local variable to some credentials. The framework
then runs some piece of code (supplied by the user) that connects to a
database. The `connectDatabase()` method, also part of the framework,
then uses the credentials bound by its caller's caller (also the
framework) to determine if its caller, the user code, may connect to
the database. It is as if the `connectDatabase()` method has an
invisible parameter to represent the caller's credentials, even though
the caller itself did not pass them.

(We assume that the `Credentials` class is not widely accessible and
cannot be instantiated by user code.)

```
class ServerFramework {
    private static final ExtentLocal<Credentials> USER_CREDENTIALS = ExtentLocal.newInstance();

    void processRequest() {
        ExtentLocal.where(ServerFramework.USER_CREDENTIALS, Credentials.DEFAULT)
                   .run(() -> { ...
                                var connection = connectDatabase();
                                ... });
    }

    Connection connectDatabase() {
        // Use the caller's credentials
        var creds = ServerFramework.USER_CREDENTIALS.get();
        if (! creds.equals ...) {
            throw new SecurityException("Invalid credentials");
        }
        return new Connection();
    }
}
```

A server framework may configure the behaviour of `run()` so that the
user code will be run in a new virtual thread. This witnesses a
thread-per-request model. Starting a new thread means that
`connectDatabase()` will run in a different thread than
`processrequest()`.  Plainly, the body of `connectDatabase()` needs to
use the extent local variable. Fortunately, the extent local variable
is inheritable such that its value is usable by a child thread. The
server framework can easily achieve this.

The `ExtentLocal.get()` operation could be thought of as
`Thread.currentThread().getExtentLocal(DatabaseConnector.USER_CREDENTIALS)`,
which clearly shows that a `ExtentLocal` instance is a key used
to look up the current thread's incarnation of an extent local.

An extent local acquires (we say: _is bound to_) a value on entry to a
extent; when that extent terminates, the previous value (or none) is
restored. In this case, the extent of `ServerFramework.USER_CREDENTIALS`'s
binding is the duration of the Lambda invoked by `run()`. In the
example above, the extent unfolds from process request, through a
lambda, to connectDatabase(). The frame for connectDatabase() is the
topmost frame; the frame for processRequest() is the bottom most
frame. If connectDatbase() called more methods, then those methods
would also be able to use the extent local variable to get the
needful. None of the methods in the extent can mutate the extent local
variable so that it holds different credentials.

One useful way to think of extent locals is as invisible, effectively
final, parameters that are passed through every method invocation.
These parameters will be accessible within the extent of a binding
operation. [ Do we need that sentence? ]

### Nested bindings

We said earlier than an extent local variable is bound to a
value. Within the extent of that binding it is possible to _re-bind_
the variable to a new value.

The example above we show the user code physically within the server
framework class. Therefore the notional user code can access the
`USER_CREDENTIALS` and attempt to rebind it. However, in real life we
couldn't. Even though the client might be able to re-bind
`ServerFramework.USER_CREDENTIALS`, there should be no way to forge
legitimate credentials, because the payload class doesn't allow
unprivileged classes to create new credentials.

It is sometimes useful to be able to re-bind an already-bound extent
local. For example, a privileged method may need to connect to a
database with a less-privileged set of credentials.

Another use for non-string results is when formatting messages for
logging. Many logging calls are for debug information, and often debug
logging is turned off. Many frameworks allow you to provide a
Supplier<String> for log messages that is only invoked if the message
is actually going to be logged, to avoid the overhead of formatting a
string that is going to be thrown away. A lazy policy object could
produce Supplier<String> rather than String itself.

```
class ServerFramework {
    private static final ExtentLocal<Credentials> USER_CREDENTIALS = ExtentLocal.newInstance();

    void processRequest() {
        ExtentLocal.where(ServerFramework.USER_CREDENTIALS, Credentials.DEFAULT)
                   .run(() -> { ...
                                var connection = connectDatabase();
                                log(() -> "Hello, World!");
                                ... });
    }

    Connection connectDatabase() {
        ... as before ...
    }
    
    void log(Supplier<String> supplier) {
      Credentials creds = ServerFramework.USER_CREDENTIALS.get();
      creds = creds.withLowerTrust();
      String s = ExtentLocal.where(ServerFramework.USER_CREDENTIALS, creds)
                 .call(() -> supplier.get());
      ... maybe print s
    }
}
```


This "shadowing" only extends until the end of the extent of the
lambda above.

(Note: This code example assumes that `USER_CREDENTIALS` is already bound
to a highly privileged set of credentials.)

(Note: Normally, user code run by the framework is not expected to
return a result, the processRequest() method uses run(), which takes a
`Runnable` and returns nothing. In contrast, the user code supplied to
`log()` is expected to return a result. Therefore, the `log()` method
uses call(), which takes a `Callable<String>` and returns a `String`.)

### In summary

Extent locals have the following properties:

* _Locally-defined extent_: The values of `x` and `y` are only bound
  in the extent of `run()`.
* _Immutability_ There is no `ExtentLocal.set()` method: extent locals,
  once bound, are effectively final.
* _Simple and fast code_: In most cases an extent-local `x.get()` is as
  fast as a local variable `x`. This is true regardless of how far
  away `x.get()` is from the point that the extent local `x` is bound.
* _Structure_: These properties also also make it easier for a reader
  to reason about programs, in much the same way that declaring a
  field of a variable `final` does.

## Uses of extent locals

## Where can extent local variables _not_ replace thread local variables?

There are cases where thread local variables are more appropriate than
extent local variables. For example, one popular use of `ThreadLocal`
is to cache objects that are expensive to create. One notorious
example is `java.text.DateFormat`, which is mutable so cannot be
shared between threads without synchronization. In this case, creating
a thread local `DateFormat` object which persists for the lifetime of
the thread might be exactly what you need:

```
public class Example2 {

    private static ThreadLocal<DateFormat> SDF = new ThreadLocal<DateFormat>() {
        protected DateFormat initialValue()
        {
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };

    static void printToday() {
        System.out.println(SDF.get().format(new Date()));
    }

    public static void main(String[] args) {
        printToday();
    }
}
```

(In hindsight, making `DateFormat` mutable was a mistake, and we'd do
better today, but it was Java 1.1 in 1997. `ThreadLocal` makes it
possible to use this utility class in a multi-threaded program.)

## API

There is more detail in the Javadoc for the API, at

http://people.redhat.com/~aph/loom_api/jdk.incubator.concurrent/jdk/incubator/concurrent/ExtentLocal.html

## Alternatives

It is possible to emulate many of the features of extent locals with
`ThreadLocal`s, albeit at some cost in memory footprint, runtime
security, and performance.

We have experimented with a modified version of `ThreadLocal` that
supports some of the characteristics of extent locals. However,
carrying the additional baggage of `ThreadLocal` results in an
implementation that is unduly burdensome, or an API that returns
`UnsupportedOperationException` for much core functionality, or both.
And we'd still have the problem of memory management.

It is better, therefore, not to do modify `ThreadLocal` but to give
extent locals a separate identity from thread locals.

## New Stuff

The run() method is the bottom-most frame of the extent in which the
extent local variable is defined.

Clearly, extent local variables are a constrained version of thread
locals. Once set, a thread local variable is defined for the entie
lifetime of its thread. an extent local variable is defined for its
extent.

## Thread inheritance

Like a thread-local variable, an extent-local variable can be 
_inheritable_. This means that the state of the extent-local variable in 
a parent thread is available to code running in child threads. The 
immutability of state in an extent-local variable means that 
inheritability in the child thread is fast and lightweight; this 
comports with programs using cheap and plentiful virtual threads as the 
child threads.

The extent local variables defined in the parent's extent (at the time
the child threads are started) are also defined in the child.

A newly created thread has read-only access the extent local variables
defined in it parent.

A child virtual thread can inherit its parent's set of extent local
variables, but as long as the inherited variables are read only,
sharing is fine.

With a million vitual threads, it would seem like there are an awful
lot of maps. But immutable maps can be shared! This is not true of
thread locals, which are mutable, so cannot.

** Thread local properties

The model where you call `remove()` is plainly dumb.

Thread locals are mutable by default, just like the local variables
they were trying to be like.

Thread locals are a less-than-global variable.

It turns out that many of the use cases for TLs have turned out to be
entirely immutable. But the requirement that a TL has to be immutable
is burned into the API.

A TL is always mutable but only optionally inheritable. The fact that
TLs are always inheritable requires inheritability to be turned
off. If a TL were immutable, they would be cheaper to inherit, which
in turn would mean inheritability could be enabled more widely.


# Orphan paragraphs

Extent locals are

* Not global
* Not per-thread
* Per a constrained slice of a thread

Global context is bad, per thread is better. Extent locals are an
intra-thread context mechanism that doesn't suffer from these
problems.

Dynamically scoped variables have been around for a long time. It's
time to being them to Java.

Dynamic scoping enables context to flow through the program from
caller to callee, to their callee, and so on. They are a way to have
invisible parameters passed through every method invocation. These
parameters are then usable in the extent of, say, request
handlers.

Virtual threads - server code. They are small, shallow, and short
lived.  The call stack of that code is a natural boundary - it just
contains the business logic.  It needs to looks things up in the
context of the framework, and it needs to call into the JDK, which
needs to know e.g. the permissions of the caller.

Setting up context is the responsibility of the sever framework.  The
server stores the credentials for a thread in an extent local.  The
application logic knows nothing about security, but the JDK will check
the current thread's permissions by looking in the extent local.

If you have a million virtual threads, the JDK connection needs to
check the credentials, and this is happening concurrently in a million
virtual threads.

Extent locals are like thread locals, but with

* Better memory management
* Better inheritability

In summary, extent local is a mechanism which exploits the idea of
dynamic scoping by allowing a variable to provide useful context to a
program, and to be unassociated automatically. Such variables would
effectively be invisible parameters passed through every method in the
call stack. This will lead to more reliable multi-threaded programs.

