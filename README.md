# Scope Locals

## Motivation

So you want to invoke a method `X` in a library which later calls back
into your code. In your callback you need some context, perhaps a
transaction ID or some `File` instances. However, `X` provides no way
to pass a reference through their code into your callback.  What are
you to do? Unfortunately, you're going to have to store the context
into what is, in effect, a global variable. A `ThreadLocal` instance
is a slight improvement on a global variable, in that you can have
multiple instances in a program, one per thread. However, this kind of
usage is still not re-entrant, so if `X` in turn calls back into your
method you'll be in trouble. And if the callback happens on a
different thread, this approach breaks down altogether.

The core idea of scope locals is to support something like a "special
variable" in Common Lisp. This is a dynamically-scoped variable, which
acquires a value on entry to a lexical scope; when that scope
terminates, the previous value (or none) is restored. We also support
inheritance for scope locals, so that parallel constructs can set a
value in the parent before threads start.

One useful way to think of scope locals is as invisible, effectively
final, parameters that are passed to every method. These parameters
will be accessible within the "dynamic scope" of a scope local's
binding operation (i.e. the set of methods invoked within the binding
scope, and any methods invoked transitively by them.) They are
guaranteed to be re-entrant.

## Dynamically-scoped values

The need for scope locals arose from Project Loom. Loom enables a new
style of Java programming, where threads are not a scarce resource to
be carefully managed by thread pools but are much more abundant,
limited only by memory. To allow us to create large numbers of threads
&mdash; potentially millions &mdash; we'll need to make all of the
per-thread structures scale well.

Thread-local variables, and in particular inheritable thread locals,
are a pain point in the design of Loom. When a new `Thread` instance
is created, its parent's set of inheritable thread-local variables is
cloned. This is necessary because a thread's set of thread locals is,
by design, mutable, so it cannot be shared. So every child thread ends
up carrying a copy of its parents entire set of thread locals, whether
the child needs them or not.

(Note: in current Java it is possible on thread creation to opt out of
inheriting any thread-local variables, but that doesn't help if you
really need one of them.)

Ideally we'd like to have a zero-copy operation when creating threads,
so that inheriting context requires only the copying of a pointer from
the creating thread (the parent) into the new thread (the child). For
this to work, the inherited context must be immutable.

Scope locals also provide us with some other nice-to-have features, in
particular:

* Strong Typing. Whenever a scope local is bound to a value, its type
  is checked. This means that an error message is delivered at the
  point an error was made rather than later. Also, there are usually
  more instances of `get()` than there are bindings.
* Immutability. The value bound to a scope local cannot change within
  a method. (It can be re-bound in a callee, of which more later.)
* Well-defined extent. A scope local is bound to a value at the start
  of a scope and its previous value (or none) is always restored at
  the end.
* Optimization opportunities. These properties allow us to generate
  excellent code. In many cases a scope-local `get()` is as fast as a
  local variable.

### Some examples

```
  // Declare scope locals x and y
  static final ScopeLocal<MyType> x = ScopeLocal.forType(MyType.class);
  static final ScopeLocal<MyType> y = ScopeLocal.forType(MyType.class);

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

(Note 1: the declarations of scope locals `x` and `y` have an explicit
type parameter. This allows us to make a strong guarantee that if any
attempt is made to bind an object of the wrong type to a scope local
we'll immediately throw a ClassCastException.)

(Note 2: `x` and `y` have the usual Java access modifiers. Even though
a scope local is implicitly passed to every method in its dynamic
scope, a method will only be able to use get() if that scope local's
name is accessible to the method. So, sensitive security information
can be passed through a stack of non-privileged invocations.)

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
            throw new SecurityException("No dice, son");
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

This "shadowing" only extends until the end of the end of the dynamic
scope of the lambda above.
 
(Note: This code example assumes that `CREDENTIALS` is already bound
to a highly-privileged set of credentials.)

### Inheritance

Scope locals are either inheritable or non-inheritable. The
inheritability of a scope local is determined by its declaration, like
so:

```
  static final ScopeLocal<MyType> x = ScopeLocal.inheritableForType(MyType.class);
```

Whenever `Thread` instances (virtual or not) are created, the set of
currently-bound inheritable scope locals in the parent thread is
automatically inherited by each child thread:

```
    private static final ScopeLocal<Credentials> CREDENTIALS 
        = ScopeLocal.inheritableForType(Credentials.class);

    void multipleThreadExample() {
        Credentials creds = new Credentials("MySecret");
        ScopeLocal.where(CREDENTIALS, creds).run(() -> {
            for (int i = 0; i < 10; i++) {
                virtualThreadExecutor.submit(() -> {
                    // ...
                    Connection connection = connectDatabase();
                    // ...
                });
            }
        });
    }
```


In addition, a `Snapshot()` operation that captures the current set
of inheritable scope locals is provided. This allows context
information to be shared with asynchronous computations.

For example, a version of `ExecutorService.submit(Callable c)` that
captures the current set of inheritable scope locals looks like this:

```
    public <V> Future<V> submitWithCurrentSnapshot(ExecutorService executor, Callable<V> aCallable) {
        var s = ScopeLocal.snapshot();
        return executor.submit(() -> ScopeLocal.callWithSnapshot(aCallable, s));
    }
```

In this example, the `snapshot()` operation captures the state of
bound scope locals so that they can be retrieved by any code in the
dynamic scope of the `submit()` method, whichever thread it's run
on. These bound values are available for use even if the parent
(i.e. the one that invoked `submitWithCurrentSnapshot()` has
terminated.

## Optimization

Scope locals have some strongly-defined properties. These allow us to
generate excellent code.

* The bound value of a scope local is immutable within a method. It
  may be re-bound in a callee, but we know that when the callee
  terminates the scope local's value will have been restored. For that
  reason, we can hoist the value of a scope local into a register at
  the start of a mathod. Repeated uses of a scope local can be as fast
  as a local variable.

* We don't have to check the type of a scope local every time we
  invoke `get()` because we know that its type was checked earlier.

## API

There is more detail in the Javadoc for the API, at

http://people.redhat.com/~aph/loom-api/api/java.base/java/lang/ScopeLocal.html
