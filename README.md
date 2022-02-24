# Scope Locals

## Summary

Enhance the Java API with scope locals, which are dynamically scoped,
effectively final, local values.

## Non-Goals

This JEP is only concerned with associating local names and values. It
doesn't attempt to replace, for example, try-with-resources. It
doesn't perform cleanup operations when a scope ends, and isn't
related to a C++-style destructor.

It is not a goal to replace existing usages of thread-local variables
without code changes. Thread-local variables may still be useful in
some contexts, but we expect scope locals to be a better fit in many
or most cases.

## Motivation

`ScopeLocal` is in essence a redesign of `ThreadLocal` with a more
modern approach. This `ScopeLocal` JEP presents a new construct that
is more lightweight, predictable and reliable than `ThreadLocal`, and
will allow better code to be generated.

`ThreadLocal`s are, more or less, thread-confined global
variables. Once set, a `ThreadLocal`'s value persists until it is
`set()` to a different value or it is removed altogether by calling
`remove()`.

In contrast,

* A scope local is bound to a value at the start of a "scope"
  (i.e. the method that binds it; more below) and its previous value
  (or none) is always restored at the end of that scope.

* Scope locals have values: they are not variables, so there is no
  `set()` method.

* These properties allow us to generate good code. In many or most
  cases a scope-local `get()` will be as fast as a local variable.

* We also support thread inheritance for scope locals, so that the
  bound values of inheritable scope locals are captured when threads
  are created, in some contexts.

## Description

The core idea of scope locals is to support something like a "special
variable" in Common Lisp. This is a dynamically scoped variable, which
acquires a value on entry to a lexical scope; when that scope
terminates, the previous value (or none) is restored. However, unlike
LISP special variables, for Java we don't want our scope locals to
have a `set()` method.

One useful way to think of scope locals is as invisible, effectively
final, parameters that are passed through every method invocation.
These parameters will be accessible within the "dynamic scope" of a
scope local's binding operation (the set of methods invoked within the
binding scope, and any methods invoked transitively by them). They are
guaranteed to be re-entrant &mdash; when used correctly.

```
  // Declare scope locals x and y
  static final ScopeLocal<MyType> x = ScopeLocal.newInstance();
  static final ScopeLocal<MyType> y = ScopeLocal.newInstance();

  ... much later

  {
    ScopeLocal.where(x, expr1)
              .where(y, expr2)
              .run(() -> ... code that uses x.get() and y.get() ...);
  }
```


* The bound values of `x` and `y` are only bound (?) while the `run()`
  method is executing.
* The value bound to a scope local cannot change within a method
  because there is no `ScopeLocal.set()` method: scope locals, once
  bound, are effectively final.
* These properties allow us to generate good code. In most cases a
  scope-local `x.get()` is as fast as a local variable `x`. This is
  true regardless of how far away `x.get()` is from the point that the
  scope local `x` is bound.
* These also also make it easier for a reader to reason about
  programs, in much the same way that declaring a field of a variable
  final does.

In this example, `run()` is said to "bind" `x` and `y` to the results
of evaluating `expr1` and `expr2` respectively. While the method
`run()` is executing, any calls to `x.get()` and `y.get()` return the
values that have been bound to them. The methods called from `run()`,
and any methods called by them, comprise the dynamic scope of `run()`.
Because scope locals are effectively final, there is no equivalent of
the `ThreadLocal.set()` method.

Please note that the code that uses `x.get()` and `y.get()` may be a
very long way away, for example in a callback somewhere. Imagine a
complex system, with many intervening method calls, between the point
where a scope local is bound to a value and the point where that value
is retrieved.

## Scope locals in threads

The need for scope locals arose from Project Loom, where threads are
lightweight and numerous.

At present, when a new `Thread` instance is created, its parent's set
of inheritable thread-local variables is cloned. This is necessary
because a thread's set of thread locals is, by design, mutable, so it
cannot be shared between threads. Every child thread ends up carrying
a local copy of its parent's entire set of (inheritable) thread
locals, whether the child needs them or not.

We'd like to have a feature that allows per-thread context information
to be inherited by a thread without an expensive cloning operation.
Some kind of immutable data structure fits this need, because the
inheriting thread needs only to copy a reference to its parent's set
of values.

## Uses of scope locals

### Hidden parameters for callbacks

You may want to invoke a method `X` in a library that later calls back
into your code. In your callback you need some context, perhaps a
transaction ID or some `File` instances. However, `X` provides no way
to pass a reference through their code into your callback. Set a
thread-local variable, then invoke `X`, then carefully `remove()` the
thread-local variable. This usage isn't ideal for thread locals
because it's not at all re-entrant: if `X` is recursively called via
your callback, it'll overwrite your already-set thread-local variable.

### Thread locals and recursion

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

### Contexts of many kinds: the notion of a "current context".

    Java Concurrency in Practice:

    "... containers associate a transaction context with an executing
    thread for the duration of an EJB call. This is easily implemented
    using a static Thread-Local holding the transaction context: when
    framework code needs to determine what transaction is currently
    running, it fetches the transaction context from this
    ThreadLocal."

Another example occurs in graphics, where there is a drawing context.

### Caches for Non-thread-safe objects that are expensive to create, e.g. `SimpleDateFormat`.

Some utility classes such as `SimpleDateFormat` are mutable and they
are not thread-safe. The `SimpleDateFormat` API specification suggests
that a new instance should be created for each thread.

### Hidden return values

It is possible to call into a method which returns a value by setting
a thread-local variable, possibly deep in a stack of method
calls. This isn't common and is unstructured (at best) but we don't
doubt that some programs do it.

### Our goals

We'd like to support as many of these use cases as we can, but only if
the basic properties of effective finality and re-entrancy can be
guaranteed.

### Some examples of scope local usage

These examples are necesarily simple for the sake of brevity, and in
many cases you wouldn't need a thread local or a scope local.

The following example uses a scope local to make credentials available
to callees.

```
    private static final ScopeLocal<Credentials> CREDENTIALS = ScopeLocal.forType(Credentials.class);

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

We also provide a shortcut for the case where only a single scope
local is set:

```
   {
       ScopeLocal.where(x, expr1, (() -> 
           ... code that uses x.get() ...);
   }

```

This is a natural fit for a `record` when you need to share a group of
values, for example:


```
    record Credentials(int userId, String password) {}
    static final ScopeLocal<Credentials> CREDENTIALS = ScopeLocal.forType(Credentials.class);

    {
        ScopeLocal.where(position,
            new Credentials(userDB.getCurrentUID(), console.askForPassword()),
                ... code that, somewhere deep, uses CREDENTIALS to connect ... ;
    }
```

We recommend this form when multiple values are shared with the same
consumer because it's likely to be more efficent and it's clear to the
reader what is intended.

(Note: `x` and `y` have the usual Java access modifiers. Even though
a scope local is implicitly passed to every method in its dynamic
scope, a method will only be able to use `get()` if that scope local's
name is accessible to the method. So, sensitive security information
can be passed through a stack of non-privileged invocations.)

## Scope locals in more detail

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

### Thread Inheritance

We intend to support inheritance of scope local bindings by some
subtasks, in particular `Thread` instances in a Structured Concurrency
context. This is described in the Structured Concurrency JEP
https://openjdk.java.net/jeps/8277129.

Here's an example that uses a `StructuredTaskScope` (from Project
Loom) to fork a number of virtual threads. A scope-local called
`invocationCounter` used to count the number of times that
`someMethod()` is invoked. This works because scope local bindings are
inherited by threads forked by a `StructuredTaskScope`.

```
    public void exampleWithInvocationCounter() throws Exception {
        ScopeLocal.where(invocationCounter, new AtomicInteger())
                .run(() -> {
                    forkSomeThreads();
                    System.out.println("" + invocationCounter.get() + " invocations");
                });
    }

    public void forkSomeThreads() {
        try (var scope = new StructuredTaskScope(null, threadFactory)) {
            for (int i = 0; i < 100; i++) {
                scope.fork(() -> someMethod());
            }
            scope.join();
        } catch (Exception e) {
        }
    }

    Void someMethod() {
        // Count the number of times this method is invoked
        if (invocationCounter.isBound()) {
            invocationCounter.get().getAndIncrement();
        }

        // ... more code here

        return null;
    }

    static final ScopeLocal<AtomicInteger> invocationCounter = ScopeLocal.newInstance();
    private ThreadFactory threadFactory = Thread.ofVirtual().factory();
```

## What works with scope locals &mdash; and what doesn't

Because scope locals have a well-defined extent, the block in which
they were bound, they can never be syntactically (or even
structurally) compatible with thread-local variables. Therefore, some
code changes will be required to switch from thread locals to scope
locals.

Please note that, for the sake of brevity, these are simple examples. In
some cases a simple refactoring would make the use of scope locals
unnecessary, but that would be far more difficult in a more complex
scenario with multiple libraries of separate authorship.

### The idea of a "current context"

These will work well, but because scope local bindings are always
removed at the end of the scope in which they were bound, you must run
an entire operation from that binding scope. So, you won't be able to
do something like this example, which has a `ThreadLocal` embedded in
a `DatabaseContext`:

```
try (final DatabaseContext ctx = new DatabaseContext()) {
  // Within this block, every use of DatabaseContext.current()
  // returns the current context.
  doSomething(ctx);
}
```

instead you'll have to do something like this, where
`DatabaseContext.run()` binds a thread local then calls a lambda:

```
DatabaseContext.run(() -> doSomething());
```

that is to say, run an entire operation in the outer scope of the
scope local binding.

### Recursion detection and counting

This case will work well too, but because scope locals have exactly
the properties required, you won't need a recursion counter to know
when to `remove()` the binding.

The following is an example that combines recursion detection and a
current context.

Firstly, `ThreadLocal` style:

```
    public final RendererContext getRendererContext() {
        // ctxTL is a thread-local variable that contains a context
        RendererContext ctx = ctxTL.get();
        if (ref != null) {
            // Check reentrance:
            if (ctx.usage == USAGE_TL_INACTIVE) {
               ctx.usage = USAGE_TL_IN_USE;
               return ctx;
            }
        }
        ctx = newContext();
        ctxTL.set(ctx);
        return ctx;
    }

   // called from here ...

    final RendererContext rdrCtx = getRendererContext();
    try {
        final Path2D.Double p2d = rdrCtx.getPath2D();
        strokeTo(rdrCtx, p2d, ...);
        return new Path2D.Double(p2d);
    } catch {
        ...
    } finally {
        // recycle the RendererContext instance
        returnRendererContext(rdrCtx);
    }
```

might turn into something like this with scope locals:

```
    public final RendererContext getRendererContext() {
        // ctxSL is a scope local that refers to a context
        if (if ctxSL.isBound()) {
            RendererContext ctx = ctxSL.get();
            // Check reentrance:
            if (ctx.usage == USAGE_TL_INACTIVE) {
               ctx.usage = USAGE_TL_IN_USE;
               return ctx;
            }
        }
        return newContext();
    }


   // called from here ...

    final RendererContext rdrCtx = getRendererContext();
    try {
        return rdCtx.call( () -> {
            final Path2D.Double p2d = rdrCtx.getPath2D();
            strokeTo(rdrCtx, p2d, ...);
            return new Path2D.Double(p2d);
        });
     } catch {
         ...
     } finally {
        // recycle the RendererContext instance
        returnRendererContext(rdrCtx);
     }

```
    
Where `RendererContext.call()` is defined like this:
    
```

    // Call r with ctxSL bound to this RendererContext
    T call(Callable<T> r) throws Exception {
        return ScopeLocal.where(ctxSL, this).call(r);
    }
```

### Hidden parameters for callbacks

These should work well, with some code changes.

Here's a simple example of using a scope local to add
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

You can do something similar with a thread-local variable, but in a
different form:

```
    interface MyLogger {
        public void log(String s);
    }
    private static final ThreadLocal<MyLogger> TL_LOGGER
            = ThreadLocal.withInitial(() -> NULL_LOGGER);

    void someMethodDeepInALibrary() {
        // ...
        TL_LOGGER.get().log("Toto, I've a feeling we're not in Kansas any more.");
        // ...
    }

    // ... called from

        try {
            TL_LOGGER.set((s) -> LOGGER.severe(s));
            this.exec();
        } finally {
            TL_LOGGER.set(NULL_LOGGER);
        }
```

Note that this isn't quite the same as the scope local example because
it's not re-entrant: if `TL_CALLBACK` was set when this code was
executed its setting would be lost. The closest thread-local equivalent of the
scope-local example above might be something like

```
        var prev = TL_LOGGER.get();
        try {
            TL_LOGGER.set((s) -> LOGGER.severe(s));
            this.exec();
        } finally {
            TL_LOGGER.set(prev);
        }
```

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

### Optimization

Scope locals have some strongly-defined properties. These can allow us
to generate excellent code for `get()` and inheritance.

* The bound value of a scope local is effectively final within a
  method. It may be re-bound in a callee, but we know that when the
  callee terminates the scope local's value will have been
  restored. For that reason, we can hoist the value of a scope local
  into a register at the start of a method. Repeated uses of a scope
  local can be as fast as using a local variable.
    

### API

There is more detail in the Javadoc for the API, at

http://people.redhat.com/~aph/loom-api/api/java.base/java/lang/ScopeLocal.html

## Alternatives

It is possible to emulate many of the features of scope locals with
`ThreadLocal`s, albeit at some cost in memory footprint, runtime
security, and performance. However, inheritable `ThreadLocal`s don't
really work with thread pools, and run the risk of leaking information
between unrelated tasks.

We have experimented with a modified version of `ThreadLocal` that
supports some of the characteristics of scope locals. However,
carrying the additional baggage of `ThreadLocal` results in an
implementation that is unduly burdensome, or an API that returns
`UnsupportedOperationException` for much core functionality, or
both. It is better, therefore, not to do that but to give scope locals
a separate identity from thread locals.
