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

