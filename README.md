# Extent-local variables

## Summary

Define a standard, lightweight way to pass contextual information from
a caller to callees in the currently executing thread, or in its child
threads. In particular, extent locals are intended
to replace `ThreadLocal`s for sharing information with a very large
number of child threads. This usage especially applies to virtual threads.

## Goals

- *Ease of use* — Provide a clear programming model to share data
  within and across threads in order to simplify reasoning about
  data flow.
- *Comprehensibility* — Make the lifetime of shared data
  clearly visible in the program code. 
- *Performance* — Treat shared data as immutable so as to allow
  sharing among large numbers of threads.
- *Robustness* — Ensure that data shared by a caller
   is only accessible to legitimate callees.

## Non-Goals

It is not a goal to change the Java Programming Language.

It is not a goal to require migration away from thread local
variables or to deprecate the ThreadLocal API.

It is not a goal for this new programming model to require use
of interface `AutoCloseable`.

## Motivation

Programs are usually built out of components that contribute independent,
complementary functionality. For example, a networked server may combine
business logic to handle service requests with trusted components such as a database
driver providing data persistence, and a logging framework that records
progress or noteworthy events. Such server components often need to share data
between themselves, independent from the business logic. As a
security measure, the server might allocate a `Permissions` token  to each
thread that handles a request. Other server components would use the token to
restrict access to the operations they provide.

Thread context data is normally communicated from a caller
to called methods in the same thread via method arguments. However,
in a case like this, where data needs to be pushed through
business logic calls it simplifies the implementation if the server can share
context with its components via some alternative channel. 

The diagram below illustrates the sort of behaviour we would like. It shows the
call stack for two different threads handling a server request. Each call stack
begins with a call to `processRequest()` at call 1. It calls business logic method
`handleRequest()` at call 2. The application logic eventually tries to open a
database connection by calling `DBDriver.open()` at call 7. At this point `DBDriver`
needs to decide whether the thread is permitted to access the database.

The requirement here is for `PERMISSIONS` to act as a direct, per-thread channel
to pass a `Permission` instance from the `Server` in call 1 to the `DBDriver` in
call 7. The value `set()` by Thread 1 in call 1 should be the value returned when
Thread 1 performs a `get()` in call 7. Thread 2 must be able to `set()` and
`get()` its own independent value. In the example, Thread 1's `Permission` object
allows database access, so the `DBDriver` proceeds to call `newConnection()` at
call 8. The `Permission` for Thread 2 disallows database access, so `DBDriver`
creates and throws an `InvalidPermissionException`. 

    Thread 1                                  Thread 2

    8. DBDriver.newConnection()                         8. InvalidPermissionException() 
    7. DBDriver.open() <-- get -------------------+     7. DBDriver.open() <--- get --------------+
       ...                                        |        ...                                    |
       ...                                 PERMISSIONS     ...                            PERMISSIONS                      
    2. AppLogic.handleRequest()                   |     2. AppLogic.handleRequest()               |
    1. Server.processRequest() -- set(DATABASE) --+     1. Server.processRequest() -- set(NONE) --+
     
### Currently Supported Options

Developers traditionally turn to `ThreadLocal` in situations like this. It
provides developers with an independent variable for each thread to
share their own copy of data.

The following example shows how a `Server` implementation might use a 
`ThreadLocal` to communicate `Permissions` from the server to the database.
`Server.processRequest()` and `DBDriver.open()` implement the methods
invoked at calls 1 and 7 in thread call stack diagram above.

    class Server {
      // 1. Provide a per-thread channel between the framework and its components
      final static ThreadLocal<Permissions> PERMISSIONS = ...;
      ...
      void processRequest(Request request, Response response) {
        int p = (request.isAuthorized() ? DATABASE : NONE);
        Permissions permissions = new Permissions(p);
        // 2. Set permissions for request thread and run app logic
        PERMISSIONS.set(permissions);
        AppLogic.handleRequest(request, response);
      }
      ...
    }

    class DBDriver {
      DBConnection open(...) throws PermissionException {
        // 3. Get permissions for request thread and check
        Permissions permissions = Server.PERMISSIONS.get();
        if (!permissions.allowed(DATABASE) throw new  PermissionException();
        // 4. Ok to connect to a database
        return DBDriver.newConnection(...);
      }
      ...
    }


The `ThreadLocal` declared at 1. in static field `PERMISSIONS` provides a
way to ensure that each thread handling a request can have its own independent
permissions. A `ThreadLocal` serves as a key that is used to look up
a `Permission` value for the current thread. So, there is not exactly _one_
incarnation of field `PERMISSIONS` shared across all instances of Foo; instead, there are
_multiple_ incarnations of the field, one per thread, and the incarnation
that is used when code sets or gets `PERMISSIONS`
depends on the thread which is executing the code.

Using a `ThreadLocal` here avoids the need to pass a `Permissions` object as
an argument to calls from the service handler through the business logic
and into the database driver or logger.

The declaration at 1. creates a `ThreadLocal` object and assigns it to static field
`PERMISSIONS`. Before it can be used the ``Server` handler thread needs to call the `set()`
method at 2. This ensures that the incarnation of field
`PERMISSIONS` specific to the handler thread identifies the right permissions
for that request. The server is now ready to execute the business logic.

The database driver calls the `get()` method at 3. to
retrieve the permissions for the current thread. It ensures that the
operation requested by the business logic is permitted before proceeding
to execute it at 4. 

This example demonstrates that `ThreadLocal`s _can_ be used to implement the
behaviour needed for this case. However, there are problems with the design
of `ThreadLocal` that affect even this sort of well-structured use.
 
- *Unconstrained Mutability*  —  Every thread-local variable is _mutable_:
  that is to say, any code that can access a `ThreadLocal`'s `get()` can
  also `set()` it to a different value, or `remove()` it altogether.
  Unconstrained mutability is prone to error or abuse. Badly designed or 
  buggy code can make it very difficult to identify where and in what
  order state will be read and updated.
 
  Such problems are possible even with a well-defined API like the `Server`
  example. If the `DBDriver` accidentally reset the permissions to NONE`
  the error would only appear the next time the business logic tried to
  perform a database operation.

  In our example, what is really needed is a simple, one-way broadcast
  with a single point of assignment in a caller (at 2.) and one or more
  clear points in called code where the data is consumed (e.g. at 3.).
  This is a very common pattern, and it is not supported well by the
  complex API of `ThreadLocal`.

  `ThreadLocal` was _specified_ to provide unconstrained mutability in order to
  support a more general model of communication than we usually need. Data is able to flow in
  either direction between a caller and called method. This generality may be 
  needed for a few difficult cases. However, in the worst case, programming
  with thread local variables can lead to spaghetti-like dataflow. The result
  is code whose structure is hard to discern, let alone maintain.
   
- *Unbounded Persistence* — Once set, a `ThreadLocal` is _persistent_: that is
  to say, the incarnation of the value for each thread that calls `set()` is retained
  for the lifetime of the thread, or until a method running on that thread
  calls `remove()`.

  Having to call `remove()` on a `ThreadLocal` to clean it up
  when it's no longer in use means that it is all too easy for values to persist
  longer than is desirable. It is unfortunately common for developers to forget
  to remove a `ThreadLocal`.  Indeed, for programs that rely on the
  unconstrained mutability of `ThreadLocal`, there may be no clear point at
  which it is safe to call `remove()`. This can lead to a long-term memory
  leak. Even though the program may have long since moved on from having
  any use for an object referenced from the `ThreadLocal`, it will not be
  garbage collected unless the thread exits.

  Clearly, it would be better if sharing of data from `set()` and
  `get()` calls could be achieved with a clearly determined lifetime,
  limiting persistence of the data to a well-bounded interval during
  execution of the thread.

- *Inheritance* — The `ThreadLocal` persistence problem can be much worse
  when using large numbers of threads because`ThreadLocal`s may be inherited
  from parent to child thread. Each thread has to allocate storage for every
  thread local variable set in that thread. If a newly created child `Thread`
  inherits the thread local it has to allocate its own storage. When an
  application uses many `Thread`s and they, in
  turn, inherit `ThreadLocals` this can add significant memory costs. 

  Child threads cannot share the storage used by the parent thread because
  `ThreadLocal`s are _mutable_. The `ThreadLocal` API requires the parent and 
  child to be able to change their own value without the change being seen
  by the other thread. This is unfortunate because in most cases child threads
  do not `set()` the `ThreadLocals` they inherit.


To sum up, `ThreadLocal` offers more complexity than is needed by many applications
and comes with some significant costs.

### Mismatch with Virtual Threads

The above problems with thread local variables have become more pressing
in the context of virtual threads. These threads are cheap and plentiful, unlike
today's platform threads which are expensive and scarce.

Platform Threads are:

* Long-running
* Heavyweight
* Pooled

Virtual Threads are:

* Short-running
* Lightweight
* Single-use

It would certainly be useful for these numerous, cheap and plentiful
threads to be able to access some context from their parent. For
example, they may share a logger on an output stream. Perhaps they may
share some kind of security policy too.

Because virtual threads are still `Thread`s, it is legitimate for a
virtual thread to carry thread-local variables. The short lifetime of
virtual threads minimises the problem of long term memory leaks via
thread locals. An explicit `remove()` isn't necessary when a thread,
virtual or not, terminates, because when a thread terminates all thread
local variables are automatically removed. However, if you have thousands
or millions of threads and every one has its own, inevitably mutable, set of
thread local variables, the memory footprint may become significant.

It would be ideal if the Java Platform provided a way to have per
thread context for millions of virtual threads that is immutable and,
given the low cost of forking virtual threads, inheritable. Because
these ideal per thread variables are immutable, their data can be
easily shared by child threads, rather than copied to child
threads. Thread local variables were the 90s realization of per thread
variables; we need a better realization of per thread variables for
the modern era.

## Description

An extent local variable is a per-thread variable that allows context
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

That is to say, m1's extent is the set of methods m1 invokes, and any
methods invoked transitively by them.

It should now be clear that the thread call stack diagram above is
actually a picture of two separate extents for two different threads.
In both cases the bottom frame of the extent is a call to method
`Server.processRequest()`. In Thread 1 the top frame is a call to
method `DBDriver.newConnection()` while in Thread 2 it is a call to
constructor `InvalidPermissionException()`

The value associated with an extent local variable is defined in the
bottom most frame of some extent, and is accessible in every frame of
that extent. The extent local is *bound* to the value.
                                   
###  Example Use Of Extent Local Variables

The `Server` example described above can easily be rewritten to
use class `ExtentLocal` instead of `ThreadLocal`.

     class Server {
      // 1. Provide a per-thread channel betwen the framework and its components
      final static ExtentLocal<Permissions> PERMISSIONS = ...;
      ...
      void processRequest(Request request, Response response) {
        int p = (request.isAdmin() ? LOG|DATABASE : LOG);
        Permissions permissions = new Permissions(p);
        // 2. Bind the extent local for the extent of the run operation
        ExtentLocal.where(PERMISSIONS, permissions)
                     .run(() -> { AppLogic.handleRequest(request, response); });
      }
      ...
    }

    class DBDriver {
      DBConnection open(...) throws PermissionException {
        // 3. Get permissions for request thread and check
        Permissions permissions = Server.PERMISSIONS.get();
        if (!permissions.allowed(DATABASE) throw new  PermissionException();
        // 4. Ok to connect to a database
        return DBDriver.newConnection(...);
      }
      ...
    }

As before, the `Server` includes an initialization at 1. which
ensures that field `PERMISSIONS` references an `ExtentLocal`. The important
difference occurs at point 2. where previously there was a call to
`ThreadLocal.set()`. This is changed to use a call to methods
`where()` and `run()` of class `ExtentLocal`. These two calls work
together to provide the immutable one-way, thread-local communication
that the application needs.

The `where()` call  *binds* `the ExtentLocal` referenced from `PERMISSIONS`
for the extent of the `run()` call. That means a call to
`PERMISSIONS.get()` executed by any method called from `run()` will return
the value passed to `where()`. So, in this case, if
`Logger.warn()` gets called, directly or indirectly, by method
`AppLogic.handleRequest()` then the `PERMISSIONS` retrieved at point 3.
will be value passed in the `where()` call at point 2.

It also means that the value retrieved by the `get()` call is specific
to whichever thread is executing the methods. As with `ThreadLocal`, an
`ExtentLocal` has an incarnation that is per thread.

The first big difference between this example and the previous one
is that the binding established by `where()` is only
visible within the extent of the code called from `run()`. If a call to
`PERMISSIONS.get()` was inserted after the call to `run()` an exception
would be thrown because `PERMISSIONS` is no longer bound. The syntax
for employing an `ExtentLocal` enforces a well defined lifetime for
data sharing, unlike the unbounded persistence provided by ThreadLocal.

The second big difference is that the binding established by `where()` is
immutable within the extent of `run()`. Methods called from `run()` can
`get()` the current binding but there is no `set()` method allowing them
to change the binding established at point 1.

Note that the `DBDriver` method looks identical. The difference is that
the call to `get()` is invoking a method belonging to `ExtentLocal` not
`ThreadLocal`.

###  Rebinding of Extent Local Variables

The immutability of `ExtentLocal` bindings means that a caller can  use
an `ExtentLocal` to reliably communicate a constant value to the methods it calls.
However, there are times when one of those called methods might
need to use the same `ExtentLocal` variable to communicate a different
value to the methods *it* calls. The
requirement is not to change the original binding but to establish a new
binding for nested calls. 

                     
As an example consider this API method of the framework `Logger`
class.

    public void log(Supplier<String> formatter);

This method formats and prints a detailed log message when logging is enabled.
If logging is disabled it prints nothing. The caller passes in a `Supplier`
argument whose `get()` method is called by `log()` to retrieve the message
text. Using a `Supplier` avoids the cost of formatting when logging is
disabled.

`log()` is a good candidate for the use of rebinding. The `Logger` code is
trusted code. However, it is expected to call the formatter, which is
untrusted code. This untrusted code should only need to do text formatting.
It should not need to do any work that requires permissions.
It would be ideal if the extent local variable `PERMISSIONS` was bound
to an empty `Permissions` instance for the extent of the `get()` call.

The code for method `log()` is shown below

    class Logger {
      void log(Supplier<String> formatter) {
        if (loggingEnabled) {
          // 1. Obtain an empty permissions instance 
          Permissions nopermission = Permissions.none();
          // 2. Rebind PERMISSIONS for extent of formatter get
          String message = ExtentLocal.where(Server.PERMISSIONS, nopermission)
                                  .call(() -> formatter.get());
          // 3. Print the formatted message
          write(logFile, "DETAIL : %s %s".format(timeStamp(), message));
          ...

The logger obtains an empty `Permissions` instance at point 1.
is obtained. At point 2. `where()` rebinds extent-local `PERMISSIONS`
to this empty `Permissions` instance for the extent of the `call()` that
follows it. The lambda passed as argument to `call()` invokes `formatter.get()`
to retrieve the formatted message.  If code called from `formatter.get()`
tries to use the database, the permissions check in the `DBDriver` code will
retrieve the empty `Permission` instance, leading to a `InvalidPermissionException`.

Notice that this example uses the paired methods `where()` and `call()`,
whereas the previous example used `where()` and `run()`. That is because in
this example the code executed below `call()` needs to return a result. 
The `String` returned by `formatter.get()` is returned from `call()` and used to
bind local variable `message`. `call()` must be passed an argument that returns a
result. `run()` must be passed an argument that does not return a result.

Once again it is clear that the syntax of `ExtentLocal` reinforces the guarantee
of a well defined lifetime for sharing by limiting rebinding to a nested extent
i.e. to the lifetime of the call to `formatter.get()` with no possibility of
changing the binding in the current extent.

###  Inheritance of Extent Local Variables

There are occasions where it would be useful for an `ExtentLocal`
to be inherited from parent thread to child thread, for much the same
reasons as it is useful to inherit a `ThreadLocal`. This works with
`ExtentLocal` variables in a way that avoids many of the problems that
arise when using a `ThreadLocal`.

An example use of inheritance is provided by API method `processQuery` of
class `DBDriver`.

    public void processQuery(DBQuery query, Consumer<DBRow> rowHandler);

Method `processQuery` runs a `query` against the database ,retrieving a
list of results of type `DBRow`.  Argument `rowHandler` has type
`Consumer<DBRow>`. That means it can be applied to each result by
calling `rowHandler.apply(row)`. In order to speed up processing of query
results, each call to `apply` is executed in its own virtual thread.

If a problem occurs for some `DBRow` then the handler code will need to use
the `Logger` to log an error message. So, the binding of `PERMISSIONS` for
the caller thread really needs to be inherited by the child virtual thread that
executes the handler. This happens automatically because the implementation
of `processQuery` uses the structured execution framework.

The code for method `processQuery` is provided below.

      void processQuery(DBQuery query, Consumer<DBRow> rowHandler) {
        // 1. Get permissions for request thread and check 
        Permissions permissions = Server.PERMISSIONS.get();
        if (!permissions.allowed(DATABASE) throw new  InvalidAccessException();
        // 2. Run the database query
        List<DBRow> rows = executeQuery(query);
        / 3. Parallelize handler with a structured fork join executor 
        try (var s = StructuredExecutor.open()) {
          for (var row : rows) {
            // 4. process each row in its own virtual thread
            s.fork(() -> rowHandler.apply(row));
          }
          // 5. wait for all virtual threads to complete
          s.join();
        }
      }

The first thing `processQuery` does at 1. is ensure the caller has
the `DATABASE` permission before proceeding to run a query against
the database at 2. The try with resources at 3. opens a `StructuredExecutor`.
This is a class provided as part of the virtual threads implementation
which allows a collections of virtual threads to be managed using a fork
join model. It gets automatically closed at the end of the try with
resources block.

Inside the `for` loop at 4. a virtual thread is forked to run the handler on
the current `DBRow`. After the for loop at 5. a call to `join()` ensures that
all the forked virtual threads have completed. This ensures that all rows
have been processed before the try block is exited.

There is one important detail in the `processQuery` code that still needs
explaining. The call to `rowHandler.apply()` happens in a forked
virtualThread. So, how can a call to `PERMISSIONS.get()` in the forked thread
retrieve the value that was `set()` in the forking thread? This works because
class `ExtentLocal` has been designed to share bindings across thread forks.
Any `ExtentLocal` bindings present in the thread that calls `fork()` will be
visible to the forked thread.

It is worth emphasising that these bindings really are *shared*. The forked
thread can reference the set of bindings established by the forking thread
without needing its own local copy.

It is also worth noting that the fork/join model offered by `StructuredExecutor`
means that a value bound in the call to to `where()` has a determinate lifetime,
as far as visibility via the `ExtentLocal` is concerned. The `Permission`
object is available while the child thread is running. The call to `join()`
at the end of the try block ensures that child threads can no longer be using
it. These two capabilities work together to avoid the problem of unbounded
persistence seen when using `ThreadLocal`. 

Using class StructuredExecutor to parallelise row processing means that query
results returning thousands or even millions of rows can be executed in
parallel. If a call to `rowHandler` needs to block, say to log a warning to
a file on disk, work can continue by switching execution to another virtual
thread with almost no overhead. 

## Migrating to extent local variables

In general, for the reasons we've listed above, extent local variables
are likely to be useful in many cases where thread local variables are
used today. We continue to support thread local variables, even with
virtual threads, despite them not being ideal when threads are very
numerous.

The first thing to do is determine whether migrating thread local to
extent local variables is appropriate. If in your application thread
local variables are used in an unstructured way so that a deep callee
`set()`s a `ThreadLocal` which is then retrieved by the caller,
migration may be difficult, and you may find there is little to be
gained.

However, in many cases extent local variables are exactly what you
need. We've already covered hidden parameters for callbacks in some
depth, but there are other good ways to use extent locals.

- *Extent locals and recursion* — Sometimes you want to be able to
  detect recursion, perhaps because a framework isn't re-entrant or
  because you want to limit recursion in some way. An extentlocal
  variable provides a way to do this: set it once, invoke a method,
  and somewhere deep in the call stack, call `ExtentLocal.isBound()` to see if the
  thread-local variable is set. More elaborately, you might want the
  extent local variable to be a recursion counter.

  The detection of recursion would also be useful in the case of flattened
  transactions: any transaction started when a transaction in progress
  becomes part of the outermost transaction.

  Another example occurs in graphics, where there is a drawing context.
  Extent local variables, because of their automatic cleanup and
  re-entrancy, are better suited to this than are thread local
  variables.


[Generally, the Gods of Java want you to migrate to extent locals.]


## Where can extent local variables _not_ replace thread local variables?

There are cases where thread local variables are more appropriate than
extent local variables. For example, one popular use of `ThreadLocal`
is to cache objects that are expensive to create. A notorious
example is `java.text.DateFormat`, which is mutable so cannot be
shared between threads without synchronization. In this case, creating
a thread local `DateFormat` object which persists for the lifetime of
the thread might be exactly what you need:

(In hindsight, making `DateFormat` mutable was a mistake, and we'd do
better today, but it was Java 1.1 in 1997. `ThreadLocal` makes it
possible to use this utility class reasonably efficiently in a
multi-threaded program.)

## In summary

Extent locals have the following properties:

* _Locally-defined extent_: The values of extent local variables are only bound
  in the extent of `run()` or `call()`.
* _Immutability_ There is no `ExtentLocal.set()` method: extent locals,
  once bound, are effectively final.
* _Simple and fast code_: In most cases an extent-local `x.get()` is as
  fast as a local variable `x`. This is true regardless of how far
  away `x.get()` is from the point that the extent local `x` is bound.
* _Structure_: These properties also also make it easier for a reader
  to reason about programs, in much the same way that declaring a
  field of a variable `final` does. The one-way nature of the channel
  from caller to callee makes it much
  easier to reason about the flow of data in a program.

## API

There is more detail in the Javadoc for the API, at

http://people.redhat.com/~aph/loom_api/jdk.incubator.concurrent/jdk/incubator/concurrent/ScopeLocal.html

## Alternatives

It is possible to emulate many of the features of extent locals with
`ThreadLocal`s, albeit at some cost in memory footprint,
security, and performance.

We have experimented with a modified version of `ThreadLocal` that
supports some of the characteristics of extent locals. However,
carrying the additional baggage of `ThreadLocal` results in an
implementation that is unduly burdensome, or an API that returns
`UnsupportedOperationException` for much core functionality, or both.
And we'd still have the problem of memory management.

It is better, therefore, not to modify `ThreadLocal` but to give
extent locals a separate identity from thread locals.

## Historical Note
The idea for Extent Locals was inspired by the way many Lisp dialects
provide support for dynamically scoped free variables, in particular
how they behave in a deep-bound, multi-threaded runtime like Interlisp-D.
