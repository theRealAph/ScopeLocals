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

[ It is reasonable to ask why all such information should not be
passed explicitly as arguments. However, this can be very
inconvenient. For example, it might be necessary to add logging to a
library, and adding a current logger to every API entry point would be
onerous. But it would not just be inconvenient. Having to explicitly
add loggers everywhere would constrain the evolution of that library's
API. ]

In such cases, there is a notion of _context_ that is set by a
higher-level caller.

### Not static fields!

The Java language has never had a really good way to do this. Static
fields might appear at first glance to be a solution. However, in a
multi-threaded application, which includes the vast majority of server
applications, it is useful to associate context with a thread, rather
than globally. Static fields are global in nature, so won't
work.

### Thread local variables

Since Java 1.2, ThreadLocals have been the standard way to associate
context with a thread.

(Story about thread local maps here.)

Unfortunately, ThreadLocals have some disadvantages when used for this
purpose.

In essence, a `ThreadLocal` is a key that may be used to set and get
the current thread's copy of a thread-local variable. Once set in a
thread, its value my be returned by `get()`. Once set, a ThreadLocal
is _persistent_: that is to say, it is retained for the lifetime of
the thread or until a method running on that thread calls `remove()`

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

### Virtual threads versus thread local variables

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
virtual thread to carry thread-local variables. The short-running
nature of virtual threads means that the crap programming model of
thread locals doesn't matter because it doesn't matter when they die
at a prodigous rate - `remove()` isn't necessary when a thread,
virtual or not, terminates.

Unfortunately, thread locals present another problem in the era of
virtual threads, because thread locals may be inheritable.
This feature, _inheritability_, is predicated on there being a
relatively small set of threads sharing domain objects. When a new
`Thread` instance is created, its parent's set of inheritable
thread-local variables is deeply copied. This is necessary because a
thread's set of thread locals is, by design, mutable, so it cannot be
shared between threads. Every child thread ends up carrying a local
copy of its parent's entire set of `InheritableThreadLocal`s.

For such uses we want sharing, but we do not want mutability. It
should be popssible for a child thread to share its parent's context,
but it's not necessary to mutate it. In contrast, thread local
variables assume mutability. While it makes sese for a parent to share
context with a million children, it makes no sense at all for them to
maintain mutable copies of it.

### The problem with unconstrained mutability

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

While using a thread local variable to store context seems reasonable
at first, it suffers from unconstrained mutability. Any callee with
access to `ThreadLocal.get()` also can call `set()` or even
`remove()`. This results in a kind of "action at a distance" where the
relationship between a caller which sets the context is impossible to
determine from the code alone.

It is far better, then, to have the structure (of what?) exposed in
the code, so that it is possible to write maintainable
programs. Maintainability is more important than programming
tricks. Reading a program is more important than writing it.

Context is a fine thing to be propagated from caller to callee, where
it should be immutable, but is is a terrible thing when a caller's
context is mutable by callees.

It would be ideal if the Java Platform provided a way to have per
thread context for millions of virtual threads that is immutable by
default and, given the low cost of forking virtual threads,
inheritable. Because these ideal per thread variables are immutable,
thir data can be easily shared by child threads, rather than copied to
child threads.

Because these ideal per thread variables are immutable and
lightweight, they align well with the thread-per-request model given
new life by virtual threads

The need for extent locals arose from Project Loom, where threads are
cheap and plentiful, rather than expensive and scarce. If you only
have a few hundred platform threads, maintaining a thread local map
seems viable. However, if you have _millions_ of threads, maintaining
millions of thread local maps becomes a significant burden, both in
terms of creating the maps and the memory they occupy.

Instead of hundreds of platform threads you have millions of virtual
threads. However, a different model of context is desirable when
programming with virtual threads.

With Project Loom's virtual threads you can keep your beloved
thread-per-request model.  Wouldn't it be terrible if virtual threads
carried over the thread locals heavyweight model of inheritability?

In summary, extent locals fix these problems with:

* Sharing, not mutation
* Automatic memory management, not manual

## Description

An extent local variable is a per thread vairable that allows context
to be set in a caller and read by callees. Unlike a thread local
variable, an extent local variable is immutable. Context can be
anything from a business object to an instance of a system-wide
logger.

The term extent local derives from the idea of extent in the Java
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
bottom most frame of an extent, and is accessible in every frame of
that extent.

### For example

The following example uses an extent local variable to make
credentials available to callees.

The ultimate caller is the server framework. It is resoponsible for
initializing an extent local variable with some credentials. The
framework then runs some piece of code (supplied by the user) thread
that connects to a database. The connectDatabase() method then uses
the credentials set by it's caller's caller to determine if its caller
to access the database. It is as if the connectDatabase() method has
an invisible parameter to represent the caller's credentials, even
though the caller itself did not pass them.

```
class DatabaseConnector {

    // Declare an extent local to hold credentials for the current thread
    private static final ExtentLocal<Credentials> CREDENTIALS = ExtentLocal.newInstance();

    Credentials creds = newCredentials();
    
    // Bind the extent local CREDENTIALS in the current thread
    // to our new credentials
    // The `run()` method here is the bottom most frame of the extent
    // in which `DatabaseConnector.CREDENTIALS.get()` will return
    // `creds`.
    ExtentLocal.where(DatabaseConnector.CREDENTIALS, creds).run(() -> {
        :
        Connection connection = connectDatabase();
        :
    });

    Connection connectDatabase() {
        // Read the caller's credentials to see if they have
        // sufficient permissions for this action.
        if (DatabaseConnector.CREDENTIALS.get().s != "MySecret") {
            throw new SecurityException("No dice, kid!");
        }
        return new Connection();
    }
}
```

[ Should this paragraph be elsewhere? ]
In this example `DatabaseConnector.CREDENTIALS.get()` has a hidden
parameter: the current thread. The `ExtentLocal.get()` operation could
be written as
`Thread.currentThread().getExtentLocal(DatabaseConnector.CREDENTIALS)`,
which more clearly shows that a `ExtentLocal` instance is a key, which
is used to look up the current thread's incarnation of an extent local.


```
class Example {

    // Define an extent local that may be bound to Integer values:
    static final ExtentLocal<Integer> X = ExtentLocal.newInstance();

    void printIt() {
        // Bind X to the value 5, then run anotherMethod()
        ExtentLocal.where(X, 5).run(() -> anotherMethod());
    }

    void anotherMethod() {
        for (int i = 0; i < 10; i++) {
            System.out.println(X.get() * i);
        }
        System.out.println(X.get());
    }
}
```

[ I hate this paragraph: ]
An `ExtentLocal` instance such as `X` above is a key that is used to
look up a value in the current thread. Despite `X` being declared
static in class `Example`, there is _not_ exactly one incarnation of
the field shared across all instances of Foo; instead, there are
_multiple_ incarnations of the field, one per thread, and the
incarnation of `X` that is used when code performs a field access
(`X.get()`) depends on the thread which is executing the code.

An extent local acquires (we say: _is bound to_) a value on entry to a
extent; when that extent terminates, the previous value (or none) is
restored. In this case, the extent of `X`'s binding is the duration of
the Lambda invoked by `run()`.

One useful way to think of extent locals is as invisible, effectively
final, parameters that are passed through every method invocation.
These parameters will be accessible within the extent of a binding
operation. [ Do we need that sentence? ]

```
  // Declare extent locals x and y
  static final ExtentLocal<MyType> x = ExtentLocal.newInstance();
  static final ExtentLocal<MyType> y = ExtentLocal.newInstance();

  ... much later

  {
    ExtentLocal.where(x, expr1)
              .where(y, expr2)
              .run(() -> ... code that uses x.get() and y.get() ...);

    // An attempt to invoke x.get() here would throw an exception
    // because x is not bound to a value outside the lambda invoked
    // by run().
  }
```

In this example, `run()` is said to "bind" `x` and `y` to the results
of evaluating `expr1` and `expr2` respectively. While the method
`run()` is executing, any calls to `x.get()` and `y.get()` return the
values that have been bound to them. The methods called from `run()`,
and any methods called by them, comprise the extent of `run()`.

Please note that the code that uses `x.get()` and `y.get()` may be a
very long way away, for example in a callback somewhere. Imagine a
complex system, with many intervening method calls, between the point
where an extent local is bound to a value and the point where that value
is retrieved. Like so:

```
    void run1() {
        ExtentLocal.where(x, new MyType("Jill"))
                  .where(y, new MyType("Sofia"))
                  .run(() -> m());
    }
    void m() {
        n();
    }
    void n() {
        o();
    }
    void o() {
        System.out.println("My name is " + x.get().toString());
        // Prints "My name is Jill"
    }
```

But when called in a different context, we see a different value for
`x.get()`

```
    void run2() {
        ExtentLocal.where(x, new MyType("Helena"))
                  .where(y, new MyType("Henri"))
                  .run(() -> p());
    }
    void p() {
        q();
    }
    void q() {
        r();
    }
    void r() {
        System.out.println("My name is " + x.get().toString());
        // Prints "My name is Helena"
    }
```

In summary, extent locals have the following properties:

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

### Hidden parameters for callbacks

You may want to invoke a method `X` in a library that later calls back
into your code. In your callback you to do some logging, so you need a
way to find the logger to use. However, `X` provides no way to pass a
reference to a logger through their code into your callback. You don't
want to turn logging on all the time, just when certain parts of the
programmer are executing.

Here's a simple example of how you'd using an extent local to add
context-sensitive logging to an application. Let's assume that you
want to log some events, but only for certain places when your
application is running.

First, declare an interface that is invoked when a loggable event
occurs, and a `ExtentLocal` instance that will refer to one:

```
    interface MyLogger {
        public void log(String s);
    }

    private static final ExtentLocal<MyLogger> SL_LOGGER
            = ExtentLocal.newInstance(MyLogger.class);
```

In your application code, call `SL_LOGGER`'s `log()` method:

```
    void someMethodDeepInALibrary() {
        // ...
        SL_LOGGER.orElse(NULL_LOGGER).log("Here's looking at you, kid.");
```

And when you want to do some logging, bind `SL_LOGGER` to do whatever
you want:

```
        ExtentLocal.where(SL_LOGGER, (s) -> LOGGER.severe(s)).run(this::exec);
```

### Securely passing credentials

### Shadowing

It is sometimes useful to be able to re-bind an already-bound extent
local. For example, a privileged method may need to connect to a
database with a less-privileged set of credentials, like so:

```
      Credentials creds = CREDENTIALS.get();
      creds = creds.withLowerTrust();
      ExtentLocal.where(CREDENTIALS, creds).run(() -> {
        :
        Connection connection = connectDatabase();
        :
      });
```

This "shadowing" only extends until the end of the extent of the
lambda above.

(Note: This code example assumes that `CREDENTIALS` is already bound
to a highly privileged set of credentials.)

### Recursion detection, and locating the current context

Sometimes you want to be able to detect recursion, perhaps because a
framework isn't re-entrant or because you want to limit recursion in
some way. A thread-local variable provides a way to do this: set it
once, invoke a method, and somwhere deep in that method, test again to
see if the thread-local variable is set. For this to work reliably
you'll probably need the thread-local variable to be a recursion
counter, which you can `remove()` when it gets to zero.

The detection of recursion is also useful in the case of flattened
transactions: any transaction started when a transaction in progress
becomes part of the outermost transaction.

This is a rather complicated example because it's adapted from
real-world library code. It contains recursion detection and the idea
of a current context, in this case a graphics rendering context.
Rendering actions call other rendering actions, but they should share
a rendering context. If there isn't a current context we should crate
one; if there is, we should use it.

```
    public final RendererContext getRendererContext() {
        // RENDER_CTX is an extent local that refers to a context
        // If there's already a bound RendererContext, use it,
        //   else create and return a new one.
        if (RENDER_CTX.isBound()) {
            return RENDER_CTX.get();
        }
        return newContext();
    }

    // called from here ...

    final RendererContext rdrCtx = getRendererContext();
    rdCtx.call( () -> {
        // ... graphics operations
    });

```

Where `RendererContext.call()` is defined like this:

```
class RendererContext {

    // Call r with RENDER_CTX bound to this RendererContext
    T call(Callable<T> r) throws Exception {
        return ExtentLocal.where(RENDER_CTX, this).call(r);
    }

    // ...
}
```

We'd like to support as many of these use cases as we can, but only if
the basic properties of effective finality and re-entrancy can be
guaranteed.

## Replacing some uses of `ThreadLocal` with extent locals

Because extent locals have a well-defined lifetime, the block in which
they were bound, they can never be used in a way that's identical to
the way thread-local variables are used. Therefore, some code changes
will be required to switch from thread locals to extent locals.

Please note that, for the sake of brevity, these are simple
examples. In some of these examples a simple refactoring would make
the use of extent locals unnecessary, but that would be far more
difficult in a more complex scenario with multiple libraries of
separate authorship.

### The idea of a "current context"

```
Java Concurrency in Practice:

"... containers associate a transaction context with an executing
thread for the duration of an EJB call. This is easily implemented
using a static Thread-Local holding the transaction context: when
framework code needs to determine what transaction is currently
running, it fetches the transaction context from this
ThreadLocal."
```

This kind of thing is well suited to the use of an extent local, but
because extent local bindings are always removed at the end of the
extent in which they were bound, you must run an entire operation from
a binding extent. So, you won't be able to do something like this
example, which has a `ThreadLocal` embedded in a `UserTransaction`:

```
  public void foo() {
      UserTransaction utx = getUserTransaction();

      // Start a transaction
      utx.begin();

      // Do work

      // Commit it
      utx.commit();
  }
```

where `UserTransaction` does something like

```
class UserTransaction {

  private static final ThreadLocal<UserTransaction> TRANSACTION = new ThreadLocal<>();

  begin() {
    ... // initialize
    TRANSACTION.set(this);
  }

  commit() {
    ... commit the transaction
    TRANSACTION.remove();
  }

  ...
}
```

instead you might do something like this, where
`UserTransaction.run()` binds an extent local then calls a lambda:

```
  public void foo() {
      UserTransaction utx = getUserTransaction();

      utx.run(() -> {

        // Do work

      });
  }
```

where `UserTransaction.run()` does something like

```
class UserTransaction {

  private static final ExtentLocal<UserTransaction> TRANSACTION = ExtentLocal.newInstance();

  void run(Runnable action) {
    ... // initialize
    ExtentLocal.where(TRANSACTION, this).run(action);
    ... commit the transaction
  }

  ...
}
```

that is to say, run an entire operation in an extent local binding.

### Caches for Non-thread-safe objects that are expensive to create

These are problematic for extent locals, perhaps because caches are one
of the few use cases for which thread-local variables are ideally
suited. If you can create caches you are likely to need in an
outermost extent that would work, but it requires some structural changes.

### Hidden return values

This isn't difficult to do with extent locals: create an empty instance
of a container class in the outer extent, call some method, and the
callee method `set()`s the value in the container. However, while it's
pretty obvious how it do this, it's rather evil.

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
`UnsupportedOperationException` for much core functionality, or
both. It is better, therefore, not to do that but to give extent locals
a separate identity from thread locals.



* New Stuff

The run() method is the bottom-most frame of the extent in which the
extent local variable is defined.

Clearly, extent local variables are a constrained version of thread
locals. Once set, a thread local variable is defined for the entie
lifetime of its thread. an extent local variable is defined for its
extent.

** Thread inheritance

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


## Alternatives

We could have had a non-settable TL that threw an
`UnsupportedOperationException`, but we'd still have the problem of
memory management.


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

