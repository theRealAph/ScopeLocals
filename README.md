Summary
-------

Introduce _scoped values_, which enable the sharing of immutable data within and across threads. They are preferred to thread-local variables, especially when using large numbers of virtual threads. This is a [preview API](https://openjdk.org/jeps/12).


History
-------

Scoped Values incubated in JDK 20 via [JEP 429](https://openjdk.org/jeps/429). In JDK&nbsp;21 this feature is no longer incubating; instead, it is a [preview API](https://openjdk.org/jeps/12).


Goals
-----

- *Ease of use* — Provide a programming model to share data both within a thread and with child threads, so as to simplify reasoning about data flow.

- *Comprehensibility* — Make the lifetime of shared data visible from the syntactic structure of code.

- *Robustness* — Ensure that data shared by a caller can be retrieved only by legitimate callees.

- *Performance* — Treat shared data as immutable so as to allow sharing by a large number of threads, and to enable runtime optimizations.


## Non-Goals

- It is not a goal to change the Java programming language.

- It is not a goal to require migration away from thread-local variables, or to deprecate the existing `ThreadLocal` API.


## Motivation

The functionality of any Java application or library is organized as a method call graph. Simple methods may only refer to constants or instance fields. However, most methods are parameterized, allowing a caller to share data with a called method by passing them as call arguments. Called methods may choose to pass this shared data on down the call chain. Arguments are always bound locally to a specific call and hence only share data within the context of a specific thread.

Most of the time method arguments are an effective and convenient way to share data. However, parameterization does not always scale well in large applications built using independently developed and maintained components. For example, a web framework might include a server component, implemented in the [thread-per-request style](https://openjdk.org/jeps/444#The-thread-per-request-style), and  a data access component, which handles persistence, connected via application-specific code.

The server and data access component might need to share an identity associated with each incoming request in order to regulate access to resources. However, it is normally the responsibility of the application request handler to initiate data access operations, such as opening a database connection. In order to share the identity using method arguments the framework must pass it to the application request handler call. The application code is then responsible for passing this argument on down through the application call chain to any point where it might call a data access component method.

There are several reasons why this can become a problem. The obvious issue is that application logic is now complicated by the need to receive and correctly pass on framework data it does not itself directly consume. When the call hierarchy and control flow in the server thread is large and complex this may require correctly routing arguments through many call chains. If instead the server thread tries to bypass the routing problem by storing the argument as object state then it must ensure that this state is only ever read by the correct thread and that it does not get overwritten or go stale before it is passed on to the data access component. Either way, the resulting code is now more brittle than it needs to be: if an update to the framework and data access component changes the way they share data then the application code may also need to be updated.

The diagram below depicts this problem scenario. The framework handles two requests, each in its own thread. Request handling flows upward, from the server component (`Server.serve(...)`) to user code (`Application.handle(...)`) to the data access component (`DBAccess.open()`). The data access component needs to share the `Identity` created by the server component in order to decide whether the thread is permitted to access the database:

- In Thread 1, a `CUSTOMER` identity created by the server component allows database access. The dashed line indicates the identity is to be shared with the data access component, which inspects it and proceeds to call `DBAccess.newConnection()`.

- In Thread 2, a `GUEST` identity created by the server component does not allow database access. The data access component inspects the identity, determines that the user code must not proceed, and throws an `InvalidIdentityException`.

<a name="Web-framework-example-Initial-extents"></a>
```
Thread 1                                 Thread 2
--------                                 --------
8. DBAccess.newConnection()              8. throw new InvalidIdentityException()
7. DBAccess.open() <----------+          7. DBAccess.open() <----------+
   ...                        |             ...                        |
5. Application.fetchOrder()   |          5. Application.fetchOrder()   |
   ...                Identity(CUSTOMER)    ...                  Identity(GUEST)
2. Application.handle(..)     |          2. Application.handle(..)     |
1. Server.serve(..) ----------+          1. Server.serve(..) ----------+
```

The correct `Identity` for each thread can be communicated from `Server.serve` to `DBAccess.open` by passing it as an argument through the chain of `Application` method calls. However, it would be cleaner, less prone to error and easier to maintain if there were some way for the two components to _directly_ share the value belonging to the current thread without involving the `Applicaton` code.

### Thread-local variables for sharing

Developers have traditionally used _thread-local variables_, introduced in Java 1.2, to help components share data without resorting to method arguments. A thread-local variable is an object of type [`ThreadLocal<T>`](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/lang/ThreadLocal.html). It is used to store a value of type `T`, written and read by calling instance methods `ThreadLocal.set(...)` and `ThreadLocal.get(...)`. Although this looks like it simply replicates the behaviour of an ordinary variable there is an important difference. A thread-local variable has multiple incarnations of its stored value, one per thread; the particular incarnation that is used depends on which thread calls its `get()` or `set(...)` methods. Code in one thread automatically reads and writes its incarnation, while code in another thread automatically reads and writes its own distinct incarnation. Typically, a thread-local variable is declared as a `final` `static` field so it can easily be reached from many components.

Here is an example of how the server component and the data access
component, both running in the same request-handling thread, can use a
thread-local variable to share an `Identity`. The server component
first declares a thread-local variable, `IDENTITY` (1). When
`Server.serve(...)` is executed in a request-handling thread, it
writes a suitable `Identity` to the thread-local variable (2), then
calls the `Application` handler. If and when user code calls `DBAccess.open()`, the
data access component reads the thread-local variable (3) to obtain
the `Identity` of the request-handling thread. The database access only
proceeds if the `Identity` is acceptable (4).

<a name="Web-framework-example-ThreadLocal-code"></a>
```
class Server {
    final static ThreadLocal<Identity> IDENTITY = new ThreadLocal<>();  // (1)

    void serve(Request request, Response response) {
        var level     = (request.isAuthorized() ? CUSTOMER : GUEST);
        var identity = new Identity(level);
        IDENTITY.set(identity);                                         // (2)
        Application.handle(request, response);
    }
}

class DBAccess {
    DBConnection open() {
        var identity = Server.IDENTITY.get();                           // (3)
        if (!identity.canOpen()) throw new InvalidIdentityException();
        return newConnection(...);                                        // (4)
    }
}
```

Using a thread-local variable avoids the need to pass an `Identity` as a method argument when the server component calls user code, and when user code calls the data access component. The thread-local variable serves as a kind of hidden method argument: A thread which calls `IDENTITY.set(...)` in `Server.serve(...)` and then `IDENTITY.get()` in `DBAccess.open()` will automatically see its own incarnation of the `IDENTITY` variable. In effect, the `ThreadLocal` field serves as a key that is used to look up a `Identity` value for the current thread.

In this example, the `IDENTITY` field has package access. This allows both framework components, declared in the same package, to access its value. The `IDENTITY` field is not accessible to `Application` classes.

### Problems with thread-local variables

Unfortunately, thread-local variables have numerous design flaws that are impossible to avoid:

- *Unconstrained mutability*  —  Every thread-local variable is mutable: Any code that can call the `get()` method of a thread-local variable can call the `set(...)` method of that variable at any time. The `ThreadLocal` API allows this in order to support a fully general model of communication, where data can flow in any direction between components. However, this can lead to spaghetti-like data flow, and to programs in which it is hard to discern which component updates shared state and in what order. The more common need, shown in the example above, is a simple one-way transmission of data from one component to others.

- *Unbounded lifetime* — Once a thread's incarnation of a thread-local variable is written via the `set(...)` method, the incarnation is retained for the lifetime of the thread, or until code in the thread calls the `remove()` method. Unfortunately, developers often forget to call `remove()`, so per-thread data is often retained for longer than necessary. In particular, if a thread pool is used, the value of a thread-local variable set in one task could, if not properly cleared, accidentally leak to an unrelated task. In addition, for programs that rely on the unconstrained mutability of thread-local variables, there may be no clear point at which it is safe for a thread to call `remove()`; this can cause a long-term memory leak, since per-thread data will not be garbage-collected until the thread exits. It would be better if the writing and reading of per-thread data occurred in a bounded period during execution of the thread, avoiding the possibility of leaks.

- *Expensive inheritance* — The overhead of thread-local variables may be worse when using large numbers of threads, because thread-local variables of a parent thread can be inherited by child threads. (A thread-local variable is not, in fact, local to one thread.) When a developer chooses to create a child thread that inherits thread-local variables, the child thread has to allocate storage for every thread-local variable previously written in the parent thread. This can add significant memory footprint. Child threads cannot share the storage used by the parent thread because thread-local variables are mutable, and the `ThreadLocal` API requires that mutation in one thread is not seen in other threads. This is unfortunate, because in practice child threads rarely call the `set(...)` method on their inherited thread-local variables.

### Toward lightweight sharing

The problems of thread-local variables have become more pressing with the availability of virtual threads ([JEP 444](https://openjdk.org/jeps/444)). Virtual threads are lightweight threads implemented by the JDK. Many virtual threads share the same operating-system thread, allowing for very large numbers of virtual threads. In addition to being plentiful, virtual threads are cheap enough to represent any concurrent unit of behavior. This means that a web framework can dedicate a new virtual thread to the task of handling a request and still be able to process thousands or millions of requests at once. In the ongoing example, the methods `Server.serve(...)`, `Application.handle(...)`, and `DBAccess.open()` would all execute in a new virtual thread for each incoming request.

It would obviously be useful for these methods to be able to share data whether they execute in virtual threads or traditional platform threads. Because virtual threads are instances of `Thread`, a virtual thread can have thread-local variables; in fact, the short-lived [non-pooled](https://openjdk.org/jeps/444#Do-not-pool-virtual-threads) nature of virtual threads makes the problem of long-term memory leaks, mentioned above, less acute. (Calling a thread-local variable's `remove()` method is unnecessary when a thread terminates quickly, since termination automatically removes its thread-local variables.) However, if each of a million virtual threads has mutable thread-local variables, the memory footprint may be significant.

In summary, thread-local variables have more complexity than is usually needed for sharing data, and significant costs that cannot be avoided. The Java Platform should provide a way to maintain immutable and inheritable per-thread data for thousands or millions of virtual threads. Because these per-thread variables would be immutable, their data could be shared by child threads efficiently. Further, the lifetime of these per-thread variables should be bounded: Any data shared via a per-thread variable should become unusable once the method that initially shared the data is finished.


## Description

A _scoped value_ allows data to be safely and efficiently shared between components in a large program without resorting to method arguments. It is a variable of type [`ScopedValue`](https://cr.openjdk.org/~alanb/sc/api/java.base/java/lang/ScopedValue.html). Typically, it is declared as a `final` `static` field so it can easily be reached from many components.

Like a thread-local variable, a scoped value has multiple incarnations, one per thread. The particular incarnation that is used depends on which thread calls its methods. Unlike a thread-local variable, a scoped value is written once and is then immutable, and is available only for a bounded period during execution of the thread.

A scoped value is used as shown below. Some code calls `ScopedValue.where(...)`, presenting a scoped value and the object to which it is to be bound. The call to `run(...)` _binds_ the scoped value, providing an incarnation that is specific to the current thread, and then executes the lambda expression passed as argument. During the lifetime of the `run(...)` call, the lambda expression, or any method called directly or indirectly from that expression, can read the scoped value via the value’s `get()` method. After the `run(...)` method finishes, the binding is destroyed.

```
final static ScopedValue<...> V = ScopedValue.newInstance();

// In some method
ScopedValue.where(V, <value>)
           .run(() -> { ... V.get() ... call methods ... });

// In a method called directly or indirectly from the lambda expression
... V.get() ...
```

The syntactic structure of the code delineates the period of time when a thread can read its incarnation of a scoped value. This bounded lifetime, combined with immutability, greatly simplifies reasoning about thread behavior. The one-way transmission of data from caller to callees — both direct and indirect — is obvious at a glance. There is no `set(...)` method that lets faraway code change the scoped value at any time. Immutability also helps performance: Reading a scoped value with `get()` is often as fast as reading a local variable, regardless of the stack distance between caller and callee.

### The meaning of "scoped"

The _scope_ of a thing is the space in which it lives — the extent or
range in which it can be used. For example, in the Java programming language, the
scope of a variable declaration is the space within the program text
where it is legal to refer to the variable with a simple name ([JLS 6.3](https://docs.oracle.com/javase/specs/jls/se19/html/jls-6.html#jls-6.3)). This kind of scope is more accurately called _lexical scope_ or _static
scope_, since the space where the variable is in scope can be
understood statically by looking for `{` and `}` characters in the program text.

Another kind of scope is called _dynamic scope_. The dynamic scope of a
thing refers to the parts of a program that can use the thing as the
program executes. This is the concept to which _scoped value_ appeals,
because binding a scoped value V in a `run(...)` method produces an
incarnation of V that is usable by certain parts of the program as it
executes, namely the methods invoked directly or indirectly by `run(...)`.
The unfolding execution of those methods defines a dynamic scope; the
incarnation is in scope during the execution of those methods, and
nowhere else.

###  Web framework example with scoped values

The framework code shown earlier can easily be rewritten to use a scoped value instead of a thread-local variable. At (1), the server component declares a scoped value instead of a thread-local variable. At (2), the server component calls `ScopedValue.where(...)` and `run(...)` instead of a thread-local variable's `set(...)` method.

<a name="Web-framework-example-ScopedValue-code"></a>
```
class Server {
    final static ScopedValue<Identity> IDENTITY
        = ScopedValue.newInstance();                                        // (1)

    void serve(Request request, Response response) {
        var level     = (request.isAdmin() ? CUSTOMER : GUEST);
        var identity = new Identity(level);
        ScopedValue.where(IDENTITY, identity)                             // (2)
                   .run(() -> Application.handle(request, response));
    }
}

class DBAccess {
    DBConnection open() {
        var identity = Server.IDENTITY.get();                             // (3)
        if (!identity.canOpen()) throw new  InvalidIdentityException();
        return newConnection(...);
    }
}
```

Together, `where(...)` and `run(...)` provide one-way sharing of data from the server component to the data access component. The scoped value passed to `where(...)` is bound to the corresponding object for the lifetime of the `run(...)` call, so `IDENTITY.get()` in any method called from `run(...)` will read that value. Accordingly, when `Server.serve(...)` calls `Application` code, and that `Application` code calls `DBAccess.open()`, the value read from the scoped value (3) is the value written by `Server.serve(...)` earlier in the thread.

The binding established by `run(...)` is usable only in code called from `run(...)`. If `IDENTITY.get()` appeared in
`Server.serve(...)` after the call to `run(...)`, an exception would be thrown because `IDENTITY` is no longer bound in
the thread.

As before, the framework relies on the language's access control to restrict access to its internal data: The `IDENTITY` field has package access, which allows the framework to share information internally between its two components. That information is both inaccessible to and hidden from user code.

###  Rebinding scoped values

The immutability of scoped values means that a caller can use a scoped value to reliably communicate a constant value to its callees in the same thread. However, there are occasions when one of the callees might need to use the same scoped value to communicate a different value to its own callees in the thread. The `ScopedValue` API allows a new binding to be established for nested calls.

As an example, consider a third component of the web framework: a logging component with a method `void log(Supplier<String> formatter)`. `Application` code passes a lambda expression to the `log(...)` method; if logging is enabled, the method calls `formatter.get()` to evaluate the lambda expression and then prints the result. Although the `Application` handler code may have permission to access the database, the lambda expression should not, since it only needs to format text. Accordingly, the scoped value that was initially bound in `Server.serve(...)` should be rebound to a guest `Identity` for the lifetime of `formatter.get()`:

<a name="Web-framework-example-Rebinding-extent"></a>
```
8. InvalidIdentityException()
7. DBAccess.open() <--------------------------+   X--------+
   ...                                        |            |
   ...                                  Identity(GUEST)    |
4. Supplier.get()                             |            |
3. Logger.log(() -> { DBAccess.open(); }) ----+      Identity(CUSTOMER)
2. Application.handle(..)                                  |
1. Server.serve(..) ---------------------------------------+
```

Here is the code for `log(...)` with rebinding. It obtains a guest `Identity` (1) and passes it as the new binding for the scoped value `IDENTITY` (2). For the lifetime of the invocation of `call` (3), `IDENTITY.get()` will read this new value. Thus, if the `Application` code passes an inapporpriate lambda expression to `log(...)` that tries to invoke  `DBAccess.open()`, the check in `DBAccess.open()` will read the guest `Identity` from `IDENTITY` and throw an `InvalidIdentityException`.

```
class Logger {
    void log(Supplier<String> formatter) {
        if (loggingEnabled) {
            var guest = Identity.createGuest();                        // (1)
            var message = ScopedValue.where(Server.IDENTITY, guest)    // (2)
                                     .call(() -> formatter.get());      // (3)
            write(logFile, "%s %s".format(timeStamp(), message));
        }
    }
}
```

(We here use `call(...)` instead of `run(...)` to invoke the formatter because the result of the lambda expression is needed.) The syntactic structure of `where(...)` and `call(...)` means that the rebinding is only visible in the nested dynamic scope introduced by `call(...)`. The body of `log(...)` cannot change the binding seen by that method itself but can change the binding seen by its callees, such as the `formatter.get(...)` method. This guarantees a bounded lifetime for sharing of the new value.

### Scoped values as capabilities

A scoped value provides a simple and robust implementation of a [capability]. The owner of a `ScopedValue` typically guards it in in a field with appropriately restricted access, such as a `private` `static` `final`. A `ScopedValue` object is typically not widely shared.

For example, suppose a framework allows running certain operations only in contexts where a user name is defined. This could be enforced as follows:

```
public class Framework {
    private Framework() {}
    private static final Framework INSTANCE = new Framework();
    public static void Framework instance() { return INSTANCE; }

    private static final ScopedValue<String> USER = ScopedValue.newInstance();
    
    public void runAsUser(String user, Runnable op) {
        ScopedValue.where(USER, user).run(op);
    }
    
    public void doOperation() {
        String user = USER.orElseThrow(() -> new IllegalStateException("User not set"));
        
        ...
    }
}
```

This ensures that only code running inside the `Runnable` passed to `runAsUser(...)` can call `doOperation()`.

[capability]: https://en.wikipedia.org/wiki/Capability-based_security

###  Inheriting scoped values

The web framework example dedicates a thread to handling each request, so the same thread can execute framework code from the server component, then user code from the application developer, then more framework code from the data access component. However, user code can exploit the lightweight nature of virtual threads by creating its own virtual threads and running its own code in them. These virtual threads will be child threads of the request-handling thread.

Data shared by a component running in the request-handling thread needs to be available to components running in child threads. Otherwise, when user code running in a child thread calls the data access component, that component — now also running in the child thread — will be unable to check the `Identity` shared by the server component running in the request-handling thread. To enable cross-thread sharing, scoped values can be inherited by child threads.

The preferred mechanism for user code to create virtual threads is the Structured Concurrency API ([JEP 453](https://openjdk.org/jeps/453)), specifically the class [`StructuredTaskScope`](https://cr.openjdk.org/~alanb/sc/api/java.base/java/util/concurrent/StructuredTaskScope.html). Scoped values in the parent thread are automatically inherited by child threads created with `StructuredTaskScope`. Code in a child thread can use bindings established for a scoped value in the parent thread with minimal overhead. Unlike with thread-local variables, there is no copying of a parent thread's scoped value bindings to the child thread.

Here is an example of scoped value inheritance occurring behind the scenes in user code. The `Server.serve(...)` method binds `IDENTITY` and calls `Application.handle(...)` just as before. However, the user code in `Application.handle` calls the `findUser()` and `fetchOrder()` methods concurrently, each in its own virtual threads, using `StructuredTaskScope.fork(...)` (1, 2). Each method calls the data access component (3), which, as before, consults the scoped value `IDENTITY` (4). Further details of the user code are not discussed here; see [JEP 453](https://openjdk.org/jeps/453#Description) for further information.

```
class Application {
    Response handle() throws ExecutionException, InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Supplier<String>  user  = scope.fork(() -> findUser());       // (1)
            Supplier <Integer> order = scope.fork(() -> fetchOrder());    // (2)
            scope.join().throwIfFailed();  // Wait for both forks
            return new Response(user.get(), order.get());
        }
    }

    String findUser() {
        ... DBAccess.open() ...                                           // (3)
    }
}

class Server {
    final static ScopedValue<Identity> IDENTITY = ScopedValue.newInstance();

    void serve(Request request, Response response) {
        var level     = (request.isAdmin() ? CUSTOMER : GUEST);
        var identity = new Identity(level);
        ScopedValue.where(IDENTITY, identity)
                   .run(() -> Application.handle(request, response));
    }
}

class DBAccess {
    DBConnection open() {
        var identity = Server.IDENTITY.get();                           // (4)
        if (!identity.canOpen()) throw new  InvalidIdentityException();
        return newConnection(...);
    }
}
```

`StructuredTaskScope.fork(...)` ensures that the binding of the scoped value `IDENTITY` made in the request-handling
thread — [when Server.serve(...) called ScopedValue.where(...)](#Web-framework-example-ScopedValue-code) — is
automatically visible to `IDENTITY.get()` in the child thread. The following diagram shows how the dynamic scope
of the binding is extended to all methods executed in the child thread:

<a name="Web-framework-example-Inheritance-extent"></a>
```
Thread 1                           Thread 2
--------                           --------
                                   8. DBAccess.newConnection()
                                   7. DBAccess.open() <----------+
                                   ...                           |
                                   ...                     Identity(CUSTOMER)
                                   4. Application.findUser()     |
3. StructuredTaskScope.fork(..)                                  |
2. Application.handle(..)                                        |
1. Server.serve(..) ---------------------------------------------+
```

The fork/join model offered by `StructuredTaskScope` means that the dynamic scope of the binding is still bounded
by the lifetime of the call to `ScopedValue.where(...).run(...)`. The `Identity` will remain in scope while the child
thread is running, and `scope.join()` ensures that child threads terminate before `run(...)` can return, destroying
the binding. This avoids the problem of unbounded lifetimes seen when using thread-local variables.

### Migrating to scoped values

Scoped values are likely to be useful and preferable in many scenarios where thread-local variables are used today. Beyond serving as hidden method arguments, scoped values may assist with:

- *Re-entrant code* — Sometimes it is desirable to detect recursion, perhaps because a framework is not re-entrant or because recursion must be limited in some way. A scoped value provides a way to do this: Set it up as usual, with `ScopedValue.where(...)` and `run(...)`, and then deep in the call stack, call `ScopedValue.isBound()` to check if it has a binding for the current thread. More elaborately, the scoped value can model a recursion counter by being repeatedly rebound.

- *Nested transactions* — Detecting recursion can also be useful in the case of flattened transactions: Any transaction started while a transaction is in progress becomes part of the outermost transaction.

- *Graphics contexts* — Another example occurs in graphics, where there is often a drawing context to be shared between parts of the program. Scoped values, because of their automatic cleanup and re-entrancy, are better suited to this than thread-local variables.

In general, we advise migration to scoped values when the purpose of a thread-local variable aligns with the goal of a scoped value: one-way transmission of unchanging data. If a codebase uses thread-local variables in a two-way fashion — where a callee deep in the call stack transmits data to a faraway caller via `ThreadLocal.set(...)` — or in a completely unstructured fashion, then migration is not an option.

There are a few scenarios that favor thread-local variables. An example is caching objects that are expensive to create and use, such as instances of `java.text.DateFormat`. Notoriously, a `DateFormat` object is mutable, so it cannot be shared between threads without synchronization. Giving each thread its own `DateFormat` object, via a thread-local variable that persists for the lifetime of the thread, is often a practical approach.

## Alternatives

It is possible to emulate many of the features of scoped values with thread-local variables, albeit at some cost in memory footprint, security, and performance.

We experimented with a modified version of `ThreadLocal` that supports some of the characteristics of scoped values. However, carrying the additional baggage of thread-local variables results in an implementation that is unduly burdensome, or an API that returns `UnsupportedOperationException` for much of its core functionality, or both. It is better, therefore, not to modify `ThreadLocal` but to introduce scoped values as an entirely separate concept.

Scoped values were inspired by the way that many Lisp dialects provide support for dynamically scoped free variables; in particular, how such variables behave in a deep-bound, multi-threaded runtime such as Interlisp-D. scoped values improve on Lisp's free variables by adding type safety, immutability, encapsulation, and efficient access within and across threads.
