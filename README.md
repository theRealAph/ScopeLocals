# Scope Locals

## Summary

Define a standard, lightweight way to pass contextual information from
a caller to callees in the currently executing thread, and in some
cases to a group of threads.

## Motivation

It is common in Java programs to need to know something about the
context in which a method is running. For example, it might be
necessary to know a transaction context for the currently-executing
thread for the duration of an EJB call. Or it might be useful for an
API to be able to check that a caller has appropriate permissions to
perform an action.

If you have some information that is specific to a thread (or a group
of threads) your choices are to pass that information to every method
that may need it, or to associate that information with a thread,
which usually means a `ThreadLocal`.

`ThreadLocal` has some disadvantages when used for this purpose. It is
rather heavyweight, requiring every thread to have its own
independently initialized (?) copy of the variable, and to keep a
table of these copies. Also, while is is possible to `remove()` a
`ThreadLocal` from a thread's table it's common for a programmer to
forget to do so: so much so that an entry in a `ThreadLocal` map is a
weak reference in order not to leak memory when a `ThreadLocal` key
becomes unreachable.

The need for scope locals arose from Project Loom, where threads are
lightweight and numerous.

When a new `Thread` instance is created, its parent's set of
inheritable thread-local variables is cloned. This is necessary
because a thread's set of thread locals is, by design, mutable, so it
cannot be shared between threads. Every child thread ends up carrying
a local copy of its parent's entire set of (inheritable) thread
locals, whether the child needs them or not.

We'd like to have a feature that allows per-thread context information
to be inherited by a thread without an expensive cloning operation.
Some kind of immutable data structure fits this need, because the
inheriting thread needs only to copy a reference to its parent's set
of values.

Here we propose a mechanism to associate a value with a name on entry
to a scope, and automatically (securely, predictably) remove that
association when the scope exits. We call such things scope locals.

## Non-Goals

This JEP is only concerned with associating local names and values. It
doesn't attempt to replace, for example, try-with-resources. It
doesn't perform cleanup operations when a scope ends, and isn't
related to a C++-style destructor.

It is not a goal to replace existing usages of thread-local variables
without code changes. Thread-local variables may still be useful in
some contexts, but we expect scope locals to be a better fit in many
or most cases.

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
of a `for` loop. In this example, th scope of `x` is that of the
method `printIt()` from the point where `x` is declared to the closing
brace of the method:

```
class Example {

    void printIt() {
        int x = 5;
        for (int i = 0; i < 10; i++) {
            System.out.println(x * i);
        }
        // It is is possible to refer to x at this point,
        // but not i, because the scope of i has ended.
        System.out.println(x);
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
removed.

The core idea of scope locals is to support dynamically scoped values,
which may be referred to anywhere within the dynamic scope that binds
a value to a scope local. If we were to imagine a variant of Java
which supported dynamically scoped values as a built-in feature, it
might look like this:

```
class Example {

    void anotherMethod() {
        for (int i = 0; i < 10; i++) {
            System.out.println(x * i);
        }
        System.out.println(x);
    }

    void printIt() {
        dynamic final int x = 5;
        anotherMethod();
    }
}
```

Although some programming languages support something like this, we
don't propose to add it to the Java language. Instead we'd like to
define a `ScopeLocal` class which does something similar, but with a
library facility rather than a language feature, and with a formal
declaration for `X`:

```
class Example {

    // Define a scope local that may be bound to Integer values:
    static final ScopeLocal<Integer> X = ScopeLocal.newInstance();

    void anotherMethod() {
        for (int i = 0; i < 10; i++) {
            System.out.println(X.get() * i);
        }
        System.out.println(X.get());
    }

    void printIt() {
        // Bind X to the value 5, then run anotherMethod()
        ScopeLocal.where(X, 5).run(() -> anotherMethod());  
    }
}
```

A scope local acquires (we say: _is bound to_) a value on entry to a
scope; when that scope terminates, the previous value (or none) is
restored. In this case, the scope of `X`'s binding is the duration of
the Lambda invoked by `run()`.

Note that references to `X` do not have to be in methods of the same
class in which `X` is declared:

```
package tests;

public class ScopeLocalDeclarations {
    public static final ScopeLocal<Integer> X = ScopeLocal.newInstance();
}
```

```
import static tests.ScopeLocalDeclarations.X;

public class Example1 {
    void printIt() {
        // Bind X to the value 5, then run anotherMethod()
        ScopeLocal.where(X, 5).run(() -> AnotherClass.anotherMethod());
    }
}

class AnotherClass {
    static void anotherMethod() {
        for (int i = 0; i < 10; i++) {
            System.out.println(X.get() * i);
        }
        System.out.println(X.get());
    }
}
```

One useful way to think of scope locals is as invisible, effectively
final, parameters that are passed through every method invocation.
These parameters will be accessible within the dynamic scope of a
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

In this example, `run()` is said to "bind" `x` and `y` to the results
of evaluating `expr1` and `expr2` respectively. While the method
`run()` is executing, any calls to `x.get()` and `y.get()` return the
values that have been bound to them. The methods called from `run()`,
and any methods called by them, comprise the dynamic scope of `run()`.

* The bound values of `x` and `y` are only bound (?) while the `run()`
  method is executing.
* There is no `ScopeLocal.set()` method: scope locals, once bound, are
  effectively final.
* These properties allow us to generate good code. In most cases a
  scope-local `x.get()` is as fast as a local variable `x`. This is
  true regardless of how far away `x.get()` is from the point that the
  scope local `x` is bound.
* These properties also also make it easier for a reader to reason
  about programs, in much the same way that declaring a field of a
  variable final does.

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
of a current context, in this case a graphics rendering context:

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

We'd like to support as many of these use cases as we can, but only if
the basic properties of effective finality and re-entrancy can be
guaranteed.

## Replacing some uses of `ThreadLocal` with scope locals

Because scope locals have a well-defined lifetime, the block in which
they were bound, they can never be syntactically (or structurally)
compatible with thread-local variables. Therefore, some code changes
will be required to switch from thread locals to scope locals.

Please note that, for the sake of brevity, these are simple
examples. In some of these examples a simple refactoring would make
the use of scope locals unnecessary, but that would be far more
difficult in a more complex scenario with multiple libraries of
separate authorship.

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
