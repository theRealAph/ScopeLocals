# Scope Locals

## Summary

Define a standard, lightweight way to pass contextual information from
a caller to callees in the currently executing thread, and in some
cases to a group of threads. In particular, scope locals are intended
to replace `ThreadLocal`s for sharing information with a very large
number of threads. This usage especially applies to Project Loom's
virtual threads.

## Motivation

In Java programs, a method may receive its input from several
sources. The primary source is the method's arguments, but there are
others, such as static fields.

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

The Java language has never had a really good way to do this. Static
fields might appear at first glance to be a solution. However, in a
multi-threaded application, which includes the vast majority of server
applications, it is useful to associate context with a thread, rather
than globally. Static fields are global in nature, so won't
work. Since Java 1.2, ThreadLocals have been the standard way to
associate context with a thread.

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
cleaned up automatically.

In practice, thread locals are managed by the `Thread` itself.  Every
thread must maintain a thread local map. This is an object that maps
from `ThreadLocal` instances to each thread's copy of that
thread-local variable. Just as a program associates data with a
thread-local variable, the Java runtime associates a `ThreadLocal`
instances with data via a thread local map.

The need for scope locals arose from Project Loom, where threads are
cheap and plentiful, rather than expensive and scarce. If you only
have a few hundred platform threads, maintaining a thread local map
seems viable. However, if you have _millions_ of threads, maintaining
millions of thread local maps becomes a significant burden, both in
terms of creating the maps and the memory they occupy.

Some developers wishing to utilize hardware to its fullest have given
up the thread-per-request model in favor of a thread-sharing
model. Instead of a request being handled by one thread from start to
finish, the request-handling code returns its thread to a pool when it
waits for an I/O operation to complete so that the thread can service
other requests. This fine-grained sharing of threads (code only holds
on to a thread when it performs calculations, not when it waits for
I/O) allows a high number of concurrent operations without consuming a
high number of threads. In this model, tasks are not bound to any
particular thread. Whenever an I/O operation completes, a thread is
taken from a pool and that thread is used for whichever task became
ready.

A task that skips from thread to thread is at odds with the idea that
task state can be kept on a thread.

In addition, thread local variables have a feature, _inheritability_,
that is predicated on a relatively small set of threads sharing domain
objects. When a new `Thread` instance is created, its parent's set of
inheritable thread-local variables is deeply copied. This is necessary
because a thread's set of thread locals is, by design, mutable, so it
cannot be shared between threads. Every child thread ends up carrying
a local copy of its parent's entire set of `InheritableThreadLocal`s.

With Project Loom's virtual threads you can keep your beloved
thread-per-request model.  Wouldn't it be terrible if virtual threads
carried over the thread locals heavyweight model of inheritability?

It would certainly be useful for these numerous cheap and plentiful
threads to be able to access some context from their parent. For
example, they may share a logger on an output stream. Perhaps they may
share some kind of security policy too.

For such uses we want sharing, but we do not want mutability. It
should be popssible for a child thread to share its parent's context,
but it's not necessary to mutate it. In contrast, thread local
variables assume mutability. While it makes sese for a parent to share
context with a million children, it makes no sense at all for them to
maintain mutable copies of it.

Fundamentally, programming with thread local variables can lead to
spaghetti-like coding, for example when used to return a hidden value
from a method to some caller. This leads to a kind of spaghetti code
whose structure is hard to discern, let alone maintain.

Context is a fine thing to be pushed downwards from caller to called
methods, but a terrible thing when pushed upwards.





We'd like to have a feature that allows per-thread context information
to be inherited by a thread without an expensive deep-copy operation.
Some kind of immutable data structure fits this need, because the
inheriting thread needs only to copy a reference to its parent's set
of values.


We'd like this feature to declare the lifetime and the region of
accessibility of the context information it refers to in an explicit
way: "between _here_, and _there_". This is in contrast with the
acessibility of a thread local, which is the current thread and (if
it's an `InheritableThreadLocal`) any children created by the curret
thread.

Here we propose a mechanism to associate a value with a name on entry
to a scope, and automatically (securely, predictably) remove that
association when the scope exits. We call such things scope locals.

## Non-Goals

It is not a goal to change the Java Programming Language.

This JEP is only concerned with associating local names and values. It
doesn't attempt to replace, for example, try-with-resources. It
doesn't perform cleanup operations when a scope ends, and isn't
related to a C++-style destructor.

It is not a goal to force virtual threads to use scope locals instead
of thread local variables. While we expect scope locals to be a better
fit in many or most cases., thread-local variables may still be useful
in some contexts,

## Description

In Java, the scope of a declaration -- the association of a name with
an entity such as a variable -- is the region of the program within
which the entity declared by the declaration can be referred to using
a simple name.

It's usual to say that Java is _lexically scoped_, in the sense that
variables (and other entities) can be referred to by a simple name
only within the region of source code in which they're declared. Here,
for example, `x` is declared in a class `Example`, and may be referred
to as `x` in any method declared in the same class. We say, therefore,
that the scope of `x` is the class `Example`:

```
class Example {
    private int x = 5;

    void printIt() {
        System.out.println(x);
    }
}
```

Java also has nested variable scopes inside a method. Here, the
variable `x` is declared in the outermost scope of the method
`printIt()`, and the variable `i` is declared in an inner scope, that
of a `for` loop. In this example, the scope of `x` is that of the
method `printIt()` from the point where `x` is declared to the closing
brace of the method:

```
class Example {

    void work() {
        int x = 5;
        for (int i = 0; i < 10; i++) {
            moreWork(x * i);
        }
        // It is is possible to refer to x at this point,
        // but not i, because the scope of i has ended.
        System.out.println(x);
    }

    void moreWork(int n) {
        System.out.println(n);
    }
}
```

(From the Java Language Specification: the scope of a local variable
declaration in a block is the rest of the block in which the
declaration appears, starting with its own initializer and including
any further declarators to the right in the local variable declaration
statement.
https://docs.oracle.com/javase/specs/jls/se17/html/jls-6.html#jls-6.3)

With lexical scoping, the region of the program within which an entity
is accessible by name is a region of the source code.

With dynamic scoping, the region of the program within which an entity
is accessible by name is a duration in time, from the point at which
the name is bound to the entity to the point at which the binding is
removed. For example, if x in the previous code were dynamically
scoped then the body of `moreWork` could access `x` despite `x` not
being passed to it, and `x` not being a field of `Example`.

The goal of this JEP is to support dynamically scoped values, which
may be referred to anywhere within the dynamic scope that binds a
value to a scope local. We'll define a `ScopeLocal` class which does
this as a library facility, like so:

```
class Example {

    // Define a scope local that may be bound to Integer values:
    static final ScopeLocal<Integer> X = ScopeLocal.newInstance();

    void printIt() {
        // Bind X to the value 5, then run anotherMethod()
        ScopeLocal.where(X, 5).run(() -> anotherMethod());
    }

    void anotherMethod() {
        for (int i = 0; i < 10; i++) {
            System.out.println(X.get() * i);
        }
        System.out.println(X.get());
    }
}
```

A `ScopeLocal` instance such as `X` above is a key that is used to
look up a value in the current thread. Despite `X` being declared
static in class `Example`, there is _not_ exactly one incarnation of
the field shared across all instances of Foo; instead, there are
_multiple_ incarnations of the field, one per thread, and the
incarnation of `X` that is used when code performs a field access
(`X.get()`) depends on the thread which is executing the code.

A scope local acquires (we say: _is bound to_) a value on entry to a
scope; when that scope terminates, the previous value (or none) is
restored. In this case, the scope of `X`'s binding is the duration of
the Lambda invoked by `run()`.

One useful way to think of scope locals is as invisible, effectively
final, parameters that are passed through every method invocation.
These parameters will be accessible within the dynamic scope of a
scope local's binding operation (the set of methods invoked within the
binding scope, and any methods invoked transitively by them).

```
  // Declare scope locals x and y
  static final ScopeLocal<MyType> x = ScopeLocal.newInstance();
  static final ScopeLocal<MyType> y = ScopeLocal.newInstance();

  ... much later

  {
    ScopeLocal.where(x, expr1)
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
and any methods called by them, comprise the dynamic scope of `run()`.

Please note that the code that uses `x.get()` and `y.get()` may be a
very long way away, for example in a callback somewhere. Imagine a
complex system, with many intervening method calls, between the point
where a scope local is bound to a value and the point where that value
is retrieved. Like so:

```
    void run1() {
        ScopeLocal.where(x, new MyType("Jill"))
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
        ScopeLocal.where(x, new MyType("Helena"))
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

In summary, scope locals have the following properties:

* _Locally-defined extent_: The values of `x` and `y` are only bound
  while the `run()` method is executing.
* _Immutability_ There is no `ScopeLocal.set()` method: scope locals,
  once bound, are effectively final.
* _Simple and fast code_: In most cases a scope-local `x.get()` is as
  fast as a local variable `x`. This is true regardless of how far
  away `x.get()` is from the point that the scope local `x` is bound.
* _Structure_: These properties also also make it easier for a reader
  to reason about programs, in much the same way that declaring a
  field of a variable final does.

## Uses of scope locals

### Hidden parameters for callbacks

You may want to invoke a method `X` in a library that later calls back
into your code. In your callback you to do some logging, so you need a
way to find the logger to use. However, `X` provides no way to pass a
reference to a logger through their code into your callback. You don't
want to turn logging on all the time, just when certain parts of the
programmer are executing.

Here's a simple example of how you'd using a scope local to add
context-sensitive logging to an application. Let's assume that you
want to log some events, but only for certain places when your
application is running.

First, declare an interface that is invoked when a loggable event
occurs, and a `ScopeLocal` instance that will refer to one:

```
    interface MyLogger {
        public void log(String s);
    }

    private static final ScopeLocal<MyLogger> SL_LOGGER
            = ScopeLocal.newInstance(MyLogger.class);
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
        ScopeLocal.where(SL_LOGGER, (s) -> LOGGER.severe(s)).run(this::exec);
```

### Securely passing credentials

The following example uses a scope local to make credentials available
to callees.

```
    private static final ScopeLocal<Credentials> CREDENTIALS = ScopeLocal.newInstance();

    Credentials creds = ...
    ScopeLocal.where(CREDENTIALS, creds).run(() -> {
        :
        Connection connection = connectDatabase();
        :
    });

    Connection connectDatabase() {
        if (CREDENTIALS.get().s != "MySecret") {
            throw new SecurityException("No dice, kid!");
        }
        return new Connection();
    }
```

### Shadowing

It is sometimes useful to be able to re-bind an already-bound scope
local. For example, a privileged method may need to connect to a
database with a less-privileged set of credentials, like so:

```
      Credentials creds = CREDENTIALS.get();
      creds = creds.withLowerTrust();
      ScopeLocal.where(CREDENTIALS, creds).run(() -> {
        :
        Connection connection = connectDatabase();
        :
      });
```

This "shadowing" only extends until the end of the dynamic
scope of the lambda above.

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
        // RENDER_CTX is a scope local that refers to a context
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
        return ScopeLocal.where(RENDER_CTX, this).call(r);
    }

    // ...
}
```

We'd like to support as many of these use cases as we can, but only if
the basic properties of effective finality and re-entrancy can be
guaranteed.

## Replacing some uses of `ThreadLocal` with scope locals

Because scope locals have a well-defined lifetime, the block in which
they were bound, they can never be used in a way that's identical to
the way thread-local variables are used. Therefore, some code changes
will be required to switch from thread locals to scope locals.

Please note that, for the sake of brevity, these are simple
examples. In some of these examples a simple refactoring would make
the use of scope locals unnecessary, but that would be far more
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

This kind of thing is well suited to the use of a scope local, but
because scope local bindings are always removed at the end of the
scope in which they were bound, you must run an entire operation from
a binding scope. So, you won't be able to do something like this
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
`UserTransaction.run()` binds a scope local then calls a lambda:

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

  private static final ScopeLocal<UserTransaction> TRANSACTION = ScopeLocal.newInstance();

  void run(Runnable action) {
    ... // initialize
    ScopeLocal.where(TRANSACTION, this).run(action);
    ... commit the transaction
  }

  ...
}
```

that is to say, run an entire operation in a scope local binding.

### Caches for Non-thread-safe objects that are expensive to create

These are problematic for scope locals, perhaps because caches are one
of the few use cases for which thread-local variables are ideally
suited. If you can create caches you are likely to need in an
outermost scope that would work, but it requires some structural changes.

### Hidden return values

This isn't difficult to do with scope locals: create an empty instance
of a container class in the outer scope, call some method, and the
callee method `set()`s the value in the container. However, while it's
pretty obvious how it do this, it's rather evil.

## API

There is more detail in the Javadoc for the API, at

http://people.redhat.com/~aph/loom_api/jdk.incubator.concurrent/jdk/incubator/concurrent/ScopeLocal.html

## Alternatives

It is possible to emulate many of the features of scope locals with
`ThreadLocal`s, albeit at some cost in memory footprint, runtime
security, and performance.

We have experimented with a modified version of `ThreadLocal` that
supports some of the characteristics of scope locals. However,
carrying the additional baggage of `ThreadLocal` results in an
implementation that is unduly burdensome, or an API that returns
`UnsupportedOperationException` for much core functionality, or
both. It is better, therefore, not to do that but to give scope locals
a separate identity from thread locals.
