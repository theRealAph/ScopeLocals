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

## New Stuff

The run() method is the bottom-most frame of the extent in which the
extent local variable is defined.

Clearly, extent local variables are a constrained version of thread
locals. Once set, a thread local variable is defined for the entie
lifetime of its thread. an extent local variable is defined for its
extent.

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

